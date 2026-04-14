# Copilot Instructions

## Commands

```bash
just build    # cross-compile for Linux/amd64 (GOOS=linux GOARCH=amd64), strips debug info
just test     # run all tests (requires a real .env file with AWS credentials — see below)
just format   # run go fmt ./...

# Run a single test or subtest
go test -v -run TestDownload ./...
go test -v -run "TestDownload/verify_err_when_region_is_empty" ./...
```

## Test Setup

Tests hit **real AWS infrastructure**. Before running:

1. `cp .env.example .env`
2. Fill in `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` in `.env`
3. The tests expect an S3 bucket named `golang-s3-lambda-test` in region `us-east-2`
4. The key `file_slash_key_name` must already exist in that bucket for `TestDownload`'s happy-path subtest

`TestMain` calls `setup()` which calls `godotenv.Load(".env")` — this calls `log.Fatalf` (not a test failure) if `.env` is missing. Similarly, `generateUploadFileReq()` calls `log.Panic` if `sample_file.csv` is absent or not exactly 369 bytes.

## Architecture

Single-file Go library (`lib.go`, package `lambda_s3`). No shared state — every exported function creates its own AWS session.

Intended upload call chain:
```
GetHeaders(lambdaReq, maxBytes) → []*multipart.FileHeader
  └─► UploadHeader(fileHeader, region, bucket, name) → *UploadRes
```

Independent operations:
```
Download(region, bucket, name) → []byte
Delete(region, bucket, name) → error
```

## Function Code Flows

### `GetHeaders(lambdaReq, maxFileSizeBytes)`

1. Copy all `lambdaReq.Headers` into a fresh `http.Header{}` — this normalizes case, working around an AWS API Gateway bug where headers arrive lowercase ([aws-lambda-go#117](https://github.com/aws/aws-lambda-go/issues/117))
2. `headers.Get("Content-Type")` empty → `ErrContentTypeHeaderMissing`
3. `mime.ParseMediaType(contentType)` error → `ErrParsingMediaType`
4. `params["boundary"]` empty → `ErrBoundaryValueMissing`
5. Build reader: default is `strings.NewReader(lambdaReq.Body)`; if `lambdaReq.IsBase64Encoded == true`, wrap it with `base64.NewDecoder` — the body encoding is handled transparently here
6. `multipart.NewReader(readerImpl, boundary).ReadForm(maxFileSizeBytes)` error → `ErrReadingMultiPartForm`
7. Loop over `form.File`, appending `form.File[fieldName][0]` — **only the first file per form field name is kept**
8. **Happy path**: returns one `*multipart.FileHeader` per unique form field name

### `UploadHeader(fileHeader, region, bucket, name)`

1. Validate: region empty → `ErrParameterRegionEmpty`; bucket empty → `ErrParameterBucketEmpty`; name empty → `ErrParameterNameEmpty`
2. `fileHeader.Open()` error → `ErrOpeningMultiPartFile`
3. `file.Read(fileContents)` where `fileContents` is a nil `[]byte` — reads 0 bytes, always returns `(0, nil)`. **`fileContents` is never used.** `ErrReadingMultiPartFile` is effectively unreachable. The actual upload body is the `file` handle itself.
4. `session.NewSession(...)` error → `ErrNewAWSSession`
5. `uploader.Upload({Bucket, Key, Body: file})` error → `ErrUploadingMultiPartFileToS3`
6. **Happy path**: returns `&UploadRes{S3Path: filepath.Join(bucket, name), S3URL: uploadOutput.Location}`
   - `S3Path` example: `"golang-s3-lambda-test/file_slash_key_name"`
   - `S3URL` example: `"https://golang-s3-lambda-test.s3.us-east-2.amazonaws.com/file_slash_key_name"`

### `Download(region, bucket, name)`

1. Validate: region → `ErrParameterRegionEmpty`; bucket → `ErrParameterBucketEmpty`; name → `ErrParameterNameEmpty`
2. `session.NewSession(...)` error → `ErrNewAWSSession`
3. `s3manager.NewDownloader` with `Concurrency=0` (sequential, no parallel part downloads)
4. Target is `aws.NewWriteAtBuffer` (in-memory)
5. `downloader.Download(...)` error → `ErrDownloadingS3File`
6. `bytesDownloaded == 0` → `ErrEmptyFileDownloaded`
7. **Happy path**: returns `writeAtBuffer.Bytes()`

### `Delete(region, bucket, name)`

1. Validate: region → `ErrParameterRegionEmpty`; bucket → `ErrParameterBucketEmpty`; name → `ErrParameterNameEmpty`
2. `session.NewSession(...)` error → `ErrNewAWSSession`
3. `s3manager.NewBatchDelete` with `BatchSize=1`, deletes a single-element `DeleteObjectsIterator`
4. **Happy path**: returns `nil`
5. **Important**: on batch delete failure, the raw error from `batcher.Delete(...)` is returned directly — unlike every other function, `Delete` does NOT wrap AWS errors in a package sentinel. `errors.Is` against package sentinels will not match delete failures.

## Key Conventions

**Sentinel errors**: All public errors are pre-declared package-level `var` values. Callers must use `errors.Is` to check them. Every function wraps AWS/stdlib errors into package sentinels — **except `Delete`**, which returns raw batcher errors on failure.

**Parameter validation order**: Every public function checks `region` → `bucket` → `name` before any AWS work.

**Test fixture**: `generateUploadFileReq()` builds a synthetic `APIGatewayProxyRequest` by reading `sample_file.csv` (369 bytes), writing it into a multipart form with boundary `"---SEAN_BOUNDARY_VALUE"` and field name `"sample_file"`, then base64-encoding the result. The returned request always has `IsBase64Encoded: true`.

**Test assertions**: Use `github.com/jgroeneveld/trial/assert`. Each function's tests enumerate error cases as subtests (`t.Run`), with the happy path as the final subtest. `TestDelete` is a top-level test (no subtests) that exercises the full upload → delete → download-confirms-gone flow.

**S3 objects used by tests**:
| Key | Used by | Pre-existing required |
|---|---|---|
| `file_slash_key_name` | `TestDownload` happy path, `TestUploadHeader` happy path | Yes (for download test) |
| `delete_me_dude` | `TestDelete` | No (uploaded then deleted) |
| `empty_file.txt` | `TestDownload` empty-file subtest | No (uploaded inline) |
