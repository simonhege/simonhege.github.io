---
title: "Couchbase Cross Datacenter Replication"
date: 2024-04-22T15:20:38+02:00
draft: false
tags: ['couchbase', 'replication', 'cross-data-center']
---

Couchbase is one of the few datastore providing and active-active replication with automatic conflict resolution. In this article I will try to
details the mechanism from high-level and look at the non-functional implications.

<!--more--> 

## The replication protocol 
XDCR runs on the target cluster, requesting data from the source bucket (including filtering capabilities), looking at each document one after the other.

This process is resilient to transient errors: it can resume from the exact point of the last successful update.


## Bidirectional replication
Bidirectional replication is achieved by performing 2 unidirectional replication, in opposite directions.


## Conflict resolution
Conflict is caused when "source and target copies of an XDCR-replicated document are updated independently of and dissimilarly to one another, each by a local application".
Two strategies for resolution exists:
- sequence-number-based (sequence number incremented for each revision, then CAS value, then TTL, then document flags)
- timestamp-based (last write wins, requires clocks sync)

Recommendation is to use timestamp-based for storage of raw measures, and sequence-number-based for storage of mutated data.

If timestamp-based conflict resolution is used, it is important to monitor clock drift. 
Also in case of failover, it is safe to redirect traffic to the other datacenter only after drift+replication latency.

## Couchbase CAS
CAS is an acronym for *Compare And Swap* and is used for optimistic locking.
CAS is an opaque 8 byte buffer. 48 bits are for the physical clock (nano seconds) and 16 bits for the logical clock (incremented each time the read physical clock is equal or lower compared to previously read physical clock). 
Each mutation has its own CAS.

## Multi-document transactions
In case a transaction is used, the uncommitted data is not propagated via XDCR. Changes will be eventually propagated, one by one. This means that only a partial
view of the transaction might be seen from the targe side.

Couchbase advise against bidirectional replication if transactions are in use in the application. If used, they advise:
- to use timestamp based conflict resolution
- to ensure application side that there will be no concurrent transactions on same set of document in 2 different clusters
- to ensure, in case of failure, to always retry on same cluster

This especially means that if multi-document transactions are used, the bidirectional replication can not be used for active-active mode, but only
for active-standby, with enough time paused at failover time (at least replication latency + time drift).

## Conclusion
The capabilities (and limitations) of Couchbase seems to be possible to replicate with proper storage of metadata per document, either with MongoDB or PostgreSQL.
One of the challenge will be the initial setup (initial snapshot, then switching to te change stream).
I'll explore this in next posts.

## References
- https://docs.couchbase.com/server/current/learn/clusters-and-availability/xdcr-overview.html
- https://docs.couchbase.com/server/current/learn/clusters-and-availability/xdcr-conflict-resolution.html
- https://docs.couchbase.com/java-sdk/current/howtos/concurrent-document-mutations.html
- https://docs.couchbase.com/server/current/learn/clusters-and-availability/xdcr-overview.html
- https://www.couchbase.com/blog/distributed-multi-document-acid-transactions/
- https://www.couchbase.com/blog/distributed-multi-document-acid-transactions-in-couchbase/
- https://docs.couchbase.com/server/current/learn/data/transactions.html
