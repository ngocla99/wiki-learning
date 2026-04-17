# Coding with Streams

> Sources: Luciano Mammino & Mario Casciaro, 2024
> Raw: [Node.js Design Patterns](../../raw/nodejs/design-patterns.md)

## Overview

Streams are one of the most important patterns in Node.js. They allow data to be processed incrementally as it arrives rather than buffering it all into memory. This enables spatial efficiency (constant memory regardless of data size), time efficiency (overlapping processing stages like an assembly line), and composability (connecting processing units via `pipe()`). Streams are everywhere in Node.js core: `fs.createReadStream()`, HTTP request/response objects, `zlib` compression, `crypto` ciphers -- all are streams.

## Buffering vs Streaming

**Buffer mode:** All data from a resource is collected into a buffer until the operation completes, then passed to the caller as one blob. This fails for large files -- V8 buffers have a hard size limit (`buffer.constants.MAX_LENGTH`).

**Streaming mode:** Data is processed incrementally as it arrives. Each chunk is passed to the consumer immediately, enabling constant memory usage regardless of total data size.

```js
// Buffer approach -- fails for files > ~2 GB
const data = await readFile(filename)
const gzipped = await gzipPromise(data)
await writeFile(`${filename}.gz`, gzipped)

// Stream approach -- works for any file size
createReadStream(filename)
  .pipe(createGzip())
  .pipe(createWriteStream(`${filename}.gz`))
```

### Spatial Efficiency

Streams process data in small chunks, so memory usage stays constant even for multi-gigabyte files. A buffered approach would attempt to load the entire file into memory and crash.

### Time Efficiency

With streams, processing stages overlap like an assembly line. While chunk N is being compressed, chunk N-1 can already be sent over the network, and chunk N+1 can be read from disk. With buffering, each stage must fully complete before the next begins.

> Streams are designed for **memory efficiency**, not necessarily speed. The abstraction adds overhead. If all data fits in memory and doesn't need to be transferred, processing it directly without streams is often faster.

### Composability

Streams share a uniform interface, so they can be connected like LEGO bricks. Adding encryption to a gzip pipeline is just one extra `.pipe()` call:

```js
// Client: read -> compress -> encrypt -> send
createReadStream(filename)
  .pipe(createGzip())
  .pipe(createCipheriv('aes192', secret, iv))
  .pipe(req)

// Server: receive -> decrypt -> decompress -> write
req
  .pipe(createDecipheriv('aes192', secret, iv))
  .pipe(createGunzip())
  .pipe(createWriteStream(destFilename))
```

## Anatomy of Streams

Every stream is an implementation of one of four base abstract classes from `node:stream`, and every stream is also an `EventEmitter`:

| Class | Purpose |
|---|---|
| **Readable** | Source of data |
| **Writable** | Data destination |
| **Duplex** | Both Readable and Writable (e.g., TCP socket) |
| **Transform** | Duplex that transforms data passing through it |

**Two operating modes:**
- **Binary mode** -- chunks are `Buffer` or `string`
- **Object mode** -- chunks are arbitrary JavaScript objects

## Readable Streams

A Readable stream represents a source of data.

### Reading Modes

**Non-flowing (paused) mode** -- the default. Attach a `readable` listener, then pull data with `read()`:

```js
process.stdin
  .on('readable', () => {
    let chunk
    while ((chunk = process.stdin.read()) !== null) {
      console.log(`Chunk: "${chunk.toString()}"`)
    }
  })
  .on('end', () => console.log('End of stream'))
```

`read()` returns `null` when the internal buffer is empty. You can pass a `size` argument to read a specific number of bytes -- useful for parsing binary protocols.

**Flowing mode** -- attach a `data` listener and chunks are pushed automatically:

```js
process.stdin
  .on('data', (chunk) => {
    console.log(`Chunk: "${chunk.toString()}"`)
  })
  .on('end', () => console.log('End of stream'))
```

Use `pause()` to switch back to non-flowing, `resume()` to re-enable flowing.

**Async iterators** -- Readable streams are async iterables:

```js
for await (const chunk of process.stdin) {
  console.log(`Chunk: "${chunk.toString()}"`)
}
console.log('End of stream')
```

### Implementing Custom Readable Streams

