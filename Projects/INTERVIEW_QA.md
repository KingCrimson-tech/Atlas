# Bitcask — Interview Q&A

 Interviewer questions about this project, with crisp answers.

---

### Q: What did you build?
**A:** A crash-safe, log-structured key-value store in Python. Append-only segment files, an in-memory KeyDir for O(1) reads, CRC-based recovery, and manual compaction. CLI-first, stdlib only, ~1k lines.

### Q: Why log-structured / append-only?
**A:** Appends are sequential I/O — fast writes, no in-place page rewrites. Overwrites create new records; the latest wins. Cost: stale versions accumulate, so compaction is needed.

### Q: What is the KeyDir?
**A:** An in-memory `dict`: `key -> (segment_id, offset, value_size, timestamp)`. It points to the record's start offset, so a `get` is one seek instead of a full-file scan. We trade RAM for O(1) reads.

### Q: Describe the record format.
**A:** A 24-byte header `>4sQIII` (big-endian) + key + value:
`magic (4B) = b"BITC" | timestamp (8B) | key_len (4B) | val_len (4B) | crc (4B) | key | value`.
CRC32 is over header-without-crc + key + value.

### Q: Why a `magic` field?
**A:** It identifies a valid record boundary and this file format. On replay, a wrong magic means corruption/torn tail — stop, don't parse garbage.

### Q: Walk me through the write path.
**A:** Encode record → maybe rotate segment → append → `flush()` + `os.fsync()` → update KeyDir. Durability happens before the KeyDir is updated, so an acked write is on disk.

### Q: Why `fsync` on every write?
**A:** Durability — a `kill -9` or power loss loses at most the in-flight write. Trade-off: puts are slow (~326 ops/sec). In production you'd batch fsyncs.

### Q: When is the offset captured?
**A:** Before writing, via `active.tell()`. That's the exact byte where the record starts, and what gets stored in the KeyDir.

### Q: Walk me through the read path.
**A:** KeyDir lookup → open segment → seek to offset → read header + payload → verify CRC → decode UTF-8. Missing key → `KeyError`; CRC failure → `ValueError`.

### Q: Why verify CRC on reads too, not just replay?
**A:** Corruption can happen after startup. Replaying once isn't enough — a read must fail loudly instead of returning garbage bytes.

### Q: How do deletes work?
**A:** Append a tombstone (a record with no value), then remove the key from the KeyDir. The tombstone is persisted so the delete survives replay; otherwise an old value would reappear.

### Q: How is a tombstone distinguished from an empty value?
**A:** Both have zero payload bytes. The high bit of `val_len` (`TOMBSTONE_FLAG = 0x80000000`) marks a tombstone. An empty value has `val_len = 0`. Real lengths stay below 2³¹, so no collision.

### Q: What happens on startup / recovery?
**A:** Sort segments by numeric ID (oldest → newest), replay records to rebuild the KeyDir, then open a *new* active segment. Replay order matters so the latest version wins.

### Q: How is a torn/corrupt tail handled?
**A:** `iter_records` stops at the first bad magic, truncation, or CRC mismatch. `_replay` truncates the file at the last good offset (`os.truncate`) and continues. No crash; the valid prefix is preserved.

### Q: Why open a new active segment instead of appending to the last one?
**A:** The replayed tail may have been truncated. Appending to it risks inconsistency, so a fresh segment ID is always opened after replay.

### Q: How does compaction work?
**A:** Snapshot the KeyDir → for each non-active segment, rewrite only live records still pointing to it into a temp file → fsync → `os.replace` atomically → delete empty segments. Active segment is never compacted.

### Q: Why snapshot the KeyDir during compaction?
**A:** Decisions about what's live are made against a stable copy, while offsets in the live KeyDir are updated to new locations. Mixing the two would lose data.

### Q: Why temp file + rename in compaction?
**A:** Atomicity. If the process dies mid-compaction, the old segment is intact or the new one is fully in place — never a half-written, corrupt file.

### Q: Why is the active segment excluded from compaction?
**A:** It's still being appended to. Rewriting it would conflict with concurrent writes and give no benefit — its records are the newest.

### Q: How is concurrency handled?
**A:** A single `threading.RLock` wraps every public method. Reads, writes, and compaction are serialized — simple and safe, but compaction blocks everything.

### Q: How do you rotate segments?
**A:** When `active.tell() + record_size > segment_size`, close the active writer, increment the ID, and open a new `seg_%06d.log`. Default size is 2 MB.

### Q: Why sort segment IDs numerically, not lexically?
**A:** `seg_000009` < `seg_000010` as strings but lexical sort breaks past 6-digit padding. Numeric order guarantees correct replay sequence.

### Q: What's in `stats()`?
**A:** `keys`, `keydir_entries`, `segments`, `disk_bytes`, `active_segment_bytes`, `active_segment_id`, `data_dir` — returned as JSON by the CLI.

### Q: What are the benchmark results?
**A:** 50k puts with fsync: ~326 ops/sec. 50k gets (50% hit): ~93,703 ops/sec, p50 0.012 ms, p95 0.023 ms. Seed 42, Ryzen 5 5600H, Python 3.14.8.

### Q: Why are reads so much faster than writes?
**A:** Reads are in-memory lookups plus one seek; writes do a blocking `fsync` each time. The asymmetry is the durability trade-off, not a bug.

### Q: What tests cover crash safety?
**A:** Torn-write truncation, CRC corruption on replay, CRC mismatch on read, persistence across restart, overwrite-latest-wins, and delete-stays-deleted after restart.

### Q: How many tests and what's the command?
**A:** 19 tests, all passing: `python -m pytest -q`. Lint with `ruff check bitcask/`.

### Q: Why stdlib only?
**A:** No runtime dependencies keeps it portable and easy to run from a fresh clone — `struct`, `zlib`, `os`, `argparse`, `threading` are enough.

### Q: What edge cases are handled?
**A:** Empty keys rejected (`ValueError`), non-string keys rejected (`TypeError`), empty values preserved, 1 MB values, Unicode/emoji keys, newlines in keys/values, idempotent `close()`.

### Q: What are the main limitations?
**A:** Single-process, one RLock, open-per-read, no hint files, no TTL, no transactions, no replication, per-write fsync cost. It's a correctness-focused prototype, not production-tuned.

### Q: What would you improve next?
**A:** Batch fsync for throughput, cached file handles for reads, hint files to speed compaction, and TTL/expiry — in that order.
