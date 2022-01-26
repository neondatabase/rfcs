Differential Layer
==================

Created on 2022-01-18
Type: for approval

Motivation
----------

Our current layer formats use more resources than we would like them to use.
Most significant is the Delta layer format, which has >28 bytes overhead
per page version; and has O(m log(n)) page@LSN worst-case lookup complexity.

Summary
-------

This RFC describes implementation details for a new implementation for the Layer
interface that targets to replace the Delta layer, taking into account the
concerns raised in RFC 016.

Non-goals
---------

This RFC does not plan to address integration of LSM-related structures into
the layer format, nor does it plan to change any fundamentals of the GC or
checkpoint logic.

Impacted Components
-------------------
The PageServer will be likely be the only impacted active component; with
special areas of interest being the Layer interface, checkpointing and garbage
collection. Any potential catalog / metadata service / layer index might also
need updates to support the (features of the) new layer format, if and when
this becomes relevant.

Current status
--------------

Currently, there are two layer formats: Delta and Image. 

Delta layers contain a sorted, indexed list of all changes to pages in a
certain segment and a range of LSNs, whereas Image layers contain a snapshot of
the pages at a certain LSN.

The Differential Layer format tries to improve and replace the Delta layer format.

Problems with Delta layer; and improvements
-------------------------------------------

### Page Version Branches
As mentioned in "Storage and file formats; Structures", storing page versions
in branches is an easy optimization which makes finding the right data easy.
This does mean that we will have one full page image in potential overhead for
each modified page in the layer, but that is not too bad (with Delta layers
this is one FPW per page in the segment, per delta layer, or we'd have
cross-file dependencies).

As there may be multiple kinds of access patterns to pages, it is beneficial to
store some metadata in the BranchPtr which discerns the kind of branch that
pointer points to. This allows for compacter storage of the actual branches,
at the cost of some extra data in the

Not all branches will describe multiple page versions: It is indeed possible
that a page will be updated only once during a checkpoint. In that case, it is
quite wasteful to store the metadata of '0 page versions'.

Distinguisable branch types could include, amongst others:

- FPI, 0 revisions
- FPI-initiated branch, N WAL records depending on that FPI
- will-init WAL record, 0 revisions
- will-init WAL record initiated branch, followed by N WAL records depending on that WAL record
- ... potential other types / physical representations.

### Branch "tail" representation

In the physical format, because the LSNs of the WAL records are sorted and stored
in increasing order, we can apply differential encoding and compression to this
array to reduce the storage overhead of these WAL records.

Additionally, because the length of the WAL record is stored in it's header, we
don't actually need to store the length of the WAL record in the branch structure;
we can read it directly from the record header, saving 4 bytes per WAL record.

With LSN array compression and the emission of the WAL record length stored 
outside the WAL record, we can reduce the storage overhead of non-FPI WAL
records from 28 bytes per record down to less than 8 bytes per record, with an
expected value of ~ 4 bytes (depending on how often a page is modified; more
frequent / less noizy WAL record generation = lower overhead).

### Page versions array
The Delta layer uses an in-memory sorted array of <lsn, pageversion> pairs to
represent the changes to a page. This is correct and allows for O(log N) version
lookups (for N = number of total page versions). This is OK, but can be improved
upon: as mentioned in "Storage and file formats; Structures", we always
need a base page image to reconstruct a page. By limiting the array of page
versions stored for each page in the Layer to Branches, we eliminate a
significant amount of entries from the search space; improving this search speed
by O(log fraction of non-image page-versions) and reducing memory usage.

Because we generally only require the latest version of a page, we store the newest
versions of the page in the front of the array, so that the access to this data
requires few extra operations.

Assuming that layer files will not grow past a certain size (2^48 - 1 bytes),
we can can use an 48-bit integer to store the location of a Branch.
When in memory, with the 8-byte LSN, we have 2 bytes left to store some more
metadata of the branch in the BranchPtr, such as type info and other relevant
information.


The per-page data layout of the Differential layer will thus be:

```text
  Page data:
    ... metadata
    page_versions: sorted array of (Lsn -> BranchPtr) pairs, in descending order of LSN
      Lsn -> BranchPtr:
        Lsn: 8 bytes
        BranchPtr: 8 bytes
          offset_to_branch: 6 bytes
          branch type specific info: 2 bytes
          ...
      Lsn -> BranchPtr -> ...
      Lsn -> BranchPtr -> ...
      Lsn -> BranchPtr -> ...
      ...
```

Note that the definition of BranchPtr is partially dependent on what data we
want to store in the BranchPtr, and what data we want to store locally in the
Branch. Either way, the overhead of one branch in the page versions array is
16 bytes.

### Page index
The Delta layer does not use an index to check if a page exists in the Layer, but
instead uses the combined index for on (page, pageversion). This means that finding
a page in a Delta layer is log(n) (for n = number of page versions in the Layer).
This is very expensive and not really sustainable when we're looking forward:
larger segments means larger amounts of page versions in a layer, which means
larger lookup times.  
In the Delta layer there is also no description on when a page was last modified,
so any failed lookup in the Delta layer must search back to the previous layer,
making such lookups in the worst case O(log N * M) for N pageversions per layer
and M layers. This currently is not too much of an issue, because this M is
limited to 1 due to how our layers work: we always have an image layer
immediately before the delta layer, which limits the number of Delta layers we
access to 1.

To limit the write amplification caused by the need for Image layers, the
Differential layer uses a mapping (implemented as bitmap for O(log1024(n)) access,
but could be any int->data map) to store the page index. This page index stores, for all pages
modified since the last snapshot, either a pointer to the page data, or the LSN
of the last time the page was modified after the last snapshot. This means that
using one lookup we can discern whether we can access the page data, and if not,
where we should look instead.

These improvements mean that the search for some old page version goes from 
`O(m log N)` to `O(log n)`; a nice improvement.

### Segment size
The segment size of the current layer formats is very small; which means that
the number of layers is relatively high, which increases the pressure on GC
and FS operations. As each layer has some constant overhead, it should be
beneficial to increase layer size to something in the order of gigabytes,
instead of megabytes.

The Differential layer format is designed for segment sizes of up to 8GiB,
almost 1000 times larger than the current segment size. 


### Read amplification
The Delta layer requires frequent Image layers to prevent the high overhead
of old page retrieval. The Differential layer does not require this, and thus it
allows us to decreases the read amplification for sparsely updated layers
significantly.
