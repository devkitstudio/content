# Queue-Based Pipeline Code

## Complete Example: Order Notification Pipeline

```python
import json
import logging
import time
from typing import List, Dict
from dataclasses import dataclass, asdict
from datetime import datetime
from enum import Enum
import redis
import requests

logger = logging.getLogger(__name__)

# Redis connection
redis_client = redis.Redis(host='localhost', port=6379, db=0)

class NotificationPriority(Enum):
    LOW = 1
    NORMAL = 2
    HIGH = 3

class Channel(Enum):
    EMAIL = "email"
    PUSH = "push"
    SMS = "sms"
    IN_APP = "in_app"

@dataclass
class NotificationJob:
    user_id: str
    title: str
    body: str
    channels: List[str]
    priority: str = "normal"
    metadata: dict = None

# ============ Notification Service (Main Entry Point) ============

class NotificationService:
    def __init__(self):
        self.queue_prefix = "notification_queue:"

    def notify(self, job: NotificationJob, event_source: str = None):
        """Main entry point for sending notifications"""

        # Log notification attempt
        logger.info(f"Processing notification for user {job.user_id}")

        # Route to channels based on priority
        if job.priority == "urgent":
            # Fast path: push only
            self._enqueue_channel("push", job)
        else:
            # Standard path: all channels
            for channel in job.channels:
                self._enqueue_channel(channel, job)

    def _enqueue_channel(self, channel: str, job: NotificationJob):
        """Queue job for specific channel"""
        queue_key = f"{self.queue_prefix}{channel}"
        redis_client.rpush(queue_key, json.dumps(asdict(job)))
        logger.info(f"Enqueued {channel} notification for user {job.user_id}")

# ============ Email Worker ============

class EmailWorker:
    def __init__(self, sendgrid_key: str):
        self.api_key = sendgrid_key
        self.base_url = "https://api.sendgrid.com/v3/mail/send"

    def process_queue(self):
        """Long-running email worker"""
        while True:
            # Batch process emails
            jobs = []
            for _ in range(10):  # Batch size of 10
                item = redis_client.lpop("notification_queue:email")
                if item:
                    jobs.append(json.loads(item))

            if not jobs:
                time.sleep(5)
                continue

            for job in jobs:
                self._send_email(job)

    def _send_email(self, job: Dict):
        """Send single email with retry logic"""
        user = self._get_user(job['user_id'])
        if not user.get('email'):
            logger.warning(f"No email for user {job['user_id']}")
            return

        email_body = {
            "personalizations": [{
                "to": [{"email": user['email']}],
                "subject": job['title']
            }],
            "from": {"email": "noreply@company.com"},
            "content": [{
                "type": "text/html",
                "value": f"<h1>{job['title']}</h1><p>{job['body']}</p>"
            }]
        }

        try:
            response = requests.post(
                self.base_url,
                json=email_body,
                headers={"Authorization": f"Bearer {self.api_key}"}
            )

            if response.status_code == 202:
                logger.info(f"Email sent to {user['email']}")
            else:
                self._retry_job(job, "email")
                logger.error(f"Email failed: {response.status_code}")

        except Exception as e:
            self._retry_job(job, "email")
            logger.error(f"Email exception: {e}")

    def _retry_job(self, job: Dict, channel: str, delay: int = 300):
        """Requeue job after delay"""
        retry_key = f"notification_queue:{channel}:retry"
        redis_client.zadd(retry_key, {json.dumps(job): time.time() + delay})

    def _get_user(self, user_id: str) -> Dict:
        # Mock: fetch from database
        return {"email": f"user{user_id}@example.com"}

# ============ Push Notification Worker ============

class PushWorker:
    def __init__(self, fcm_credentials: str):
        self.fcm_key = fcm_credentials

    def process_queue(self):
        """Process push notifications"""
        while True:
            # Batch collect jobs
            jobs = []
            for _ in range(50):  # Batch 50 push notifications
                item = redis_client.lpop("notification_queue:push")
                if item:
                    jobs.append(json.loads(item))

            if not jobs:
                time.sleep(2)
                continue

            # Group by user for batching
            by_user = {}
            for job in jobs:
                user_id = job['user_id']
                if user_id not in by_user:
                    by_user[user_id] = []
                by_user[user_id].append(job)

            # Send batch
            for user_id, user_jobs in by_user.items():
                self._send_batch(user_id, user_jobs)

    def _send_batch(self, user_id: str, jobs: List[Dict]):
        """Send multiple notifications to same user"""
        user = self._get_user(user_id)
        tokens = user.get('push_tokens', [])

        if not tokens:
            logger.warning(f"No push tokens for user {user_id}")
            return

        for job in jobs:
            payload = {
                "notification": {
                    "title": job['title'],
                    "body": job['body']
                },
                "tokens": tokens
            }

            try:
                # Simulate Firebase API call
                response = self._call_firebase(payload)
                if response['success']:
                    logger.info(f"Push sent to user {user_id}")
                else:
                    self._cleanup_dead_tokens(user_id, response['failed_tokens'])
            except Exception as e:
                logger.error(f"Push failed: {e}")

    def _call_firebase(self, payload: Dict) -> Dict:
        # Mock Firebase response
        return {
            'success': True,
            'failed_tokens': []
        }

    def _cleanup_dead_tokens(self, user_id: str, dead_tokens: List[str]):
        """Remove expired tokens from user"""
        for token in dead_tokens:
            logger.info(f"Removing dead token for user {user_id}")

    def _get_user(self, user_id: str) -> Dict:
        return {
            'push_tokens': [f'token_{user_id}_1', f'token_{user_id}_2']
        }

# ============ SMS Worker ============

class SMSWorker:
    def __init__(self, twilio_sid: str, twilio_token: str):
        self.account_sid = twilio_sid
        self.auth_token = twilio_token
        self.base_url = "https://api.twilio.com/2010-04-01/Accounts"

    def process_queue(self):
        """Process SMS with rate limiting"""
        rate_limit = RateLimiter(calls=100, period=60)  # 100 SMS/minute

        while True:
            item = redis_client.lpop("notification_queue:sms")
            if not item:
                time.sleep(5)
                continue

            rate_limit.wait()

            job = json.loads(item)
            self._send_sms(job)

    def _send_sms(self, job: Dict):
        """Send SMS message"""
        user = self._get_user(job['user_id'])
        if not user.get('phone'):
            logger.warning(f"No phone for user {job['user_id']}")
            return

        try:
            response = requests.post(
                f"{self.base_url}/{self.account_sid}/Messages.json",
                data={
                    "From": "+1234567890",
                    "To": user['phone'],
                    "Body": job['body'][:160]  # SMS limit
                },
                auth=(self.account_sid, self.auth_token)
            )

            if response.status_code == 201:
                logger.info(f"SMS sent to {user['phone']}")
            else:
                logger.error(f"SMS failed: {response.status_code}")

        except Exception as e:
            logger.error(f"SMS exception: {e}")

    def _get_user(self, user_id: str) -> Dict:
        return {'phone': f"+1555000{user_id % 10000:04d}"}

# ============ In-App Worker ============

class InAppWorker:
    def process_queue(self):
        """Process in-app notifications (direct DB insert)"""
        while True:
            item = redis_client.lpop("notification_queue:in_app")
            if not item:
                time.sleep(2)
                continue

            job = json.loads(item)
            self._insert_in_app(job)

    def _insert_in_app(self, job: Dict):
        """Insert notification into database"""
        notification = {
            'user_id': job['user_id'],
            'title': job['title'],
            'body': job['body'],
            'read': False,
            'created_at': datetime.utcnow()
        }
        # db.in_app_notifications.insert_one(notification)
        logger.info(f"In-app notification created for user {job['user_id']}")

# ============ Rate Limiter ============

class RateLimiter:
    def __init__(self, calls: int, period: int):
        self.calls = calls
        self.period = period
        self.window = []

    def wait(self):
        """Enforce rate limit"""
        now = time.time()
        # Remove old entries
        self.window = [t for t in self.window if t > now - self.period]

        if len(self.window) >= self.calls:
            sleep_time = self.period - (now - self.window[0])
            if sleep_time > 0:
                time.sleep(sleep_time)
            self.window = []

        self.window.append(time.time())

# ============ Usage Example ============

if __name__ == "__main__":
    service = NotificationService()

    # Send notification
    job = NotificationJob(
        user_id="user123",
        title="Order Confirmed",
        body="Your order #456 has been confirmed",
        channels=["email", "push", "in_app"],
        priority="normal"
    )

    service.notify(job)

    # Start workers (in separate processes)
    # email_worker = EmailWorker("sg_key")
    # email_worker.process_queue()

    # push_worker = PushWorker("fcm_creds")
    # push_worker.process_queue()

    # sms_worker = SMSWorker("twilio_sid", "twilio_token")
    # sms_worker.process_queue()

    # in_app_worker = InAppWorker()
    # in_app_worker.process_queue()
```

