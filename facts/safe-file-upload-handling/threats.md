# File Upload Security Threats & Mitigations

## 1. Zip Bomb (Denial of Service)

A compressed archive that expands to massive size when extracted.

```
File: bomb.zip (100KB on disk)
Uncompressed: 5TB
```

### Attack Example
```bash
# Create a 1GB file of zeros (compresses well)
dd if=/dev/zero bs=1M count=1000 | gzip > bomb.gz

# Result: 1GB compresses to ~1MB
# Unzip it on target: runs out of disk space
```

### Mitigation
```javascript
// 1. Limit decompressed size before extracting
const maxDecompressed = 500 * 1024 * 1024; // 500MB

// 2. Monitor extraction size in real-time
let extracted = 0;
archive.on('data', (chunk) => {
  extracted += chunk.length;
  if (extracted > maxDecompressed) {
    stream.destroy(); // Stop extraction
    throw new Error('Archive too large');
  }
});

// 3. Use separate volume with quota
// Store uploads on separate partition with limited space

// 4. Implement extraction timeout
const timeout = 10000; // 10 seconds
setTimeout(() => process.abort(), timeout);
```

## 2. Path Traversal (Directory Escape)

Archive contains files that escape the intended directory.

```
Archive contents:
├── image.jpg
├── ../../../../etc/passwd      ← Escape!
└── ../../var/www/html/shell.php ← RCE!
```

### Attack Example
```bash
# Create archive with path traversal
zip -r exploit.zip image.jpg
# Edit zip to include: ../../../../etc/shadow
# Extract: writes to /etc/shadow
```

### Mitigation
```javascript
// 1. Validate all paths before extraction
const entry = archive.entry;

if (entry.path.includes('..') ||
    entry.path.startsWith('/') ||
    path.resolve(basePath, entry.path) !== path.resolve(basePath, entry.path)) {
  throw new Error('Path traversal detected');
}

// 2. Use normalized path
const safePath = path.join(basePath, path.basename(entry.path));

// 3. Whitelist allowed directories
const allowedDirs = ['/uploads/user123/'];
if (!safePath.startsWith(allowedDirs[0])) {
  throw new Error('Invalid directory');
}
```

## 3. MIME Type Spoofing

Client claims file is one type, but it's actually another.

```javascript
// WRONG - trust client
const filename = req.file.originalname;
if (!filename.endsWith('.jpg')) {
  return res.status(400).send('Only JPG allowed');
}

// Client: Uploads "shell.php" renamed to "image.jpg"
// Server accepts it, uploads to web-accessible directory
// Attacker: Accesses http://server.com/uploads/image.jpg -> executes PHP
```

### Mitigation
```javascript
// 1. Detect actual file type
const fileType = require('file-type');
const detected = await fileType.fromBuffer(buffer);

if (detected.mime !== 'image/jpeg') {
  throw new Error('MIME type mismatch');
}

// 2. Check magic bytes
function verifyJPEG(buffer) {
  // JPEG starts with FF D8 FF
  return buffer[0] === 0xFF && buffer[1] === 0xD8 && buffer[2] === 0xFF;
}

// 3. Validate file signature (magic numbers)
const signatures = {
  'image/jpeg': [0xFF, 0xD8, 0xFF],
  'image/png': [0x89, 0x50, 0x4E, 0x47],
  'application/pdf': [0x25, 0x50, 0x44, 0x46] // %PDF
};

function isValidSignature(buffer, mime) {
  const sig = signatures[mime];
  if (!sig) return false;
  return sig.every((byte, i) => buffer[i] === byte);
}

// 4. Never execute uploaded files
// Store in non-web-accessible directory
// Serve through explicit download handler
```

## 4. Executable Upload (Remote Code Execution)

Uploading executable files that get executed on server.

```javascript
// WRONG - store in web directory and execute
app.post('/upload', upload.single('file'), (req, res) => {
  const filepath = path.join(__dirname, 'public', req.file.filename);
  fs.writeFileSync(filepath, req.file.buffer);

  // If attacker uploads shell.php and file has execute permissions...
  // Accessing /public/shell.php executes the code!
});

// WRONG - execute uploaded files
exec(`convert ${filepath} -resize 100x100 ${output}`); // ImageMagick injection
```

