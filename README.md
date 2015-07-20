Cuerda: a scalable buffer
=========================

Buffer?  Like, an editor buffer.  Like if you want to edit a
multi-gigabyte text file.  With guaranteed worst-case performance for
the usual operations.  Cuerda is a string type into which you can
lazily load the contents of a multi-gigabyte file, attach metadata to
substrings, efficiently insert and delete text anywhere in the string,
index around as you please, and then efficiently write it back out.

You can write a simple text editor just using a regular string for the
file you’re editing, unless you’re in a language like Microsoft BASIC
or Turbo Pascal or something where your strings are limited to 255
bytes.  But it won’t perform very well, because the regular string
data type is optimized for small strings, so when you insert and
delete far from the end of a long string, it will be slow.

Also, editors have metadata associated with text in the
buffer — fonts, bookmarks, syntax-highlighting state, line numbers,
and so on.  This metadata should move with that text when it moves,
for example because stuff is inserted before it.

One well-known approach to the scalable-string problem is the
[Rope][0], where you never modify existing data (the data structure is
“persistent”), and you represent your string as a binary tree of
concatenations at whose leaves you find chunks of characters.  Using a
Rope as your string type means that the fundamental string operations
(substring, concatenation, indexing) are relatively fast (exactly how
fast depends on your tree-rebalancing strategy) and consume little
memory, and you never run afoul of your generational GC’s write
barrier, but your memory usage is unpredictable and difficult to
analyze because of sharing.  Also, you get “undo” more or less for
free.  Xanadu implemented not only editing but also version control
and transclusion with ropes.  (Or it was intended to do so; I never
quite understood how far along the implementation of the Ent, as it
was called, got.)

[0]: http://scienceblogs.com/goodmath/2009/01/26/ropes-twining-together-strings/

However, memory usage is not the main problem with ropes.  The main
problem is that editors need to attach metadata to substrings,
including hyperlink destinations, and then mutate that metadata, but
that mutable metadata moves around with the characters it’s attached
to.  And there’s no clear way to do that with ropes.

(Another, perhaps smaller, problem with ropes is that they have very
poor locality of reference, so their constant factor on modern CPUs is
very slow.)

Cuerda solves these problems, at the expense of losing ropes’ ability
to inexpensively refer to older versions of the data, and beating the
hell out of your garbage collector’s write barrier, if you have one.
Cuerda is a mutable string type that supports metadata that refers to
particular spans in the string, called “ranges”,
analogous to XEmacs “extents”,
which move with the
insertions and deletions in the string.  Cuerda has good worst-case
efficiency from cuerdas of about 32 bytes up to cuerdas of at least a
tebibyte, and with up to about 64 bytes of text per range.

Example
-------

Yeah, I’ll totally put an example here.

API
---

There are four kinds of objects in the API: cuerdas, slices, ranges,
and allocation disciplines.  A cuerda is a mutable string, and slices
and ranges are subsets of cuerdas.  Slices can be ephemerally created
and become invalid when the cuerda changes, while ranges must be
explicitly destroyed, are updated when the cuerda changes, and are
associated with some metadata.  Finally, an allocation discipline is a
way that the cuerda will allocate or deallocate when necessary.

XXX should a range just be a kind of slice?  Should the cuerda objects
be exposed at all, or are ranges enough?

XXX Also missing: an explicit search interface (so search doesn’t
involve multiple function calls per byte) and maybe some more
convenient interface.

### Basic cuerda creation and destruction ###

`cuerda *cuerda_new()` allocates a new empty cuerda with `malloc` and
returns it.  This is a convenience wrapper for the more elaborate
`cuerda_make` interface.  Constant time, assuming the allocator is
constant-time; this same assumption is made in all of what follows.

`cuerda_new` returns `NULL` if `malloc` fails.  However, you’re
probably fooling yourself if you think your program using dynamic
allocation is prepared to handle allocation failures.  It’s better in
the vast majority of cases to set jemalloc’s `opt.abort` or the
nonportable equivalent in your allocator.

