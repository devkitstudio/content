# Code Examples: Password Hashing

## Argon2 (Recommended)

```typescript
import argon2 from 'argon2';

class AuthenticationService {
  // Configuration for security
  private argonOptions = {
    type: argon2.argon2id,  // Most resistant to GPU attacks
    memoryCost: 65536,       // 64 MB
    timeCost: 2,             // 2 iterations
    parallelism: 4,          // 4 threads
    saltLength: 16,          // 16 bytes of random salt
    raw: false               // Return encoded hash
  };

  async hashPassword(password: string): Promise<string> {
    // Validate password strength
    if (!this.isStrongPassword(password)) {
      throw new Error('Password too weak');
    }

    try {
      const hash = await argon2.hash(password, this.argonOptions);
      return hash;
    } catch (error) {
      throw new Error('Failed to hash password');
    }
  }

  async verifyPassword(password: string, hash: string): Promise<boolean> {
    try {
      return await argon2.verify(hash, password);
    } catch (error) {
      // Invalid hash format
      return false;
    }
  }

  private isStrongPassword(password: string): boolean {
    // Minimum 12 characters, mix of types
    const minLength = 12;
    const hasUppercase = /[A-Z]/.test(password);
    const hasLowercase = /[a-z]/.test(password);
    const hasNumbers = /[0-9]/.test(password);
    const hasSpecial = /[!@#$%^&*]/.test(password);

    const strength = [
      hasUppercase,
      hasLowercase,
      hasNumbers,
      hasSpecial
    ].filter(Boolean).length;

    return password.length >= minLength && strength >= 3;
  }
}

// Usage
const auth = new AuthenticationService();

// During registration
const password = 'MySecure!Pass123';
const hash = await auth.hashPassword(password);
await db.users.create({ email: 'user@example.com', passwordHash: hash });

// During login
const user = await db.users.findOne({ email: 'user@example.com' });
const isValid = await auth.verifyPassword(password, user.passwordHash);
if (!isValid) {
  throw new Error('Invalid password');
}
```

## Bcrypt (Fallback)

```typescript
import bcrypt from 'bcrypt';

class LegacyAuthService {
  private saltRounds = 12;  // 2^12 = 4096 rounds

  async hashPassword(password: string): Promise<string> {
    return await bcrypt.hash(password, this.saltRounds);
  }

  async verifyPassword(password: string, hash: string): Promise<boolean> {
    return await bcrypt.compare(password, hash);
  }

  // Increase cost over time (upgrade old hashes)
  async upgradeHashIfNeeded(
    password: string,
    oldHash: string,
    userId: string
  ): Promise<void> {
    // If cost < 12, rehash and update database
    const costRegex = /^\$2[aby]\$(\d{2})\$/;
    const match = oldHash.match(costRegex);
    const currentCost = match ? parseInt(match[1], 10) : 0;

    if (currentCost < 12) {
      const newHash = await this.hashPassword(password);
      await db.users.update(
        { id: userId },
        { passwordHash: newHash }
      );
    }
  }
}

// Usage
const auth = new LegacyAuthService();
const hash = await auth.hashPassword('MyPassword123');

// Verify
const isValid = await auth.verifyPassword('MyPassword123', hash);

// Upgrade old bcrypt(10) to bcrypt(12)
if (user.passwordCost === 10) {
  await auth.upgradeHashIfNeeded(
    plainPassword,
    user.passwordHash,
    user.id
  );
}
```

## PBKDF2 (FIPS Compliance)

```typescript
import crypto from 'crypto';

class FIPSAuthService {
  // OWASP 2023 recommendations
  private iterations = 600000;
  private algorithm = 'sha256';
  private saltLength = 32;
  private keyLength = 64;

  async hashPassword(password: string): Promise<string> {
    return new Promise((resolve, reject) => {
      const salt = crypto.randomBytes(this.saltLength);

      crypto.pbkdf2(
        password,
        salt,
        this.iterations,
        this.keyLength,
        this.algorithm,
        (err, derivedKey) => {
          if (err) reject(err);

          // Format: iterations$salt$hash
          const hash = `${this.iterations}$${salt.toString('hex')}$${derivedKey.toString('hex')}`;
          resolve(hash);
        }
      );
    });
  }

  async verifyPassword(password: string, hash: string): Promise<boolean> {
    return new Promise((resolve) => {
      try {
        const [iterations, saltHex, hashHex] = hash.split('$');
        const salt = Buffer.from(saltHex, 'hex');
        const storedHash = Buffer.from(hashHex, 'hex');

        crypto.pbkdf2(
          password,
          salt,
          parseInt(iterations, 10),
          this.keyLength,
          this.algorithm,
          (err, derivedKey) => {
            if (err) {
              resolve(false);
            } else {
              // Constant-time comparison
              resolve(crypto.timingSafeEqual(derivedKey, storedHash));
            }
          }
        );
      } catch (error) {
        resolve(false);
      }
    });
  }

  // Increase iterations yearly
  async upgradeHashIfNeeded(
    password: string,
    oldHash: string,
    userId: string
  ): Promise<void> {
    const [iterations] = oldHash.split('$');
    const oldIterations = parseInt(iterations, 10);

    // Upgrade if iterations < current recommendation
    if (oldIterations < this.iterations) {
      const newHash = await this.hashPassword(password);
      await db.users.update(
        { id: userId },
        { passwordHash: newHash, hashUpdatedAt: new Date() }
      );
    }
  }
}
```

