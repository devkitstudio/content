# Signed URLs: Secure File Access Pattern

## Problem with Public Buckets

```
Bucket: mybucket.s3.amazonaws.com
Public bucket → Anyone can list/read files

Attacker:
- Enumerates bucket: GET /?prefix=admin/
- Downloads files: GET /admin/passwords.csv
- Searches for patterns: /backups/, /secrets/, /api-keys/
- No authentication needed
```

## Signed URL Solution

**Principle**: Generate time-limited, cryptographically signed URLs that grant temporary access.

```
1. User authenticated to your app
2. App generates signed URL (valid 15 minutes)
3. App returns signed URL to frontend
4. Frontend requests file using signed URL
5. S3 validates signature, expiration, and permissions
6. File is served or access denied
7. After 15 minutes, URL expires and becomes invalid
```

## How Signing Works

```
1. Request Details:
   - Method: GET
   - Bucket: mybucket
   - Key: files/document.pdf
   - Expiry: 900 seconds from now

2. String to Sign:
   GET\n
   \n
   \n
   \n
   \n
   /mybucket/files/document.pdf?X-Amz-Algorithm=...

3. Signature = HMAC-SHA256(SecretKey, StringToSign)
   Signature = a1b2c3d4...

4. Signed URL:
   https://mybucket.s3.amazonaws.com/files/document.pdf
   ?X-Amz-Algorithm=AWS4-HMAC-SHA256
   &X-Amz-Credential=AKIAIOSFODNN7EXAMPLE/...
   &X-Amz-Date=20260406T120000Z
   &X-Amz-Expires=900
   &X-Amz-Signature=a1b2c3d4...
```

## Key Characteristics

| Property | Value |
|----------|-------|
| **Authentication** | AWS Access Key + Secret Key |
| **Authorization** | IAM policy of key that signed it |
| **Expiration** | 15 minutes to 7 days (configurable) |
| **Signature** | HMAC-SHA256 (unbreakable without secret) |
| **User Identity** | Not included (anonymous access) |
| **IP Restrictions** | Optional (x-amz-security-token) |
| **Conditions** | Content-type, bucket, key, etc. |

## When to Use Signed URLs

| Scenario | Use Signed URL? |
|----------|-----------------|
| Download document (user owns) | Yes |
| Download public file | No (use public URL) |
| Upload file to S3 | Yes (POST signature) |
| Temporary API access | Yes |
| Confidential files | Yes (short expiry) |

## Best Practices

1. **Short Expiration**: 15-30 minutes for normal files, 5 minutes for sensitive
2. **Regenerate**: Don't cache URLs, generate new one per request
3. **Use Separate Key**: Create IAM user with only necessary S3 permissions
4. **Log Access**: CloudTrail logs who generated URL and when
5. **Scope by Resource**: Sign only the specific file/prefix user needs
6. **Content-Type Validation**: Include in signature to prevent MIME type exploitation
7. **Versioning**: Support file versions to allow old URLs to expire safely
