# Multi-Channel Notification System

## Architecture Overview

```
                  ┌──────────────────────┐
                  │   Event Source       │
                  │ (OrderPlaced, etc.)  │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │ Notification Service │
                  │ (Dispatcher)         │
                  └──────────┬───────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
    ┌─────────┐         ┌────────┐        ┌──────────┐
    │  Email  │         │  Push  │        │   SMS    │
    │ Queue   │         │ Queue  │        │  Queue   │
    └────┬────┘         └───┬────┘        └────┬─────┘
         │                  │                   │
         ▼                  ▼                   ▼
    ┌─────────┐         ┌────────┐        ┌──────────┐
    │SendGrid │         │Firebase│        │   Twilio │
    │ (sync)  │         │ (async)│        │  (sync)  │
    └─────────┘         └────────┘        └──────────┘

In-App: Direct DB insert (in notification dispatcher)
```

## Key Challenges

| Channel | Challenge | Solution |
|---------|-----------|----------|
| **Email** | Slow (2-5s per send), high failure rate | Queue + retries + bounce handling |
| **Push** | Provider rate limits, token expiration | Batch processing, remove dead tokens |
| **SMS** | Expensive ($0.01-0.05 per SMS) | User preferences, don't oversend |
| **In-app** | Not critical if missed | Direct DB insert, best-effort |

## Design Principles

1. **Decouple from source:** Event triggers dispatcher, dispatcher handles channels
2. **Respect preferences:** User can disable each channel
3. **Degrade gracefully:** If SMS fails, don't block email
4. **Idempotent:** Resending same notification twice = okay
5. **Audit trail:** Log every send attempt

## Notification Dispatcher

```python
from enum import Enum
from dataclasses import dataclass
from datetime import datetime

class NotificationChannel(Enum):
    EMAIL = "email"
    PUSH = "push"
    SMS = "sms"
    IN_APP = "in_app"

@dataclass
class Notification:
    user_id: str
    title: str
    body: str
    channels: list[NotificationChannel]
    metadata: dict = None
    priority: str = "normal"  # normal, high, urgent

class NotificationDispatcher:
    def __init__(self, queue_service, db, logger):
        self.queue = queue_service
        self.db = db
        self.logger = logger

    def dispatch(self, notification: Notification):
        # Get user preferences
        user = self.db.users.find_one(notification.user_id)
        preferences = user.notification_preferences

        # Create notification record
        notif_record = {
            'user_id': notification.user_id,
            'title': notification.title,
            'body': notification.body,
            'created_at': datetime.utcnow(),
            'status': 'pending',
            'channels': {}
        }

        # Dispatch to each channel
        for channel in notification.channels:
            # Skip if user disabled channel
            if not preferences.get(f'enable_{channel.value}', True):
                continue

            # Skip SMS for opt-out users
            if channel == NotificationChannel.SMS:
                if not user.phone_verified or not preferences.get('enable_sms'):
                    continue

            # Dispatch
            try:
                if channel == NotificationChannel.EMAIL:
                    self._dispatch_email(notification, user)
                elif channel == NotificationChannel.PUSH:
                    self._dispatch_push(notification, user)
                elif channel == NotificationChannel.SMS:
                    self._dispatch_sms(notification, user)
                elif channel == NotificationChannel.IN_APP:
                    self._dispatch_in_app(notification, user)

                notif_record['channels'][channel.value] = {'status': 'queued'}
            except Exception as e:
                self.logger.error(f"Failed to dispatch {channel.value}: {e}")
                notif_record['channels'][channel.value] = {
                    'status': 'failed',
                    'error': str(e)
                }

        # Save notification record
        self.db.notifications.insert_one(notif_record)
        return notif_record['_id']

    def _dispatch_email(self, notification: Notification, user):
        # Queue for async processing (SendGrid can be slow)
        self.queue.push('email.queue', {
            'user_id': user['_id'],
            'email': user['email'],
            'title': notification.title,
            'body': notification.body,
            'template': 'generic_notification',
            'retry_count': 0,
            'max_retries': 3
        })

    def _dispatch_push(self, notification: Notification, user):
        # Queue for batching
        self.queue.push('push.queue', {
            'user_id': user['_id'],
            'title': notification.title,
            'body': notification.body,
            'tokens': user.get('push_tokens', []),
            'priority': notification.priority
        })

    def _dispatch_sms(self, notification: Notification, user):
        # Queue with rate limiting
        self.queue.push('sms.queue', {
            'user_id': user['_id'],
            'phone': user['phone'],
            'message': f"{notification.title}: {notification.body}",
            'retry_count': 0,
            'max_retries': 1
        })

    def _dispatch_in_app(self, notification: Notification, user):
        # Direct insert (no queue needed)
        self.db.in_app_notifications.insert_one({
            'user_id': user['_id'],
            'title': notification.title,
            'body': notification.body,
            'read': False,
            'created_at': datetime.utcnow()
        })
```

