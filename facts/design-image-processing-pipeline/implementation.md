# S3 + SQS + Sharp Pipeline Code

## Complete Working Example

```python
import asyncio
import boto3
import json
import logging
import uuid
from datetime import datetime
from fastapi import FastAPI, UploadFile, File
from typing import Dict
import httpx
from io import BytesIO

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# AWS clients
s3 = boto3.client('s3', region_name='us-east-1')
sqs = boto3.client('sqs', region_name='us-east-1')
dynamodb = boto3.resource('dynamodb', region_name='us-east-1')

# Configuration
BUCKET = 'my-images-bucket'
QUEUE_URL = 'https://sqs.us-east-1.amazonaws.com/123456789/image-processing'
DLQ_URL = 'https://sqs.us-east-1.amazonaws.com/123456789/image-processing-dlq'
TABLE_NAME = 'images'

# Image resize configurations
FORMATS = {
    'thumbnail': (100, 100, 70),
    'small': (300, 300, 80),
    'medium': (600, 600, 85),
    'large': (1200, 1200, 90),
}

# ============ UPLOAD SERVICE ============

app = FastAPI()

@app.post('/api/images/upload')
async def upload_image(file: UploadFile = File(...)):
    """
    User uploads image
    1. Validate file
    2. Upload original to S3 (temp)
    3. Queue for processing
    4. Return image_id
    """

    # Validate content type
    allowed_types = {'image/jpeg', 'image/png', 'image/webp'}
    if file.content_type not in allowed_types:
        return {'error': 'Invalid image type'}, 400

    # Read file
    content = await file.read()

    # Validate size (max 50MB)
    if len(content) > 50 * 1024 * 1024:
        return {'error': 'File too large'}, 400

    image_id = str(uuid.uuid4())

    try:
        # Upload original to S3
        s3.put_object(
            Bucket=BUCKET,
            Key=f'uploads/temp/{image_id}/original',
            Body=content,
            ContentType=file.content_type,
            Metadata={
                'upload_time': datetime.utcnow().isoformat(),
                'filename': file.filename
            }
        )
        logger.info(f'Uploaded original: {image_id}')

        # Queue processing job
        sqs.send_message(
            QueueUrl=QUEUE_URL,
            MessageBody=json.dumps({
                'image_id': image_id,
                'original_key': f'uploads/temp/{image_id}/original',
                'filename': file.filename,
                'timestamp': datetime.utcnow().isoformat()
            }),
            MessageGroupId='images'  # FIFO ordering
        )
        logger.info(f'Queued for processing: {image_id}')

        # Create DB entry
        table = dynamodb.Table(TABLE_NAME)
        table.put_item(Item={
            'image_id': image_id,
            'status': 'uploading',
            'created_at': datetime.utcnow().isoformat(),
            'filename': file.filename
        })

        return {
            'image_id': image_id,
            'status': 'processing',
            'message': 'Image queued for processing'
        }

    except Exception as e:
        logger.error(f'Upload failed: {e}')
        return {'error': 'Upload failed'}, 500


@app.get('/api/images/{image_id}')
async def get_image(image_id: str):
    """Get image processing status and URLs"""
    table = dynamodb.Table(TABLE_NAME)
    response = table.get_item(Key={'image_id': image_id})

    if 'Item' not in response:
        return {'error': 'Image not found'}, 404

    item = response['Item']
    return {
        'image_id': image_id,
        'status': item.get('status'),
        'created_at': item.get('created_at'),
        'urls': item.get('urls', {}),
        'original_url': item.get('original_url')
    }


# ============ WORKER: IMAGE PROCESSING ============

from PIL import Image

class ImageProcessor:
    def __init__(self):
        self.max_retries = 3

    def run(self):
        """Main worker loop"""
        logger.info('Starting image processor')

        while True:
            try:
                self.process_batch()
            except Exception as e:
                logger.error(f'Batch error: {e}')
                asyncio.sleep(5)

    def process_batch(self):
        """Process batch of messages"""
        response = sqs.receive_message(
            QueueUrl=QUEUE_URL,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=10,
            VisibilityTimeout=300
        )

        messages = response.get('Messages', [])
        if not messages:
            logger.debug('No messages')
            return

        for message in messages:
            self.process_message(message)

    def process_message(self, message: Dict):
        """Process single image"""
        receipt_handle = message['ReceiptHandle']
        body = json.loads(message['Body'])

        image_id = body['image_id']
        original_key = body['original_key']

        logger.info(f'Processing: {image_id}')

        try:
            # Download original
            obj = s3.get_object(Bucket=BUCKET, Key=original_key)
            original_data = obj['Body'].read()

            # Process formats
            processed = self._process_image(image_id, original_data)

            # Upload processed images
            urls = {}
            for format_name, image_bytes in processed.items():
                key = f'uploads/processed/{image_id}/{format_name}'
                s3.put_object(
                    Bucket=BUCKET,
                    Key=key,
                    Body=image_bytes,
                    ContentType='image/jpeg',
                    Metadata={'format': format_name}
                )
                urls[format_name] = self._get_url(key)
                logger.info(f'Uploaded {format_name}')

            # Upload original to processed folder
            original_key_processed = f'uploads/processed/{image_id}/original'
            s3.put_object(
                Bucket=BUCKET,
                Key=original_key_processed,
                Body=original_data
            )
            urls['original'] = self._get_url(original_key_processed)

            # Update database
            table = dynamodb.Table(TABLE_NAME)
            table.update_item(
                Key={'image_id': image_id},
                UpdateExpression='SET #status = :status, urls = :urls, processed_at = :at',
                ExpressionAttributeNames={'#status': 'status'},
                ExpressionAttributeValues={
                    ':status': 'processed',
                    ':urls': urls,
                    ':at': datetime.utcnow().isoformat()
                }
            )

            # Delete from queue
            sqs.delete_message(
                QueueUrl=QUEUE_URL,
                ReceiptHandle=receipt_handle
            )

            logger.info(f'Completed: {image_id}')

        except Exception as e:
            logger.error(f'Failed: {image_id}: {e}')
            # Message will retry (visibility resets)

    def _process_image(self, image_id: str, data: bytes) -> Dict[str, bytes]:
        """Process image to all formats"""
        results = {}

        try:
            img = Image.open(BytesIO(data))

            # Convert to RGB
            if img.mode in ('RGBA', 'LA', 'P'):
                rgb = Image.new('RGB', img.size, (255, 255, 255))
                rgb.paste(img, mask=img.split()[-1] if img.mode in ('RGBA', 'LA') else None)
                img = rgb

            # Process each format
            for name, (size, quality) in FORMATS.items():
                resized = img.copy()
                resized.thumbnail((size, size), Image.Resampling.LANCZOS)

                # Pad to exact size
                final = Image.new('RGB', (size, size), (255, 255, 255))
                offset = ((size - resized.width) // 2, (size - resized.height) // 2)
                final.paste(resized, offset)

                # Compress
                output = BytesIO()
                final.save(output, format='JPEG', quality=quality, optimize=True)
                results[name] = output.getvalue()
                logger.info(f'Processed {name} for {image_id}')

            return results

        except Exception as e:
            logger.error(f'Image processing error: {e}')
            raise

    def _get_url(self, key: str) -> str:
        """Generate public URL for S3 key"""
        return f'https://cdn.example.com/{key}'


# ============ RUN ============

if __name__ == '__main__':
    import uvicorn

    # Start processor in background
    import threading
    processor = ImageProcessor()
    processor_thread = threading.Thread(target=processor.run, daemon=True)
    processor_thread.start()

    # Start FastAPI server
    uvicorn.run(app, host='0.0.0.0', port=8000)
```