### Mitigation
```javascript
// 1. Whitelist file extensions strictly
const allowed = ['jpg', 'jpeg', 'png', 'gif', 'pdf'];
const ext = path.extname(req.file.originalname).toLowerCase().slice(1);
if (!allowed.includes(ext)) {
  throw new Error('File type not allowed');
}

// 2. Store outside web root
const uploadDir = '/data/uploads'; // Not under /public or /var/www/html

// 3. Disable script execution
// In web server (nginx/Apache) - never execute scripts in upload directory
// nginx.conf:
location /uploads {
  # Disable PHP execution
  location ~ \.php$ {
    deny all;
  }
  # Disable script execution
  location ~ \.(sh|exe|bat)$ {
    deny all;
  }
}

// 4. Use object storage (S3) - can't execute files
const s3Url = await uploadToS3(buffer);
// S3 serves as download, never as executable

// 5. Sanitize command injection
// WRONG:
exec(`convert ${userFile} -resize 100x100 ${output}`);

// RIGHT:
exec('convert', [userFile, '-resize', '100x100', output]);
// Or use library:
sharp(buffer).resize(100, 100).toBuffer();
```

## 5. Malware & Polyglot Files

Files that contain malicious code disguised as legitimate files.

```
Polyglot file example:
├── Valid JPEG header (passes image validation)
├── JPEG image data
└── Embedded executable/script at end (ignored by image viewers)
```

### Mitigation
```javascript
// 1. Malware scanning (ClamAV)
const clamscan = require('clamscan');

async function scanForMalware(buffer) {
  const result = await clamscan.scanBuffer(buffer);
  return !result.isInfected;
}

// 2. Use trusted libraries for processing
// DON'T use ImageMagick with user input (command injection)
// DO use sharp (safe, popular)
const sharp = require('sharp');
const processed = await sharp(buffer).resize(100, 100).toBuffer();

// 3. Re-encode files (removes embedded content)
const reencoded = await sharp(buffer).jpeg({ quality: 90 }).toBuffer();
// This strips anything after JPEG data

// 4. Sandbox file processing
// Use separate container/service to process uploads
// If malware is detected, only that container is affected
```

## 6. Storage Exhaustion

Legitimate files that consume all available disk space.

```javascript
// WRONG - no quotas
app.post('/upload', upload.single('file'), (req, res) => {
  const size = req.file.size;
  saveToDiskinish(req.file.buffer); // Could be 1TB
});

// Result: Disk full -> entire server down
```

### Mitigation
```javascript
// 1. Implement per-user quotas
const USER_QUOTA = 5 * 1024 * 1024 * 1024; // 5GB per user

async function checkQuota(userId, fileSize) {
  const used = await FileRecord.aggregate([
    { $match: { userId } },
    { $group: { _id: null, total: { $sum: '$size' } } }
  ]);

  const usedSize = used[0]?.total || 0;
  if (usedSize + fileSize > USER_QUOTA) {
    throw new Error('Quota exceeded');
  }
}

// 2. Monitor disk space
const df = require('diskusage');
const { available } = await df.check('/data');

if (available < MIN_FREE_SPACE) {
  throw new Error('Insufficient disk space');
}

// 3. Use object storage with billing limits
// S3 charges per GB stored
// Billing alerts trigger when usage is high
```

## Summary: Complete Checklist

- [ ] Validate file size before accepting
- [ ] Verify MIME type (re-check actual bytes, not client claim)
- [ ] Check magic numbers/file signatures
- [ ] Scan for malware (ClamAV)
- [ ] Prevent path traversal (no `..`, no absolute paths)
- [ ] Check zip bomb decompressed size
- [ ] Limit archive entry count
- [ ] Store outside web root
- [ ] Disable script execution in upload directory
- [ ] Use object storage (S3) when possible
- [ ] Re-encode images (sharp library)
- [ ] Implement per-user quotas
- [ ] Monitor disk space
- [ ] Rate limit uploads
- [ ] Log all uploads for auditing
