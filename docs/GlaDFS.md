# Glas Distribution Filesystem (Not Likely to Happen)

Glas distributions would benefit from a filesystem with good support for structure sharing and deduplication, snapshots for history, efficient merges and diffs.

However, although developing a cool new filesystem is a fun project, the return on investment is lower than simply using a good package manager. Also, there are existing filesystems that already get close to the goals and can deliver a useful subset of the benefits. Such as Btrfs or Tahoe-LAFS.

Still, I've spent some time thinking about this and decided to write my ideas down. Maybe I'll get back to it later.

## Proposal

Leverage ideas of log-structured merge trees, merkle trees, and radix trees to construct an immutable, persistent structure with cheap snapshots that be shared within a network. Remote nodes can be encrypted, and securely accessed by hash of the encryption key.

Implement a simple memory-mapped database above this model, similar to LMDB. This should be a superior option to implementing using files within another filesystem. We can add support for network sharing at the database layer.

Then, develop a FUSE/WinFSP filesystem above the database.

## Representation

### Binary Tree Node

Start at a tree node. Each tree node has:

* extended shared prefix 
* optional external node reference
* optional deletion, definition, or update
* map from bytes to child tree nodes

To represent our options compactly and ensure extensibility, I add a bitfield (varnat) for the binary node prefix. Required bits:

* delete - symbol is deleted
* define - symbol is defined (delete + define = update)
* remote - reference to remote node (prototype / override)

See *Optimizing Bitfield* below.

### Binary Representation

For small binaries, we'll use a direct length-data encoding. 

For large binaries, we'll use a sequence of fragments encoding. Each fragment may be one of:

* small binary (with length)
* reference to raw binary, with size
* reference to sequence of fragments, with aggregate size

Sizes support indexed and ranged access. 

The motive for large binaries is to support structure for similar binaries, at least when they're large enough for this to matter. We can also support rope-like updates - including insertions and deletions - anywhere within the binary.

A sequence of fragments will simply have a fragment count followed by that many fragments. Each fragment must have one byte for fragment type.

### References

A reference shall be encoded as a *pair* of small binaries. The first binary encodes the protocol, while the second encodes the description. The protocol will usually be under ten bytes, the description under a hundred bytes. 

In some usage contexts, it might be favorable to abandon provider-independent storage in favor of small references. For example, references could represent block storage addresses, with blocks of 512 bytes or 4kB. 

With smaller references, we can effectively use smaller node sizes, ropes with smaller chunks. We would still benefit from structure sharing.

### The Map of Children 

The map of children will be encoded as a sized sequence of (byte, offset) pairs.

Inlining children require scanning through all children to find the right child. By using an offset, we instead scan through a few offsets then jump to the correct child. This supports efficient queries on the binary representation.

### Optimizing Bitfield

We can leverage the bitfield to improve performance. 

* empty map of children - leaf nodes don't have children, and most nodes are leaf nodes. This saves almost one byte per node, on average.
* prefix via pointer - moving large key prefixes to the binaries region will reduce paging and improve search performance.

More dubious:

* empty prefixes - saves one byte for branching of compact keys (e.g. numeric 00000-99999). This is rather specialized, and the limited savings would be overshadowed by the branching overheads.
* empty definitions - unless we start favoring files as flags, this is unlikely to see much use in practice. The savings would be negligible, too.
* small vs. large definitions - we could save a couple bytes for single-fragment small-binary definitions. But the saved bytes wouldn't be within the critical search path, so the benefits are limited to space savings for small files. Might be worthwhile.

These could be added if empirical evidence shows they'd be worthwhile. The lowest 7 bits for our bitfield are the most precious, so I won't sell them without clear benefits.

### Varnat Encoding

A 'Varnat' encodes an arbitrary natural number using 7 bits per byte. 

* most-significant bit is 0 for last byte, 1 for others.
* byte order is most significant to least significant

Note that byte order is opposite protocol buffers, simplifying read operation. 

### Offset Encoding

An offset is encoded as a varnat for the number of bytes to jump after reading the offset. There is no null - a `0` offset means 'next byte'. There is no negative offset - pointers must use forward jumps only.

### Layout

A goal of this representation is to support efficient search directly on the binary. An important consideration is how memory is paged and cached by the operating system and CPU. We'll want to support a very tight critical search,

The layout goal is to minimize paging when searching a key path, and to avoid unnecessary scanning or backtracking. 

A node will be represented by:

* header byte (bitfield)
* prefix (offset or inline, via bit)
* remote reference offset (if remote bit)
* definition offset (if define bit)
* map of children (if non-empty bit)

The smallest node is two bytes: deletion header byte, empty prefix. Empty prefix will be rare in context of a filesystem-like context. The map of children is encoded after the definition so we can jump straight to the definition where appropriate.

Children will be encoded favoring depth-first and lexicographic order. All definitions and remote references are effectively encoded in another section of the file. This design complicates the writer to improve reader performance. 

## Search

When a symbol define/delete/update is not found within the current node, we'll search the nearest remote node, with the appropriate relative prefix. Relevantly, we do not search the entire chain of remote nodes, just the closest one. 

To support this, the node associated with a subtree must always contain all keys associated with that subtree. Before creating a remote node, we must extract and merge the relevant subtree from an existing remote node.

## Diff

Diffs would depend on comparing both tree structure and references. We wouldn't need to load references to continue the diff except to show even deeper differences.

## Merge

Mostly, we'd merge with the remote. This requires loading the remote, then overriding definitions from the remote with the updates. 

Definition updates merge

* define fby update => define
* define fby delete => erase
* update fby delete => delete
* update fby update => update
* delete fby define => update

Things like define and delete would merge to erasure, 

## Tree-Level Manipulations

We can erase entire prefix, or copy to a prefix that is unused. This can be leveraged for snapshots, history, etc.

## Patches

A node can directly model a patch, if needed.

## Security

Security depends on the reference model. For example, we can support 

We can feasibly secure this model by encrypting remote storage and using a secure hash of the encryption key as the lookup key. The encryption might include a secure hash, so we can further authenticate the data, but it should also include some randomized salt to prevent data prediction attacks. Structure sharing, then, would require a shared history of updates. 

## Garbage Collection

References will need to be garbage collected. This is quite a challenge in context of encrypted, distributed storage. However, it can be achieved locally without much difficulty.

I would emphasize locality of GC, and making heuristic decisions to release content that can be downloaded again if needed.
