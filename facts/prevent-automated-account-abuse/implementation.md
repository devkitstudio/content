# Code Examples

## Rate Limiting with Redis

```typescript
import RedisRateLimiter from 'ioredis';

class RegistrationRateLimiter {
  private redis = new RedisRateLimiter();

  async checkLimit(ip: string, email: string): Promise<boolean> {
    const ipKey = `rate:reg:ip:${ip}`;
    const emailKey = `rate:reg:email:${email}`;

    const [ipCount, emailCount] = await Promise.all([
      this.redis.incr(ipKey),
      this.redis.incr(emailKey)
    ]);

    // Set expiry on first increment
    if (ipCount === 1) this.redis.expire(ipKey, 3600);
    if (emailCount === 1) this.redis.expire(emailKey, 86400);

    return ipCount <= 5 && emailCount <= 3;
  }
}

// Usage in Express
app.post('/api/register', async (req, res) => {
  const ip = req.ip;
  const { email } = req.body;

  const allowed = await limiter.checkLimit(ip, email);
  if (!allowed) {
    return res.status(429).json({ error: 'Too many registration attempts' });
  }

  // Continue with registration...
});
```

## reCAPTCHA v3 Integration

```typescript
import axios from 'axios';

class CaptchaVerifier {
  async verify(token: string, ip: string): Promise<boolean> {
    const response = await axios.post(
      'https://www.google.com/recaptcha/api/siteverify',
      {
        secret: process.env.RECAPTCHA_SECRET_KEY,
        response: token,
        remoteip: ip
      }
    );

    // Score 0.0 = bot, 1.0 = legitimate user
    return response.data.success && response.data.score > 0.5;
  }
}

// Frontend
const verifyAndRegister = async (formData) => {
  const token = await grecaptcha.execute(
    process.env.REACT_APP_RECAPTCHA_SITE_KEY,
    { action: 'register' }
  );

  const response = await fetch('/api/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ ...formData, captchaToken: token })
  });
};
```

## Device Fingerprinting

```typescript
import { v4 as uuid } from 'uuid';

interface DeviceFingerprint {
  userAgent: string;
  timezone: string;
  canvasHash: string;
  acceptLanguage: string;
}

class DeviceTracker {
  async generateFingerprint(req: Request): Promise<string> {
    const fingerprint: DeviceFingerprint = {
      userAgent: req.get('user-agent') || '',
      timezone: req.body.timezone,
      canvasHash: req.body.canvasHash,
      acceptLanguage: req.get('accept-language') || ''
    };

    return crypto
      .createHash('sha256')
      .update(JSON.stringify(fingerprint))
      .digest('hex');
  }

  async checkDeviceAbuse(fingerprint: string): Promise<boolean> {
    const count = await db.deviceAccounts.count({
      where: { fingerprint, createdAt: { gt: now - 24h } }
    });

    return count < 10; // Max 10 accounts per device per day
  }
}

// Frontend collection
const getCanvasHash = async () => {
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d');
  ctx!.textBaseline = 'top';
  ctx!.font = '32px "Arial"';
  ctx!.textBaseline = 'alphabetic';
  ctx!.fillStyle = '#f60';
  ctx!.fillRect(125, 1, 62, 20);
  ctx!.fillStyle = '#069';
  ctx!.fillText('Device Fingerprint', 2, 15);

  return canvas.toDataURL();
};
```

## Email Verification Flow

```typescript
import { randomInt } from 'crypto';
import nodemailer from 'nodemailer';

class EmailVerification {
  async generateToken(): Promise<string> {
    return String(randomInt(100000, 999999));
  }

  async sendVerification(email: string): Promise<string> {
    const token = await this.generateToken();
    const expires = new Date(Date.now() + 15 * 60 * 1000); // 15 minutes

    await db.verificationTokens.create({
      email,
      token,
      expires
    });

    await nodemailer.createTransport().sendMail({
      to: email,
      subject: 'Verify your account',
      html: `<p>Your verification code: <strong>${token}</strong></p>`
    });

    return token;
  }

  async verify(email: string, token: string): Promise<boolean> {
    const record = await db.verificationTokens.findOne({
      where: { email, token, expires: { gt: new Date() } }
    });

    if (record) {
      await db.verificationTokens.delete({ id: record.id });
      return true;
    }
    return false;
  }
}
```

## IP Reputation Check

```typescript
import axios from 'axios';

class IPReputation {
  async checkReputation(ip: string): Promise<'allow' | 'challenge' | 'block'> {
    try {
      const response = await axios.get(
        `https://api.abuseipdb.com/api/v2/check`,
        {
          params: { ipAddress: ip, maxAgeInDays: 90 },
          headers: { Key: process.env.ABUSEIPDB_API_KEY }
        }
      );

      const abuseScore = response.data.data.abuseConfidenceScore;

      if (abuseScore > 75) return 'block';
      if (abuseScore > 25) return 'challenge'; // Require CAPTCHA
      return 'allow';
    } catch (error) {
      return 'allow'; // Fail open on API errors
    }
  }
}
```

## Complete Registration Middleware

```typescript
app.post('/api/register', async (req, res) => {
  const { email, password, captchaToken, deviceFingerprint } = req.body;
  const ip = req.ip!;

  try {
    // 1. Check IP reputation
    const ipReputation = await ipChecker.checkReputation(ip);
    if (ipReputation === 'block') {
      return res.status(403).json({ error: 'Registration blocked' });
    }

    // 2. Rate limiting
    const allowed = await limiter.checkLimit(ip, email);
    if (!allowed) {
      return res.status(429).json({ error: 'Too many attempts' });
    }

    // 3. CAPTCHA verification
    const captchaValid = await captcha.verify(captchaToken, ip);
    if (!captchaValid && ipReputation === 'challenge') {
      return res.status(400).json({ error: 'CAPTCHA failed' });
    }

    // 4. Device fingerprinting
    const fingerprint = await deviceTracker.generateFingerprint(req);
    const deviceSafe = await deviceTracker.checkDeviceAbuse(fingerprint);
    if (!deviceSafe) {
      return res.status(403).json({ error: 'Device blocked' });
    }

    // 5. Email verification
    const verificationToken = await emailVerifier.sendVerification(email);

    // 6. Create account
    const user = await db.users.create({
      email,
      password: await bcrypt.hash(password, 12),
      fingerprint,
      verificationToken,
      verified: false
    });

    res.json({ userId: user.id, message: 'Check email for verification' });
  } catch (error) {
    res.status(500).json({ error: 'Registration failed' });
  }
});
```