Extend `Readable` and implement `_read(size)`. Use `this.push(chunk)` to fill the internal buffer. Push `null` to signal end-of-stream:

```js
import { Readable } from 'node:stream'

export class RandomStream extends Readable {
  constructor(options) {
    super(options)
  }

  _read(size) {
    const chunk = generateRandomString(size)
    this.push(chunk, 'utf8')
    if (shouldStop()) {
      this.push(null) // signal EOF
    }
  }
}
```

**Key options:** `encoding`, `objectMode` (default `false`), `highWaterMark` (default 16 KB -- upper limit of internal buffer).

**Simplified construction** -- pass a `read()` function in options instead of subclassing:

```js
const myStream = new Readable({
  read(size) {
    this.push('some data')
    this.push(null)
  },
})
```

**From iterables** -- `Readable.from()` creates a stream from arrays, generators, or async iterators. It defaults to object mode:

```js
const stream = Readable.from([
  { name: 'Everest', height: 8848 },
  { name: 'K2', height: 8611 },
])
```

> Avoid loading large arrays into memory. Use generators or async iterators with `Readable.from()` for lazy data loading.

## Writable Streams

A Writable stream represents a data destination (file, socket, stdout, etc.).

### Writing to a Stream

```js
writable.write(chunk, [encoding], [callback])
writable.end([chunk], [encoding], [callback])
```

`write()` pushes data in. `end()` signals no more data; its callback is equivalent to the `finish` event.

### Backpressure

When data is written faster than the stream can consume, it accumulates in an internal buffer. `write()` returns `false` when the buffer exceeds `highWaterMark`, signaling the caller to pause. When the buffer drains, a `drain` event fires, signaling it's safe to resume:

```js
function generateMore() {
  do {
    const chunk = produceData()
    const shouldContinue = res.write(chunk)
    if (!shouldContinue) {
      // Buffer full -- wait for drain
      return res.once('drain', generateMore)
    }
  } while (hasMoreData())
  res.end()
}
generateMore()
```

Backpressure is **advisory** -- you can ignore it, but the buffer will grow indefinitely and consume excessive memory.

### Implementing Custom Writable Streams

Extend `Writable` and implement `_write(chunk, encoding, cb)`. Call `cb()` when done, or `cb(error)` on failure:

```js
import { Writable } from 'node:stream'

export class ToFileStream extends Writable {
  constructor(options) {
    super({ ...options, objectMode: true })
  }

  _write(chunk, _encoding, cb) {
    fs.writeFile(chunk.path, chunk.content)
      .then(() => cb())
      .catch(cb)
  }
}
```

## Duplex Streams

A Duplex stream is both Readable and Writable (e.g., a TCP socket). The read and write sides are independent -- there is no inherent relationship between data written in and data read out. Implement both `_read()` and `_write()`.

**Options:** Same as Readable + Writable, plus `allowHalfOpen` (default `true`) -- if `false`, ending one side ends both. Use `readableObjectMode` and `writableObjectMode` to mix binary and object mode.

## Transform Streams

Transform streams are a special Duplex where input is transformed and pushed to the output. Data written to the Writable side flows through a transformation and becomes available on the Readable side.

### Implementing Transform Streams

Implement `_transform(chunk, encoding, cb)` and optionally `_flush(cb)`:

```js
import { Transform } from 'node:stream'

export class ReplaceStream extends Transform {
  constructor(searchStr, replaceStr, options) {
    super({ ...options })
    this.searchStr = searchStr
    this.replaceStr = replaceStr
    this.tail = ''
  }

  _transform(chunk, _encoding, cb) {
    const pieces = (this.tail + chunk).split(this.searchStr)
    const lastPiece = pieces[pieces.length - 1]
    const tailLen = this.searchStr.length - 1
    this.tail = lastPiece.slice(-tailLen)
    pieces[pieces.length - 1] = lastPiece.slice(0, -tailLen)
    this.push(pieces.join(this.replaceStr))
    cb()
  }

  _flush(cb) {
    this.push(this.tail)
    cb()
  }
}
```

The `tail` variable handles matches that span chunk boundaries. `_flush()` is called before the stream ends, giving a final chance to emit buffered data.

### Filtering with Transform Streams

**Pattern -- Transform filter:** Call `this.push()` conditionally to allow only matching data to reach the next pipeline stage:

