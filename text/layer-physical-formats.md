Storage and file formats; Structures
==========================================

### On fast reconstruction

To reconstruct a page, we need to have

- A base page image @ some LSN.
- Any subsequent WAL records that modify that page, up to the requested LSN.

It makes sense to store the WAL in approximately that format: `(base_page_image, [WALRecord; N])`.

As long as they are stored as such, this prevents N random seeks through large files; saving on syscalls and latency.


// TODO: Because we generally request only the latest page version (or at least: one of the latest page versions)

### On fast localization of restoration data

To reconstruct a page, we need to find the reconstruction data fast; which also means as few syscalls as possible. 
Going through many (potentially not yet opened) files is thus not a feasable solution. As such, I think we need to
know, at a global or local level, where to find the latest authorative information of any page.

As such, I think we need to maintain, for each segment file in each layer of the LSM:

1. What units of underlying data were changed from the underlying layer *in this segment*
2. What units of underlying data were changed from the underlying layer *in previous, uncompacted, segments of this
   layer*, and the LSN at which that data last changed (for O(1) lookup into the next, more granular, layer).
3. (Implied by the above: If the page we’re looking for is not in (1) or (2), we can search in the next lower layer)

... where a ‘unit of underlying data’ is a postgresql block for the lowest (oldest / most precise / smallest keyspace)
LSM segments, and the segment of the lower layer for the higher (wider) layers.

### On fast localization of data #2

When we *do* eventually find some of the data we need to restore the requested page version, then that data should be
complete enough to restore the full page; i.e. for some WAL / delta records found in one file, we should not have to
go to another file to request the base page version. This implies that each page with deltas in a delta file will also
have a base image in that same delta file. *This indeed wastes some space (with a bad pathological ~ 400x worst case of
a single minimal WAL record on that page in this layer), but this is important in the long run; as it saves us from
cross-file dependencies.*

### On disk space savings methods: Physical representation of page version metadata

As Branches are always replayed in ascending order of LSN, we should store the WAL in ascending order as well; so that
any retrieval of the records (once located) is sequential.

To support fast `getpage@LSN`-requests; we can store the LSN seperated from the records they describe; so that this
data can be compressed and accessed seperately, without a need for further seeking through the file for the next 
record to find out you don't need that record because the LSN is out of the applicable range.

When storing LSNs seperate from the records they describe, we can use common compression operations to reduce the
size of this metadata.

Wal record lengths are generally small, so we can use variable-length integer encoding to store this field. Depending
on the encoding used we can save a few bytes per record, up to 12.5% of the overhead in the case of a single XlogRecord
without further data.

### On PageServer optimizations: only retain replayable records for some amount of time

At some point, storing more deltas is not worth the effort; it becomes clear that we’re not going to see requests from
Compute with LSN < X. At that point, we could replay all completed segments <X onto existing image layers; so that we
can drop the segments locally and save local storage space.

### On Pageserver optimizations: write amplification / image layers

When we have no need for intermediate image layers between layers which include
deltas (due to e.g. low overhead of finding the right page version), we can
decrease write amplification of creating and maintaining image layers by
maintaining one "base" image layer, whose pages are modified and updated when
the layers containing deltas above that image layer are removed by GC. This means
that instead of O(blocks in segment) we only need to write out 
O(blocks changed in segment) changes; which can be significantly less. This will do even
better if we keep around several differential layers for each segment, and
truncate many at once.
This of course requires that no page@lsn request will hit that image layer for
the `[old, new]` lsn range for changed pages while the changes are being applied.

Do note, though, that this is not guaranteed to work when some form of page
compression is applied: the sizes and offsets of pages cannot be guaranteed, and
space might thus be unavailable in such cases.

### Colophon

Lineage: All WAL and Page Images that apply to one page, in some Zenith Timeline
Branch: A Page Image with all WAL up to the next Page Image (if present) in a lineage.

