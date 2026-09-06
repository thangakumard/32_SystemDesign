# Merkle Trees

## What they are

A Merkle tree is a tree data structure used to efficiently detect and locate
differences between two large sets of data (a directory of files, a chunked
large file, a database's key range) without transferring or comparing the
full data set. It sits alongside other tree variants — B-trees, binary
search trees, tries, AVL trees — but is purpose-built for **change
tracking and synchronization**, not lookup or ordering.

Real-world uses:
- **Git** — tracking which files/blobs in a repo have changed.
- **Cursor** (and similar code-indexing tools) — detecting which parts of a
  codebase changed so only the diff needs to be re-indexed/re-synced with a
  remote server.
- More generally: any client/server or peer/peer setup that needs to keep
  large data sets in sync (blockchains, distributed databases like Cassandra
  / DynamoDB for anti-entropy repair, IPFS, backup tools).

## Structure

- **Leaf nodes** = the actual data units — one per file, or one per chunk if
  a single large file is split into pieces.
- Each leaf stores the **hash of its content** (e.g., MD5, SHA-256).
- **Internal nodes** represent directories/groups and store a hash computed
  from their children: `hash(child1_hash + child2_hash + ...)` (concatenate
  children's hashes, then hash the result).
- This repeats up to a single **root hash**, which is a fingerprint of the
  *entire* data set.

Example from the transcript — a small project:

```
project/
├── src/
│   ├── api/
│   │   └── app.ts
│   └── scraper.ts
└── test/
```

- `app.ts` → hash of its file contents (leaf).
- `api/` (parent of `app.ts`) → hash(app.ts's hash).
- `src/` → hash(api's hash + scraper.ts's hash).
- `root` → hash(src's hash + test's hash).

Every node's hash depends on everything beneath it, so a change to any leaf
propagates a new hash all the way up to the root.

## Why this helps: efficient diffing

Say `scraper.ts` changes locally. Its hash changes, which changes `src`'s
hash, which changes the `root` hash. Now:

1. Compare **root hashes** (client vs. server). If they match, nothing
   changed anywhere — done in one comparison.
2. If they differ, ask the server for the hashes of the root's **immediate
   children** (`src`, `test`). Compare each — `test` matches (unchanged,
   skip it entirely), `src` doesn't.
3. Recurse into `src`: ask for its children's hashes (`api`, `scraper.ts`).
   `api` matches, `scraper.ts` doesn't.
4. `scraper.ts` is a leaf with no further children → it's the file that
   changed. Sync just that file.

This is effectively a **tree walk that prunes entire matching subtrees**,
so you never touch data that hasn't changed. In a sync loop, this
comparison could run on an interval (e.g., every 10 seconds) — most cycles
just compare root hashes and find everything already in sync.

## Complexity / why it's efficient

- Naive sync: compare or transfer all `n` files → **O(n)** work every time,
  regardless of how much actually changed.
- Merkle sync: cost is proportional to **the number of nodes on the path(s)
  to the changed leaves**, not the total data set. For a balanced tree with
  `n` leaves, locating one changed leaf costs roughly **O(log n)** hash
  comparisons; only the changed leaf's actual content is transferred.
- The savings scale with how *localized* the changes are — one changed file
  in a directory of thousands is cheap to find and sync; if everything
  changed, you eventually walk (and transfer) most of the tree anyway.

## Tradeoffs

- **Bandwidth/I-O vs. bookkeeping**: you avoid re-transferring unchanged
  data, at the cost of maintaining/recomputing the tree's hashes as data
  changes, and round trips to exchange hash levels.
- **Not a search/ordering structure**: unlike a BST or B-tree, a Merkle tree
  isn't optimized for lookups by key — its job is equality/diff detection.
- **Flexible shape**: the tree doesn't need to mirror a literal directory
  structure (as in the example) — a single large file can be chunked
  arbitrarily and each chunk treated as a leaf, so the technique generalizes
  beyond "one file = one leaf."
- **Multi-party sync**: the same idea extends beyond one client/one server
  to a cluster of servers keeping each other in sync, though that adds
  coordination complexity.

## Quick contrast with other trees

| Structure | Primary purpose |
|---|---|
| Binary search tree / AVL tree | Ordered lookup, insert, delete in O(log n) |
| B-tree | Ordered lookup optimized for disk/block storage |
| Trie (prefix tree) | Prefix-based lookup (autocomplete, routing) |
| **Merkle tree** | **Efficiently detect *what* differs between two data sets** |
