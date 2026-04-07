# Secure File Upload Pipeline

## 1. Validate File Before Accepting

```javascript
const express = require('express');
const multer = require('multer');
const fs = require('fs/promises');
const path = require('path');
const fileType = require('file-type');

// Configure multer with safeguards
const upload = multer({
  // Don't save immediately - use memory storage
  storage: multer.memoryStorage(),

  // Limit file size (10MB example, adjust as needed)
  limits: {
    fileSize: 10 * 1024 * 1024, // 10MB
    files: 1
  },

  // Only accept specific MIME types
  fileFilter: (req, file, callback) => {
    const allowedMimes = [
      'image/jpeg',
      'image/png',
      'image/webp',
      'application/pdf'
    ];

    if (allowedMimes.includes(file.mimetype)) {
      callback(null, true);
    } else {
      callback(new Error(`File type ${file.mimetype} not allowed`));
    }
  }
});

// Upload endpoint with validation
app.post('/api/files/upload', upload.single('file'), async (req, res) => {
  try {
    const file = req.file;

    // 1. Re-verify file type (don't trust client)
    const detectedType = await fileType.fromBuffer(file.buffer);

    if (!detectedType) {
      return res.status(400).json({ error: 'File type could not be determined' });
    }

    const allowedMimes = ['image/jpeg', 'image/png', 'image/webp', 'application/pdf'];
    if (!allowedMimes.includes(detectedType.mime)) {
      return res.status(400).json({
        error: `File type ${detectedType.mime} not allowed. Detected MIME mismatch.`
      });
    }

    // 2. Scan for malware (integrate ClamAV or similar)
    const isSafe = await scanForMalware(file.buffer);
    if (!isSafe) {
      logger.warn(`Malware detected in upload from ${req.user.id}`);
      return res.status(400).json({ error: 'File failed security scan' });
    }

    // 3. Generate safe filename (no path traversal)
    const safeFilename = generateSafeFilename(file.originalname, detectedType.ext);

    // 4. Store file securely (use S3, not local filesystem)
    const fileUrl = await storeFile(safeFilename, file.buffer, detectedType.mime);

    // 5. Record metadata
    await FileRecord.create({
      userId: req.user.id,
      originalName: file.originalname,
      storageName: safeFilename,
      mimeType: detectedType.mime,
      size: file.buffer.length,
      url: fileUrl
    });

    res.json({
      success: true,
      url: fileUrl,
      filename: safeFilename
    });
  } catch (error) {
    logger.error('File upload error:', error);
    res.status(500).json({ error: 'Upload failed' });
  }
});
```

## 2. Generate Safe Filenames (No Path Traversal)

```javascript
const crypto = require('crypto');

function generateSafeFilename(originalName, ext) {
  // WRONG - allows path traversal
  // return originalName; // Could be "../../etc/passwd.txt"

  // RIGHT - generate random filename
  const timestamp = Date.now();
  const randomHash = crypto.randomBytes(8).toString('hex');
  const safe = `${timestamp}-${randomHash}.${ext}`;

  return safe;
}

// Example:
// Input: "../../../../etc/passwd.txt"
// Output: "1702656000000-a1b2c3d4e5f6g7h8.txt"
```

## 3. Validate After Decompression (Anti-Zip Bomb)

```javascript
const archiver = require('archiver');
const { Extract } = require('unzipper');

async function extractAndValidate(zipBuffer, maxExtractedSize = 1000 * 1024 * 1024) {
  // 1GB max decompressed size

  return new Promise((resolve, reject) => {
    let totalExtracted = 0;
    const files = [];

    zipBuffer
      .pipe(new Extract({ path: tmpdir }))
      .on('entry', (entry) => {
        // Check for path traversal in archive
        if (entry.path.includes('..') || path.isAbsolute(entry.path)) {
          return reject(new Error('Path traversal detected in archive'));
        }

        // Track extracted size
        totalExtracted += entry.size;

        // Stop if size exceeds limit
        if (totalExtracted > maxExtractedSize) {
          return reject(new Error('Archive expands beyond size limit (zip bomb detected)'));
        }

        files.push({
          path: entry.path,
          size: entry.size
        });
      })
      .on('finish', () => resolve(files))
      .on('error', reject);
  });
}

// Streaming extraction for large files
async function extractZipSafely(zipBuffer) {
  const maxExtractedSize = 500 * 1024 * 1024; // 500MB
  const maxEntriesInArchive = 10000;

  const extraction = await extractAndValidate(
    zipBuffer,
    maxExtractedSize
  );

  if (extraction.length > maxEntriesInArchive) {
    throw new Error(`Archive contains too many files (${extraction.length} > ${maxEntriesInArchive})`);
  }

  return extraction;
}
```