## Docker Compose for Full Stack

```yaml
version: '3.8'
services:
  redis:
    image: redis:latest
    ports:
      - "6379:6379"

  notification-service:
    build: .
    environment:
      - REDIS_HOST=redis
      - SENDGRID_KEY=${SENDGRID_KEY}
      - FCM_CREDENTIALS=${FCM_CREDENTIALS}
      - TWILIO_SID=${TWILIO_SID}
      - TWILIO_TOKEN=${TWILIO_TOKEN}
    depends_on:
      - redis

  email-worker:
    build:
      context: .
      dockerfile: Dockerfile.worker
    command: python -m workers.email_worker
    environment:
      - REDIS_HOST=redis
      - SENDGRID_KEY=${SENDGRID_KEY}
    depends_on:
      - redis
    scale: 2  # 2 email workers

  push-worker:
    build:
      context: .
      dockerfile: Dockerfile.worker
    command: python -m workers.push_worker
    environment:
      - REDIS_HOST=redis
      - FCM_CREDENTIALS=${FCM_CREDENTIALS}
    depends_on:
      - redis
    scale: 3  # 3 push workers

  sms-worker:
    build:
      context: .
      dockerfile: Dockerfile.worker
    command: python -m workers.sms_worker
    environment:
      - REDIS_HOST=redis
      - TWILIO_SID=${TWILIO_SID}
      - TWILIO_TOKEN=${TWILIO_TOKEN}
    depends_on:
      - redis
    scale: 1  # 1 SMS worker (rate limited)
```

## Metrics & Monitoring

```python
# Track notification performance
class NotificationMetrics:
    def record_sent(self, channel: str, success: bool, latency_ms: float):
        redis_client.incr(f"metrics:sent:{channel}:{'success' if success else 'failure'}")
        redis_client.rpush(f"metrics:latency:{channel}", latency_ms)

    def get_success_rate(self, channel: str) -> float:
        success = redis_client.get(f"metrics:sent:{channel}:success") or 0
        total = success + (redis_client.get(f"metrics:sent:{channel}:failure") or 0)
        return (int(success) / int(total) * 100) if total > 0 else 0
```

This design scales to handle millions of notifications across multiple channels independently.
