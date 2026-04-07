## Complete CSV Processing Pipeline

```typescript
import fs from 'fs';
import { Transform, pipeline } from 'stream';
import { parse } from 'csv-parse';
import { stringify } from 'csv-stringify';

interface User {
  id: string;
  name: string;
  email: string;
}

// Step 1: Parse CSV
const csvParser = parse({
  columns: true,
  skip_empty_lines: true
});

// Step 2: Transform (filter + enrich)
const dataTransform = new Transform({
  objectMode: true,
  transform(record: User, encoding, callback) {
    if (record.email && record.email.includes('@')) {
      record.name = record.name.toUpperCase();
      this.push(record);
    }
    callback();
  }
});

// Step 3: Output as JSONL
const jsonlStringify = new Transform({
  objectMode: true,
  transform(record, encoding, callback) {
    this.push(JSON.stringify(record) + '\n');
    callback();
  }
});

// Connect pipeline
pipeline(
  fs.createReadStream('input.csv'),
  csvParser,
  dataTransform,
  jsonlStringify,
  fs.createWriteStream('output.jsonl'),
  (err) => {
    if (err) {
      console.error('Pipeline failed:', err);
    } else {
      console.log('Processing complete');
    }
  }
);
```

## Streaming to Database

```typescript
import { pool } from './db';
import { Transform } from 'stream';

const batchInsert = new Transform({
  objectMode: true,
  highWaterMark: 100,
  async transform(record: User, encoding, callback) {
    try {
      await pool.query(
        'INSERT INTO users (id, name, email) VALUES ($1, $2, $3)',
        [record.id, record.name, record.email]
      );
      this.push(record);
      callback();
    } catch (err) {
      callback(err);
    }
  }
});

pipeline(
  fs.createReadStream('input.csv'),
  csvParser,
  batchInsert,
  (err) => {
    if (err) console.error('Insert failed:', err);
    else console.log('All records inserted');
  }
);
```

## Monitoring & Error Handling

```typescript
const monitoringTransform = new Transform({
  objectMode: true,
  transform(record, encoding, callback) {
    processedCount++;
    if (processedCount % 1000 === 0) {
      console.log(`Processed: ${processedCount} records`);
    }
    this.push(record);
    callback();
  }
});

pipeline(
  fs.createReadStream('huge.csv'),
  csvParser,
  monitoringTransform,
  dataTransform,
  fs.createWriteStream('output.jsonl'),
  (err) => {
    if (err) {
      console.error('Pipeline failed at record:', processedCount);
      process.exit(1);
    }
  }
);
```

## Key Settings

```typescript
// Readable stream options
fs.createReadStream('file.csv', {
  highWaterMark: 256 * 1024,  // 256KB chunks (default 64KB)
  encoding: 'utf8'
});

// Transform stream options
new Transform({
  objectMode: true,           // Work with objects, not buffers
  highWaterMark: 100,         // Buffer 100 objects max
  transform(chunk, enc, cb) { // Process each chunk
    this.push(transformed);
    cb();
  }
});
```
