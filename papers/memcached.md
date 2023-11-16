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

## Solution 1: Handling latency

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
The packet drops in UDP mode is considered as cache-miss. However, client will NOT try to invalidate the cache as it will overwhelm the cache for all the UDP miss. All the UPDATE/DELETE operations are still done in TCP mode for reliability.

`mcrouter` also contains [connection pooling](https://github.com/facebook/mcrouter/wiki/Features#connection-pooling) which opens a long, single connection with a memcache server.
This has great benefits, compared to letting each client open up new connection for each memcache fetch/set request. This greatly benefits the memory usage, which would've been consumed by TCP connection otherwise.

### 1.4 Incast congestion control

Memcache uses sliding window mechanism to control the number of concurrent requests that can be sent to the server. If client sends all the requests unchecked, it may overwhelm the server since lot of responses will be generated, causing incast congestion.

In a sliding window mechanism, a window is only allowed to send certain number of requests, based on the size of window. For ex, Let's say the initial window size is 3, and client wants to retrieve memcache keys from 1 to 10. In this case, the client will only send 3 requests to the server (key 1,2,3). Once the successful response of any of them is returned (for eg, response for key1 is returned), the next request (key4) will be sent.

This way, successful responses indicate that the sliding window has more available space for requests, hence grows in size. If responses are not received in a timely manner, it is considered as congestion and the size of window shrinks (as requests are pending to be completed, hence lesser available space for new requests).

If window size is smaller (eg 100), then large number of "queued" requests will exist on server. This means the client will spend more time waiting for requests to be dispatched to the server, thus increasing latency. On the contrary, if the window size is too large (eg. 500), then lot of requests will be simultaneously sent, causing potential incast congestion. Hence facebook uses a balance of between these two extremes (eg 300) to maintain the best ratio of congestion and traffic.

## Solution 2: Handling load

In terms of fb, the most load is generated by sending read queries directly to db (because of cache miss). The cache faces 2 kind of load issues which may
result in reads going to db:

1. Stale reads: This happens when concurrent writes in memcache gets reorder, causing an expected earlier update to go later, setting a stale value in cache.
2. Thundering herd: Happens when certain keys in cache experiences heavy writes (eg: celebrity page). In such cases, reads gets invalidated a lot, causing requests to go to db

### 2.1 Leases
Facebook uses `lease tokens` to solve this issue. As any key is fetched, it is provided with a lease token. The same token should be present during writes. If token is expired (due to some concurrent write for same key), the request is dropped. The client retries to get updated value.

Limiting the number of tokens issued per unit time also fixes herd issue. For eg: fb grants cache token once every 10 seconds. If any read happens for a token during heavy write, it is told to retry. Since writes are usually milli-seconds long, the next read usually gets the updated data (instead of invalidated)

### 2.2 Pooling

Memcache can be further partitioned based on the key's access patterns. For instance, we can have following categories:
1. Frequently accessed keys having inexpensive cache-miss, can go to __smaller pool__
2. Less frequently accessed keys having expensive cache-miss, can go to __larger pool__
3. All the other keys can be routed to a default pool of memcached, which acts as a general purpose caching layer.

#### 2.2.1 Partitioning vs Replication

Fb prefers to replicate the data in memcache, instead of partitioning, in certain cases such as:
1. App fetches large number of keys in single request
2. Data set is so small that it can be fit in one or two memcache servers
3. Request rate is higher than single server can manage

The reason for point 1 can be understood with an example. Consider a memcached server, which has 100 keys and can handle 500k/s writes.
Suddenly, requirement grows to 1M/s writes. Assuming that the application sends GET for all of 100 keys in a single request, and that we have created a new
memcached server, we have 2 options:
    
    __option 1__: We partition the data to have 50 keys per memcache instance.
    In this scenario, both memcache servers will experience 1M/s hits for 50 keys each.

    __option 2__: We replicate all 100 keys to both servers. In this case, each server only needs to handle 500K/s requests.
    
    
We know the latency difference in fetching single key vs multiple keys at memcache server level is a matter of microseconds, the latency of overall 1M request will be very less in option2.

