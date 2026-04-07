# Code Examples: Signed URLs

## AWS S3 Signed URLs (Node.js)

```typescript
import AWS from 'aws-sdk';
import { Request, Response } from 'express';

// Configure S3 client with restricted IAM user
const s3 = new AWS.S3({
  accessKeyId: process.env.S3_ACCESS_KEY,
  secretAccessKey: process.env.S3_SECRET_KEY,
  region: 'us-east-1'
});

class S3FileService {
  private bucketName = 'my-secure-bucket';
  private defaultExpiration = 900; // 15 minutes

  // Generate signed URL for download
  async getDownloadUrl(
    fileKey: string,
    expirationSeconds: number = this.defaultExpiration
  ): Promise<string> {
    const params = {
      Bucket: this.bucketName,
      Key: fileKey,
      Expires: expirationSeconds,
      // Optional: Require specific content type
      ResponseContentType: this.getContentType(fileKey)
    };

    return new Promise((resolve, reject) => {
      s3.getSignedUrl('getObject', params, (err, url) => {
        if (err) reject(err);
        else resolve(url);
      });
    });
  }

  // Generate signed URL for upload
  async getUploadUrl(
    fileKey: string,
    contentType: string,
    expirationSeconds: number = 300 // 5 minutes
  ): Promise<string> {
    const params = {
      Bucket: this.bucketName,
      Key: fileKey,
      ContentType: contentType,
      Expires: expirationSeconds,
      // Require this content type on upload
      Conditions: [
        ['content-length-range', 0, 104857600], // Max 100 MB
        ['eq', '$Content-Type', contentType]
      ]
    };

    return new Promise((resolve, reject) => {
      s3.getSignedUrl('putObject', params, (err, url) => {
        if (err) reject(err);
        else resolve(url);
      });
    });
  }

  // Generate multi-part upload initiation
  async getMultiPartUploadUrl(
    fileKey: string,
    contentType: string
  ): Promise<{ uploadId: string; signedUrl: string }> {
    // Initiate multipart upload
    const initResponse = await s3
      .createMultipartUpload({
        Bucket: this.bucketName,
        Key: fileKey,
        ContentType: contentType,
        ServerSideEncryption: 'AES256'
      })
      .promise();

    const uploadId = initResponse.UploadId!;

    // Generate signed URL for first part
    const params = {
      Bucket: this.bucketName,
      Key: fileKey,
      UploadId: uploadId,
      PartNumber: 1,
      Expires: 3600
    };

    const signedUrl = await new Promise<string>((resolve, reject) => {
      s3.getSignedUrl('uploadPart', params, (err, url) => {
        if (err) reject(err);
        else resolve(url);
      });
    });

    return { uploadId, signedUrl };
  }

  private getContentType(filename: string): string {
    const ext = filename.split('.').pop()?.toLowerCase();
    const types: Record<string, string> = {
      pdf: 'application/pdf',
      doc: 'application/msword',
      docx: 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
      xls: 'application/vnd.ms-excel',
      xlsx: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
      jpg: 'image/jpeg',
      png: 'image/png',
      zip: 'application/zip'
    };
    return types[ext || ''] || 'application/octet-stream';
  }
}

// Express routes
const fileService = new S3FileService();

app.get('/api/download/:fileId', authenticateUser, async (req, res) => {
  const { fileId } = req.params;
  const userId = req.user.id;

  // Verify user has permission
  const file = await db.files.findOne({ id: fileId });
  if (!file || file.ownerId !== userId) {
    return res.status(403).json({ error: 'Not authorized' });
  }

  try {
    // Generate short-lived signed URL
    const url = await fileService.getDownloadUrl(
      file.s3Key,
      300  // 5 minutes for sensitive files
    );

    res.json({ downloadUrl: url, expiresIn: 300 });
  } catch (error) {
    res.status(500).json({ error: 'Failed to generate URL' });
  }
});

app.get('/api/upload-url/:filename', authenticateUser, async (req, res) => {
  const { filename } = req.params;
  const userId = req.user.id;

  // Validate filename
  if (!filename || filename.includes('..')) {
    return res.status(400).json({ error: 'Invalid filename' });
  }

  try {
    const contentType = 'application/octet-stream';
    const s3Key = `users/${userId}/${Date.now()}_${filename}`;

    const url = await fileService.getUploadUrl(s3Key, contentType);

    // Store pending upload in database
    await db.uploads.create({
      userId,
      s3Key,
      originalFilename: filename,
      status: 'pending'
    });

    res.json({ uploadUrl: url, s3Key, expiresIn: 300 });
  } catch (error) {
    res.status(500).json({ error: 'Failed to generate upload URL' });
  }
});
```

## AWS SDK v3 (Modern)

```typescript
import {
  S3Client,
  GetObjectCommand,
  PutObjectCommand
} from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3Client = new S3Client({
  region: 'us-east-1',
  credentials: {
    accessKeyId: process.env.S3_ACCESS_KEY!,
    secretAccessKey: process.env.S3_SECRET_KEY!
  }
});

async function getDownloadSignedUrl(bucket: string, key: string): Promise<string> {
  const command = new GetObjectCommand({
    Bucket: bucket,
    Key: key
  });

  return getSignedUrl(s3Client, command, { expiresIn: 900 });
}

async function getUploadSignedUrl(bucket: string, key: string): Promise<string> {
  const command = new PutObjectCommand({
    Bucket: bucket,
    Key: key,
    ServerSideEncryption: 'AES256'
  });

  return getSignedUrl(s3Client, command, { expiresIn: 300 });
}
```