`void cuerda_free(cuerda *)` destroys all the nodes in the indicated
cuerda, and deallocates the cuerda itself using the same allocator.
This is a convenience wrapper for `cuerda_destroy`.  It takes O(N)
time in the data in the cuerda.

### Slices ###

Any access to the contents of a cuerda uses a slice to indicate which
part of the contents are to be accessed.  Slices are small ephemeral
objects, and they are generally copied to avoid memory-management
headaches.  Creating and modifying slices does not write to a cuerda;
writing to a cuerda invalidates all previously existing slices on it.

`cuerda_slice cuerda_all(cuerda *)` returns a slice including the
entire cuerda in constant time.

`cuerda_slice cuerda_slice_advance(cuerda_slice, ptrdiff_t nbytes)`
returns a copy of the given slice with its beginning moved forward by
at most the given number of bytes, including backward if negative.  If
this would advance it past the end of the slice, it is moved forward
to the end of the slice instead; if this would move it backward before
the beginning of the cuerda, it is moved to the beginning of the
cuerda instead.  O(log(N)) in the size of the cuerda.

`cuerda_slice cuerda_slice_extend(cuerda_slice, ptrdiff_t nbytes)` is
analogous to `cuerda_slice_advance`, but moves the end rather than the
beginning.  If this would move it backward before the beginning of the
slice, it is moved to the beginning of the slice instead; if this
would move it forward past the end of the cuerda, it is moved to the
end of the cuerda instead.  O(log(N)) in the size of the cuerda.

`size_t cuerda_length(cuerda_slice)` returns the number of bytes in
the given slice, like `strlen`.  O(log(N)) in the size of the cuerda.

XXX iterating over chunks of bytes in a slice?

XXX overlaps, intersects, contains?

### Ranges ###

`cuerda_range *cuerda_new_range(cuerda_slice, int tag, void
*user_data)` dynamically allocates a new range pointing to the given
slice with the given metadata.  This is a convenience wrapper for
`cuerda_make_range`.  It returns NULL on allocation failure.  Takes
O(log(N)) time in the size of the cuerda.

`void cuerda_free_range(cuerda_range *)` destroys and deallocates a
range.  This is a convenience wrapper for `cuerda_destroy_range`.
O(log(N)) time in the size of the cuerda.

The more important performance consideration is that if you leave
ranges sitting around, they take up a surprising amount of memory, and
also slow down all kinds of mutation operations in their vicinity,
although not nearly as much as you probably expect if you’re used to
thinking about markers in Elisp.
So you should probably free your ranges at the
earliest opportunity.  If you love them, at least.

`cuerda_slice cuerda_get_range(cuerda_range *)` returns the slice that
the given range currently covers.  Constant time.  XXX what to do if
the cuerda is already destroyed?

`int cuerda_set_range(cuerda_range *, cuerda_slice)` makes the given
range cover an arbitrary different slice.  O(log(N)) time in the size
of the cuerda in the worst case.  It returns true unless there was an
allocation failure, which can totally happen, so watch out.

The `cuerda_range` structure itself has public fields `cuerda`, `tag`,
and `user_data`.

XXX iterating over the ranges in a slice

### Getting bytes from and putting bytes into cuerdas ###

`int cuerda_copy(cuerda_slice dest, cuerda_slice src)` replaces the
bytes contained in `dest` with a copy of the contents of `src`, which
need not be the same length, and returns
true unless there was an allocation failure.  `dest` and `src` may belong
to the same cuerda or different cuerdas.  I don’t know what it will do if they
overlap yet.  Takes O(N+log(M)) time, with N being
the size of the data copied, and M being the size of the destination cuerda.

If `dest` is empty, this is simple insertion; if `src` is empty, it
is simple deletion.  The combined operation is provided because it can
be more efficient.  If `dest` is `cuerda_all(c)`, this
is analogous to `strcpy`; if it is an empty slice at the end of a cuerda,
this is analogous to `strcat`.

The following variants provide similar functionality with other
sources of data:

