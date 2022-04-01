# Reduction of the GetPage@LSN problem to GetPage@LatestLSN

## 1. Motivation

The Socrates paper [1] from Microsoft mentions that "Serving GetPage@LSN requests is also straightforward. The Page Server simply follows the protocol of Section 4.4", and I wish I could say the same about our system. Our storage format is complicated and still unfinished. At first glance it looks like our problem is significantly harder because our GetPage@LSN **cannot** return a newer version than what was requested. We're serving historical data, and Socrates is serving latest data.

My goal here is to show that our GetPage@LSN problem reduces to GetPage@LatestLSN.

## 2. Context
Why does our pageserver need to serve historical data? Because we want the ability to serve read nodes at some point, and they will lag. Not all read replicas will reuse an existing pageserver, but there are at least a few cases where that seems useful:
1. Not much read traffic, but read replica helps with isolating read-only queries
2. Lots of read traffic, and primary is already the biggest instance available

These two use cases are enough to justify our desire to keep our options open.

## 3. Definitions
The following terms are used in section 4.

Definition: Let `hot_wal_size` be a very conservative upper bound on how much our read replicas can lag behind, in terms of WAL size. To allow for ~30 sec lag, `hot_wal_size` should be about 15-200GB. These numbers are derived from theoretical upper bounds on write speed at safekeepers, which limit our write troughput. For now the exact value doesn't matter, since this is a workable range for the sake of this document.

Definition: A page version is **hot** if it has been modified within the last `hot_wal_size` gigabytes of wal, and **cold** otherwise.

## 4. The problem reduction
We define 2 helper lemmas and use them in the final proof below.

### 4.1 Lemma 1
Statement: The pageserver will only get two kinds of get_page queries:
a) get a hot page at some LSN within `hot_wal_size`
b) get the latest cold version of a page

Proof: It follows from the definition of `hot_wal_size` as an upper bound on replica lag. 

### 4.2 Lemma 2
Statement: Answering queries of type (a) is easy.

Proof: The `hot_wal_size` fits in RAM. One implementation of (a) is to just read the `PageReconstructData` from ram, and optimize our wal redo enough so that get_page is pretty fast. Some people believe wal redo can be as fast as memcopy. Others believe that there alternative in-memory data structures that obviate the need for redo. I'm sure there are other approaches, but in the end this is easier than finding `PageReconstructData` from disk. I refrain from picking a solution in order to avoid strawman arguments.

### 4.3 The reduction
Statement: The GetPage@LSN problem reduces to GetPage@LatestLSN.

Proof: Assume we have an implementation of GetPage@LatestLSN. To prove our reduction we need to derive an equally efficient implementation of GetPage@LSN.

By Lemma 1, we can split queries into two kinds, and match on the kind of query we have:
a) By Lemma 2, we can solve kind (a) with just some memory overhead (that I'm sure we can afford).
b) We take the WAL stream that lags `hot_wal_size` behind head, and feed it into our system for answering GetPage@LatestLSN (that we assumed exists). To answer the query of kind (b), we directly query this system.

## 5. More nuance on layered timmelines
Our layered timeline additionally helps with branching and cloud storage. I want to address these two problems in a non-layered repo, since my solution here doesn't specifically require layers (but it could).

### 5.1 Branching
Currently a branch holds up garbage collection of old layers, effectively bloating the SSD with a full database image that can't be deleted. If we don't have layers to rely on, we can just phyiscally copy whatever SSD files we have to support our branches, and that will have equivalent costs. To make sure that branch **creation** is fast, we can use copy-on-write semantics on these files.

### 5.2 Cloud storage
The assumed GetPage@LatestLSN solution is assumed to come with its own durability guarantees. The Microsoft pageservers also store to cloud storage.

## 6. Conclusion
If my reasoning is correct, it means we have many opportunities to simplify our designs and stop worrying about irrelevant problems (layer_map, PageReconstructData from disk, etc.) I'm not suggesting we build anything right now, just trying to simplify how we think, and learn something if I'm wrong.

## 6. References

[1] https://www.microsoft.com/en-us/research/uploads/prod/2019/05/socrates.pdf
