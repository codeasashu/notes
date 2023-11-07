# Scaling Memcache at Facebook

[paper](https://research.facebook.com/publications/scaling-memcache-at-facebook/)

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