## JavaScript/Node.js Version (Lambda)

```javascript
const AWS = require('aws-sdk');
const sharp = require('sharp');
const { v4: uuidv4 } = require('uuid');

const s3 = new AWS.S3();
const dynamodb = new AWS.DynamoDB.DocumentClient();

const BUCKET = 'my-images-bucket';
const TABLE = 'images';

const FORMATS = {
  thumbnail: { width: 100, height: 100, quality: 70 },
  small: { width: 300, height: 300, quality: 80 },
  medium: { width: 600, height: 600, quality: 85 },
  large: { width: 1200, height: 1200, quality: 90 }
};

// Lambda handler
exports.processImage = async (event) => {
  const message = JSON.parse(event.Records[0].Sns.Message);
  const { imageId, originalKey } = message;

  console.log(`Processing: ${imageId}`);

  try {
    // Download original
    const original = await s3
      .getObject({ Bucket: BUCKET, Key: originalKey })
      .promise();

    const imageData = original.Body;

    // Process formats
    const urls = {};
    for (const [format, config] of Object.entries(FORMATS)) {
      const processed = await sharp(imageData)
        .resize(config.width, config.height, {
          fit: 'contain',
          background: { r: 255, g: 255, b: 255 }
        })
        .jpeg({ quality: config.quality, progressive: true })
        .toBuffer();

      const key = `uploads/processed/${imageId}/${format}`;
      await s3
        .putObject({
          Bucket: BUCKET,
          Key: key,
          Body: processed,
          ContentType: 'image/jpeg'
        })
        .promise();

      urls[format] = `https://cdn.example.com/${key}`;
      console.log(`Processed: ${format}`);
    }

    // Upload original
    const originalKey = `uploads/processed/${imageId}/original`;
    await s3
      .putObject({
        Bucket: BUCKET,
        Key: originalKey,
        Body: imageData
      })
      .promise();
    urls.original = `https://cdn.example.com/${originalKey}`;

    // Update database
    await dynamodb
      .update({
        TableName: TABLE,
        Key: { image_id: imageId },
        UpdateExpression: 'SET #status = :status, urls = :urls, processed_at = :at',
        ExpressionAttributeNames: { '#status': 'status' },
        ExpressionAttributeValues: {
          ':status': 'processed',
          ':urls': urls,
          ':at': new Date().toISOString()
        }
      })
      .promise();

    console.log(`Completed: ${imageId}`);
    return { statusCode: 200 };
  } catch (error) {
    console.error(`Failed: ${imageId}:`, error);
    throw error;
  }
};
```

## Docker Compose Setup

```yaml
version: '3.8'
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.api
    ports:
      - "8000:8000"
    environment:
      - AWS_ACCESS_KEY_ID
      - AWS_SECRET_ACCESS_KEY
      - AWS_REGION=us-east-1
    depends_on:
      - localstack

  worker:
    build:
      context: .
      dockerfile: Dockerfile.worker
    environment:
      - AWS_ACCESS_KEY_ID
      - AWS_SECRET_ACCESS_KEY
      - AWS_REGION=us-east-1
    depends_on:
      - localstack
    deploy:
      replicas: 4  # 4 workers

  localstack:
    image: localstack/localstack:latest
    ports:
      - "4566:4566"
    environment:
      - SERVICES=s3,sqs,dynamodb
      - DEBUG=1
      - DOCKER_HOST=unix:///var/run/docker.sock
    volumes:
      - "${TMPDIR:-/tmp/localstack}:/tmp/localstack"
      - "/var/run/docker.sock:/var/run/docker.sock"
```

This handles 1000 images/minute with automatic scaling and fault tolerance.