## Migration from MD5 to Argon2

```typescript
// This is the scenario: You have MD5 hashes, need to upgrade

class PasswordMigration {
  // Step 1: Identify old MD5 hashes (no prefix)
  // Step 2: On login, rehash if needed
  // Step 3: Gradually upgrade database

  async authenticateAndUpgrade(
    email: string,
    plainPassword: string
  ): Promise<User> {
    const user = await db.users.findOne({ email });
    if (!user) {
      throw new Error('User not found');
    }

    // Check if hash is old MD5 (no prefix) or new Argon2/bcrypt
    const isOldHash = !user.passwordHash.includes('$');

    if (isOldHash) {
      // Verify with old MD5
      const md5Hash = crypto
        .createHash('md5')
        .update(plainPassword)
        .digest('hex');

      if (md5Hash !== user.passwordHash) {
        throw new Error('Invalid password');
      }

      // Immediately rehash with Argon2
      const newHash = await argon2.hash(plainPassword, {
        type: argon2.argon2id,
        memoryCost: 65536,
        timeCost: 2
      });

      await db.users.update(
        { id: user.id },
        {
          passwordHash: newHash,
          migratedAt: new Date(),
          hashAlgorithm: 'argon2id'
        }
      );
    } else {
      // Verify with Argon2/bcrypt
      const isValid = await argon2.verify(user.passwordHash, plainPassword);
      if (!isValid) {
        throw new Error('Invalid password');
      }
    }

    return user;
  }

  // Background job to migrate remaining hashes
  async migrateRemainingHashes(): Promise<{ migrated: number }> {
    const oldUsers = await db.users.findAll({
      where: { hashAlgorithm: null }
    });

    let migrated = 0;

    for (const user of oldUsers) {
      try {
        // You need password reset to migrate these
        // Or use a background migration with manual password reset
        // For now, mark that they need upgrade on next login
        migrated++;
      } catch (error) {
        console.error(`Failed to migrate user ${user.id}`, error);
      }
    }

    return { migrated };
  }
}

// Usage
const migration = new PasswordMigration();
const user = await migration.authenticateAndUpgrade(
  'user@example.com',
  'plainPassword'
);
```

## Rate Limiting on Failed Logins

```typescript
import RedisRateLimiter from 'redis';

class RateLimitedAuth {
  private redis = new RedisRateLimiter();
  private maxAttempts = 5;
  private lockoutDuration = 900; // 15 minutes

  async authenticateWithRateLimit(
    email: string,
    password: string
  ): Promise<User> {
    const lockoutKey = `login_lockout:${email}`;
    const attemptsKey = `login_attempts:${email}`;

    // Check if account is locked
    const isLockedOut = await this.redis.get(lockoutKey);
    if (isLockedOut) {
      throw new Error('Account temporarily locked. Try again later.');
    }

    // Check attempt count
    const attempts = await this.redis.incr(attemptsKey);
    if (attempts === 1) {
      await this.redis.expire(attemptsKey, this.lockoutDuration);
    }

    if (attempts > this.maxAttempts) {
      // Lock account
      await this.redis.setex(
        lockoutKey,
        this.lockoutDuration,
        '1'
      );
      throw new Error('Too many failed attempts. Account locked.');
    }

    // Authenticate
    const user = await this.authenticate(email, password);

    if (!user) {
      throw new Error('Invalid credentials');
    }

    // Clear attempts on success
    await Promise.all([
      this.redis.del(attemptsKey),
      this.redis.del(lockoutKey)
    ]);

    return user;
  }

  private async authenticate(email: string, password: string): Promise<User | null> {
    const user = await db.users.findOne({ email });
    if (!user) return null;

    const isValid = await argon2.verify(user.passwordHash, password);
    return isValid ? user : null;
  }
}
```

## Constant-Time Comparison

```typescript
// WRONG: String comparison leaks timing information
function compareWrong(a: string, b: string): boolean {
  return a === b;  // Returns early if first char differs
  // Time varies based on where strings differ
}

// CORRECT: Constant-time comparison
import crypto from 'crypto';

function compareCorrect(a: string, b: string): boolean {
  // crypto.timingSafeEqual prevents timing attacks
  try {
    return crypto.timingSafeEqual(
      Buffer.from(a, 'utf8'),
      Buffer.from(b, 'utf8')
    );
  } catch (error) {
    // Buffers different length
    return false;
  }
}

// Usage in password verification
const hash = await db.users.findOne({ email }).passwordHash;
const isValid = crypto.timingSafeEqual(
  Buffer.from(hash),
  Buffer.from(computedHash)
);
```
