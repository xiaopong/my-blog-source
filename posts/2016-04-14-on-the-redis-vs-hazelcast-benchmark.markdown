---
title:  On the Redis vs Hazelcast benchmark
author: xp
---
I have read the [Redis vs Hazelcast benchmark](https://hazelcast.com/resources/benchmark-redis-vs-hazelcast/) with a lot of interest, as we use both cache frameworks in our projects for a few years now. However, we are still on older versions, namely v3.2 for Hazelcast and v2.8.x for Redis. We like both of them, a lot. Both have their strengths, and issues, as I wouldn't really call them weaknesses. Although we have heavily used Hazelcast cluster, but we haven't tested the new Redis cluster, so I would not be able to comment on the Redis cluster performance.

We do rely heavily on the near cache feature in Hazelcast, and we do know it is a very nice performance enhancer, but to see it outperform Redis by 500% is quite amazing.

After all these years with these two frameworks, we have learned that, as nice as they are, you really need to test them thoroughly for your own use cases. No generic benchmark could give you a definitive answer on which framework you should use, the benchmark should only serve as an indicator.

We have used Hazelcast to cache a lot of things, from string-based key/value to Java objects, to images (yes, we do use it to cache millions of thumbnail images). Here are what we like about Hazelcast:

- Native Java API. If you program in Java, nothing can beat its native Java API and data structures. It's so simple and natural to use.
- Well thought-out data structures. Hazelcast has a rich set of well thought-out data structures, and they are as easy to use as the Java library that programmers are familiar with.
- Out of the box cluster. Any programmer can have the cluster up and running in five minutes. What more can you expect?
- Near cache. This is really a performance enhancer, and with the right percentage of data in near cache, it can significantly reduce network overhead.
- Predicate. The predicates make complicated searches easy, and complex searches possible.
- On-heap and off-heap memory. The open source version only provides on-heap memory, but if you need to cache a huge amount of data, off-heap memory is the way to go. You have to pay for the enterprise version for that. However, we implemented our own version of off-heap memory management, since the API was pretty straightforward.

We manage millions of objects and images in Hazelcast, and it has been reliable, easy to use and performance was great. However, as nice as it is, you really need to be aware of its internal implementation. And one of the performance hindrance is the serialization and deserialization of the cached data. If you cache only simple data, it works great. Now, if you are caching Java objects, and especially complicated Java objects, you could quickly kill Hazelcast's performance by serializing and deserializing objects. One way to get around it is to break the object into smaller pieces, but by paying a cost in management hassle and network overhead. Especially, if you are using off-heap memory, then this serialization/deserialization problem is basically unavoidable.

Another problem that you must watch out for is how frequent your cached data get changed. If these are write-once, read-many data, then Hazelcast is great. That's why we even use it to cache thumbnail in image-heavy projects. Once loaded, thumbnail images are never changed. But in one case where we have a list of frequently used objects whose status and information change at a pretty fast pace, we quickly found that Hazelcast could not keep up with the changes, even in moderate workload. We broke the objects into smaller pieces, but at some point, coordination and management of so many small pieces of data made it not worth trying. At the end, we moved these data to Redis, the Redis' hashes solved the problem nicely, without performance penalty.

A third pet peeve of mine in Hazelcast is in adding new node to a live cluster. You have millions of objects in cache, you foresee that new workloads are coming in, so you thought it would be simple to just add a few more nodes to the cluster to spread the workload and increase the bandwidth, right? Wrong, adding a new node to a live cluster brings the whole cluster to its knee. When a new node joins the cluster, every node in the cluster suddenly becomes busy calculating what needs to synchronize, and how much, and how to do it. All CPU cores are peaked, all memory allocated fully used up, network ports are almost jammed, all requests to the cluster timed out. This kind of issue is certainly not specific to Hazelcast, as we ran into similar problem with [Ceph](http://ceph.com/), [Cassandra](http://cassandra.apache.org/) and other cluster framework with automatic data partitioning. But it is still very annoying.

Hazelcast certainly has its own share of issues, but the ones described above are why you always must have a combination of caching solutions, and can't rely on a single framework.

In the end, I just want to re-state again, if I haven't before, that overall, Hazelcast is very neat to use.
