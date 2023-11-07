# Scaling Memcache at Facebook

[paper](https://research.facebook.com/publications/scaling-memcache-at-facebook/)

## Definitions

1. Incast congestion
    Incast congestion is a network congestion scenario that occurs when multiple clients send simultaneous requests to multiple servers, and all these requests arrive at the same time or in a short time frame. This sudden burst of traffic can overwhelm the network and the servers, leading to delays and packet loss.
    
    Example: Imagine you have a data center with multiple web servers that all need to retrieve data from a single database server. If, for some reason, all web servers decide to query the database server at the exact same moment, the incoming traffic can cause congestion and delays in the data center's network. This is because all web servers are effectively competing for network resources, leading to potential packet collisions, queuing delays, and network congestion. Incast congestion can significantly increase the latency for the requests, causing performance issues.

## Introduction

Social media site such as facebook encounters a lot of reads for a lot of users. It can go upto several billion reads per second.
Such a load can not be dependent on databases as it will be challanging to scale.

Also, huge data in databases demands redundancy for better consistency. Thus, it limits the availability. As such, these heavy-read
sites rely a lot on caches to make reads. Memcache is one such in-memory cache.

This paper discusses 4 points
1. evolution of Facebook’s memcached based architecture
2. enhancements to memcached (for performance and efficiency)
3. operability (ability to operate at scale)
4. characterization of production workloads

## Memcache Uses

Generally, memcache is used as 2 types:

1. Query cache: on read, check in cache, if not found, cache from db. On write, delete the cache
2. Generic cache: Put heavy computations in cache (such as results from machine learning)

### Problem statement

Memcached is distributed as single server binary, with no distributed support (i.e. no server-to-server cordination)
Hence, facebook needed to develop configuration, aggregation, and routing services to organize memcached instances 
into a distributed system

Facebook categorizes the problem at 3 levels:

1. Level 1: heavy workload and wide fan-out (due to huge horizontal scaling)
2. Level 2: Due to level 1, scaling memcache is required, which creates replication issues (cluster level)
3. Level 3: Consistent user experience across all the clusters (global level)

Across all the levels, design of memcache is constained by operational complexity and fault-tolerance. Hence:
1. Only changes that provides operational or user-facing benefits are accepted
2. It is okay to read stale data at the expense of preventing backend services from heavy loads.

## Solution 1: Handling latency and load

### 1.1 Scaling
Facebook uses several memcached servers to handle latency issues. However, each server only keeps partioned data (using consistent hashing).
This creates [incast congestion](#incast-congestion) issue since all the frontend webservers are requesting for several memcached servers at
the same time.
To address this, fb uses better memcache clients, which handles proxying, keeping active servers in request routing, and request-batching techniques.

### 1.2 Parallel execution
Suppose you were to fetch your profile page of facebook, which includes lot of data such as profile details, profile and cover pic, news feed.
and friends list. Concurrently fetching all the items may overlap some items which has already been fetched. For eg: Your news feed may contain details
of your friend, which is also contained while fetching your friend list.
In order to not fetch same items twice, facebook uses the strategy called DAG (Direct Acylic Graph). This graph organises the dependent items and hence:
1. independent items can be fetched in batches (hence parallelizing fetches)
2. dependent items can be fetched, leaving out already fetched data in step 1

Such as DAG in application code helps facebook to further reduce latencies by reducing and parallelizing queries.

### 1.3 Client-Server communication

Facebook uses `mcrouter` as a client proxy to communicate with `memcached` servers, instead of making memcached-to-memcached communication possible.
Reason is to leave communication complexities out of `memcached` and put it in a proxy, outside of it.

`mcrouter` uses 2 modes of communication: TCP and UDP. Since UDP is connectionless, if it is used for GET requests, it greatly improves the latencies.
The packet drops in UDP mode is considered as cache-miss. However, client will NOT try to invalidate the cache as it will overwhelm the cache for all the UDP miss.

