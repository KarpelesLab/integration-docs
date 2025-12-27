# KLB Upload Protocol Specification

This document describes the low-level upload protocol used by the KLB platform. It is intended for developers implementing server-side APIs that need to accept file uploads from KLB-compatible clients.

## Important Note

**In most cases, you do not need to implement this protocol yourself.** The KLB platform provides SDKs for various languages (JavaScript, PHP, Go, etc.) that handle file uploads automatically. These SDKs manage chunking, retries, progress tracking, and protocol negotiation transparently.

This documentation is for:
- Implementing custom upload endpoints outside the standard KLB infrastructure
- Understanding the underlying protocol for debugging purposes
- Building alternative client implementations

If you are using a KLB platform SDK, file uploads are handled automatically when calling API endpoints that accept file parameters.

## Design Principle

**File data is never sent directly to API endpoints.** Instead, the API endpoint acts as a negotiation layer that returns instructions for where and how to upload the actual file data. This design:

- Allows unlimited file sizes (up to ~5TB, the S3 per-object limit)
- Keeps API servers lightweight and stateless
- Enables direct-to-storage uploads (S3, dedicated upload servers)
- Avoids request body size limits on API endpoints
- Supports resumable uploads and parallel chunk transfers

The API endpoint only receives file metadata (name, size, type) and returns upload instructions. The file bytes flow through a separate path entirely.

---

## Protocol Overview

The KLB upload protocol is a two-phase process:

1. **Negotiation Phase**: Client sends file metadata, server responds with upload instructions
2. **Transfer Phase**: Client uploads file data according to the instructions, then calls completion endpoint

The server can choose between two upload methods based on its infrastructure and the file characteristics.

## Negotiation Phase

### Client Request

The client calls an API endpoint with file metadata:

| Parameter | Type | Description |
|-----------|------|-------------|
| `filename` | string | Name of the file being uploaded |
| `size` | number | Size of the file in bytes |
| `type` | string | MIME type of the file |
| `lastModified` | number | Unix timestamp of last modification |

Additional application-specific parameters may be included.

### Server Response

The server must return one of two response formats to indicate which upload method to use.

---

## Method 1: Direct PUT Upload

This method instructs the client to upload file data directly to a URL via HTTP PUT requests.

### Response Format

```json
{
  "result": "success",
  "data": {
    "PUT": "https://your-server.com/upload/unique-upload-id",
    "Complete": "Your/Api/Endpoint:handleComplete",
    "Blocksize": 5242880
  }
}
```

### Response Fields

| Field | Required | Description |
|-------|----------|-------------|
| `PUT` | **Yes** | URL where the client will PUT the file data |
| `Complete` | **Yes** | API endpoint the client will call after upload completes |
| `Blocksize` | No | Maximum size per block in bytes. If omitted, the entire file is uploaded in one request |

### Transfer Behavior

**Single-block upload** (when file size ≤ Blocksize, or Blocksize not provided):
- Client sends one `PUT` request to the `PUT` URL
- `Content-Type` header is set to the file's MIME type
- No `Content-Range` header is used

**Multi-block upload** (when file size > Blocksize):
- File is split into blocks of `Blocksize` bytes
- Each block is uploaded via `PUT` to the same URL
- Each request includes: `Content-Range: bytes {start}-{end}/*`
- Clients may upload up to 3 blocks concurrently

### Request Headers Sent by Client

| Header | Value | Condition |
|--------|-------|-----------|
| `Content-Type` | File MIME type | Always |
| `Content-Range` | `bytes {start}-{end}/*` | Multi-block uploads only |

### Server Requirements for PUT Endpoint

Your PUT endpoint must:
- Accept `PUT` requests with binary body data
- Parse `Content-Range` headers for partial uploads
- Track which byte ranges have been received
- Return `200 OK` on successful receipt
- Return appropriate error codes (4xx, 5xx) on failure

### Completion Phase

After all data is uploaded, the client calls the `Complete` endpoint:

```
POST {Complete}
```

Your completion handler should:
1. Verify all byte ranges have been received
2. Assemble the complete file if chunked
3. Perform post-processing (hash computation, database records, etc.)
4. Return the final result

**Expected completion response:**
```json
{
  "result": "success",
  "data": {
    "Blob__": "blob-abc123-xyz789",
    "SHA256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "Size": "1048576",
    "Mime": "application/octet-stream"
  }
}
```

The response `data` object is returned to the application. The specific fields depend on your API design.

---

## Method 2: AWS S3 Multipart Upload

This method delegates file storage to AWS S3 using multipart upload. The client uploads directly to S3, with your server providing request signing.

### Response Format

```json
{
  "result": "success",
  "data": {
    "Cloud_Aws_Bucket_Upload__": "clabu-abc123-defg-hijk-lmno-pqrstuvw",
    "Bucket_Endpoint": {
      "Host": "my-bucket.s3.us-east-1.amazonaws.com",
      "Name": "my-bucket",
      "Region": "us-east-1"
    },
    "Key": "uploads/unique-file-key.bin"
  }
}
```

### Response Fields

| Field | Required | Description |
|-------|----------|-------------|
| `Cloud_Aws_Bucket_Upload__` | **Yes** | Unique identifier for this upload session |
| `Bucket_Endpoint.Host` | **Yes** | S3 bucket hostname |
| `Bucket_Endpoint.Name` | **Yes** | S3 bucket name |
| `Bucket_Endpoint.Region` | **Yes** | AWS region |
| `Key` | **Yes** | Object key in S3 |

### Transfer Behavior

