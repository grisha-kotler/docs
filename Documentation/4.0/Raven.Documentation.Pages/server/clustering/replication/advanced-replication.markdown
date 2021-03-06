# Advanced Replication Topics

## Replication Handshake Procedure

Whenever the replication process between two databases is initiated it has to determine the process state.  

1. The first message is a request to establish a TCP connection of type _replication_ with a _protocol version_.  
2. The _destination_ verifies that the protocol version matches and that the request is authorized.  
3. Once the _source_ gets the OK message it queries the _destination_ about the latest ETag it got from him.  
4. The _destination_ sends back a heartbeat message with both the latest `ETag` he got from the _source_ and the current [Change Vector](../../../server/clustering/replication/change-vector) of the database.  
5. The `ETag` is used as a starting point for the replication process but it is then been filtered by the _destination's_ current `change vector`,
meaning we will skip documents with higher `ETag` and lower `Change Vector`, this is done to prevent the [Ripple Effect](../../../server/clustering/replication/advanced-replication#preventing-the-ripple-effect).  

## Preventing the Ripple Effect

RavenDB [Database Group](../../../server/clustering/distribution/distributed-database#distributed-database) is a fully connected graph of replication channels, meaning that if there are `n` nodes in a `Database Group` there are `n*(n-1)` replication channels.  
We wanted to prevent the case where inserting data into one database will cause the data to propagate multiple times through multiple paths to all the other nodes.  
We have managed to do so by delaying the propagation of data coming from the replication logic itself.  

If the sole source of incoming data is replication we will not replicate it right away, we will wait up to `15 seconds` before sending data.  
This will allow the destination to inform us about his current change vector and most of the time the data will get filtered at the source.  
On a stable system the steady state will have a `Spanning Tree` of replication channels, `n-1` of the fastest channels available that are doing the actual work and the rest are just sending heartbeats.  

