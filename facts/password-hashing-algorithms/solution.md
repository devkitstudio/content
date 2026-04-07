# Why MD5 is Broken and What to Use Instead

## Why MD5 is Dangerous

### 1. Cryptographically Broken
- **Collision attacks**: Two different inputs produce same hash
- **2004**: Wang & Yu proved collisions are feasible
- **2009**: Md5CRK project found collisions in seconds
- Makes rainbow tables and precomputed attacks possible

### 2. Speed is a Problem, Not a Feature
- MD5 hashes billions of passwords per second
- **Goal**: Make brute force attacks slow, not fast
- Fast hashing enables: `1M attempts/second / GPU = 278 hours max password`
- Modern GPUs can try 100+ billion MD5 hashes/second

### 3. No Salt Protection by Default
- MD5 doesn't have built-in salt
- If you manually add salt, it's still fast
- Without work factor, salt is ineffective

### 4. Rainbow Tables Available
- Precomputed tables of common password → MD5 hash mappings
- Lookup attacks take seconds: `password → MD5 → lookup → plaintext`
- Example: "password" MD5 is known hash

## Comparison of Algorithms

| Algorithm | Speed | Parallelizable | Salt | Work Factor | Status |
|-----------|-------|-----------------|------|------------|--------|
| **MD5** | 10 billion/sec | Yes | No | N/A | Broken |
| **SHA-256** | 1 billion/sec | Yes | No | N/A | NOT for passwords |
| **bcrypt** | 1000/sec | No | Yes | Yes | Excellent |
| **scrypt** | 1/sec | No | Yes | Yes | Excellent |
| **argon2** | 1/sec | No | Yes | Yes | Best choice |

## Why Each Algorithm Fails/Succeeds

### PBKDF2 (Password-Based Key Derivation Function 2)
- **Pros**: Standardized (NIST), tunable iterations, salted
- **Cons**: Designed for encryption keys, not passwords
- **Use case**: When you need FIPS compliance
- **Speed**: ~100k iterations = 100,000 attempts/sec on GPU

### bcrypt (Blowfish-based)
- **Pros**:
  - Automatically includes salt
  - Work factor (cost) increases with time
  - GPU-resistant (small memory usage)
  - Proven track record since 1999
- **Cons**:
  - Slower than scrypt/argon2
  - Output limited to 72 bytes
  - Designed before modern attacks
- **Use case**: Legacy systems, backward compatibility
- **Speed**: ~1000 attempts/sec (tunable)

### scrypt
- **Pros**:
  - Memory-hard: Requires significant RAM
  - GPU resistance through memory hardness
  - Tunable cost parameters
- **Cons**:
  - More complex than bcrypt
  - Less widely implemented
  - Slower startup (needs RAM allocation)
- **Use case**: High-security scenarios
- **Speed**: ~1 attempt/sec with N=16384

### Argon2 (RECOMMENDED)
- **Pros**:
  - Winner of Password Hashing Competition (2015)
  - Memory-hard and time-hard
  - GPU and ASIC resistant
  - Adjustable parameters for future-proofing
  - Proven against side-channel attacks
- **Cons**: Newer (2015), less legacy support
- **Use case**: New systems, modern authentication
- **Speed**: ~1 attempt/sec (tunable)

## Recommended Work Factors

### bcrypt
- Cost: 12 (2^12 rounds)
- Takes ~250ms to hash
- Takes ~250ms per login

```
bcrypt: 12 iterations = 1M hashing attempts/year/GPU
Attack: 12 years with single GPU
Attack: 1 week with 600 GPUs
Recommended: Cost 14 if you can afford 1 second hash time
```

### Argon2
- Time cost: 2 iterations
- Memory cost: 65536 KB (64 MB)
- Parallelism: 4 threads
- Output: 32 bytes
- Takes ~500ms to hash

```
Argon2: 500ms per hash = 2 hashes/second
Attack: 50 years with single GPU (16GB memory)
Memory requirement blocks GPU parallelism
```

### PBKDF2
- Iterations: 600,000+ (OWASP 2023 recommendation)
- Algorithm: HMAC-SHA256
- Takes ~100ms to hash

```
PBKDF2: 600k iterations = 100ms hash time
Attack: 60+ years with single GPU
Upgrade iterations yearly
```

## Summary Table

| Scenario | Algorithm | Reason |
|----------|-----------|--------|
| **New Project** | Argon2 | Best security, future-proof |
| **Enterprise** | Argon2 or bcrypt | Argon2 better, bcrypt if compatibility needed |
| **Legacy System** | bcrypt | Proven, available, good enough |
| **FIPS Required** | PBKDF2 | Only NIST-approved option |
| **Never** | MD5, SHA-1, SHA-256 | Not designed for passwords |

## Password Requirements Checklist

- Use Argon2 (or bcrypt as fallback)
- Never reuse passwords
- Enforce minimum entropy (not character rules)
- Rate limit login attempts
- Implement account lockout
- Add 2FA for sensitive accounts
- Hash database backups
