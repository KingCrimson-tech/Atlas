# Bitcask, From First Principles: A Deep Dive into a Crash-Safe Key-Value Store in Python

*A guide for someone who has never seen this project before — written so that, after reading it once, you can come back to any section and remember not just **what** the code does, but **why** it does it that way.*

---

## Table of contents

1. [The 30-second pitch](#1-the-30-second-pitch)
2. [Why build this at all?](#2-why-build-this-at-all)
3. [The one big idea: append, remember, replay](#3-the-one-big-idea-append-remember-replay)
4. [Architecture map: the four moving parts](#4-architecture-map-the-four-moving-parts)
5. [The record format: the contract everything rests on](#5-the-record-format-the-contract-everything-rests-on)
6. [The write path: `put`](#6-the-write-path-put)
7. [The read path: `get`](#7-the-read-path-get)
8. [Startup: replay and crash recovery](#8-startup-replay-and-crash-recovery)
9. [Deletion and statistics](#9-deletion-and-statistics)
10. [Compaction: reclaiming the garbage](#10-compaction-reclaiming-the-garbage)
11. [Concurrency, lifecycle, and edge cases](#11-concurrency-lifecycle-and-edge-cases)
12. [Tests: what is actually being protected](#12-tests-what-is-actually-being-protected)
13. [Benchmarks: honest numbers](#13-benchmarks-honest-numbers)
14. [The repository as a learning artifact](#14-the-repository-as-a-learning-artifact)
15. [Where the docs and the code disagree (and other rough edges)](#15-where-the-docs-and-the-code-disagree-and-other-rough-edges)
16. [Quick reference](#16-quick-reference)
17. [The final mental model](#17-the-final-mental-model)

---

## 1. The 30-second pitch

This repository is a small, from-scratch implementation of **Bitcask**, a *log-structured key-value store*. It is written in Python using only the standard library. It exposes five commands through a CLI — `put`, `get`, `delete`, `stats`, and `compact` — and it is designed to survive a process crash (`kill -9`) without losing any write that it already acknowledged.

The entire design fits into three sentences:

> Every write is **appended** to a file; nothing is ever modified in place. An in-memory dictionary, the **KeyDir**, remembers where the latest version of each key lives. On startup, the files are **replayed** from oldest to newest to rebuild that dictionary, and any half-written tail is detected with a checksum and cut off.

If you understand those three sentences and the trade-offs behind them, you understand this project. The rest of this blog unpacks them carefully.

Concretely, the thing you can run looks like this:

```shell
python -m bitcask.cli --dir ./data put user:1 '{"name":"Chinmay"}'
python -m bitcask.cli --dir ./data get user:1        # → {"name":"Chinmay"}
python -m bitcask.cli --dir ./data stats             # → JSON with keys, segments, bytes
python -m bitcask.cli --dir ./data delete user:1
python -m bitcask.cli --dir ./data compact           # → JSON with bytes_reclaimed
```

---

## 2. Why build this at all?

### Why not just a Python dictionary?

A `dict` is a perfectly good index — while the process is alive. The moment the process exits, it is gone. There is no such thing as "crash safety" for something that was never written down. A dict can tell you where a value is *in RAM*; it cannot tell you where it is *on disk tomorrow*.

### Why not SQLite?

SQLite is a complete, mature answer — which is exactly why it is the wrong tool for *learning*. It hides B-trees, page caches, write-ahead logs, and transaction isolation between you and the bytes. If something goes wrong, you cannot point at one file and say "the answer is in there, in order." Writing the deliberately simplified version teaches you things that merely *calling* the complete version cannot.

### Why "log-structured"?

Think of a notebook of facts:

```
user:1 -> {"name": "Chinmay"}
user:2 -> {"name": "Ada"}
user:1 -> {"name": "Chinmay", "role": "admin"}
```

When `user:1` changes, you do **not** cross out the old line. You write a new line at the bottom. The current value of `user:1` is simply the *last* line that mentions it. That notebook is a **log**, and a store built on "never modify, only append; the latest append wins" is **log-structured**.

Three reasons this beats updating a file in place:

1. **Writes are sequential.** Appending to the end of a file is the friendliest possible request to a disk. In-place updates require seeking to a random offset, reading, modifying, and writing back — slow and failure-prone, with write amplification even on SSDs.
2. **Crashes leave a clean story.** If a process dies during an in-place update, you may have half an old record and half a new one, with no way to know which is real. With an append-only log, only the *tail* can be damaged: the last record may be incomplete, but everything before it is intact. Detect the incomplete tail with a checksum, chop it off, and every acknowledged write survives.
3. **Overwrites are trivial to record.** Changing a key means appending a new version; you never have to find and patch the old bytes. The cost is space — old versions become garbage — which is why we eventually need compaction. Correctness never depends on reclaiming that space.

The original Bitcask design (Basho/Riak, 2010) added the clever part: keep an **in-memory index** so you never have to scan the notebook. That index is the KeyDir.

---

## 3. The one big idea: append, remember, replay

Every subsystem in this project is one of these three verbs:

| Verb | What it means | Where it lives |
|---|---|---|
| **Append** | A write is a new framed, checksummed record at the end of a segment file; `flush()` + `fsync()` before returning. | `log.py` (`SegmentWriter`), `db.py` (`put`, `delete`) |
| **Remember** | `KeyDir[key] = (segment_id, offset, value_size, timestamp)` records exactly where the latest version of a key lives — four integers, O(1) lookup. | `db.py` (`self._keydir`) |
| **Replay** | On startup, walk every segment oldest → newest, rebuild `KeyDir`, and truncate any torn tail at the last good offset. | `log.py` (`iter_records`), `db.py` (`_replay`) |

Everything else — tombstones, compaction, CRC-on-read, the RLock — is a consequence of making those three verbs honest under crashes.

Two supporting rules make replay work:

- **Segment IDs are creation order.** Files are named `seg_000001.log`, `seg_000002.log`, … zero-padded to six digits. Sorting by numeric ID gives chronological write order, so replaying in that order naturally makes the newest record for each key win. (The code sorts by parsed integer ID, not by filename string, to avoid the classic `seg_10.log < seg_2.log` lexical trap.)
- **Every record carries a CRC32.** A mismatch means "this record and everything after it in this file is suspect." That single rule is the entire crash-recovery story.

---

## 4. Architecture map: the four moving parts

```
   argv                                     ┌──────────────────────────┐
   ┌──────────────┐                          │  KeyDir  (in-memory)     │
   │  CLI         │  put/get/delete/...      │  key → (seg, offset,     │
   │  (cli.py)    │ ───────────────────────► │          size, ts)       │
   └──────────────┘                          └────────────┬─────────────┘
          │                                               │ lookup / update
          │ builds one Bitcask                            │
          ▼                                               ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │  Bitcask  (db.py) — the public API, owns the RLock               │
   │  put   get   delete   stats   compact   close                    │
   └───────┬──────────────────────────────────────────────────────────┘
           │ encode/decode, append, replay
           ▼
   ┌──────────────────────────┐        ┌──────────────────────────┐
   │  log.py (pure bytes)     │        │  data/segments/          │
   │  encode_record           │  ───►  │  seg_000001.log          │
   │  decode_record           │  ◄───  │  seg_000002.log ...      │
   │  SegmentWriter           │        │  (append-only, fsynced)  │
   │  iter_records            │        └──────────────────────────┘
   └──────────────────────────┘
```

The module boundaries are deliberate and worth memorizing:

- **`bitcask/log.py` speaks only bytes.** It knows the record format, how to pack/unpack a record, how to append to a file, and how to walk a file. It never sees a `str` key or a `Bitcask` object. This is why the format can be tested with no filesystem, no database, and no business logic.
- **`bitcask/db.py` speaks strings and owns the index.** It is the only file that knows about `KeyDir`, the five public operations, rotation, replay, compaction, and the `RLock`. It encodes `str → bytes` on the way into `log.py` and decodes on the way out.
- **`bitcask/cli.py` is a thin wrapper.** `argparse`, one method call, print the result. No storage logic lives here. `--dir` is a global flag placed *before* the subcommand. Values go to stdout; diagnostics (`KeyError: not found`) go to stderr with exit code 1; `stats` and `compact` print JSON to stdout so scripts can pipe them.
- **`data/` is runtime state, not source.** Segment files are binary, machine-specific, and gitignored. If you ever commit a `seg_*.log`, something has gone wrong.

---

## 5. The record format: the contract everything rests on

This is the most important section. If the bytes are wrong, nothing built on top can be right.

Every record is a **24-byte big-endian header followed by the key and value bytes**:

```
| magic (4B) | timestamp (8B) | key_len (4B) | val_len (4B) | crc (4B) | key | value |
|<-------------------------- header: 24 bytes ------------------------->|<k>|< v >|
```

Packed with `struct.Struct(">4sQIII")` — `>` means big-endian / network byte order, which disables native padding so the header is *always* exactly 24 bytes on every machine.

| Field | Size | Format | Meaning |
|---|---|---|---|
| `magic` | 4 B | `4s` | Always `b"BITC"`. Four bytes that say "this is a Bitcask record"; change it if the format ever changes. |
| `timestamp` | 8 B | `Q` | Unix seconds at write time (`int(time.time())`), unsigned 64-bit. |
| `key_len` | 4 B | `I` | Key length **in bytes** (UTF-8 encoded, not characters). |
| `val_len` | 4 B | `I` | Value length, **with a twist** — see tombstones below. |
| `crc` | 4 B | `I` | CRC32 over the rest of the record. |
| `key` | `key_len` B | — | Key bytes, UTF-8. |
| `value` | real length B | — | Value bytes, UTF-8. Absent for tombstones. |

### The CRC rule

```
crc = zlib.crc32(header_without_crc + key + value_bytes) & 0xffffffff
```

where `header_without_crc` is the packed `magic + timestamp + key_len + val_len` — the **first 20 bytes** of the header (`struct.Struct(">4sQII")`). A thing cannot be a checksum of itself, so the CRC field is excluded by construction.

Why cover the header and not just the payload? Because if `key_len` were corrupted, a reader would slice the payload at the wrong offset and happily return the wrong bytes. Covering everything means *any* single-byte flip anywhere in the record is detected. On decode, the code recomputes the CRC from the **actual bytes read off disk** (`buf[:20] + key + value`), not from re-packed fields — verifying reality, not our interpretation of it.

### Tombstones vs. empty values (an important refinement)

The original tutorial material says "`val_len = 0` means tombstone." The **as-built code is better than that** and uses the high bit of the 4-byte `val_len` as a flag:

```python
TOMBSTONE_FLAG = 0x80000000
```

- A **deleted** key stores `val_len == TOMBSTONE_FLAG` (real payload length 0, no value bytes).
- An **empty string value** `""` stores `val_len == 0`.
- A normal value stores its real byte length.

Real lengths stay below 2³¹, so the flag can never collide with a genuine length. Decode recovers the truth with `is_tombstone = bool(val_len & TOMBSTONE_FLAG)` and `real_val_len = val_len & ~TOMBSTONE_FLAG`. Record size is unchanged either way (24 + key + 0 bytes of payload).

This matters because a plain "length zero = delete" scheme silently turns `put("k", "")` into a delete — a bug that is invisible until someone stores an empty value and restarts the process. The code has a test specifically for this (see §12).

### A worked example

`put("user:1", '{"name":"Chinmay"}')` at time `1700000000`, landing at byte 1024 of segment 1:

- key: `b"user:1"` → 6 bytes
- value: `b'{"name":"Chinmay"}'` → 18 bytes

```
| BITC | 1700000000 | 6 | 18 | crc | user:1 | {"name":"Chinmay"} |
|<------------ 24-byte header ------------>|< 6 B >|<------ 18 B ------>|
|<-------------------------- 48 bytes total -------------------------->|
```

And the in-memory index entry becomes:

```python
KeyDir["user:1"] = (1, 1024, 18, 1700000000)
#                    │    │     │      └─ timestamp
#                    │    │     └──────── value size in bytes
#                    │    └────────────── record START offset (the magic), not the value
#                    └─────────────────── segment id
```

That four-integer tuple is the *entire* in-memory index. Nothing more is needed for O(1) reads.

---

## 6. The write path: `put`

```python
def put(self, key: str, value: str) -> None:
    _require_key(key)
    with self._lock:
        key_bytes = key.encode("utf-8")
        value_bytes = value.encode("utf-8")
        record = encode_record(int(time.time()), key_bytes, value_bytes)
        self._maybe_rotate(len(record))
        offset = self._active.append(record)          # write + flush + fsync
        self._keydir[key] = (self._active_id, offset, len(value_bytes), ts)
```

Four decisions hide in those few lines.

### 1. Durability: `flush()` + `os.fsync()` before returning

When Python's `write()` returns, the bytes are in the OS page cache — RAM. A power loss would erase them. `flush()` pushes Python's buffer to the OS; `os.fsync()` pushes the OS cache to physical disk. `SegmentWriter.append()` does both on **every single record**:

```python
def append(self, record: bytes) -> int:
    offset = self.tell()        # capture BEFORE writing
    self._file.write(record)
    self._file.flush()
    os.fsync(self._file.fileno())
    return offset
```

The ordering rule is absolute: **fsync before updating `KeyDir`**. If the index were updated first and the machine died before the fsync, the index would claim a write is durable that is not. Index-after-fsync means the index never lies. The cost is throughput — we will see ~326 puts/sec in the benchmark — and that trade-off is deliberate.

### 2. Offset capture: `tell()` before the append

The `KeyDir` offset must point at the record's *magic byte*. The only moment you know that position is **before** writing. Capture it after the write and you point at the end of the record — i.e. the start of the *next* record — and every later read fails in a confusing way while writes look perfectly fine. `append()` captures and returns the offset itself, so "capture + write" stays one atomic operation under the lock.

### 3. Segments and rotation

Data is split into files named `seg_%06d.log` in the `data/segments/` directory. The active segment has a soft size limit (2 MiB by default). The code checks *before* writing whether the record will fit:

```python
def _maybe_rotate(self, record_size: int) -> None:
    if self._active.tell() > 0 and self._active.tell() + record_size > self.segment_size:
        self._rotate()   # close current file, increment id, open next file
```

This "will it fit?" check is a small improvement over the tutorial's earlier "rotate after crossing the limit" version: for normal-sized records a segment no longer overshoots the configured size. (A single record larger than `segment_size` still has to go somewhere, so it becomes the sole occupant of a segment.) Rotation closes the current writer, bumps the ID, and opens the next zero-padded filename. Because IDs increase in creation order, replay order is automatically chronological.

### 4. Overwrites create garbage, and that's fine

```
put a 1   → record at offset 0   → KeyDir[a] = (1, 0, ...)
put a 2   → record at offset 26  → KeyDir[a] = (1, 26, ...)   # file GREW
```

Nothing was edited. The old record is now dead weight on disk. Disk usage grows monotonically until compaction — an explicit, accepted trade-off for crash-safe sequential writes.

---

## 7. The read path: `get`

A read has two steps, and the split is the whole point of the design:

1. **Index lookup (memory, O(1)).** `KeyDir[key]` gives `(segment_id, offset, value_size, timestamp)`. No disk touched yet. If the key is absent, `get` raises `KeyError` immediately — the index is authoritative for what is live, and there is no fallback scan.
2. **Seek and read one record (disk).** Open the segment, `seek(offset)`, read the 24-byte header, use its `key_len`/`val_len` to know how many payload bytes to read (this is what length-prefixing is for), then verify and decode.

The code then does its own careful validation rather than trusting the disk:

```python
magic, _ts, key_len, val_len, crc = HEADER.unpack(header)
if magic != MAGIC:                       raise ValueError("... bad magic")
is_tombstone = bool(val_len & TOMBSTONE_FLAG)
real_val_len = val_len & ~TOMBSTONE_FLAG
payload = f.read(key_len + real_val_len)
if len(payload) < key_len + real_val_len: raise ValueError("... short payload")
expect = zlib.crc32(header[:20] + payload) & 0xFFFFFFFF
if expect != crc:                        raise ValueError("... CRC mismatch")
if is_tombstone:                         raise KeyError(key)
return payload[key_len:].decode("utf-8")
```

Three things to notice:

- **CRC verification happens on every read, not just at startup.** A disk byte can flip *after* replay. Returning a corrupted value would be worse than failing — callers would believe a lie. A CRC mismatch raises `ValueError` ("corrupt record at segment N offset M"), distinct from `KeyError` ("not here"). Values are never silently returned from suspect bytes.
- **Cheap checks come first** (short header, bad magic, short payload) so a torn tail produces a precise error.
- **The public API only ever speaks `str`.** `db.py` owns the UTF-8 decode; `log.py` never sees a string. That is why emoji keys, newline-containing keys, and 100 KB values all work without special handling — bytes don't care about delimiters.

**Why open-per-read in v1?** Production Bitcask implementations cache file handles. Here each `get` opens and closes the segment file. That is a few extra syscalls, but it eliminates an entire class of bugs: no handle lifecycle to manage on rotation/close/error, and no stale handles reading from files that compaction deleted. The project's rule is *start with the simplest correct version, measure, then optimize with evidence*. The benchmark shows reads at ~93k ops/sec, so the simple version is not the bottleneck.

One deliberate non-feature: `get` does not cross-check that the key bytes at the offset match the key requested. The offset came from `KeyDir[key]` written by `put` for that exact key — if the bytes disagreed, the CRC would already have to be lying.

---

## 8. Startup: replay and crash recovery

This is where "crash-safe" becomes real. `KeyDir` lives only in RAM; the segment files are the source of truth. Rebuilding the index from the log is not a hack — it is *the* recovery mechanism of a log-structured store.

### Order matters

```
put a=1  (seg 1, offset 0)
put a=2  (seg 1, offset 30)
put b=3  (seg 2, offset 0)
```

Replaying oldest → newest and repeatedly overwriting `KeyDir[key]` naturally leaves `a` pointing at the newest record. Replay newest → oldest would resurrect stale values. The code discovers segments with `glob("seg_*.log")`, sorts them by parsed numeric ID, and walks them in that order.

### Torn writes and the truncation rule

An acknowledged write is fsynced, so it should be on disk — **except possibly the tail**. If the process dies mid-`write()`, a segment can end with half a header, a header claiming a long value that never arrived, or garbage that fails the CRC. This is a **torn write**.

`iter_records(path)` walks a file and yields `(offset, ts, key, value)` for each good record, stopping silently at the first truncated/CRC-mismatched one — it never raises for corruption. The caller then applies the rule:

> On the first bad record, stop replaying that file, truncate it to the end of the last good record with `os.truncate(path, last_good)`, and keep going. Never crash.

Why truncate rather than ignore the bad bytes? Two reasons:

1. The next `append` would otherwise start *after* the garbage, so every future replay would hit that garbage first and stop early, losing all later writes.
2. The file size should reflect real data; leaving garbage wastes space and confuses `stats()`.

Why stop at the first bad record instead of resyncing? Records are variable-length with framed headers but no per-record footer. Once the stream desyncs, the next `b"BITC"` you find could be inside a value. Skipping forward risks resurrecting deleted keys. The CRC makes detection reliable, and the readable prefix is always preserved. A diagnostic ("replayed N segments, M keys, truncated K bytes from …") goes to **stderr**, keeping stdout clean for scripts.

### Always start a fresh active segment

After replay, the code deliberately opens a **new** active segment with ID `max(existing) + 1` rather than appending to the last replayed file. Invariant: *replayed segments are immutable; only the newest segment is writable*. This avoids reopening a file whose torn tail was just cut off, and keeps rotation trivial. The cost is one extra small empty file per restart — which is why a CLI workload of 200 one-shot `put` invocations produces ~201 segments. Fine for v1.

### The three outcomes a `get` can produce after recovery

| Situation | Result |
|---|---|
| Key exists in a good record | Value returned |
| Key was deleted (tombstone replayed) | `KeyError` — deletion survives restart |
| Key's record was in a torn/corrupt tail | Truncated away; `KeyError` — never garbage |

---

## 9. Deletion and statistics

### Delete: append a tombstone, never erase

The naive delete — `del KeyDir[key]` — works until restart: replay finds the old `put` record again and the key **resurrects**. Since the log is append-only, deletion must append a marker that replay will interpret as "remove this key." That marker is a **tombstone**: a normal record with the key and the tombstone flag set on `val_len`, with no value bytes.

Replay logic is already built for it:

```python
if value is None:                          # tombstone
    self._keydir.pop(key, None)            # pop(..., None) so double-delete is a no-op
else:
    self._keydir[key] = (seg_id, offset, len(value), ts)
```

The code **always appends a tombstone on delete**, even if the key does not currently exist. Skipping non-existent deletes seems like an optimization but is incorrect: a `delete` that follows a `put` living in an older segment must shadow that older record, so a marker must exist in the log. Correctness beats saving ~30 bytes.

Sequence worth internalizing: `put a=1; delete a; put a=3` → replay sees put, pop, put → `get("a") == "3"`. Delete-then-reput works because a tombstone only pops what came before it; later puts repopulate the entry.

### Stats: cheap counts, no scanning

`stats()` reads `len(KeyDir)`, counts registered segment files, `stat()`s them for sizes, and reports the active segment's current offset. It never looks at record contents, so it is cheap enough to call at any time. It returns:

```json
{
  "keys": 2,
  "keydir_entries": 2,
  "segments": 1,
  "disk_bytes": 120,
  "active_segment_bytes": 60,
  "active_segment_id": 1,
  "data_dir": "./data"
}
```

Note the gap that motivates compaction: `disk_bytes` counts *every version ever written*, while `keys` counts only live ones.

---

## 10. Compaction: reclaiming the garbage

Run 1,000 overwrites across 10 keys and `disk_bytes` explodes while `keys` stays 10. Without reclamation the disk fills with dead versions. Compaction (also called *merge* in Bitcask literature) is the deferred cleanup step.

**The theory in one paragraph:** a record is *live* only if `KeyDir` currently points at exactly that `(segment_id, offset)`. Every other record — overwritten, tombstoned, or superseded — can be dropped. So compaction walks each old (non-active) segment, rewrites only its live records into a temp file, and atomically replaces the old file with the temp file.

| Requirement | Why |
|---|---|
| **Live-check:** `loc.segment == seg_id AND loc.offset == offset` | Guarantees only the newest version survives; stale versions and tombstones are skipped |
| **Snapshot `KeyDir` first** | Decisions must be made against a stable copy while the live dict is being updated with new offsets |
| **Exclude the active segment** | New writes go there during the merge; rewriting/deleting it would race with `put` |
| **Temp file + fsync + `os.replace` + directory fsync** | A crash leaves either the old segment or the complete new one, never a half-written segment in place |
| **Hold the `RLock` for the whole merge** | v1 simplicity: block reads/writes for a few seconds instead of building concurrent merging |

The algorithm:

```python
live_snapshot = dict(self._keydir)
for seg_id in sorted(self._segments):
    if seg_id == self._active_id:
        continue
    tmp_writer = SegmentWriter(old_path.with_suffix(".tmp"), seg_id)
    for offset, ts, key_bytes, value in iter_records(old_path):
        if value is None:                     # tombstone — never live
            continue
        loc = live_snapshot.get(key_bytes.decode("utf-8"))
        if loc and loc[0] == seg_id and loc[1] == offset:
            new_offset = tmp_writer.append(encode_record(ts, key_bytes, value))
            self._keydir[key] = (seg_id, new_offset, len(value), ts)  # repoint
    tmp_writer.close()
    os.fsync(tmp_fd)                          # make the new data durable
    os.replace(tmp_path, old_path)            # atomic on POSIX
    self._fsync_dir(self.segments_dir)        # make the rename durable (best effort)
    if old_path.stat().st_size == 0:          # nothing live → drop the segment
        old_path.unlink()
        del self._segments[seg_id]
```

The result is `{"segments_merged": N, "bytes_reclaimed": B}` printed as JSON by the CLI. The acceptance workload (1,000 overwrites on 10 keys, 16 KiB segments) reclaims >75% of disk space, keeps all 10 keys readable, keeps deleted keys deleted, and survives a restart afterward.

A subtle but important point: the live check uses **absolute `(seg_id, offset)` identity**, not just "is this key present in the snapshot?" That prevents a key rewritten earlier in the same merge from being matched again in a later segment. A leftover `.tmp` file from a crashed merge is harmless — it does not match the `seg_*.log` glob and is ignored.

---

## 11. Concurrency, lifecycle, and edge cases

**One `RLock` for the whole object.** Every public method (`put`, `get`, `delete`, `stats`, `compact`, `close`) holds `self._lock` for its entire body. The critical case is compaction: it snapshots the KeyDir and rewrites thousands of records. If a `put` interleaved, the snapshot could go stale and a live key's new location could be overwritten with an old one — silent data loss. The GIL makes single dict operations atomic, but not a thousand of them. The lock is *re-entrant* so methods can call helpers without deadlocking. Private helpers do not lock themselves; the lock lives only at the public boundary. Consequence: `compact()` blocks readers and writers. Slow, but correct, and multi-process access is explicitly out of scope.

**`close()` and the context manager.** Each `Bitcask` keeps its active segment file open. Without cleanup, opening hundreds of instances in tests or a benchmark hits the OS file-descriptor limit (`OSError: [Errno 24]`). `close()` flushes and closes the writer, sets it to `None`, and is idempotent; `__enter__`/`__exit__` make `with Bitcask(...) as db:` release the handle on both happy and exception paths. The CLI uses the `with` form for exactly this reason.

**Edge cases, handled deliberately:**
- Empty keys are rejected with `ValueError` (`_require_key`) on `put`, `get`, and `delete` — an empty key is almost certainly a bug or a missing argument. Non-string keys raise `TypeError`.
- Empty *values* are first-class, thanks to the tombstone flag discussed in §5.
- Unicode, emoji, embedded newlines, and large values need no special code — framing is by byte length, not by delimiters. Tests prove it.

---

## 12. Tests: what is actually being protected

The suite currently has **19 passing tests** (`python -m pytest -q` → `19 passed`). Every test uses pytest's `tmp_path` fixture — never `./data` — so tests are isolated and never touch a developer's real segments. No `time.sleep()`, no unseeded randomness, no network.

They form a matrix where each group protects a specific failure mode:

| Test group | What it protects against |
|---|---|
| Encode/decode roundtrip (incl. tombstone and empty value) | CRC formula drift; empty value colliding with delete |
| `put`/`get`, persistence across restart, overwrite-latest-wins | Basic regressions in the core path |
| Delete stays deleted; delete-then-reput; delete of nonexistent | Tombstones failing to survive replay; resurrection bugs |
| Torn tail truncated; CRC corruption drops the bad record | Changes to `iter_records` / truncation logic |
| CRC mismatch on `get` raises `ValueError`; `close()` idempotent | Silent corruption on read; leaked handles |
| Compaction reclaims >75%; compaction preserves live keys | Data loss from a stale KeyDir snapshot |
| Stats counts | Fields drifting after new features |
| Unicode/emoji/newlines/100 KB/empty value | UTF-8 and framing bugs |
| Empty key rejected (×3 ops); non-string key rejected | API contract erosion |

The tests are the project's real specification. If you change the header layout, the CRC formula, replay order, or the live-check, the matrix tells you immediately which invariant you broke.

---

## 13. Benchmarks: honest numbers

`benchmark.py` is a reproducible harness: `random.seed(42)`, 50,000 puts (each with its own fsync), then 50,000 gets with a 50/50 hit/miss split, reporting `p50` and `p95` latency in addition to throughput. Run it 2–3 times and report the median. Hardware and Python version are recorded alongside the numbers.

From the README (median of 3 runs, AMD Ryzen 5 5600H, 14 GiB RAM, Fedora Linux x86_64, Python 3.14.8):

| Workload | Throughput | p50 | p95 |
|---|---|---|---|
| put (50k, fsync per write) | ~326 ops/sec | — | — |
| get (50k, 50% hit) | ~93,703 ops/sec | 0.012 ms | 0.023 ms |

Resulting store: 50,000 keys, 2 segments, 2,177,780 bytes on disk.

**How to read these numbers.** The put figure is low *on purpose*: a real `fsync` per write costs hundreds of microseconds to milliseconds. That is the durability trade-off chosen in the write path — every acknowledged write is on physical disk. Batching fsyncs (stretch goal D) would raise throughput but weaken the guarantee. Gets are fast because they are just a dict lookup plus `open`/`seek`/`read` with no fsync; open-per-read has not shown up as a bottleneck. Reporting `p95` matters because averages hide the tail that systems people care about. Reproduce with:

```shell
python benchmark.py
```

---

## 14. The repository as a learning artifact

This code was not written in one shot; the repo is structured as a **build-along tutorial** in the spirit of *Write Yourself a Git*.

- **`plan.md`** is the single source of truth for **scope**: the weekend schedule, risk table, module contracts, interview questions, definition of done, and stretch goals. Its bluntest rule: *do not add HTTP, transactions, or replication until the MVP is done and tested.*
- **`docs/00-index.md` … `docs/10-benchmarks-final.md`** (entry point also `TUTORIAL.md`) are the single source of truth for **code and concepts**: ten chapters, each following the same rhythm — idea first, a small diff labelled with its file, a *Try It* box, and a *Checkpoint* that must pass before moving on. If plan and docs disagree, the tutorial says stop and fix one before writing more code.
- **`bitcask/`** is the package you build by following the chapters.
- **Git history** is phased on purpose: record format → write path → read path → recovery → delete/stats → compaction → hardening → benchmark/README, tagged `v0.1.0-weekend`. A recruiter reading `git log --oneline` sees the phases, not one giant commit.

The tutorial's scope discipline is itself part of the project: **in scope** is five CLI commands, append-only segments, replay recovery, manual compaction, tests, a benchmark, and a README. **Out of scope** (hard no until MVP is green) includes hint files, HTTP, TTL, transactions, replication, multi-process access, and SQL. Stretch goals A–D are written down but explicitly *not* built.

---

## 15. Where the docs and the code disagree (and other rough edges)

An honest deep dive should tell you where the narrative and the implementation drifted. None of these break correctness, but knowing them prevents confusion when you read the chapters next to the source:

- **Tombstone representation.** Chapters 3 and 5 describe `val_len = 0` as the tombstone. The code (and README) uses the high-bit `TOMBSTONE_FLAG` so empty string values remain storable. The docs even flag this as a known v1 limitation; the implementation fixed it. Trust the code on this point.
- **Rotation timing.** Chapter 4 rotates *after* a write crosses the segment limit; the code rotates *before* the write with `_maybe_rotate(record_size)`. The code is slightly better (segments respect the size limit).
- **CRC on read.** Chapter 5 delegates verification to `decode_record`; the as-built `get` inlines explicit checks with precise error messages. The algorithm is the same; the error reporting is richer. This does mean the CRC formula is expressed in more than one place — something I would consolidate next.
- **`__init__.py` is empty.** Chapter 2 says it re-exports `Bitcask` and `__version__`; as built, `from bitcask import Bitcask` will not work. Use `from bitcask.db import Bitcask`.
- **Minor leftovers.** `benchmark.py` imports `statistics` without using it; `_replay` computes a `replayed_keys` set that is never read; `_segment_has_live` is an unused helper; indentation in parts of `db.py` is inconsistent. Harmless, but they are the first things I would clean up.
- **Doc test count.** Chapter 9/10 speak of a 12-test matrix; the current suite has 19 tests (extra cases for tombstone-vs-empty-value, on-read CRC, idempotent close, compaction preserving live keys, non-string keys). More coverage is a good thing; just don't be surprised by the number.

---

## 16. Quick reference

**Run it:**

```shell
python -m bitcask.cli --dir ./data put <key> <value>
python -m bitcask.cli --dir ./data get <key>
python -m bitcask.cli --dir ./data delete <key>
python -m bitcask.cli --dir ./data stats
python -m bitcask.cli --dir ./data compact
python -m pytest -q
ruff check bitcask/
python benchmark.py
```

**Constants and invariants:**

| Thing | Value |
|---|---|
| Magic | `b"BITC"` |
| Header | `struct.Struct(">4sQIII")`, 24 bytes, big-endian |
| CRC coverage | first 20 bytes + key + value; `zlib.crc32(...) & 0xffffffff` |
| Tombstone | `val_len & 0x80000000` set, no value bytes |
| Empty value | `val_len == 0`, value bytes absent, decodes to `b""` |
| KeyDir entry | `(segment_id, record_start_offset, value_size, timestamp)` |
| Segment name | `seg_%06d.log`; IDs are chronological; sort numerically |
| Rotation | before write, when `tell() + record_size > segment_size` |
| Durability order | append → flush → fsync → **then** update KeyDir |
| Recovery | replay oldest→newest; `os.truncate` at last good offset; open a new active segment |
| Concurrency | one `RLock` around every public method |

**File map:**

| File | Responsibility |
|---|---|
| `bitcask/log.py` | Bytes only: record constants, `encode_record`, `decode_record`, `SegmentWriter`, `iter_records` |
| `bitcask/db.py` | `Bitcask` class: KeyDir, five operations, replay, rotation, compaction, lock, lifecycle |
| `bitcask/cli.py` | `argparse` only; global `--dir`; JSON / stdout / stderr conventions |
| `bitcask/test_db.py` | The 19-test behavioral matrix |
| `benchmark.py` | Seeded 50k put + 50k get benchmark |
| `README.md` | Short public-facing summary with real benchmark numbers |
| `docs/`, `TUTORIAL.md` | The ten-chapter build-along tutorial |
| `plan.md` | Scope, schedule, contracts, definition of done |
| `data/`, `benchmark-data/` | Runtime segments (gitignored) |

---

## 17. The final mental model

If you remember nothing else, remember this story:

A `put` frames your key and value into a 24-byte-header record with a CRC, appends it to the end of a segment, fsyncs it, and then writes four integers into an in-memory dictionary so a later `get` can seek straight to it. A `delete` appends a flagged empty record so that replay knows to forget the key. On restart, the store replays its segments in creation order, rebuilding the dictionary and chopping off any half-written tail at the last checksum-valid record — which is why an acknowledged write survives `kill -9`. Over time, dead versions accumulate; `compact` rewrites only the records the dictionary still points at into fresh files and renames them into place, so the disk shrinks without ever risking live data. One lock serializes all of it; one `with` block guarantees cleanup.

Everything else in this repository — the docs, the tests, the benchmarks, the phased git history — exists to explain, verify, and defend that story. That is the whole project.

*Start with `docs/00-index.md` if you want to rebuild it yourself; come back to this blog when you want the whole picture in one place.*
