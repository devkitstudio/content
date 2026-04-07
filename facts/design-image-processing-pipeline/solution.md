# High-Throughput Image Processing Pipeline

## Requirements

- 1,000 images/minute = 16-17 images/second
- 5 output formats per image (thumbnail, small, medium, large, original)
- Processing should not block upload
- Handle failures gracefully

## Architecture

```
┌──────────────────────────────────────┐
│   User Uploads Image (HTTP POST)     │
└────────────┬─────────────────────────┘
             │
             ▼
┌──────────────────────────────────────┐
│   Upload Service (FastAPI)           │
│   - Validate image (size, type)      │
│   - Generate unique ID               │
│   - Upload raw to S3 (temp folder)   │
│   - Queue processing job             │
│   - Return: image_id (for tracking)  │
└────────────┬─────────────────────────┘
             │
             ▼
┌──────────────────────────────────────┐
│   SQS Queue (AWS)                    │
│   - Decouples upload from processing │
│   - Handles backpressure             │
│   - Retries failed jobs              │
└────────────┬─────────────────────────┘
             │
        ┌────┴────┬────────┬────────┐
        │          │        │        │
        ▼          ▼        ▼        ▼
     Worker 1  Worker 2  Worker 3  Worker N
     (Process) (Process) (Process) (Process)
        │          │        │        │
        └────┬─────┴────┬───┴────┬──┘
             │          │        │
             ▼          ▼        ▼
        S3: thumbnail
        S3: small
        S3: medium
        S3: large
        S3: original

        + Update database with URLs
```

## Upload Service

```python
from fastapi import FastAPI, UploadFile, File
import boto3
import uuid
from datetime import datetime

app = FastAPI()
s3_client = boto3.client('s3')
sqs_client = boto3.client('sqs')

BUCKET_NAME = "images-bucket"
TEMP_PREFIX = "uploads/temp/"
QUEUE_URL = "https://sqs.us-east-1.amazonaws.com/123456789/image-processing"
MAX_IMAGE_SIZE = 50 * 1024 * 1024  # 50MB

@app.post("/upload")
async def upload_image(file: UploadFile = File(...)):
    # Validate
    if file.content_type not in ['image/jpeg', 'image/png', 'image/webp']:
        return {"error": "Invalid image type"}, 400

    file_size = 0
    content = await file.read()
    file_size = len(content)

    if file_size > MAX_IMAGE_SIZE:
        return {"error": "Image too large"}, 400

    # Generate unique ID
    image_id = str(uuid.uuid4())

    # Upload raw image to S3 temp folder
    try:
        s3_client.put_object(
            Bucket=BUCKET_NAME,
            Key=f"{TEMP_PREFIX}{image_id}/original.jpg",
            Body=content,
            ContentType=file.content_type,
            Metadata={
                'upload_time': datetime.utcnow().isoformat(),
                'original_filename': file.filename
            }
        )
    except Exception as e:
        logger.error(f"S3 upload failed: {e}")
        return {"error": "Upload failed"}, 500

    # Queue processing job
    try:
        sqs_client.send_message(
            QueueUrl=QUEUE_URL,
            MessageBody=json.dumps({
                'image_id': image_id,
                's3_key': f"{TEMP_PREFIX}{image_id}/original.jpg",
                'timestamp': datetime.utcnow().isoformat()
            }),
            MessageGroupId='images'  # FIFO queue
        )
    except Exception as e:
        logger.error(f"Queue failed: {e}")
        # But don't fail upload - image is already in S3
        # Process will retry

    return {
        "image_id": image_id,
        "status": "uploading",
        "message": "Image queued for processing"
    }
```

## Worker: Image Processing