```js
_transform(record, _enc, cb) {
  if (record.country === this.country) {
    this.push(record)
  }
  cb() // always call cb(), even when filtering out
}
```

### Aggregating with Transform Streams

**Pattern -- Streaming aggregation:** Accumulate in `_transform()`, emit only in `_flush()`:

```js
_transform(record, _enc, cb) {
  this.total += Number.parseFloat(record.profit)
  cb() // no push here
}

_flush(cb) {
  this.push(this.total.toString())
  cb()
}
```

A complete pipeline combining filtering and aggregation:

```js
createReadStream('data.csv')
  .pipe(csvParser)
  .pipe(new FilterByCountry('Italy'))
  .pipe(new SumProfit())
  .pipe(process.stdout)
```

## PassThrough Streams

A PassThrough stream outputs every chunk without modification. Despite seeming trivial, it has several practical uses.

### Observability

Insert a PassThrough into a pipeline to monitor data flow without altering it:

```js
const monitor = new PassThrough()
monitor.on('data', chunk => { bytesWritten += chunk.length })

createReadStream(filename)
  .pipe(createGzip())
  .pipe(monitor)          // observe compressed bytes
  .pipe(createWriteStream(`${filename}.gz`))
```

### Late Piping

Use a PassThrough as a placeholder when an API requires a stream but data isn't available yet:

```js
const contentStream = new PassThrough()
upload('file.br', contentStream) // API starts consuming immediately

// Data flows later when pipeline is set up
createReadStream(filepath)
  .pipe(createBrotliCompress())
  .pipe(contentStream)
```

This pattern also lets you transform a function that accepts a Readable into one that returns a Writable:

```js
function createUploadStream(filename) {
  const connector = new PassThrough()
  upload(filename, connector)
  return connector // caller writes to this
}
```

### Lazy Streams

When creating many streams at once (e.g., for archiving), each may open expensive resources (file descriptors). Libraries like `lazystream` use a PassThrough internally to defer the actual stream creation until data is first consumed.

## Pipes and Error Handling

### pipe()

```js
readable.pipe(writable, [options])
```

`pipe()` connects a Readable to a Writable, automatically handling backpressure. It returns the destination stream, enabling chaining (when the destination is also Readable, like a Transform).

**Critical limitation:** Error events do not propagate through `pipe()`. Each stream needs its own error listener, and failing streams must be manually destroyed to avoid resource leaks.

### pipeline()

The `pipeline()` utility from `node:stream` (or its Promise version from `node:stream/promises`) solves all error-handling problems:

```js
import { pipeline } from 'node:stream/promises'

await pipeline(
  process.stdin,
  createGunzip(),
  uppercasify,
  createGzip(),
  process.stdout
)
```

`pipeline()` registers proper error and close listeners on every stream. On error, all streams are properly destroyed and resources released. Always prefer `pipeline()` over raw `pipe()` chains in production code.

## Async Control Flow with Streams

Streams can serve as elegant alternatives to traditional async control flow patterns. See [Async Control Flow](async-control-flow.md) for non-stream async patterns.

### Sequential Execution

By default, `_transform()` is not called with the next chunk until the current callback completes. This property enables sequential async iteration:

```js
export function concatFiles(dest, files) {
  return new Promise((resolve, reject) => {
    const destStream = createWriteStream(dest)
    Readable.from(files)
      .pipe(new Transform({
        objectMode: true,
        transform(filename, _enc, done) {
          const src = createReadStream(filename)
          src.pipe(destStream, { end: false })
          src.on('error', done)
          src.on('end', done) // next file only after current finishes
        },
      }))
      .on('finish', () => { destStream.end(); resolve() })
  })
}
```

**Pattern:** Use a stream, or combination of streams, to iterate over a set of asynchronous tasks in sequence.

### Unordered Concurrent Execution

To process chunks concurrently, invoke `done()` immediately in `_transform()` instead of waiting for the async operation to complete:

```js
_transform(chunk, enc, done) {
  this.running++
  this.userTransform(chunk, enc, this.push.bind(this), this._onComplete.bind(this))
  done() // immediately signal readiness for next chunk
}

_flush(done) {
  if (this.running > 0) {
    this.terminateCb = done // defer finish until all tasks complete
  } else {
    done()
  }
}
```

