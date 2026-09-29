# Writing My Own Git: Building `gut` from First Principles

*How a 1600-line Python file taught me that Git is not magic — it's just a content-addressed filesystem with some clever bookkeeping on top.*

---

## Table of Contents

1. [Why build Git from scratch?](#1-why-build-git-from-scratch)
2. [The architecture of `gut`](#2-the-architecture-of-gut)
3. [Part 1: The repository](#3-part-1-the-repository)
4. [Part 2: Objects — the content-addressed filesystem](#4-part-2-objects--the-content-addressed-filesystem)
5. [Part 3: Commits — key-value lists with messages](#5-part-3-commits--key-value-lists-with-messages)
6. [Part 4: Trees — snapshots of directories](#6-part-4-trees--snapshots-of-directories)
7. [Part 5: References — naming the unnameable hashes](#7-part-5-references--naming-the-unnameable-hashes)
8. [Part 6: The index — Git's staging area is a binary file](#8-part-6-the-index--gits-staging-area-is-a-binary-file)
9. [Part 7: status, add, rm, and commit — putting it all together](#9-part-7-status-add-rm-and-commit--putting-it-all-together)
10. [What surprised me most](#10-what-surprised-me-most)

---

## 1. Why build Git from scratch?

I use Git every single day. I type `git commit`, `git push`, occasionally paste a hash into `git checkout`, and mostly treat it as a black box that "saves my work." At some point the curiosity got the better of me: **what actually happens when I run `git commit`?**

So I followed the excellent [*Write Yourself a Git*](https://wyag.thb.lt) tutorial and built my own implementation, called **`gut`**. The entire thing lives in one Python file (`libgut.py`) with a two-line executable wrapper:

```python
#!/usr/bin/env python3

import libgut
libgut.main()
```

That's it. No dependencies beyond the standard library (`zlib`, `hashlib`, `configparser`, `argparse`). And yet by the end, this toy can initialize repositories, hash files, make commits, resolve references, create tags, show status, and print history.

The big reveal up front: **Git is a content-addressed filesystem.** Everything else — branches, tags, staging areas, commits — are conventions layered on top of that one idea.

---

## 2. The architecture of `gut`

The whole program follows one simple pattern: an argument parser at the top, subparsers for each command, and *bridge functions* (`cmd_*`) that translate parsed arguments into actual library calls.

```python
def main(argv=sys.argv[1:]):
    args = argparser.parse_args(argv)
    match args.command:
        case "add":
            cmd_add(args)
        case "cat-file":
            cmd_cat_file(args)
        case "check-ignore":
            cmd_check_ignore(args)
        case "commit":
            cmd_commit(args)
        case "hash-object":
            cmd_hash_object(args)
        case "init":
            cmd_init(args)
        case "log":
            cmd_log(args)
        ...
```

Every supported command — `init`, `hash-object`, `cat-file`, `log`, `ls-tree`, `ls-files`, `status`, `add`, `rm`, `commit`, `tag`, `show-ref`, `rev-parse`, `check-ignore`, `checkout` — gets its own subparser and its own bridge function.

---

## 3. Part 1: The repository

A repository is just **a working tree plus a `.git` directory full of metadata**. So the first data structure models exactly that:

```python
class GitRepository(object):
    # a repo has a worktree showing the file structure and a subtree
    # called git sub directory to store the metadata
    worktree = None
    gitdir = None
    conf = None

    def __init__(self, path, force=False):
        self.worktree = path
        self.gitdir = os.path.join(path, ".git")

        if not (force or os.path.isdir(self.gitdir)):
            raise Exception(f"Not a Git repository {path}")

        # read config file in .git/config
        self.conf = configparser.ConfigParser()
        cf = repo_file(self, "config")

        if cf and os.path.exists(cf):
            self.conf.read([cf])
        elif not force:
            raise Exception("Configuration file missing")

        if not force:
            vers = int(self.conf.get("core", "repositoryformatversion"))
            if vers != 0:
                raise Exception("Unsupported repositoryformatversion: {vers}")
```

Creating a repository (`gut init`) means creating a specific directory skeleton:

```python
assert repo_dir(repo, "branches", mkdir=True)
assert repo_dir(repo, "objects", mkdir=True)
assert repo_dir(repo, "refs", "tags", mkdir=True)
assert repo_dir(repo, "refs", "heads", mkdir=True)

# .git/HEAD
with open(repo_file(repo, "HEAD"), "w") as f:
    f.write("ref: refs/heads/master\n")
```

That `.git/HEAD` line is worth pausing on. HEAD is a *file containing text*. It doesn't contain a commit — it contains a *pointer to another file*. This "indirect reference" trick turns out to be how branches work, but more on that later.

One utility I ended up using everywhere is `repo_find`, which walks up the directory tree looking for `.git` — this is why you can run git commands from any subdirectory of your project:

```python
def repo_find(path=".", required=True):
    path = os.path.realpath(path)

    # found
    if os.path.isdir(os.path.join(path, ".git")):
        return GitRepository(path)

    # a recursion base
    parent = os.path.realpath(os.path.join(path, ".."))

    if parent == path:
        if required:
            raise Exception("No git directory")
        else:
            return None

    # recursive case
    return repo_find(parent, required)
```

---

## 4. Part 2: Objects — the content-addressed filesystem

This is the heart of everything.

A regular filesystem finds files by *name*. Git's object store finds files by *content*: the name (a SHA-1 hash) is derived from what's inside the file. Change the content, and you get an entirely new file. Identical content is stored exactly once.

A git object is just bytes with a small header:

```
<type> <size-in-ascii>\x00<contents>
```

Where `<type>` is one of `blob`, `commit`, `tree`, or `tag`.

### Reading an object

Objects live at `.git/objects/<first-two-hash-chars>/<rest-of-hash>`, zlib-compressed:

```python
def object_read(repo, sha):
    # read the sha from git repository repo and retuen a GitObject
    path = repo_file(repo, "objects", sha[0:2], sha[2:])

    if not os.path.isfile(path):
        return None

    with open(path, "rb") as f:
        raw = zlib.decompress(f.read())

        # read object type
        x = raw.find(b" ")
        fmt = raw[0:x]

        # read and validate object size
        y = raw.find(b"\x00", x)
        size = int(raw[x:y].decode("ascii"))
        if size != len(raw) - y - 1:
            raise Exception(f"malformed object {sha}: bad length")
        ...
```

### Writing an object

Writing is the mirror image — serialize, prepend header, hash, compress, store:

```python
def object_write(obj, repo=None):
    # serialize
    data = obj.serialize()
    # header addition
    header = obj.fmt + b" " + str(len(data)).encode() + b"\x00"
    result = header + data
    # hashing
    sha = hashlib.sha1(result).hexdigest()

    if repo:
        # compute path
        path = repo_file(repo, "objects", sha[0:2], sha[2:], mkdir=True)

        if not os.path.exists(path):
            with open(path, "wb") as f:
                # compress and write
                f.write(zlib.compress(result))

    return sha
```

Note the idempotence for free: since the filename *is* the hash of the content, re-writing identical content is a no-op. Deduplication isn't a feature; it's a consequence of the design.

### Blobs: the simplest object

A blob stores raw file contents and nothing else — no filename, no permissions:

```python
class GitBlob(GitObject):
    fmt = b"blob"

    def serialize(self):
        return self.blobdata

    def deserialize(self, data):
        self.blobdata = data
```

That's the whole class. A blob is literally "some bytes."

### Seeing it in action

Let me hash a file into a fresh repo:

```console
$ gut init demo
$ echo "print('hello gut')" > main.py
$ gut hash-object -w main.py
7024cb570f5a1d3074a195841f16998cca65fc42
$ find .git/objects -type f
.git/objects/70/24cb570f5a1d3074a195841f16998cca65fc42
```

Decompressing that file by hand shows the exact format described above — header, NUL byte, payload:

```python
>>> import zlib
>>> raw = zlib.decompress(open(".git/objects/70/24cb…fc42","rb").read())
>>> repr(raw)
"blob 20\x00print('hello gut')\n\n"
```

And `cat-file` reads it back out:

```python
def cat_file(repo, obj, fmt=None):
    obj = object_read(repo, object_find(repo, obj, fmt=fmt))
    sys.stdout.buffer.write(obj.serialize())
```

---

## 5. Part 3: Commits — key-value lists with messages

A commit object has this textual structure:

```
tree <sha>
parent <sha>
author Chinmay <me@example.com> 1787586745 +0530
committer Chinmay <me@example.com> 1787586745 +0530

Initial commit: main script and utils
```

It's a list of key-value pairs followed by a free-form message. The parsing function is recursive and treats the message specially — storing it under the key `None`:

```python
def kvlm_parse(raw, start=0, dct=None):
    if not dct:
        dct = dict()
        # You CANNOT declare the argument as dct=dict() or all call to
        # the functions will endlessly grow the same dict.

    spc = raw.find(b' ', start)
    nl = raw.find(b'\n', start)

    # Base case: blank line means the remainder is the message.
    if (spc < 0) or (nl < spc):
        assert nl == start
        dct[None] = raw[start+1:]
        return dct

    # Recursive case: read a key-value pair and recurse.
    key = raw[start:spc]

    # Continuation lines begin with a space ("\n "), which are
    # logically part of the same value.
    end = start
    while True:
        end = raw.find(b'\n', end+1)
        if raw[end+1] != ord(' '): break

    value = raw[spc+1:end].replace(b'\n ', b'\n')

    # Don't overwrite existing data contents
    if key in dct:
        if type(dct[key]) == list:
            dct[key].append(value)
        else:
            dct[key] = [ dct[key], value ]
    else:
        dct[key]=value

    return kvlm_parse(raw, start=end+1, dct=dct)
```

With KVLM in place, a commit is a tiny class:

```python
class GitCommit(GitObject):
    fmt=b'commit'

    def deserialize(self, data):
        self.kvlm = kvlm_parse(data)

    def serialize(self):
        return kvlm_serialize(self.kvlm)

    def init(self):
        self.kvlm = dict()
```

Reading back a real commit made by `gut commit -m "..."`:

```console
$ gut cat-file commit b41537a2dab947b385ef569ec2a485616fb6fca1
tree b62215e1149fa471537e9b026795b0fe68a78c6c
author Chinmay <chinmayrpatil123@gmail.com> 1787586745 +0530
committer Chinmay <chinmayrpatil123@gmail.com> 1787586745 +0530

Initial commit: main script and utils
```

Notice something profound here: **a commit does not contain files. It contains the hash of a tree, plus the hash of its parent commit.** History emerges purely from these parent links. That insight makes `gut log` trivial — it's just a graph walk, which is why I render it as Graphviz dot notation:

```python
def log_graphviz(repo, sha, seen):

    if sha in seen:
        return
    seen.add(sha)

    commit = object_read(repo, sha)
    message = commit.kvlm[None].decode("utf8").strip()
    ...
    print(f"  c_{sha} [label=\"{sha[0:7]}: {message}\"]")
    assert commit.fmt==b'commit'

    if not b'parent' in commit.kvlm.keys():
        # Base case: the initial commit.
        return

    parents = commit.kvlm[b'parent']

    if type(parents) != list:
        parents = [ parents ]

    for p in parents:
        p = p.decode("ascii")
        print (f"  c_{sha} -> c_{p};")
        log_graphviz(repo, p, seen)
```

Output from a real repo:

```
digraph gutlog{
  node[shape=rect]
  c_b41537a2dab947b385ef569ec2a485616fb6fca1 [label="b41537a: Initial commit: main script and utils"]
}
```

Pipe that into `dot -Tsvg` and you get an actual picture of your commit DAG.

---

## 6. Part 4: Trees — snapshots of directories

If blobs are files, trees are directories. A tree is a sorted list of entries, each being: `mode`, a space, a path, a NUL byte, and then **20 raw bytes of binary SHA** (not hex!). Parsing one entry:

```python
def tree_parse_one(raw, start=0):
    # Find the space terminator of the mode
    x = raw.find(b' ', start)
    assert x-start == 5 or x-start==6

    # Read the mode
    mode = raw[start:x]
    if len(mode) == 5:
        # Normalize to six bytes.
        mode = b"0" + mode

    # Find the NULL terminator of the path
    y = raw.find(b'\x00', x)
    # and read the path
    path = raw[x+1:y]

    # Read the 20-byte SHA in network (big-endian) order and convert
    # it to the 40-char hexadecimal string git uses.
    raw_sha = int.from_bytes(raw[y+1:y+21], "big")
    sha = format(raw_sha, "040x")
    return y+21, GitTreeLeaf(mode, path.decode("utf8"), sha)
```

One subtle detail I learned the hard way: **tree sorting puts a `/` after directory names**, so `src/` sorts before `src.txt` even though `.` sorts before `/` in plain ASCII. Git's ordering rule reproduced via Python's sort key:

```python
def tree_leaf_sort_key(leaf):
    # To ensure directories are sorted before files with the same
    # name prefix, append a trailing slash to directory names. This
    # reproduces git's tree ordering rules using Python's `key`.
    if leaf.mode.startswith(b"4"):
        return leaf.path + "/"
    else:
        return leaf.path
```

Dumping a real tree with `gut ls-tree`:

```console
$ gut ls-tree HEAD
100644 blob 7024cb570f5a1d3074a195841f16998cca65fc42	main.py
040000 tree f22673df40736638241649c54768cbbdd7a2542c	src
```

The raw tree bytes (via `cat-file tree`) show the binary SHAs hiding under the readable parts:

```
00000000  31 30 30 36 34 34 20 75  74 69 6c 2e 70 79 00 46  |100644 util.py.F|
00000010  93 ad 3c f8 b0 90 3b 98  49 7f b8 9b 8b 52 4f bf  |..<.....;....RO.|
00000020  1b 93 f4                                          |...|
```

See those bytes after `\x00`? That's the SHA of `util.py`, stored as raw bytes — `46 93 ad 3c …` becomes `f493ad3c…` in hex.

Trees nest inside trees, which makes `checkout` a pleasant recursion:

```python
def tree_checkout(repo, tree, path):
    for item in tree.items:
        obj = object_read(repo, item.sha)
        dest = os.path.join(path, item.path)

        if obj.fmt == b'tree':
            os.mkdir(dest)
            tree_checkout(repo, obj, dest)
        elif obj.fmt == b'blob':
            with open(dest, 'wb') as f:
                f.write(obj.blobdata)
```

---

## 7. Part 5: References — naming the unnameable hashes

Nobody wants to type `b41537a2dab947b385ef569ec2a485616fb6fca1`. References are simply **files whose contents are a hash**. A branch is nothing more than a file in `.git/refs/heads/`:

```python
def ref_resolve(repo, ref):
    path = repo_file(repo, ref)

    # Sometimes, an indirect reference may be broken.  This is normal
    # in one specific case: we're looking for HEAD on a new repository
    # with no commits.  In that case, .git/HEAD points to "ref:
    # refs/heads/main", but .git/refs/heads/main doesn't exist yet.
    if not os.path.isfile(path):
        return None

    with open(path, 'r') as fp:
        data = fp.read()[:-1]
    if data.startswith("ref: "):
        return ref_resolve(repo, data[5:])
    else:
        return data
```

That recursion is the whole branch mechanism: HEAD says `ref: refs/heads/master`, so we follow the pointer until we hit a hash. **A branch is literally a 41-byte text file.**

Real output from a repo where I created a tag named `v1.0`:

```console
$ gut show-ref
b41537a2dab947b385ef569ec2a485616fb6fca1 refs/heads/master
b41537a2dab947b385ef569ec2a485616fb6fca1 refs/tags/v1.0
```

Name resolution (`object_resolve`) handles everything users might type — `HEAD`, short hashes, tags, branches, remote branches — and collects candidates so ambiguity can be detected:

```python
def object_resolve(repo, name):
    candidates = list()
    hashRE = re.compile(r"^[0-9A-Fa-f]{4,40}$")

    if not name.strip():
        return None

    # Head is nonambiguous
    if name == "HEAD":
        return [ ref_resolve(repo, "HEAD") ]

    # If it's a hex string, try for a hash.
    if hashRE.match(name):
        name = name.lower()
        prefix = name[0:2]
        path = repo_dir(repo, "objects", prefix, mkdir=False)
        if path:
            rem = name[2:]
            for f in os.listdir(path):
                if f.startswith(rem):
                    candidates.append(prefix + f)

    # Try for references.
    as_tag = ref_resolve(repo, "refs/tags/" + name)
    if as_tag:
        candidates.append(as_tag)
    ...
```

There's even a bonus subtlety in `object_find`: when you ask for a *tree*, it knows how to follow a tag → commit → chain to get there.

Tags come in two flavors, and implementing both clarified the difference forever:

```python
def tag_create(repo, name, ref, create_tag_object=False):
    # get the GitObject from the object reference
    sha = object_find(repo, ref)

    if create_tag_object:
        # create tag object (commit)
        tag = GitTag()
        tag.kvlm = dict()
        tag.kvlm[b'object'] = sha.encode()
        tag.kvlm[b'type'] = b'commit'
        tag.kvlm[b'tag'] = name.encode()
        tag.kvlm[b'tagger'] = b'gut <gut@example.com>'
        tag.kvlm[None] = b"A tag generated by gut, which won't let you customize the message!\n"
        tag_sha = object_write(tag, repo)
        # create reference
        ref_create(repo, "tags/" + name, tag_sha)
    else:
        # create lightweight tag (ref)
        ref_create(repo, "tags/" + name, sha)
```

A lightweight tag is just a pointer file. An annotated tag (`GitTag`) is an *actual object* — it subclasses `GitCommit` because it uses the same KVLM format! — that points at another object. One is a sticky note, the other is a document about the thing it names.

---

## 8. Part 6: The index — Git's staging area is a binary file

This was the biggest surprise of the whole project. The staging area isn't a concept — it's a concrete binary file at `.git/index`, starting with the magic bytes `DIRC` ("DirCache").

Each entry is 62 fixed bytes plus the filename, padded to 8-byte alignment. Reading it is pure binary-format work:

```python
with open(index_file, 'rb') as f:
    raw = f.read()

header = raw[:12]
signature = header[:4]
assert signature == b"DIRC" # Stands for "DirCache"
version = int.from_bytes(header[4:8], "big")
assert version == 2, "gut only supports index file version 2"
count = int.from_bytes(header[8:12], "big")
```

Each entry stores far more than "file → hash". It caches the *filesystem stat metadata* too — ctime, mtime, device ID, inode, uid/gid, size:

```python
ctime_s = int.from_bytes(content[idx: idx+4], "big")
ctime_ns = int.from_bytes(content[idx+4: idx+8], "big")
mtime_s = int.from_bytes(content[idx+8: idx+12], "big")
...
dev = int.from_bytes(content[idx+16: idx+20], "big")
ino = int.from_bytes(content[idx+20: idx+24], "big")
mode = int.from_bytes(content[idx+26: idx+28], "big")
mode_type = mode >> 12
assert mode_type in [0b1000, 0b1010, 0b1110]
mode_perms = mode & 0b0000000111111111
...
sha = format(int.from_bytes(content[idx+40: idx+60], "big"), "040x")
flags = int.from_bytes(content[idx+60: idx+62], "big")
```

Why cache all that stat data? Because it powers `git status`'s fast path. When checking whether a file changed, you first compare timestamps/inodes — cheap metadata checks — and only fall back to hashing the file when metadata differs:

```python
stat = os.stat(full_path)

# Compare metadata
ctime_ns = entry.ctime[0] * 10**9 + entry.ctime[1]
mtime_ns = entry.mtime[0] * 10**9 + entry.mtime[1]
if (stat.st_ctime_ns != ctime_ns) or (stat.st_mtime_ns != mtime_ns):
    # If different, deep compare.
    with open(full_path, "rb") as fd:
        new_sha = object_hash(fd, b"blob", None)
        same = entry.sha == new_sha
```

You can inspect all of it with `gut ls-files --verbose`.

---

## 9. Part 7: status, add, rm, and commit — putting it all together

Once objects, trees, refs, and the index exist, the everyday commands fall out naturally. `status` is a **three-way comparison**: HEAD ↔ index ↔ worktree.

```python
def cmd_status(_):
    repo = repo_find()
    index = index_read(repo)

    cmd_status_branch(repo)
    cmd_status_head_index(repo, index)
    print()
    cmd_status_index_worktree(repo, index)
```

The HEAD↔index half compares the flattened HEAD tree against index entries:

```python
def cmd_status_head_index(repo, index):
    print("Changes to be committed:")

    head = tree_to_dict(repo, "HEAD")
    for entry in index.entries:
        if entry.name in head:
            if head[entry.name] != entry.sha:
                print("  modified:", entry.name)
            del head[entry.name] # Delete the key
        else:
            print("  added:   ", entry.name)

    # Keys still in HEAD are files that we haven't met in the index,
    # and thus have been deleted.
    for entry in head.keys():
        print("  deleted: ", entry)
```

Live output from my test repo after editing and re-staging `main.py`:

```
On branch master.
Changes to be committed:
  modified: main.py

Changes not staged for commit:

Untracked files:
```

`add` is delightfully minimal once you have `rm`: it removes paths from the index and re-adds them with freshly hashed blobs:

```python
def add(repo, paths, delete=True, skip_missing=False):

    # First remove all paths from the index, if they exist.
    rm (repo, paths, delete=False, skip_missing=True)
    ...
    for (abspath, relpath) in clean_paths:
        with open(abspath, "rb") as fd:
            sha = object_hash(fd, b"blob", repo)

            stat = os.stat(abspath)
            ...
            entry = GitIndexEntry(ctime=(ctime_s, ctime_ns), mtime=(mtime_s, mtime_ns),
                                  dev=stat.st_dev, ino=stat.st_ino,
                                  mode_type=0b1000, mode_perms=0o644, uid=stat.st_uid,
                                  gid=stat.st_gid, fsize=stat.st_size, sha=sha,
                                  flag_assume_valid=False, flag_stage=False, name=relpath)
            index.entries.append(entry)

    # Write the index back
    index_write(repo, index)
```

And finally `commit` — the moment of truth. It flattens the index into nested trees bottom-up:

```python
# Get keys (= directories) and sort them by length, descending.
# This means that we'll always encounter a given path before its
# parent, which is all we need, since for each directory D we'll
# need to modify its parent P to add D's tree.
sorted_paths = sorted(contents.keys(), key=len, reverse=True)
```

…then builds the commit object pointing at the root tree and the current HEAD:

```python
def cmd_commit(args):
    repo = repo_find()
    index = index_read(repo)
    # Create trees, grab back SHA for the root tree.
    tree = tree_from_index(repo, index)

    # Create the commit object itself
    commit = commit_create(repo,
                           tree,
                           object_find(repo, "HEAD"),
                           gitconfig_user_get(gitconfig_read()),
                           datetime.now(),
                           args.message)

    # Update HEAD so our commit is now the tip of the active branch.
    active_branch = branch_get_active(repo)
    if active_branch: # If we're on a branch, we update refs/heads/BRANCH
        with open(repo_file(repo, os.path.join("refs/heads", active_branch)), "w") as fd:
            fd.write(commit + "\n")
    else: # Otherwise, we update HEAD itself.
        with open(repo_file(repo, "HEAD"), "w") as fd:
            fd.write(commit + "\n")
```

Read that last block carefully, because it's the punchline of the entire project: **making a commit means writing one hash into a text file.** Branches don't "contain" commits — they move. Every `git commit` you've ever run was, mechanically speaking, appending a line to a file in `.git/refs/heads/`.

Even `.gitignore` support fits in ~90 lines — parse patterns into `(pattern, include?)` tuples, read rules from `.git/info/exclude`, the global config, *and any `.gitignore` files already tracked in the index*, then match with `fnmatch`:

```python
def check_ignore1(rules, path):
    result = None
    for (pattern, value) in rules:
        if fnmatch(path, pattern):
            result = value
    return result
```

Last-match-wins semantics is exactly what makes `!` negation lines work.

---

## 10. What surprised me most

Building `gut` dismantled several myths I'd absorbed about Git:

1. **Git is not a delta-storage system at its core.** Each commit snapshots the full tree. Content-addressing means unchanged files share blobs across commits automatically — deduplication falls out of hashing rather than being engineered in.

2. **Branches are trivially cheap.** A branch is a 41-byte file. Creating one costs nothing; "moving" a branch is overwriting that file. There is no branch data structure anywhere in Git.

3. **The staging area is a real file with a real binary format.** Version 2 of it, with `DIRC` magic bytes, stat caching, and 8-byte alignment padding. Reading it taught me more about systems programming than anything else in the project.

4. **Commits form a Merkle DAG.** Because each commit references its parent's hash, and the tree hashes it contains cover all content, every commit cryptographically vouches for the entire history beneath it. Tamper with any blob anywhere and the root hash changes.

5. **Most of Git's UX is convention.** `HEAD`, `master`, `v1.0` — all just names resolving through files. Once `object_resolve` handled "try HEAD, try hashes, try tags, try branches," the mental model clicked.

The complete implementation is ~1600 lines of dependency-free Python covering `init`, `hash-object`, `cat-file`, `log`, `ls-tree`, `checkout`, `show-ref`, `tag`, `rev-parse`, `ls-files`, `check-ignore`, `status`, `add`, `rm`, and `commit` — fully interoperable with the real Git's object database, since both write the exact same formats.

If you've ever felt Git was arbitrary or hostile, I can't recommend building your own strongly enough. After writing `gut`, `git rebase` stopped being scary — it's just constructing new commits with different parents. Everything is.

---

*Thanks to [thblt's *Write Yourself a Git*](https://wyag.thb.lt) for the roadmap.*
