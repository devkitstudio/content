## Stream Types

**Readable Streams**: Read data in chunks
- File system, network requests, user input
- Emits `data`, `end`, `error` events
- Can pause/resume with backpressure handling

**Writable Streams**: Write data in chunks
- File system, HTTP responses, databases
- Handles buffering and flow control
- Returns false when buffer is full → pause reading

**Transform Streams**: Read → modify → write
- Compress, parse, filter, aggregate data
- Reduces memory: processes one chunk at a time
- Perfect for CSV parsing, JSON transformation

## Why Streams Matter

**Without streams** (naive approach):
```javascript
const fs = require('fs');

// DANGER: Loads entire file into memory!
const data = fs.readFileSync('huge-file.csv', 'utf8');
const lines = data.split('\n');
lines.forEach(line => {
  // Process line
});
// Result: 10GB file = 10GB RAM spike
```

**With streams** (correct approach):
```javascript
const fs = require('fs');

// Only 64KB in memory at a time
fs.createReadStream('huge-file.csv', { highWaterMark: 64 * 1024 })
  .on('data', (chunk) => {
    // Process chunk (64KB max)
  });
// Result: constant ~64KB RAM usage
```

## Backpressure Handling

Critical for piping streams safely:

```javascript
const readable = fs.createReadStream('input.csv');
const writable = fs.createWriteStream('output.csv');

readable.on('data', (chunk) => {
  const canContinue = writable.write(chunk);

  if (!canContinue) {
    // Buffer full, pause reading
    readable.pause();
  }
});

writable.on('drain', () => {
  // Buffer emptied, resume reading
  readable.resume();
});
```

**Or use pipe** (handles backpressure automatically):
```javascript
fs.createReadStream('input.csv')
  .pipe(fs.createWriteStream('output.csv'));
```

## Transform Stream Pattern

For processing CSV row-by-row:

```javascript
const { Transform } = require('stream');

const parseCSV = new Transform({
  transform(chunk, encoding, callback) {
    const lines = chunk.toString().split('\n');

    lines.forEach((line, i) => {
      if (line.trim()) {
        const [id, name, email] = line.split(',');
        this.push(JSON.stringify({ id, name, email }) + '\n');
      }
    });

    callback();
  }
});

fs.createReadStream('users.csv')
  .pipe(parseCSV)
  .pipe(fs.createWriteStream('users.jsonl'));
```

## Memory Profile

- **No streams**: 10GB file → 10GB+ RAM peak
- **Streams with 64KB chunks**: ~65-128KB RAM
- **Streams with backpressure**: Predictable, bounded memory