**Caveat:** Output order is not preserved. Only use when chunk order does not matter (common with object streams, rare with binary).

### Unordered Limited Concurrent Execution

Add a concurrency limit by only calling `done()` when a slot is available:

```js
_transform(chunk, enc, done) {
  this.running++
  this.userTransform(chunk, enc, this.push.bind(this), this._onComplete.bind(this))
  if (this.running < this.concurrency) {
    done()
  } else {
    this.continueCb = done // hold until a task completes
  }
}
```

The `_onComplete` handler decrements `running` and invokes the saved `continueCb` to unblock the next item.

### Ordered Concurrent Execution

Process chunks concurrently but emit results in the original order. This requires an internal reorder buffer. The `parallel-transform` npm package implements this:

```js
import parallelTransform from 'parallel-transform'

const checkUrls = parallelTransform(8, async function (url, done) {
  try {
    await fetch(url, { method: 'HEAD', signal: AbortSignal.timeout(5000) })
    this.push(`${url} is up\n`)
  } catch (err) {
    this.push(`${url} is down\n`)
  }
  done()
})

await pipeline(fileLines, checkUrls, outputFile)
```

> Be aware of slow items: depending on implementation, they either block the pipeline or grow the reorder buffer indefinitely. `parallel-transform` opts for predictable memory usage by capping the buffer at the concurrency limit.

## Piping Patterns

### Combining Streams

Package an entire pipeline as a single Duplex stream using `compose()` from `node:stream`:

```js
import { compose } from 'node:stream'

export function createCompressAndEncrypt(password, iv) {
  const key = scryptSync(password, 'salt', 24)
  return compose(
    createGzip(),
    createCipheriv('aes192', key, iv)
  )
}
```

Writes enter the first stream; reads come from the last. Errors propagate to the composite. This enables black-box reuse and simplified error management.

**`compose()` vs `pipeline()`:** `compose()` is lazy -- it builds the chain but doesn't start data flow. Use `compose()` to package a reusable pipeline as one stream. Use `pipeline()` to wire a source to a destination and wait for completion.

### Forking Streams

Pipe a single Readable into multiple Writable streams:

```js
const inputStream = createReadStream(filename)
inputStream.pipe(sha1Stream).pipe(createWriteStream(`${filename}.sha1`))
inputStream.pipe(md5Stream).pipe(createWriteStream(`${filename}.md5`))
```

**Caveats:**
- Both forks receive the same chunk references -- avoid mutating data
- Backpressure is governed by the **slowest** branch; one blocking destination blocks everything
- Piping after consumption has started means the new stream only receives future chunks (use a PassThrough as a buffer if needed)

### Merging Streams

Pipe multiple Readable streams into a single Writable. Use `{ end: false }` to prevent the destination from closing when any single source ends, then manually `end()` the destination when all sources are done:

```js
let endCount = 0
for (const source of sources) {
  const sourceStream = createReadStream(source)
  sourceStream.on('end', () => {
    if (++endCount === sources.length) {
      destStream.end()
    }
  })
  sourceStream.pipe(destStream, { end: false })
}
```

With binary streams, merged chunks may interleave. For ordered merging (concatenation), consume sources sequentially, or use the `multistream` npm package.

### Multiplexing and Demultiplexing

Combine multiple logical channels into a single physical stream (mux), then separate them at the other end (demux). Uses packet switching: each packet has a header with channel ID + data length, followed by the payload.

**Multiplexer (client):**

```js
function multiplexChannels(sources, destination) {
  for (let i = 0; i < sources.length; i++) {
    sources[i].on('readable', function () {
      let chunk
      while ((chunk = this.read()) !== null) {
        const outBuff = Buffer.alloc(1 + 4 + chunk.length)
        outBuff.writeUInt8(i, 0)            // 1 byte: channel ID
        outBuff.writeUInt32BE(chunk.length, 1) // 4 bytes: data length
        chunk.copy(outBuff, 5)              // remaining: data
        destination.write(outBuff)
      }
    })
  }
}
```

**Demultiplexer (server):** Read the header fields using non-flowing mode with exact byte counts (`source.read(1)`, `source.read(4)`, `source.read(currentLength)`), then route each packet to the correct destination stream based on channel ID.