1. Client initiates S3 multipart upload
2. File is split into parts (minimum 5MB per part, targeting max ~10,000 parts)
3. Client uploads parts directly to S3
4. Client completes S3 multipart upload
5. Client calls your server's completion handler

### Required Server Endpoint: Request Signing

The client cannot sign AWS requests directly (it lacks credentials). Your server must provide a signing endpoint:

```
POST Cloud/Aws/Bucket/Upload/{Cloud_Aws_Bucket_Upload__}:signV4
```

**Request from client:**
```json
{
  "method": "PUT",
  "host": "my-bucket.s3.us-east-1.amazonaws.com",
  "uri": "/uploads/unique-file-key.bin?partNumber=1&uploadId=xyz",
  "headers": { ... },
  "hash": "sha256-of-request-body"
}
```

**Expected response:**
```json
{
  "result": "success",
  "data": {
    "authorization": "AWS4-HMAC-SHA256 Credential=AKID.../aws4_request, SignedHeaders=..., Signature=..."
  }
}
```

Your signing endpoint must:
- Validate that the upload session exists and is authorized
- Generate AWS Signature Version 4 for the request
- Never expose AWS credentials to the client

### Completion Phase

After S3 multipart upload completes, the client calls:

```
POST Cloud/Aws/Bucket/Upload/{Cloud_Aws_Bucket_Upload__}:handleComplete
```

Your handler should:
1. Verify the upload completed successfully in S3
2. Record file metadata in your database
3. Return the final result (same format as Method 1)

---

## Error Handling

Clients typically implement automatic retry logic. Your server should return appropriate HTTP status codes:

| Status | Meaning | Expected Client Behavior |
|--------|---------|--------------------------|
| 200 | Success | Continue to next step |
| 4xx | Client error | May retry depending on client configuration |
| 5xx | Server error | May retry depending on client configuration |

Transient failures (network issues, temporary server overload) should use 5xx codes to encourage retry.

---

## Block Sizing Considerations

### For PUT Method

- Without `Blocksize`: entire file uploaded in one request
- Recommended `Blocksize`: 5MB - 100MB depending on expected file sizes
- Smaller blocks: better resume capability, more HTTP overhead
- Larger blocks: less overhead, but must restart entire block on failure

### For AWS Method

Block sizing is handled automatically by compliant clients:
- Minimum: 5MB (AWS S3 requirement)
- Target: files divided into ~10,000 parts maximum
- For streaming uploads with unknown size: 526MB blocks (supports up to ~5TB)

---

## Implementation Checklist

### Minimal PUT Implementation

- [ ] API endpoint that returns `PUT` and `Complete` in response
- [ ] PUT endpoint that accepts binary data
- [ ] Completion handler that returns upload result

### Full PUT Implementation (chunked uploads)

- [ ] API endpoint that returns `PUT`, `Complete`, and `Blocksize`
- [ ] PUT endpoint that parses `Content-Range` headers
- [ ] Storage for tracking received byte ranges
- [ ] Logic to assemble chunks into complete file
- [ ] Completion handler with integrity verification

### AWS S3 Implementation

- [ ] API endpoint that returns AWS bucket info
- [ ] `signV4` endpoint for AWS request signing
- [ ] Secure storage of AWS credentials (never exposed to client)
- [ ] `handleComplete` endpoint
- [ ] Verification that upload exists in S3

---

## Example: Minimal PUT Server (Node.js/Express)

```javascript
const uploads = new Map();

// Initial request - returns upload instructions
app.post('/api/upload/init', (req, res) => {
    const { filename, size, type } = req.body;
    const uploadId = crypto.randomUUID();

    uploads.set(uploadId, {
        filename,
        size,
        type,
        data: Buffer.alloc(0)
    });

    res.json({
        result: 'success',
        data: {
            PUT: `https://yourserver.com/api/upload/${uploadId}`,
            Complete: `Upload/${uploadId}:handleComplete`,
            Blocksize: 5242880  // 5MB chunks
        }
    });
});

// PUT endpoint - receives file data
app.put('/api/upload/:id', (req, res) => {
    const upload = uploads.get(req.params.id);
    if (!upload) return res.status(404).send('Upload not found');

    const chunks = [];
    req.on('data', chunk => chunks.push(chunk));
    req.on('end', () => {
        const data = Buffer.concat(chunks);
        const range = req.headers['content-range'];

        if (range) {
            // Parse "bytes start-end/*" and store at correct offset
            const match = range.match(/bytes (\d+)-(\d+)/);
            if (match) {
                const start = parseInt(match[1]);
                // Expand buffer if needed and copy data
                if (start + data.length > upload.data.length) {
                    const newBuf = Buffer.alloc(start + data.length);
                    upload.data.copy(newBuf);
                    upload.data = newBuf;
                }
                data.copy(upload.data, start);
            }
        } else {
            upload.data = data;
        }

        res.status(200).send('OK');
    });
});

// Completion handler - finalizes upload
app.post('/api/Upload/:id\\:handleComplete', async (req, res) => {
    const upload = uploads.get(req.params.id);
    if (!upload) return res.status(404).send('Upload not found');

    const hash = crypto.createHash('sha256').update(upload.data).digest('hex');

    // Save file, create database record, etc.

    res.json({
        result: 'success',
        data: {
            Blob__: `blob-${req.params.id}`,
            SHA256: hash,
            Size: String(upload.data.length),
            Mime: upload.type
        }
    });

    uploads.delete(req.params.id);
});
```

This example is simplified for illustration. Production implementations should include:
- Authentication and authorization
- Request size limits
- Timeout handling for incomplete uploads
- Persistent storage for upload state
- Proper error handling