```python
import boto3
import sqs
import json
import logging
from PIL import Image
import io
from datetime import datetime
import time

logger = logging.getLogger(__name__)

s3_client = boto3.client('s3')
sqs_client = boto3.client('sqs')

BUCKET_NAME = "images-bucket"
TEMP_PREFIX = "uploads/temp/"
PROCESSED_PREFIX = "uploads/processed/"
QUEUE_URL = "https://sqs.us-east-1.amazonaws.com/123456789/image-processing"

# Output formats
FORMATS = {
    'thumbnail': {'size': (100, 100), 'quality': 70},
    'small': {'size': (300, 300), 'quality': 80},
    'medium': {'size': (600, 600), 'quality': 85},
    'large': {'size': (1200, 1200), 'quality': 90},
    'original': {'size': None, 'quality': 95}  # Keep original
}

class ImageWorker:
    def __init__(self):
        self.max_retries = 3

    def process_queue(self):
        """Main worker loop"""
        while True:
            try:
                messages = sqs_client.receive_message(
                    QueueUrl=QUEUE_URL,
                    MaxNumberOfMessages=10,
                    WaitTimeSeconds=10,
                    VisibilityTimeout=300  # 5 minutes to process
                )

                if 'Messages' not in messages:
                    logger.info("No messages in queue")
                    continue

                for message in messages['Messages']:
                    try:
                        self.process_message(message)
                    except Exception as e:
                        logger.error(f"Failed to process message: {e}")
                        # Don't delete, let it retry

            except Exception as e:
                logger.error(f"Queue error: {e}")
                time.sleep(5)

    def process_message(self, message):
        """Process single image"""
        receipt_handle = message['ReceiptHandle']
        body = json.loads(message['Body'])

        image_id = body['image_id']
        s3_key = body['s3_key']

        logger.info(f"Processing image: {image_id}")

        try:
            # Download from S3
            response = s3_client.get_object(
                Bucket=BUCKET_NAME,
                Key=s3_key
            )
            image_data = response['Body'].read()

            # Process all formats
            results = self.process_image(image_id, image_data)

            # Upload processed images to S3
            for format_name, image_bytes in results.items():
                s3_key = f"{PROCESSED_PREFIX}{image_id}/{format_name}.jpg"
                s3_client.put_object(
                    Bucket=BUCKET_NAME,
                    Key=s3_key,
                    Body=image_bytes,
                    ContentType='image/jpeg',
                    Metadata={'format': format_name}
                )
                logger.info(f"Uploaded {format_name} to {s3_key}")

            # Update database
            self.update_database(image_id, results)

            # Delete message from queue
            sqs_client.delete_message(
                QueueUrl=QUEUE_URL,
                ReceiptHandle=receipt_handle
            )

            logger.info(f"Successfully processed {image_id}")

        except Exception as e:
            logger.error(f"Error processing {image_id}: {e}")
            # Message will be retried (visibility timeout resets)
            raise

    def process_image(self, image_id: str, image_bytes: bytes) -> dict:
        """Convert image to all formats"""
        results = {}

        try:
            # Open original image
            original_image = Image.open(io.BytesIO(image_bytes))

            # Convert to RGB if needed
            if original_image.mode in ('RGBA', 'LA', 'P'):
                rgb_image = Image.new('RGB', original_image.size, (255, 255, 255))
                rgb_image.paste(original_image, mask=original_image.split()[-1] if original_image.mode in ('RGBA', 'LA') else None)
                original_image = rgb_image

            # Process each format
            for format_name, config in FORMATS.items():
                if format_name == 'original':
                    # Keep original as-is, just re-encode
                    output = io.BytesIO()
                    original_image.save(
                        output,
                        format='JPEG',
                        quality=config['quality'],
                        optimize=True
                    )
                    results[format_name] = output.getvalue()
                else:
                    # Resize
                    size = config['size']
                    resized = original_image.copy()
                    resized.thumbnail(size, Image.Resampling.LANCZOS)

                    # Pad to exact size (center)
                    final_image = Image.new('RGB', size, (255, 255, 255))
                    offset = (
                        (size[0] - resized.width) // 2,
                        (size[1] - resized.height) // 2
                    )
                    final_image.paste(resized, offset)

                    # Compress
                    output = io.BytesIO()
                    final_image.save(
                        output,
                        format='JPEG',
                        quality=config['quality'],
                        optimize=True
                    )
                    results[format_name] = output.getvalue()

                logger.info(f"Processed {format_name} for {image_id}")

            return results

        except Exception as e:
            logger.error(f"Image processing error for {image_id}: {e}")
            raise

    def update_database(self, image_id: str, results: dict):
        """Update DB with processed image URLs"""
        db.images.update_one(
            {'_id': image_id},
            {'$set': {
                'status': 'processed',
                'processed_at': datetime.utcnow(),
                'urls': {
                    format_name: f"https://cdn.example.com/images/{image_id}/{format_name}.jpg"
                    for format_name in results.keys()
                }
            }}
        )

# Run worker
if __name__ == '__main__':
    worker = ImageWorker()
    worker.process_queue()
```

## Scaling Strategy

```
Load: 1,000 images/minute = 16-17/second
Average processing time: 2-5 seconds per image

Workers needed: 16 * 5 = 80 worker capacity
                ÷ CPU cores per worker (4) = 20 workers

Instance setup:
├─ t3.large (2 vCPU): 1 image/sec = ~17 needed
├─ Or c5.xlarge (4 vCPU): 2 images/sec = ~9 needed
├─ Or use Lambda (auto-scales)
└─ Use spot instances to reduce cost 70%
```

## Monitoring & Dead Letter Queue

```python
class ProcessingMetrics:
    def record_upload(self):
        cloudwatch.put_metric_data(
            'ImageProcessing',
            'UploadsPerMinute',
            value=1
        )

    def record_success(self, format_name: str, duration_ms: float):
        cloudwatch.put_metric_data(
            'ImageProcessing',
            'ProcessingTime',
            value=duration_ms
        )

    def record_failure(self, reason: str):
        # Send to Dead Letter Queue for investigation
        dlq_client.send_message(
            QueueUrl=DLQ_URL,
            MessageBody=json.dumps({
                'reason': reason,
                'timestamp': datetime.utcnow().isoformat()
            })
        )
```

## Cost Optimization

| Component | Cost | Note |
|-----------|------|------|
| S3 storage (1M images × 5 formats × 100KB avg) | $5K/month | Use lifecycle policies to archive old images |
| SQS | $5/month | Cheap for 1M messages/month |
| Lambda (processing) | $2K-5K/month | Cheaper than EC2 for bursty workload |
| Data transfer | $1K/month | Optimize with regional buckets |
| **Total** | **$8-11K/month** | Scales linearly with images |

## Failure Handling

1. **Failed image processing:**
   - Automatically retry up to 3 times (SQS visibility)
   - If still failing, move to Dead Letter Queue
   - Alert ops to investigate

2. **S3 outage:**
   - Queue fills up, workers wait
   - No data loss (messages persist)
   - Resume automatically when S3 recovers

3. **Database down:**
   - Processing completes but DB update fails
   - Message goes to DLQ
   - Replay after DB recovery
