Cluster size limits
==================

## Summary

One of the resource consumption limits for free-tier users is a cluster size limit.

To enforce it, we need to calculate the timeline size and check if the limit is reached before relation create/extend operations.
If the limit is reached, the query must fail some meaningful error/warning.
We may want to exempt some operations from the quota to allow users to free space to fit back into the limit.

Stateless compute node where the validation is performed is separate from the storage where the usage can be calculated, 
so we need to exchange cluster size information between those components.

## Motivation

Enforce limit on the maximum size of a PostgreSQL instance to limit free tier users (and other tiers in the future).
First of all this is needed to control the cost of our free tier production.
Another reason to limit resourses is risk management — we haven’t (fully) tested and optimized zenith for really big clusters,
so we don’t want to give users access to the functionality that we don’t think is ready.

## Components

pageserver - calculate size consumed by a timeline, add it into feedback message.
safekeeper - pass feedback message from pageserver to compute.
compute - receive feedback message, enforce size limit based on GUC `zenith.max_cluster_size`.
console - set and update `zenith.max_cluster_size setting`

## Proposed implementation

First of all, it's necessary to define timeline size. Options are:

1. Count all data, including SLRUs. (not including WAL)
Here we think of it as of a physical disk underneath postgres cluster.
This is how `LOGICAL_TIMELINE_SIZE` metric is implemented in pageserver right now and the draft PR uses it.

2. Count only relation data. As in pg_database_size().

This approach is more user-friendly, because it is the data that is really affected by user.
To implement it, we will need to refactor timeline_size counter, or rather add another counter. 
I suggest to use this approach.


Timeline size is updated during wal digestion, it is not versioned and is valid at the last_received_lsn moment.
Then this size should be reported to compute node.

current_instance_size value is included into walreceiver's custom feedback message: `ZenithFeedback`

(PR about protocol changes in review https://github.com/zenithdb/zenith/pull/1037).

This message is received by safekeeper and propagated to compute node as a part of `AppendResponse`.

Finally, when compute node receives the `current_instance_size` from safekeeper (or from pageserver directly), it updates the global variable.

And then every zenith_extend() operation checks if limit is reached `(current_instance_size > zenith.max_cluster_size)` and throws `ERRCODE_DISK_FULL` error if so.
(see Postgres error codes [https://www.postgresql.org/docs/devel/errcodes-appendix.html](https://www.postgresql.org/docs/devel/errcodes-appendix.html))

We can allow autovacuum processes to bypass this check, simply checking `IsAutoVacuumWorkerProcess()`.

It would be nice to allow manual VACUUM and VACUUM FULL to bypass the check too, but it's uneasy to distinguish these operations at the low level.

### **Reliability, failure modes and corner cases**

1. `current_instance_size` is valid at the last received and digested by pageserver lsn.
    
    If pageserver lags behind compute node, current_instance_size will lag too. This lag can be tuned using backpressure, but it is not expected to be 0 all the time.
    
    So transactions that happen in this lsn range may cause limit overflow. Especially, operations that generate (i.e CREATE DATABASE) or free (i.e. TRUNCATE) a lot of data pages while generating a small amount of WAL. Are there other operations like this?
    
    TODO: figure out how to handle them.
    
    One of the options is to update the counter locally for such operations, while pageserver lags.
    
2. How to distinguish VACUUM and VACUUM FULL operations to allow them even if limit is reached?


### **Security implications**

We treat compute as untrusted component, that’s why we try to isolate it with secure container runtime or a VM.
Malicious user may change the `zenith.max_cluster_size`, so we need extra size limit check on the safekeeper's side.

If `max_timeline_size` limit is reached on safekeeper, it means that compute node is compromised
so treat this as a hard limit and set this higher than `zenith.max_cluster_size`.
Safekeeper gets `current_timeline_size` from pageserver's feedback, so it will lag a little bit, but it's ok.