`int cuerda_splice_file(cuerda_slice dest, int
fd, size_t fbegin, size_t fend)`, analogously, lazily splices a byte
range from the given file descriptor into the cuerda.  Because it’s
lazy, this takes O(log(N)) time in the previous size of the cuerda.
The cuerda takes ownership of the file descriptor — that is,
thereafter it will seek around in the file and read from it, and the
file will be closed when the cuerda is destroyed.  So don’t use the
file descriptor elsewhere thereafter.  Also, if the contents of the
file changes while the cuerda exists, the results may be
indeterminate.

XXX I don’t currently have a good way to report I/O errors.

`int cuerda_splice_bytes_lazily(cuerda_slice dest, char *bytes,
char *bend)`, analogously, lazily splices a region
of memory into the cuerda.  You should guarantee that the memory will
not change until you remove this region from the cuerda; if it
changes, results may be indeterminate.  Also takes O(log(N)) time in
the previous size of the cuerda.

`int cuerda_splice_bytes(cuerda_slice dest, char *bytes,
char *bend)`, analogously, eagerly splices a region of memory
into the cuerda, immediately copying it into cuerda nodes.  This takes
O(log(N) + M) time, where N is the previous size of the cuerda, and M
is `bend-bytes`.

`void cuerda_get(char *dest, cuerda_slice src)`
copies the specified byte range from the cuerda into the space
starting at `dest`.  This does not NUL-terminate `dest`; it writes
strictly `cuerda_len(src)` bytes to `dest`, and cuerdas support NUL bytes
just like any other byte.  NUL-terminate it yourself if you need that.
This takes O(log(N) + M) time, where N is the total size of the
cuerda, and M is the `cuerda_len(src)`.

`void cuerda_write(int fd, cuerda_slice src)` writes
the specified byte range from the cuerda to the given file descriptor.
God knows what kind of time this will take.  It may hang with `nfs
server bozo not responding still trying` until you fix your NFS
server.

XXX I don’t currently have a good way to report I/O errors here
either.  At least this time I have a return value I could use.

### Marks ###

XXX this section is obsolete but I haven’t fixed the ranges section
yet

`cuerda_mark *cuerda_first_mark(cuerda *, size_t begin, size_t end)`
returns a pointer to the first mark within the given region, or `NULL`
if there are none.  O(log(N)) time in the size of the cuerda.

`cuerda_mark *cuerda_next_mark(cuerda_mark *, size_t end)` returns the
next mark following the given mark before the position `end`, or
`NULL` if there is none.  O(N) time in the distance to `end`, since
this could potentially traverse an arbitrarily large number of
leafnodes.

### More flexible but less convenient functions ###

`int cuerda_make(cuerda *dest, cuerda_allocator alloc)` sets up a new
cuerda in the uninitialized memory at `dest` to represent emptiness,
using the given allocator discipline to request new nodes.  It returns
true unless there is an allocation failure.  Constant time.

`void cuerda_destroy(cuerda *)` deallocates all the nodes of the
specified cuerda, but doesn’t try to deallocate the cuerda itself.
O(N) time.

`int cuerda_make_range(cuerda_range *dest, cuerda_slice, int
tag, void *user_data)` sets up a new range in the uninitialized memory
at `dest`.  It returns true unless there is an allocation failure.
O(log(N)) in cuerda size.

`void cuerda_destroy_range(cuerda_range *)` destroys a range (detaching
it from its cuerda) but doesn’t try to deallocate it.  O(log(N)) in
cuerda size.

An allocator discipline `cuerda_allocator` is a struct specifying how
to allocate and deallocate nodes; you can initialize the standard one
on most systems as:

    cuerda_allocator std = {malloc, free};

I think strict standards compliance requires wrapping `malloc` and
`free` in two-argument functions here.

The full definition is as follows:

    typedef struct {
        void *(*malloc)(size_t, void *user_data);
        void (*free)(void *, void *user_data);
        void *user_data;
    } cuerda_allocator;

When the `malloc` and `free` members are invoked, the `user_data`
supplied is passed to them as a second argument.  Most C
implementations use the same calling convention for nearly all
functions, which has to be caller-pops-args in order to support
varargs, so the system `malloc` and `free` will **never know** if
cuerda invokes them with this extra argument!

### Threading ###

