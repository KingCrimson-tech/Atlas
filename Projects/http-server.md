# I Built an HTTP Server From Scratch. Here's Everything You Need to Know.

*(A beginner-friendly, interview-ready deep dive into the "Raw HTTP Server" project.)*

---

## Table of contents

1. [The big picture: what did I actually build?](#1-the-big-picture)
2. [The basic concepts (with simple examples)](#2-the-basic-concepts)
   - [What is a server?](#what-is-a-server)
   - [What is TCP and why "raw TCP"?](#what-is-tcp)
   - [What is HTTP?](#what-is-http)
   - [What is a socket?](#what-is-a-socket)
   - [The request/response cycle](#the-requestresponse-cycle)
   - [Parsing: why it's harder than it looks](#parsing-why-its-harder-than-it-looks)
   - [Keep-alive and pipelining](#keep-alive-and-pipelining)
   - [Backpressure](#backpressure)
   - [Multiple CPU cores with cluster](#multiple-cpu-cores-with-cluster)
   - [Graceful shutdown](#graceful-shutdown)
   - [Status codes and methods](#status-codes-and-methods)
   - [Path traversal (a security idea)](#path-traversal)
3. [The project: file by file](#3-the-project-file-by-file)
4. [How a request travels through the code](#4-how-a-request-travels-through-the-code)
5. [The interesting problems I solved](#5-the-interesting-problems-i-solved)
6. [Testing](#6-testing)
7. [Benchmarks: how fast is it?](#7-benchmarks)
8. [What I deliberately did NOT build](#8-what-i-deliberately-did-not-build)
9. [Interview cheat sheet](#9-interview-cheat-sheet)
10. [How to run it](#10-how-to-run-it)

---

## 1. The big picture

Normally, when you want an HTTP server in Node.js, you write:

```js
import http from 'node:http';
http.createServer((req, res) => res.end('hello')).listen(3000);
```

That works great, but **Node does all the hard stuff for you**: it reads the raw
bytes from the network, figures out where the request line ends, parses the
headers, handles keep-alive, deals with slow clients, and so on.

I wanted to understand what Node is actually doing under the hood. So I built a
server that skips `node:http` completely and works directly with **raw TCP
sockets**. That means I had to write, by hand:

- a **parser** that reads the incoming bytes and understands the HTTP request,
- a **response writer** that sends back correctly formatted HTTP,
- **keep-alive** and **pipelining** support,
- **backpressure handling** so the server doesn't crash under load,
- **static file serving** (reading files from disk) with a security guard,
- **multi-core scaling** using Node's `cluster` module,
- and a **benchmark harness** to compare my server against Node's built-in one.

The goal was not to replace Node's `http` module. The goal was to *learn* by
building it myself, and to have something real to show for it.

> **One sentence version for a resume:** "Built an HTTP/1.1 server from scratch on
> raw TCP sockets in Node.js with a hand-written parser, keep-alive, pipelining,
> backpressure handling, and multi-core clustering."

---

## 2. The basic concepts

Let's go through every idea used in the project, one at a time, with simple
examples. If you understand this section, you can answer most interview
questions about the project.

### What is a server?

A server is just a program that **waits for other programs (clients) to connect
to it and then talks to them**. Your browser is a client. When you type
`http://localhost:3000`, your browser connects to my server program and asks it
for a page. My server sends the page back.

The server doesn't "do" anything magical. It listens on a **port** (like a numbered
door), and when someone knocks, it responds.

### What is TCP and why "raw TCP"?

**TCP** is one of the basic ways computers send data over the internet. Think of
it as a **phone call**:

- You first have to *connect* (dial the number).
- Then you can send data back and forth.
- The data arrives **in order** and **without errors** (TCP checks this for you).
- When you're done, you *hang up*.

The important part: TCP deals in a **stream of bytes**. It does not know anything
about HTTP, web pages, or requests. It just delivers bytes in order.

Most Node servers use `node:http`, which sits on top of TCP and understands HTTP
for you. "Raw TCP" means I skipped that layer. I used Node's `node:net` module,
which only gives me the bytes, and then I wrote the HTTP logic myself.

Why do this? Because it forces you to understand HTTP properly, and you control
every detail.

**Example** — with `node:net` I just get chunks of bytes:

```js
import net from 'node:net';
net.createServer((socket) => {
  socket.on('data', (chunk) => {
    // chunk is a Buffer of raw bytes like:
    // <Buffer 47 45 54 20 2f 20 48 54 54 50 2f 31 2e 31 ...>
    console.log(chunk);
  });
}).listen(3000);
```

Those bytes happen to spell `GET / HTTP/1.1...` but `node:net` has no idea. My
parser is what gives them meaning.

### What is HTTP?

**HTTP** is the language the browser and server speak. It's plain text (for
HTTP/1.1). A request looks like this:

```http
GET /json HTTP/1.1
Host: localhost
User-Agent: curl
Connection: close
```

Breaking it down:

- **Line 1 — the request line:** `METHOD PATH VERSION`
  - `GET` = the method (what I want to do)
  - `/json` = the path (what I want)
  - `HTTP/1.1` = the protocol version
- **Lines 2–4 — headers:** extra information as `Name: value` pairs.
- **Then a blank line** (`\r\n\r\n`) marks the end of the headers.
- **Then optionally a body** (data sent with the request, e.g. for `POST`).

The server replies with something like:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 42
Connection: keep-alive
Date: Tue, 17 Nov 2026 07:00:00 GMT

{"message":"Hello from a hand-written HTTP parser"}
```

Same structure: a **status line**, **headers**, a **blank line**, then the **body**.

Notice the special characters `\r\n`. That is "carriage return + line feed" —
the official way HTTP separates lines. `\r\n\r\n` means "blank line".

### What is a socket?

A **socket** is one end of a connection. When a client connects, Node gives my
server a socket object. I can:

- listen to `socket.on('data', ...)` to receive bytes,
- call `socket.write(...)` to send bytes back,
- call `socket.end()` to close the connection.

One socket = one conversation with one client.

### The request/response cycle

The whole flow is:

1. Client opens a TCP connection to port 3000.
2. Client sends an HTTP request (bytes).
3. My server's parser reads those bytes and turns them into a JavaScript object.
4. The router looks at the path and picks a handler.
5. The handler makes a response object (status, headers, body).
6. The response writer turns that object back into HTTP bytes and writes them to
   the socket.
7. Either the connection closes, or it stays open for another request
   (keep-alive).

### Parsing: why it's harder than it looks

Here is the key problem. TCP is a **stream**. It does not deliver your request in
nice clean pieces. If I send:

```http
GET /json HTTP/1.1\r\nHost: x\r\n\r\n
```

TCP might deliver it to my server like this:

```
chunk 1: "GET /js"
chunk 2: "ON HTTP/1.1\r\nHo"
chunk 3: "st: x\r\n\r\n"
```

Or all at once. Or one byte at a time. The parser must produce the **same result
no matter how the bytes are split**.

A beginner mistake is to do `chunk.toString().split('\r\n')` on each chunk.
That breaks when a header is split across two chunks. My parser avoids this by
keeping a **buffer** of all the bytes seen so far and only acting when it has a
complete piece.

I use a **state machine**. That's a fancy name for a program that is always in
one of a few "states" and moves between them:

```
REQUEST_LINE  --(find \r\n)-->  HEADERS  --(find \r\n\r\n)-->  BODY  --(read Content-Length bytes)-->  DONE
```

- In `REQUEST_LINE`, I wait until I see `\r\n`, then parse `GET /json HTTP/1.1`.
- In `HEADERS`, I read header lines one by one until I see a blank line.
- In `BODY`, if the request has a `Content-Length`, I wait until that many bytes
  have arrived.
- Then I emit one request and go back to `REQUEST_LINE`.

**The cursor trick.** Imagine you're reading a book with your finger. You don't
rip out each page after reading it — you just move your finger forward. My parser
does the same: it keeps one big buffer and an **offset** (my finger) saying how
far I've read. This is faster than constantly cutting the buffer into smaller
pieces. Every so often, when most of the buffer is already read, I throw away the
old part (this is called **compacting**).

**Simple example of why splitting matters.** Suppose I feed the request one byte
at a time:

```js
for (const byte of buffer) {
  results.push(...parser.feed(Buffer.from([byte])));
}
```

On the first many calls, the parser finds no `\r\n`, so it returns nothing (it just
stores the bytes). Only when the final byte of `\r\n\r\n` arrives does it return a
complete request. My tests check this exact case.

### Keep-alive and pipelining

**Without keep-alive:** every request needs a brand-new connection. That's
wasteful — opening a connection takes time.

**With keep-alive:** after answering one request, the connection stays open and
can be reused for more requests. HTTP/1.1 uses keep-alive by default. A client
can also send `Connection: close` to say "close it after this one".

**Pipelining** is when the client sends several requests back-to-back on the same
connection *without waiting for the answers*:

```
request 1, request 2, request 3  --->  server  --->  response 1, response 2, response 3
```

My server must answer them **in order**. Since my handlers are asynchronous (they
can finish in a different order), I use a **promise queue**:

```js
this.queue = this.queue.then(() => this.process(req, keepAlive));
```

Each request is chained onto the previous one, so even if request 2 finishes
first, response 1 is written first. Think of it like a queue at a ticket counter —
you're served in the order you arrived.

### Backpressure

This is one of the most important ideas in the project.

`socket.write(data)` does not always write the data immediately. It puts it in a
buffer and returns:

- `true` — "I've got this, you can keep sending."
- `false` — "my buffer is full, stop sending for a moment."

If you ignore `false` and keep writing, memory grows until the program crashes.
**Backpressure** means respecting that signal: when `write()` returns `false`, I
stop and wait for the socket's `drain` event before continuing:

```js
if (!socket.write(chunk)) {
  stream.pause();                       // stop reading from the file
  socket.once('drain', () => stream.resume()); // continue when the socket is ready
}
```

**Real-world analogy:** imagine pouring water from a jug into a funnel. If the
funnel fills up, you stop pouring until the liquid drains through. If you keep
pouring, it spills everywhere (that's the crash).

For small responses (like `/json`), I write the whole thing in one go and wait for
`drain` if needed. For files, I stream them with `fs.createReadStream`, which
naturally supports pause/resume.

### Multiple CPU cores with cluster

A single Node process uses one CPU core. A machine might have 8 or 12 cores.
**Cluster** lets me start several copies (workers) of my server that all listen
on the same port. Node shares the incoming connections among them (round-robin).

```
          master (primary)
          /     |      \
     worker  worker  worker     <- all listening on port 3000
```

- The **master** forks N workers (N = number of CPUs).
- If a worker crashes, the master starts a new one (**respawn**). This is good for
  reliability.
- Each worker is a completely separate process with its own memory.

Why not just one process? More workers = more requests handled at the same time,
using all the CPU cores.

### Graceful shutdown

When you press Ctrl+C or stop the server, you don't want to cut off requests that
are still being answered. **Graceful shutdown** means:

1. Stop accepting new connections.
2. Let the current ones finish.
3. If something hangs, force-close after a timeout.

The master catches `SIGTERM` / `SIGINT`, tells every worker to shut down, and has
a fallback timer that force-kills anything stuck. This avoids angry users seeing
half-finished responses.

### Status codes and methods

**Methods** say what the client wants:

- `GET` — give me something (most common).
- `POST` — send you some data (like a form).
- `HEAD` — give me the headers only, no body. (Useful for checking if a page
  exists without downloading it.)
- `PUT`, `PATCH`, `DELETE` — update / partially update / delete.

**Status codes** are the server's answer:

- `200 OK` — fine, here's your stuff.
- `400 Bad Request` — your request was nonsense.
- `404 Not Found` — no such page.
- `405 Method Not Allowed` — this page exists, but not for that method (e.g.
  `PUT /`).
- `413 Payload Too Large` — your body is too big.
- `431` — too many / too large headers.
- `501 Not Implemented` — I don't support this feature (chunked request bodies).
- `505` — I don't speak that HTTP version.

A nice detail: when I return `405`, I also send an `Allow` header saying which
methods *are* allowed. That's polite and correct HTTP.

### Path traversal (a security idea)

If I serve files from a `public/` folder, a nasty client could ask for:

```
GET /static/../package.json
```

The `../` means "go up one folder". If I blindly joined that to my folder path,
the client could read any file on my computer, not just files in `public/`. This
is called **path traversal**.

My guard does this:

1. Work out the full, resolved path using `path.resolve()` (which collapses
   `..`).
2. Check that the result is still **inside** the `public` folder.
3. If it's outside, return `403 Forbidden`.

So `/static/../package.json` gets a `403`, and the file stays safe. My integration
test checks exactly this.

---

## 3. The project: file by file

Here's what each file does:

| File | Job |
| --- | --- |
| `src/config.js` | All settings in one place (port, limits, timeouts, paths). Uses environment variables so you can change things without editing code. |
| `src/master.js` | The cluster primary. Starts N workers, respawns them if they die, handles graceful shutdown. |
| `src/worker.js` | One worker. Owns the TCP server and each connection. Runs the parser, router and response writer. |
| `src/parser.js` | The core. Turns raw bytes into request objects. The state machine. |
| `src/response.js` | Turns response objects into HTTP bytes. Handles backpressure. |
| `src/router.js` | Decides which handler runs for which path/method. Also has a method-not-allowed check. |
| `src/static.js` | Serves files from disk with a MIME map and the path-traversal guard. |
| `src/errors.js` | Small helper for consistent JSON error responses. |
| `test/parser.test.js` | 21 unit tests for the parser. |
| `test/integration.test.js` | 11 tests that start a real server and talk to it over raw TCP. |
| `test/helpers.js` | Helpers to build requests and feed them in chunks. |
| `bench/run.js` | The autocannon benchmark harness. |
| `bench/baseline-http.js` | A plain Node `http` server to compare against. |
| `scripts/lint.js` | A tiny dependency-free syntax checker (`node --check` on every file). |
| `results/benchmark.md` | The measured results table. |

`package.json` has the scripts: `start`, `start:worker`, `test`, `lint`, `bench`.

---

## 4. How a request travels through the code

Let's trace `GET /json` from start to finish:

1. **Connection arrives.** `net.createServer` calls `new Connection(socket)` in
   `worker.js`. The socket gets `setNoDelay(true)` and a header timeout.

2. **Bytes arrive.** `socket.on('data')` fires with a `Buffer`. `Connection.onData`
   calls `this.parser.feed(chunk)`.

3. **Parser works.** It walks the state machine, finds the request line, headers
   and (if any) body, and returns:
   ```js
   {
     method: 'GET', path: '/json', query: '',
     httpVersion: 'HTTP/1.1', headers: { host: 'localhost' },
     body: <Buffer >, keepAlive: true, rawLength: 42
   }
   ```

4. **Queued.** `Connection.enqueue` chains this onto the promise queue so
   pipelined responses stay in order.

5. **Routed.** `router.js` looks up `/json`, sees `GET` is allowed, and the handler
   returns:
   ```js
   { status: 200, headers: {...}, body: '{"message":...}' }
   ```

6. **Written.** `writeResponse()` builds the status line and headers, adds
   `Content-Length` and `Date`, and writes the bytes to the socket. It waits for
   `drain` if the socket is full.

7. **Keep-alive.** Since the client didn't say `Connection: close`, the socket
   stays open (with a shorter idle timeout) for the next request.

For a **file** like `/static/index.html`, step 5 calls `serveStatic()`, which
`stat`s the file, checks the path guard, and returns a `stream` plus the file
size. Step 6 pipes that stream to the socket with pause/resume for backpressure.

---

## 5. The interesting problems I solved

**1. Bytes arriving in weird pieces.** The state machine plus a growing buffer
with an offset means the parser gives the same result whether a request arrives in
one chunk, ten chunks, or one byte at a time.

**2. A 50× speedup from one line.** At first, static files were slow: about
**41 ms** each. Why? For a streamed file, I write the response head and the file
body as *two separate writes*. TCP's **Nagle's algorithm** held the second small
piece back until an acknowledgement arrived (delayed ACK), adding ~40 ms. Adding
`socket.setNoDelay(true)` (which turns Nagle off) dropped latency to ~1 ms and
raised throughput from about 243 to over 12,000 requests/second. This was the
best "aha!" moment of the project.

**3. Slow clients (slowloris).** A malicious client could start a request and
never finish it, eating memory. I protect against this with a header timeout and
size limits (buffer, headers, body, request line). If a client is too slow or too
big, the socket is destroyed.

**4. Ordered pipelining.** Wrapping each request in a promise chain means answers
always come out in request order, even when handlers finish out of order.

**5. Fair benchmarking.** My first baseline preloaded the HTML file into memory,
which made it look much faster than my server (which reads from disk). I changed
the baseline to also stream from disk so the comparison is fair.

---

## 6. Testing

I used Node's built-in test runner (`node --test`) — no extra test library.

**Parser tests (21).** These feed requests in every possible way: all at once,
one byte at a time, and every fixed chunk size. They check keep-alive, `Connection:
close`, HTTP/1.0, query splitting, duplicate headers, bodies, pipelining, split
leftovers, and all the error cases (bad request line, bad `Content-Length`,
chunked bodies, size limits).

**Integration tests (11).** These actually spawn the server as a child process
and talk to it with a raw TCP client (not `curl` and not `fetch`). They check
every route, static file streaming, path traversal (`403`), `POST /echo`,
pipelining, malformed requests (`400`), wrong methods (`405`), and `HEAD`.

Why both? Unit tests prove the parser is correct in isolation. Integration tests
prove the whole system works together over a real network connection.

Run them with:

```bash
npm test
```

Currently: **32 tests, all passing.**

---

## 7. Benchmarks

I used **autocannon**, a load-testing tool, to compare my server against Node's
built-in `http` module. Both ran single-process for a fair fight, and both
streamed the static file from disk.

| Metric | My server | Node `http` |
| --- | --- | --- |
| Peak `/json` throughput | **38,725 req/s** | 30,936 req/s |
| Peak `/static` throughput | 13,561 req/s | **16,716 req/s** |
| Avg latency at 100 connections | 2.47 ms | 3.02 ms |
| p99 latency at 100 connections | 5 ms | 5 ms |
| Errors | 0% | 0% |

**What this means:**

- My server is actually *faster* on the in-memory JSON route. The hand-written
  parser and writing the response in one buffer avoids some of Node's overhead.
- Node's built-in server is a bit faster on the disk route, because its file
  streaming code is very mature.
- Both handled 200 concurrent connections with zero errors.

Some terms:

- **Throughput (req/s)** — how many requests per second.
- **p50 / p99** — 50% / 99% of requests were faster than this. p99 is a good way
  to see the "worst typical" experience.
- **Concurrency** — how many connections are open at once.

You can reproduce it:

```bash
DURATION_MS=15000 WARMUP_MS=3000 REPS=3 \
  CONNECTIONS=10,50,100,200,500 npm run bench
```

The results go into `results/benchmark.md`.

---

## 8. What I deliberately did NOT build

A mature signal in interviews is knowing your limits. I intentionally left out:

- **Chunked request bodies** (request bodies where the length isn't known upfront)
  — I return `501 Not Implemented` for these.
- **HTTPS/TLS** — no encryption.
- **HTTP/2** — I only speak HTTP/1.1.
- **Compression, range requests, caching/ETag, rate limiting.**

There is also no metrics endpoint, no CI, and no Docker — I kept the project
small and runnable with plain Node.

Saying "I left this out on purpose, and here's why" is much better than pretending
it's a complete product.

---

## 9. Interview cheat sheet

If someone asks about my resume bullet, here's how to answer.

**"What did you build?"**
> An HTTP/1.1 server from scratch on raw TCP sockets in Node.js. I skipped the
> `node:http` module and wrote the parser, response writer, keep-alive, pipelining
> and backpressure handling myself. It scales across CPU cores with `cluster` and
> I benchmarked it against Node's built-in server.

**"Why build it from scratch instead of using Express or node:http?"**
> To learn how HTTP really works. Using a framework hides all the interesting
> parts. I wanted to understand parsing bytes, handling slow clients, and writing
> responses correctly.

**"What was the hardest part?"**
> Making the parser work when TCP splits a request across chunks, including one
> byte at a time. I solved it with a state machine and a growing buffer with an
> offset. Writing lots of tests for every split point was key.

**"What is backpressure and how did you handle it?"**
> When `socket.write()` returns false, the socket's buffer is full. If you ignore
> it, memory explodes. I stop writing (or pause the file stream) and wait for the
> `drain` event before continuing.

**"How does it scale?"**
> I use Node's `cluster` module to fork one worker per CPU core. They share port
> 3000. If one crashes, the master respawns it.

**"What is keep-alive / pipelining?"**
> Keep-alive reuses one connection for many requests instead of making a new one
> each time. Pipelining sends several requests without waiting for each answer;
> my promise queue makes sure I answer them in order.

**"How did you make it fast?"**
> Two big things: writing small JSON responses in a single buffer, and turning
> off Nagle's algorithm with `setNoDelay(true)` for streamed files (that alone
> cut static-file latency from 41 ms to 1 ms). I measured everything with
> autocannon — I peaked around 38,000 req/s on `/json` with zero errors.

**"How did you test it?"**
> 32 tests with Node's built-in runner: 21 parser unit tests (including
> byte-by-byte splits) and 11 integration tests that talk to a real server over
> raw TCP.

**"Any security considerations?"**
> Yes — path traversal. A user could request `/static/../somefile` to try to read
> files outside my public folder. I resolve the path and verify it stays inside
> the folder, otherwise I return `403`.

**"What would you add next?"**
> Chunked request decoding, HTTPS, a token-bucket rate limiter, and slow-client
> hardening tests. I left them out on purpose to keep the scope focused.

**"What does a 405 response include?"**
> An `Allow` header listing the methods that are valid for that route. Small
> detail, but it shows I care about doing HTTP properly.

---

## 10. How to run it

```bash
npm install
npm start            # clustered server on port 3000
curl http://localhost:3000/
curl http://localhost:3000/json
curl http://localhost:3000/health
curl http://localhost:3000/static/index.html
```

Single process (no cluster): `npm run start:worker`
Tests: `npm test`
Lint: `npm run lint`
Benchmark: `npm run bench`

Settings like `PORT`, `WORKERS`, `MAX_BODY_SIZE` and the timeouts can be changed
with environment variables (see `src/config.js`).

---

*That's the whole project. If you understand the ideas in section 2 and the cheat
sheet in section 9, you can confidently talk about this server in an interview.*