## Channel Adapters

### Email Adapter (SendGrid)

```python
import sendgrid
from sendgrid.helpers.mail import Mail, Email, To, Content

class EmailAdapter:
    def __init__(self, api_key):
        self.sg = sendgrid.SendGridAPIClient(api_key)

    def send(self, email_job):
        try:
            mail = Mail(
                from_email=Email("noreply@company.com"),
                to_emails=To(email_job['email']),
                subject=email_job['title'],
                html_content=self._render_template(email_job)
            )

            response = self.sg.send(mail)
            return {'success': response.status_code == 202}

        except Exception as e:
            # Retry logic handled by queue
            raise

    def _render_template(self, job):
        return f"""
        <h1>{job['title']}</h1>
        <p>{job['body']}</p>
        """

# Email consumer/worker
def email_worker():
    redis = redis.Redis()

    while True:
        job = redis.lpop('email.queue')
        if not job:
            time.sleep(1)
            continue

        job = json.loads(job)
        adapter = EmailAdapter(API_KEY)

        try:
            result = adapter.send(job)
            logger.info(f"Email sent to {job['email']}")
        except Exception as e:
            if job['retry_count'] < job['max_retries']:
                job['retry_count'] += 1
                redis.rpush('email.queue', json.dumps(job))
            else:
                logger.error(f"Email failed: {job['email']}")
```

### Push Adapter (Firebase)

```python
from firebase_admin import messaging

class PushAdapter:
    def __init__(self, credentials_path):
        firebase_admin.initialize_app(
            firebase_admin.credentials.Certificate(credentials_path)
        )

    def send_batch(self, push_jobs):
        # Batch multiple tokens efficiently
        for job in push_jobs:
            tokens = job['tokens']

            # Send to all tokens at once
            message = messaging.MulticastMessage(
                notification=messaging.Notification(
                    title=job['title'],
                    body=job['body']
                ),
                tokens=tokens
            )

            try:
                response = messaging.send_multicast(message)
                self._handle_errors(job['user_id'], response)
            except Exception as e:
                logger.error(f"Push failed for user {job['user_id']}: {e}")

    def _handle_errors(self, user_id, response):
        # Remove dead tokens
        if response.failure_count > 0:
            failures = response.failures
            dead_tokens = [f.exception.registration_token for f in failures]
            db.users.update_one(
                {'_id': user_id},
                {'$pullAll': {'push_tokens': dead_tokens}}
            )
```

### SMS Adapter (Twilio)

```python
from twilio.rest import Client

class SMSAdapter:
    def __init__(self, account_sid, auth_token):
        self.client = Client(account_sid, auth_token)

    def send(self, sms_job):
        try:
            message = self.client.messages.create(
                body=sms_job['message'],
                from_="+1234567890",
                to=sms_job['phone']
            )
            return {'success': True, 'sid': message.sid}
        except Exception as e:
            logger.error(f"SMS failed: {e}")
            raise

# SMS worker with rate limiting
def sms_worker():
    rate_limiter = RateLimiter(calls=100, period=60)  # 100 SMS/minute

    while True:
        job = queue.pop('sms.queue')
        if not job:
            time.sleep(1)
            continue

        rate_limiter.wait()

        adapter = SMSAdapter(TWILIO_SID, TWILIO_TOKEN)
        try:
            result = adapter.send(job)
            logger.info(f"SMS sent to {job['phone']}")
        except Exception as e:
            if job['retry_count'] < job['max_retries']:
                job['retry_count'] += 1
                queue.push('sms.queue', job)
```

## User Preferences

```python
# Database schema
{
  "user_id": "user123",
  "notification_preferences": {
    "enable_email": True,
    "enable_push": True,
    "enable_sms": True,
    "enable_in_app": True,
    "email_frequency": "immediate",  # immediate, daily, weekly
    "quiet_hours": {
      "enabled": True,
      "start": "22:00",  # 10 PM
      "end": "08:00"     # 8 AM
    },
    "categories": {
      "order_updates": True,
      "promotions": False,
      "news": False
    }
  }
}
```

## Idempotency Key

```python
# Prevent duplicate notifications
def dispatch_with_idempotency(notification: Notification, idempotency_key: str):
    # Check if already sent
    existing = db.notifications.find_one({
        'idempotency_key': idempotency_key
    })

    if existing:
        return existing['_id']

    # New notification
    notif_record = {
        'idempotency_key': idempotency_key,
        # ... rest of notification
    }

    db.notifications.insert_one(notif_record)
    dispatcher.dispatch(notification)
```

## Scalability Notes

- **Queue-based design:** Each channel scales independently
- **Batching:** Push notifications are batched by Firebase
- **Rate limiting:** SMS respects provider limits
- **Monitoring:** Track success rate per channel
- **Dead letter queue:** Failed messages after retries go to DLQ for investigation