Cuerda doesn’t lock.  In the usual case where a cuerda is confined to
a single thread, this is what you want.  If you’re going to access a
cuerda from multiple threads, go ahead, but make sure you don’t have
multiple threads modifying it at a time, or any other threads reading
it while some thread is modifying it.

This is tricky because anything you lazily splice into a cuerda
involves implicit modification even when you just read it.  So you
have to be extra mutexy with cuerdas that might have lazy splices
still active; none of this multiple-readers nonsense.  Otherwise,
multiple readers are fine.

Creating and moving ranges in a cuerda counts as modifying it.
Creating and manipulating `cuerda_slice` objects for it does not.

Also, keep in mind that, if you’re using threads,
you probably need some memory fence
instructions to make sure those modifications are completely visible,
like the ones introduced by pretty much whatever locking mechanism you
use.  Use whatever you want.  Just don’t expect me to be interested in
your sick perversions.  Remember, ken created fork() for a reason.

Internal design overview
------------------------

A cuerda is a mutable in-memory B-tree, in which each non-root node
has a pointer to its parent, and each parent knows the total number of
bytes of text in each child’s subtree.  There are two types of leaf
nodes: those that contain literal bytes in memory and those that
lazily load data from an open file or area of memory.

A range consists of the attached metadata and two marks, internal
objects that do not escape the cuerda API.

Each leaf node
has an array of pointers to the marks in that leaf node; each mark has
a pointer to its leaf node, a byte offset within that leaf node, a
type tag, and a pointer to some arbitrary application-specific
metadata associated with that mark and that type tag.

The root node’s parent pointer points at the cuerda object, because
when we split the root node, we need a way to point the cuerda object
at the new root node.

The cuerda object itself contains pointers to the first and last leaf
nodes in order to make traversal from the beginning constant-time.

Because nodes have parent pointers, they cannot be shared between
cuerdas.

Leaf nodes are split when they contain either too many marks or too
much text.

Except in a leaf-node-only tree, all tree nodes are 504 bytes, a
strange number chosen, phyllotactically, to minimize cache-line
collisions, while having a total size in a reasonable range.

On a 64-bit machine, internal nodes contain an 8-byte parent pointer
followed by 31 (child-pointer, subtree-weight) pairs, 16 bytes each.
This permits a branching factor of 31.

Normal leaf nodes also contain a parent pointer, and then they are
divided between text bytes and an array of mark pointers, using three
counts: a current number of text bytes, a current number of bytes of
slack space after the text bytes, and a current number of mark
pointers, which run to the end of the node.  (If all leaf nodes were
the same size, this third would be unnecessary.)  These three numbers
are packed together into a 32-bit integer, leaving 492 bytes for text
and mark pointers.

With this approach, a tebibyte of text fits into 2,234,779,732 leaf
nodes.  (So we could probably get by identifying each node inside a
given tree with a 32-bit unsigned number, but we’d still need more
than that for the subtree weights, at least 5 bytes per subtree
weights.  Consider this for later.  This would be especially helpful
for leaf-node parent pointers, where it would improve memory
efficiency by almost 1%.)

Nodes are split when they become overfull and merged when they become
less than ⅓ full.  The number of mark pointers per leaf node is
limited to 16, so if you have a lot of marks, you can end up splitting
a leaf node even if it’s mostly empty space.  This is because updating
marks after an insertion or deletion involves expensive random
accesses to memory.

A 31-way B-tree with 2,234,779,732 leaf nodes needs to have 7 levels
of internal nodes, which levels can contain respectively up to 1, 31,
961, 29,791, 923,521, 28,629,151, and 887,503,681 internal nodes.  A
7-internal-level tree of this form can accommodate 27,512,614,111 leaf
nodes, and therefore in the worst case (where all the leaf nodes are ⅓
full) can accommodate 4,512,068,714,204 bytes of combined text.  It
contains only 917,087,137 internal nodes totaling 462,211,917,048
bytes, a bit over 10% of the text size.