## Cloudflare R2 Signed URLs

```typescript
import { S3Client, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

// Cloudflare R2 is S3-compatible
const r2Client = new S3Client({
  region: 'auto',
  endpoint: `https://${process.env.CLOUDFLARE_ACCOUNT_ID}.r2.cloudflarestorage.com`,
  credentials: {
    accessKeyId: process.env.R2_ACCESS_KEY_ID!,
    secretAccessKey: process.env.R2_SECRET_ACCESS_KEY!
  }
});

class R2FileService {
  async getSignedDownloadUrl(bucket: string, key: string): Promise<string> {
    const command = new GetObjectCommand({
      Bucket: bucket,
      Key: key
    });

    // R2 supports same presigner interface
    return getSignedUrl(r2Client, command, { expiresIn: 900 });
  }

  async getPublicUrl(bucket: string, key: string): Promise<string> {
    // R2 public bucket URL (if configured)
    return `https://${bucket}.cloudflarestorage.com/${key}`;
  }
}
```

## Custom Signed URL Implementation

```typescript
import crypto from 'crypto';

class CustomSignedUrlService {
  private secretKey = process.env.SIGNATURE_SECRET;

  // Generate custom signed URL (without AWS)
  generateSignedUrl(
    fileId: string,
    userId: string,
    expirySeconds: number = 900
  ): string {
    const expiresAt = Date.now() + expirySeconds * 1000;

    // Signature components
    const signature = crypto
      .createHmac('sha256', this.secretKey!)
      .update(`${fileId}:${userId}:${expiresAt}`)
      .digest('hex');

    // URL safe encoding
    const token = Buffer.from(
      JSON.stringify({
        fileId,
        userId,
        expiresAt,
        signature
      })
    ).toString('base64url');

    return `https://api.example.com/files/download/${token}`;
  }

  // Verify and decode signed URL
  verifySignedUrl(token: string): { fileId: string; userId: string } | null {
    try {
      const decoded = JSON.parse(
        Buffer.from(token, 'base64url').toString('utf8')
      );

      // Check expiration
      if (Date.now() > decoded.expiresAt) {
        return null; // Expired
      }

      // Verify signature
      const expectedSignature = crypto
        .createHmac('sha256', this.secretKey!)
        .update(`${decoded.fileId}:${decoded.userId}:${decoded.expiresAt}`)
        .digest('hex');

      if (decoded.signature !== expectedSignature) {
        return null; // Invalid signature
      }

      return { fileId: decoded.fileId, userId: decoded.userId };
    } catch (error) {
      return null;
    }
  }
}

// Usage
const urlService = new CustomSignedUrlService();

app.get('/files/download/:token', (req, res) => {
  const decoded = urlService.verifySignedUrl(req.params.token);

  if (!decoded) {
    return res.status(403).json({ error: 'Invalid or expired URL' });
  }

  // Fetch and serve file
  const file = fs.createReadStream(`/files/${decoded.fileId}`);
  file.pipe(res);
});
```

## React Frontend Usage

```typescript
import React, { useState } from 'react';

export const FileDownloadComponent: React.FC<{ fileId: string }> = ({ fileId }) => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleDownload = async () => {
    setLoading(true);
    setError(null);

    try {
      // Request signed URL from backend
      const response = await fetch(`/api/download/${fileId}`, {
        credentials: 'include'
      });

      if (!response.ok) {
        throw new Error('Failed to get download URL');
      }

      const { downloadUrl, expiresIn } = await response.json();

      // Create temporary link and trigger download
      const link = document.createElement('a');
      link.href = downloadUrl;
      link.download = 'file';
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);

      // Note: URL expires after expiresIn seconds
      // Don't share this URL with others
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Unknown error');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      <button onClick={handleDownload} disabled={loading}>
        {loading ? 'Generating URL...' : 'Download File'}
      </button>
      {error && <p style={{ color: 'red' }}>{error}</p>}
    </div>
  );
};

export const FileUploadComponent: React.FC = () => {
  const [file, setFile] = useState<File | null>(null);
  const [uploading, setUploading] = useState(false);

  const handleUpload = async () => {
    if (!file) return;

    setUploading(true);

    try {
      // Get signed upload URL from backend
      const uploadResponse = await fetch(
        `/api/upload-url/${encodeURIComponent(file.name)}`,
        { credentials: 'include' }
      );

      if (!uploadResponse.ok) throw new Error('Failed to get upload URL');

      const { uploadUrl, s3Key } = await uploadResponse.json();

      // Upload directly to S3 using signed URL
      const uploadResult = await fetch(uploadUrl, {
        method: 'PUT',
        body: file,
        headers: {
          'Content-Type': file.type
        }
      });

      if (!uploadResult.ok) throw new Error('Upload failed');

      // Notify backend of successful upload
      await fetch('/api/upload-complete', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ s3Key }),
        credentials: 'include'
      });

      setFile(null);
      alert('File uploaded successfully');
    } catch (error) {
      alert('Upload failed: ' + (error as Error).message);
    } finally {
      setUploading(false);
    }
  };

  return (
    <div>
      <input
        type="file"
        onChange={(e) => setFile(e.target.files?.[0] || null)}
        disabled={uploading}
      />
      <button onClick={handleUpload} disabled={uploading || !file}>
        {uploading ? 'Uploading...' : 'Upload'}
      </button>
    </div>
  );
};
```

## Bucket Policy for Signed URL Safety

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyPublicAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Sid": "AllowSignedUrlAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_ID:user/s3-app-user"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/users/*"
    }
  ]
}
```