For **object streams**, multiplexing is simpler: set a `channelID` property on each object. Demultiplexing reads the property and routes accordingly.

## Readable Stream Utilities

The `Readable` class exposes functional-style helper methods (chainable, returning new Readable streams):

| Method | Description |
|---|---|
| `.map(fn)` | Transform each chunk (supports async fn) |
| `.flatMap(fn)` | Like map, but fn can return streams/iterables that are flattened |
| `.filter(fn)` | Keep only chunks where fn returns truthy |
| `.forEach(fn)` | Side effects per chunk (doesn't produce a new stream) |
| `.some(fn)` | Promise: true if any chunk satisfies fn |
| `.every(fn)` | Promise: true if all chunks satisfy fn |
| `.find(fn)` | Promise: first chunk satisfying fn |
| `.drop(n)` | Skip first n chunks |
| `.take(n)` | Emit at most n chunks, then end |
| `.reduce(fn, init)` | Accumulate to a single Promise value |

Example -- the CSV filtering/aggregation pipeline rewritten with helpers:

```js
const totalProfit = await byLine
  .drop(1)                                           // skip header
  .map(chunk => {
    const [type, country, profit] = chunk.toString().split(',')
    return { type, country, profit: Number.parseFloat(profit) }
  })
  .filter(record => record.country === 'Italy')
  .reduce((acc, record) => acc + record.profit, 0)
```

These helpers are convenient for simple transformations. For complex, reusable logic, custom Transform streams remain the better choice.

## Web Streams

The WHATWG Streams Standard provides a browser-native streaming API with three types that mirror Node.js streams:

| Web Stream | Node.js Equivalent |
|---|---|
| `ReadableStream` | `Readable` |
| `WritableStream` | `Writable` |
| `TransformStream` | `Transform` |

Web Streams are the standard in browsers (used by `fetch`, etc.). Node.js also implements them, giving two competing APIs.

### Interoperability

Convert between the two APIs using static methods:

```js
import { Readable, Writable, Transform } from 'node:stream'

// Node.js -> Web
const webReadable = Readable.toWeb(nodeReadable)
const webWritable = Writable.toWeb(nodeWritable)
const webTransform = Transform.toWeb(nodeTransform)

// Web -> Node.js
const nodeReadable = Readable.fromWeb(webReadable)
const nodeWritable = Writable.fromWeb(webWritable)
const nodeTransform = Transform.fromWeb(webTransform)
```

These conversions **wrap** the source stream (Adapter pattern) -- both the original and the converted stream emit the same chunks.

## Stream Consumer Utilities

Sometimes you need to buffer an entire stream into memory (e.g., parsing a JSON response). The `node:stream/consumers` module provides helpers that return Promises:

| Method | Accumulates as |
|---|---|
| `consumers.buffer(stream)` | `Buffer` |
| `consumers.text(stream)` | `string` |
| `consumers.json(stream)` | Parsed JSON object |
| `consumers.arrayBuffer(stream)` | `ArrayBuffer` |
| `consumers.blob(stream)` | `Blob` |

```js
import consumers from 'node:stream/consumers'

const req = request('http://example.com/data.json', async res => {
  const data = await consumers.json(res)
  console.log(data)
})
req.end()
```

If using `fetch`, the response object has built-in consumers: `res.json()`, `res.text()`, `res.blob()`, `res.arrayBuffer()`.

## Key Patterns Summary

| Pattern | Technique |
|---|---|
| Transform filter | `this.push()` conditionally in `_transform()` |
| Streaming aggregation | Accumulate in `_transform()`, emit in `_flush()` |
| Late piping | PassThrough as placeholder for future data |
| Lazy streams | PassThrough that defers real stream creation until first read |
| Sequential async | Stream's natural chunk-by-chunk processing |
| Concurrent async | Call `done()` immediately in `_transform()` |
| Limited concurrent | Hold `done()` when at concurrency limit |
| Ordered concurrent | Concurrent processing + reorder buffer before emit |
| Combining | `compose()` to package a pipeline as a single Duplex |
| Forking | One Readable piped to multiple Writables |
| Merging | Multiple Readables piped to one Writable (`{ end: false }`) |
| Mux/Demux | Packet headers (channel ID + length) on a shared stream |