(XXX what if all the nodes are only ⅓ full?  Then it’s just a 10-way
B-tree!  And you only have 164 × 10⁷ ≈ 1.6GB of text.  Maybe I should
use a larger branching factor in part to tamp down on that worst-case
case, and also to reduce the 2.4% overhead in the leaf nodes and this
10% overhead of internal nodes.  See below in section “re-analyzing
with larger nodes”)

So the worst-case time for inserting a byte at the beginning of a
cuerda of less than 4.5 terabytes (of text and marks) is as follows:

1. Split the leaf node, involving the allocation and initialization of
   another 504-byte leaf node, the copying of (up to) 246 bytes of
   text into it, and the insertion of the byte into the original node,
   which involves moving all 246 remaining bytes.
2. Split the parent nodes on all 7 levels of internal nodes, except
   that the root is not split, each one involving the moving of 504
   bytes of pointers either into a new node or 16 bytes to the right.
3. Increment the 7 levels of subtree weights.
4. Iterate over the marks in the two leaf nodes modified to update
   their offsets and possibly which nodes they point to.  There cannot
   have been more than 16 mark pointers in the original leaf node, so
   this involves modifying up to 256 bytes of memory in up to 16
   places.

In total, this involves reading 8*504 + 16*16 = 4288 bytes of memory
from 24 random places (8 nodes and 16 marks) and writing the same
amount to those same places plus 8 more (16 nodes and 16 marks).  On a
cold cache, with each random memory access costing 100ns, I would
expect this to take about 4μs.  (If you have tebibytes of RAM, though,
maybe it will take longer than 100ns to access it.)

However, it’s impossible to hit this worst-case time consistently,
because even if you repeatedly insert and delete at the beginning of
the cuerda, you won’t repeatedly unsplit the higher-level nodes.  You
could repeatedly insert and delete an 83-byte string at the beginning
so that the leafnode alternates between merging and splitting, but
that won’t repeatedly split its parent node; it will only repeatedly
split and merge the leaf node, involving copying about 750 bytes each
time.  This is admittedly almost 10× bigger than the 83 bytes you’re
inserting and deleting, but that’s a reasonable level of overhead,
particularly since, in an absolute sense, we’re talking about 250
nanoseconds or so.

Even if you insert at different places in the cuerda chosen to be the
most costly, the worst you can do is 31 of these 4μs ultra-splittings
in a row.  But then there are 31× as many opportunities to split one
layer less of parent nodes, etc.

If you insert at *random* places, the situation is much better!  491
of 492 byte insertions will simply copy bytes inside the leaf node
(less than 246 bytes on average), and of the remaining 0.2%, 30 of 31
will copy those bytes and then do a single layer of splitting.

“Virtual” leaf nodes contain, rather than bytes of text, a way to get
the bytes of text, either from an open file or from some area of
memory.  When accessed, these bytes are copied into normal leaf nodes,
which can result in splitting parent nodes.

Re-analyzing with bigger nodes
------------------------------

Suppose we use 1096-byte nodes instead?

Now normal leafnodes contain 12 bytes of parent pointer and count
fields, but then have space for 1084 bytes of text and markers.  1084
bytes of text might reasonably contain up to 34 markers, which kind of
sucks because it means that inserting into the leaf node might require
adjusting offsets at 34 random memory addresses.

One tebibyte of text stored 1084 bytes per leafnode requires only
1,014,309,620 leafnodes, less than half the previous number.

Internal nodes now contain an 8-byte parent pointer followed by 68
(child-pointer, subtree-weight) pairs, so now our branching factor is
68 instead of 31.  This gets us to 1,014,309,620 leafnodes in only 6
levels of nodes (5 internal): 1, 68, 4624, 314,432, 21,381,376,
1,453,933,568; there are a total of 21,700,501 internal nodes in a
full tree, totaling 23,783,749,096 bytes, which is only about 2% of
the tebibyte of text data they index.

Having only 5 levels of internal nodes instead of 7 means that the
worst case of underfull internal nodes is not as bad.  It reduces our
branching factor to 23, so varying levels of nodes have respectively
1, 23, 529, 12,167, 279,841, and 6,436,343 nodes; the leaf nodes there
are capable of holding 6,976,995,812 bytes, but will only hold
2,329,956,166 bytes if they are minimally (⅓) full.

