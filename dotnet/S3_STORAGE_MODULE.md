# Module: File Storage (S3)

Adds file upload/download support via Amazon S3.

---

## Server Additions

### S3Service

Core operations:
- Upload file (stream or byte array) to a key path
- Download file as stream
- Generate presigned upload URL (for direct client-to-S3 uploads)
- Generate presigned download URL (for time-limited access)
- Delete file
- Multipart upload support for large files

### AppSettings Update

Add `StorageBucketName` property, populated from `STORAGE_BUCKET_NAME` environment variable.

### NuGet Package

```xml
<PackageReference Include="AWSSDK.S3" />
```

---

## Infrastructure Additions

### StorageStack (new stack)

- S3 bucket: `{projectname}-storage-{stage}-{account}-{region}`
- Versioning enabled, AES256 encryption
- Lifecycle rules: old version deletion (30 days), Infrequent Access transition (180 days)
- CORS configuration for cross-origin uploads from Web client
- Bucket policy enforcing secure transport (`aws:SecureTransport`)
- Expose: `Bucket`

Key decisions:
- **Public-read prefixes**: If certain files should be publicly accessible (e.g., profile pictures, thumbnails), add bucket policy statements granting `s3:GetObject` for those prefixes. Otherwise, keep the bucket fully private and use presigned URLs for all access
- **Lifecycle rules**: Adjust retention periods based on data sensitivity and storage costs
- **Encryption**: AES256 (default) or AWS KMS for stricter compliance requirements

### CDK Entry Point Update

Add `StorageStack` as an independent stack. Pass `StorageBucket` to `ApiStack` via props.

### ApiStack Update

- Add `STORAGE_BUCKET_NAME` Lambda environment variable
- Grant S3 read/write + multipart permissions to the Lambda execution role:
  - `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`
  - `s3:ListMultipartUploadParts`, `s3:AbortMultipartUpload`

---

## Web Additions

For file uploads from the browser, either:
1. **Server-proxied**: POST files to an API endpoint, server uploads to S3
2. **Direct-to-S3**: Request a presigned upload URL from the API, then PUT directly to S3 from the browser (requires CORS on the bucket)

---

## Mobile Additions

Same two patterns as Web. For large files (images, videos), direct-to-S3 with presigned URLs avoids Lambda memory/timeout limits.

---

## File Additions

```
Server/src/Server/
  Services/
    IS3Service.cs
    S3Service.cs

Infrastructure/
  Stacks/
    StorageStack.cs
```