## 4. Store in Secure Location (Avoid Local Filesystem)

```javascript
// WRONG - store in web-accessible directory
const wrongPath = path.join(__dirname, 'public/uploads', filename);

// WRONG - store unencrypted
fs.writeFileSync(wrongPath, fileBuffer);

// RIGHT - use object storage (S3)
const AWS = require('aws-sdk');
const s3 = new AWS.S3();

async function storeFile(filename, buffer, mimeType) {
  const key = `uploads/${new Date().getFullYear()}/${filename}`;

  const params = {
    Bucket: process.env.S3_BUCKET,
    Key: key,
    Body: buffer,
    ContentType: mimeType,
    ServerSideEncryption: 'AES256', // Encrypt at rest
    ACL: 'private' // Not public
  };

  const result = await s3.upload(params).promise();

  // Return signed URL that expires in 1 hour
  const url = s3.getSignedUrl('getObject', {
    Bucket: process.env.S3_BUCKET,
    Key: key,
    Expires: 3600
  });

  return url;
}
```

## 5. Scan for Malware (ClamAV Integration)

```javascript
const NodeClam = require('clamscan');

async function initMalwareScanner() {
  const clamscan = await new NodeClam().init({
    clamdscan: {
      host: process.env.CLAMAV_HOST || 'localhost',
      port: process.env.CLAMAV_PORT || 3310
    }
  });

  return clamscan;
}

const scanner = initMalwareScanner();

async function scanForMalware(buffer) {
  try {
    const { isInfected, viruses } = await scanner.scanBuffer(buffer);

    if (isInfected) {
      logger.warn(`Malware detected: ${viruses.join(', ')}`);
      return false;
    }

    return true;
  } catch (error) {
    logger.error('Malware scan error:', error);
    // Fail open vs closed - depending on your security posture
    // Fail closed (safer): reject if scan fails
    return false;
  }
}
```

## 6. Rate Limiting on Uploads

```javascript
const rateLimit = require('express-rate-limit');

const uploadLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 uploads per 15 min
  message: 'Too many uploads, please try again later',
  keyGenerator: (req) => req.user.id // Per user, not per IP
});

app.post('/api/files/upload', uploadLimiter, upload.single('file'), uploadHandler);
```

## 7. Complete Validation Pipeline

```javascript
async function validateFileUpload(file, user) {
  const errors = [];

  // Size check
  if (file.size > MAX_FILE_SIZE) {
    errors.push(`File too large: ${file.size} > ${MAX_FILE_SIZE}`);
  }

  // MIME type verification
  const detected = await fileType.fromBuffer(file.buffer);
  if (!detected || !ALLOWED_MIMES.includes(detected.mime)) {
    errors.push(`Invalid file type: ${detected?.mime || 'unknown'}`);
  }

  // Magic number check (file signature)
  if (!isValidFileSignature(file.buffer, detected?.ext)) {
    errors.push('File signature does not match extension');
  }

  // Malware scan
  const isSafe = await scanForMalware(file.buffer);
  if (!isSafe) {
    errors.push('File failed malware scan');
  }

  // For archives: check contents
  if (detected?.mime === 'application/zip') {
    try {
      await extractAndValidate(file.buffer, MAX_DECOMPRESSED_SIZE);
    } catch (err) {
      errors.push(`Invalid archive: ${err.message}`);
    }
  }

  // Check user quota
  const userUsage = await FileRecord.aggregate([
    { $match: { userId: user.id } },
    { $group: { _id: null, totalSize: { $sum: '$size' } } }
  ]);

  const currentUsage = userUsage[0]?.totalSize || 0;
  if (currentUsage + file.size > USER_QUOTA) {
    errors.push('Storage quota exceeded');
  }

  return { valid: errors.length === 0, errors };
}

// Usage
app.post('/api/files/upload', async (req, res) => {
  const file = req.file;
  const { valid, errors } = await validateFileUpload(file, req.user);

  if (!valid) {
    return res.status(400).json({ errors });
  }

  // Proceed with upload...
});
```