From this strange situation, it would be possible to fill up a subtree
until it forces the root node to split, bringing us to an undesirable
6 levels of internal nodes.  (And this is also true with the 504-byte
design.)

A full-depth split now would involve splitting 6 levels of 1096-byte
nodes instead of 8 levels of 504-byte nodes, so about 6K rather than
about 4K.  This is not an improvement.  Worse, we end up having to
update twice as many marks, which could be the majority of the
expense.

This is making pointer compression look increasingly appealing!

Re-analyzing with pointer compression
-------------------------------------

Suppose you have 504-byte nodes, but you decide to pump up the
branching factor by hook or by crook.

First, you can store the nodes in arrays of nodes, and name them by
array indices rather than by memory addresses.  If your tebibyte of
text needs up to about 3 billion nodes to store it, and it does, you
can use 32-bit indices to name the nodes.

Second, you never have 2⁶⁴ bytes of text in a subtree, or even close.
A tebibyte is 2⁴⁰, and that only needs 5 bytes.  But you could very
reasonably use a variable-length number encoding for subtree weight.
A really simple one uses 7 bits per byte to hold digits and 1 bit per
byte to signal termination; this is generally cheaper than having a
count byte.  The vast majority of subtree weights are in the bottom
couple of layers of internal nodes, and those weights are pretty
small.  A leaf node can have up to 492 bytes (9 bits of weight), and a
bottom-level internal node can have up to, say, 64 leaf nodes (15 bits
of weight).

You can probably actually get better efficiency using a couple of flag
bits in the internal node to indicate whether all subtree weights
under it are stored in 16, 32, or 40 bits.  You can probably steal
those flag bits from somewhere.

That allows the bottom couple of levels of internal nodes to use
4-byte pointers and 2-byte subtree-weight fields, for 6 bytes per
branch.  And the parent-node pointer has reduced to 4 bytes!  (Hmm,
maybe not, since we don’t know which cuerda we’re in when we follow a
marker pointer.)

So if we have 496 bytes of 6-byte branches, we can do 82-way branching
in the bottom couple of levels of the internal nodes.  So each
bottom-level internal node handles 82 leafnodes (up to 40,344 bytes),
and each next-level internal node handles 82 of those (up to 3,308,208
bytes).  Above that we need 32-bit subtree weights, or at least 24-bit
ones (but let’s use 32), so each third-level internal node divides its
496 bytes of data into 62 branches, or up to 205,108,896 bytes in all.
That’s still well inside the 32-bit range, so the fourth-level
internal nodes can still branch 62 ways, each handling 12,716,751,552
bytes, which is just about the size of my entire RAM.  Then a
fifth-level node needs 5-byte subtree weights, so it can only branch
55 ways, for a total of 699,421,335,360 bytes, which is still not a
tebibyte, although it’s a lot bigger than my current RAM.  But a
sixth-level node can handle dozens of tebibytes.

The downside of using arrays in this way is that we have to manage the
allocation.  But this is not an impossible problem.  The cuerda object
points to an array of pointers to node arrays, increasing in size by
factors of 2, and there’s an allocation index that points to the next
uninitialized node, and a linked list of already-freed nodes (from
coalescing) to allocate from before bumping the allocation index.

However, if you use a real allocator like jemalloc for this, it may do
a better job at avoiding the allocation of memory that isn’t being
used, and you don’t have to do address arithmetic to dereference
pointers between nodes.  You just have 8 bytes per pointer.

If we can align the nodes to 16-byte boundaries and allocate them
within a specified region of virtual memory that’s no more than 2³⁶
bytes in size, then you can subtract a base address and shift the
offset right by four bits to get a 32-bit value; this approach is what
the JVM calls “compressed oops”, and it gives you up to 64 gibibytes
of memory space with 32-bit pointers.  The upcoming 4.0.0 release of
jemalloc will hopefully support custom chunk allocators and give you 4
bytes per pointer and also no address arithmetic for dereferencing.
In the meantime, you could potentially just use `mmap()`.
