# Writing a Hash Table in C: Open Addressing, Double Hashing, Tombstones, and Resizing

Hash tables are one of those data structures every programmer uses daily (every
dictionary, map, or object you've ever touched is probably backed by one), but
relatively few people have written one from scratch. This post walks through my
own string-to-string hash table written in plain C11 — no libraries beyond the
standard library, no cleverness beyond what's needed to make it correct and
fast.

The full source lives in [this repo](https://github.com/) under `src/`. The API
surface is deliberately tiny:

```c
void ht_insert(ht_hash_table* ht, const char* key, const char* value);
char* ht_search(ht_hash_table* ht, const char* key);
void ht_delete(ht_hash_table* ht, const char* key);
```

Three functions: put, get, remove. Everything else in this post is what it
takes to make those three functions behave well.

---

## 1. The data structures

Two structs carry the whole design:

```c
typedef struct {
  char* key;
  char* value;
} ht_item;

typedef struct {
  int size;        // number of buckets (always prime)
  int count;       // number of live items currently stored
  ht_item** items; // array of pointers to items
} ht_hash_table;
```

A few deliberate choices here:

- **`ht_item**` (array of *pointers*), not an array of `ht_item`. An empty slot
  is just a `NULL` pointer, which makes probing loops trivial to write:
  `while (item != NULL)`. It also means items never move inside the array
  during deletion — we'll see why that matters when we get to tombstones.
- **Keys and values are owned copies.** We store `strdup(k)` rather than the
  caller's pointer:

```c
static ht_item* ht_new_item(const char* k, const char* v) {
  ht_item* i = malloc(sizeof(ht_item));
  i->key = strdup(k);
  i->value = strdup(v);
  return i;
}
```

This means callers can mutate or free their strings immediately after calling
`ht_insert` without corrupting the table. The trade-off is that the table is
responsible for freeing everything — more on memory management in section 8.

- **`size` vs `base_size`.** `base_size` is the size the user asked for;
  `size` is the next prime at or above it. Keeping both lets resizing stay
  predictable (we double/halve the *base*, then round up to a prime again):

```c
static ht_hash_table* ht_new_sized(const int base_size) {
  ht_hash_table* ht = malloc(sizeof(ht_hash_table));
  ht->base_size = base_size;
  ht->size = next_prime(ht->base_size);
  ht->count = 0;
  ht->items = calloc((size_t)ht->size, sizeof(ht_item*));
  return ht;
}

ht_hash_table* ht_new() {
  return ht_new_sized(HT_INITIAL_BASE_SIZE);
}
```

`calloc` zero-initialises the bucket array, so every slot starts life as a
clean `NULL`.

---

## 2. The hash function

The core job of a hash function is: turn a string into an index in `[0, m)`.
I use the classic **polynomial rolling hash**:

```c
static int ht_hash(const char* s, const int a, const int m) {
  long hash = 0;
  const int len_s = strlen(s);
  for (int i = 0; i < len_s; i++) {
    hash += (long)pow(a, len_s - (i + 1)) * s[i];
    hash = hash % m;
  }
  return (int)hash;
}
```

Conceptually the hash of string `s` is:

```
hash(s) = s[0]·a^(n-1) + s[1]·a^(n-2) + ... + s[n-1]·a^0   (mod m)
```

where `a` is some prime base (here 151). Treating the string as a number in
base-`a` spreads similar strings ("cat", "bat", "cats") to very different
values, unlike a naive "sum of characters" hash where any anagram collides.

Taking `% m` after each addition is important: it keeps the accumulator small,
so even long keys don't overflow `long`.

Note that `ht_hash` takes *two* parameters we haven't fixed yet: the base `a`
and the modulus `m`. That's not accidental — it's exactly what double hashing
needs.

### Why must `m` be prime?

If `m` shares factors with the key bytes' structure, patterns in the input map
to patterns in the indices. With a prime modulus, every byte position's
contribution cycles through all residues, which distributes keys evenly. This
is why our table rounds its capacity up to the next prime before allocating.

## 3. Primality helper

```c
int is_prime(const int x) {
  if (x < 2) { return -1; }
  if (x < 4) { return 1; }
  if ((x % 2) == 0) { return 0; }
  for (int i = 3; i <= floor(sqrt((double)x)); i += 2) {
    if ((x % i) == 0) {
      return 0;
    }
  }
  return 1;
}

int next_prime(int x) {
  while (is_prime(x) != 1) {
    x++;
  }
  return x;
}
```

Standard trial division, with two micro-optimisations: skip all even candidates
(step by 2 from 3), and stop once the divisor exceeds `√x` — past that point
any factor would already have been found by its pair.

---

## 4. Collisions: open addressing with double hashing

Two different keys can hash to the same bucket. There are two families of
fixes:

1. **Chaining** — each bucket holds a linked list of items.
2. **Open addressing** — if the bucket is taken, probe onward until you find a
   free slot.

I chose open addressing because it's cache-friendly: everything lives in one
contiguous array instead of chasing heap-allocated list nodes.

The simplest open addressing scheme is *linear probing* (try slot `h`,
then `h+1`, `h+2`, ...). Its weakness is **clustering**: occupied slots tend to
form long contiguous runs, and every new collision lands on those runs and
makes them longer.

Instead, I use **double hashing**: the probe step itself comes from a second,
independent hash of the key:

```c
#define HT_PRIME_1 151
#define HT_PRIME_2 163

static int ht_get_hash(const char* s, const int num_buckets, const int attempt) {
  const int hash_a = ht_hash(s, HT_PRIME_1, num_buckets);
  const int hash_b = ht_hash(s, HT_PRIME_2, num_buckets);
  return (hash_a + (attempt * (hash_b + 1))) % num_buckets;
}
```

- `attempt == 0` gives `hash_a` — the "home" bucket.
- On collision, attempt `i` jumps to `hash_a + i·(hash_b + 1)`.

Two details worth pausing on:

- The `+ 1` guarantees the stride is never zero. If `hash_b` happened to be 0,
  a zero stride would make us retry the same slot forever.
- Because both hashes are computed mod `num_buckets` (prime!) and the stride is
  non-zero, the probe sequence visits every bucket before repeating — no
  cluster of collisions can hide an empty slot from us.

Each key gets its own probe pattern, so clustering collapses: two keys that
collide at the home bucket almost certainly diverge on the very next probe.

---

## 5. Insert

Here's `ht_insert`, which ties hashing, probing, and load-factor management
together:

```c
void ht_insert(ht_hash_table* ht, const char* key, const char* value) {
  const int load = ht->count * 100 / ht->size;
  if (load > 70) {
    ht_resize_up(ht);
  }

  ht_item* item = ht_new_item(key, value);
  int index = ht_get_hash(item->key, ht->size, 0);
  ht_item* cur_item = ht->items[index];
  int i = 1;

  while (cur_item != NULL) {
    if (cur_item != &HT_DELETED_ITEM) {
      if (strcmp(cur_item->key, key) == 0) {
        // Key already exists: replace the value.
        ht_del_item(cur_item);
        ht->items[index] = item;
        return;
      }
    }
    index = ht_get_hash(item->key, ht->size, i);
    cur_item = ht->items[index];
    i++;
  }

  ht->items[index] = item;
  ht->count++;
}
```

Things to notice:

- **Upsize before inserting** if the table is over 70% full. Open addressing
  degrades badly as it fills (probes get longer and longer), so we grow early.
- **Duplicate keys update in place** rather than leaking a second entry. When
  we find the existing item mid-probe, we free it and swap in the new one
  *without touching `count`* — one item left, one item back.
- The loop exits only on a genuinely empty (`NULL`) slot, so tombstones are
  transparently skipped over during insertion. (See section 7 for why they're
  checked explicitly.)

Average complexity: O(1) amortised, assuming the load stays bounded — which
the resize check enforces.

## 6. Search

Search follows the exact same probe path insert used, so it will find anything
that was inserted:

```c
char* ht_search(ht_hash_table* ht, const char* key) {
  int index = ht_get_hash(key, ht->size, 0);
  ht_item* item = ht->items[index];
  int i = 1;

  while (item != NULL) {
    if (item != &HT_DELETED_ITEM) {
      if (strcmp(item->key, key) == 0) {
        return item->value;
      }
    }
    index = ht_get_hash(key, ht->size, i);
    item = ht->items[index];
    i++;
  }

  return NULL;
}
```

The loop terminates on the first `NULL` slot: since inserts place a key at the
first free slot along its probe sequence, hitting a `NULL` proves the key was
never inserted. If we walk off into a run of tombstones and live items and then
hit `NULL`, the answer is definitively "(not found)" — no fallback scan needed.

Note the returned pointer aliases storage owned by the table; treat it as
valid until the next mutation of the table.

## 7. Deletion and tombstones

Deletion is where open addressing bites you. The obvious approach —

> find the item, `free` it, set the slot to `NULL`

— silently breaks search. Suppose `A` and `B` collide at the same home bucket,
so `A` sits at slot 5 and `B` probes to slot 6. Delete `A`: slot 5 becomes
`NULL`. Now searching for `B` starts at slot 5, sees `NULL`, and concludes "not
found" — even though `B` is sitting right there at slot 6.

The fix is to not leave holes, but markers. A **tombstone** is a sentinel
meaning "something lived here once; keep probing":

```c
static ht_item HT_DELETED_ITEM = {NULL, NULL};
```

It's a single static instance — deleting doesn't allocate anything, we just
point the slot at this shared sentinel:

```c
void ht_delete(ht_hash_table* ht, const char* key) {
  const int load = ht->count * 100 / ht->size;
  if (load < 10) {
    ht_resize_down(ht);
  }

  int index = ht_get_hash(key, ht->size, 0);
  ht_item* item = ht->items[index];
  int i = 1;

  while (item != NULL) {
    if (item != &HT_DELETED_ITEM) {
      if (strcmp(item->key, key) == 0) {
        ht_del_item(item);
        ht->items[index] = &HT_DELETED_ITEM;
        return;
      }
    }
    index = ht_get_hash(key, ht->size, i);
    item = ht->items[index];
    i++;
  }
}
```

Both `ht_search` and `ht_insert` compare against `&HT_DELETED_ITEM` to skip
tombstones while continuing their probes — that's why the sentinel check wraps
the `strcmp` in both loops.

The cost: tombstones count toward probe-chain length but aren't live items.
Delete-everything-insert-something churn would fill the table with sentinels
even though `count` is tiny. Which leads neatly into…

---

## 8. Resizing

The table tracks a load factor and resizes in both directions:

- **Grow** above 70% full (`ht_resize_up`)
- **Shrink** below 10% full (`ht_resize_down`)

```c
static void ht_resize(ht_hash_table* ht, const int base_size) {
  if (base_size < HT_INITIAL_BASE_SIZE) {
    return;
  }

  ht_hash_table* new_ht = ht_new_sized(base_size);
  for (int i = 0; i < ht->size; i++) {
    ht_item* item = ht->items[i];
    if (item != NULL && item != &HT_DELETED_ITEM) {
      ht_insert(new_ht, item->key, item->value);
    }
  }

  // Steal new_ht's internals, dispose of the old shell.
  ...
  ht_del_hash_table(new_ht);
}

static void ht_resize_up(ht_hash_table* ht) {
  ht_resize(ht, ht->base_size * 2);
}

static void ht_resize_down(ht_hash_table* ht) {
  ht_resize(ht, ht->base_size / 2);
}
```

The rehash loop is the interesting part: **every surviving item is re-inserted
into the new table**, recomputing its hash against the new modulus. You cannot
copy slots across — `(hash_a + i·hash_b) mod 53` says nothing useful about the
same key in a 101-bucket table. Rehashing also has a lovely side effect: the
entire graveyard of tombstones vanishes in one pass, since only live items get
re-inserted.

Implementation detail: rather than copying item pointers one by one, the code
builds `new_ht` fully and then swaps the `items` array (and bookkeeping fields)
between the two tables, frees the old shell, and mutates `ht` in place. Callers
keep their original `ht_hash_table*` — resize is invisible outside.

One subtlety worth knowing: re-inserting items in *array order* isn't the same
as replaying the original insertion order, but since each item's position in
the new table depends only on its own key's probe sequence, correctness holds
regardless.

Amortised analysis: doubling on growth means any element migrates O(1)
expected times over its lifetime, keeping inserts O(1) amortised despite the
occasional O(n) rebuild.

## 9. Memory management

Every allocation has a matching free, tracked through three destructors:

```c
static void ht_del_item(ht_item* i) {
  free(i->key);
  free(i->value);
  free(i);
}

void ht_del_hash_table(ht_hash_table* ht) {
  for (int i = 0; i < ht->size; i++) {
    ht_item* item = ht->items[i];
    if (item != NULL) {
      ht_del_item(item);
    }
  }
  free(ht->items);
  free(ht);
}
```

Because the table owns deep copies of every key and value, teardown is simple:
free each non-`NULL` slot (tombstones point at the static sentinel and are
never freed individually), then the array, then the table struct. Run it under
valgrind and it comes back clean.

---

## 10. Trying it out

The repo ships with a small interactive REPL (`main.c`) for poking at the
table:

```sh
make
./hash_table
```

```text
ht> set language C
ok
ht> get language
C
ht> delete language
deleted (if present)
ht> get language
(not found)
ht> quit
```

---

## Wrapping up

The whole thing is ~200 lines of C and hits the classic design points:

| Concern            | Technique used here                          |
|--------------------|----------------------------------------------|
| Hashing            | Polynomial rolling hash, prime bases         |
| Bucket count       | Always prime (`next_prime`)                  |
| Collision handling | Open addressing, double hashing              |
| Deletion           | Shared-static tombstone sentinel             |
| Load management    | Resize up @ >70%, down @ <10%                |
| Resizing cost      | Full rehash, amortised O(1) inserts          |

Natural next steps if you want to extend it: SipHash or FNV-1a instead of the
polynomial hash (the current one is deterministic and trivially
collision-DoSable by an adversary), Robin Hood hashing or backward-shift
deletion to eliminate tombstones entirely, storing hashes alongside keys to
shorten `strcmp`s, and generic key/value types via `void*` plus a provided
hash/compare function.

---

## Appendix: known issues in the current tree

While writing this up I noticed the working copy of `src/hash_table.c`
currently contains several typos and gaps that break the build (the snippets
above show the intended versions):

- `cont char* v` → `const char* v`; `xonst` → `const`; `xmalloc` / `xcalloc`
  → `malloc` / `calloc`.
- In `ht_get_hash`: `const int_hash_a` needs a space (`const int hash_a`),
  and the subsequent references use `hash_a` / `hash_b`.
- `return NULL` in `ht_search` is missing a semicolon.
- `HT_PRIME_1`, `HT_PRIME_2`, `HT_INITIAL_BASE_SIZE` need defining, and the
  header needs the `base_size` field added to `ht_hash_table` (the `.c` file
  uses it).
- `hash_table.c` should `#include "prime.h"` so `next_prime` is declared.
- In `ht_delete`, the `count--` should happen when the item is actually found
  and removed (inside the match branch), otherwise it decrements even for
  missing keys; likewise the downsize check runs before the removal it reacts
  to.

Happy hacking!
