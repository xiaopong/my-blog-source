---
title:  How to implement off-heap memory storage for Hazelcast
author: xp
---
In the last post, I did a small comparison of on-heap and off-heap memory storage of Hazelcast for byte arrays and plain Java objects. The benchmark methodology was far from being scientific, as it was meant for my own understanding of the Hazelcast internals and the effectiveness of my own implementation of off-heap memory management mechanism.

Hazelcast is a very elegant framework, and with an off-heap storage to get around the Java GC cycles, it is an excellent framework for distributed big data caching.

In this post, I'm going to provide some instructions on how to create your own off-heap storage implementation for Hazelcast. It is assumed here that you are familiar with Java programming, as well as the concept of memory management overall. If you have forgotten what you have learned from your CS classes, you might want to dust out some text books now. It is also assumed that you are familiar with how to allocate off-heap memory in Java. There are already a lot of information and instructions regarding this domain on the internet, and this is beyond the scope of this post anyway.

This post is not intended to provide you a fully implemented off-heap storage, it is meant to help you get familiar with the necessary preparation work, should you need to implement your own version. However, if you have the budget, getting the Hazelcast enterprise license is probably a safer route to take.

Let's remember what I have described in the last post, regarding the Hazelcast internal logic. Hazelcast provides a set of built-in data structures, such as map, queue, list, etc. When we are dealing with these data structures, such as a map, which is a key/value store, the logic of the implementation is managed by Hazelcast. And the management of the implementation logic includes the management of the key set, eviction policy, etc. The off-heap storage, when you are using off-heap storage, is used only for storing data contents. Therefore, for the key/value pair, the key part is managed in heap memory, while the value part is stored in off-heap memory. This clean distinction between the data structure logic from the data storage allows you to swap the storage implementation easily.

Hazelcast is meant to run as a cluster. At start up, each node in the cluster needs to initialize its own environment, including its memory storage model, before joining the cluster. And this is where we can plug in our own initialization code to swap in our own off-heap storage.

First of all, you need to create a node initializer class that implements the `com.hazelcast.instance.NodeInitializer` interface:

```Java
public interface NodeInitializer {

    void beforeInitialize(Node node);

    void printNodeInfo(Node node);

    void afterInitialize(Node node);

    SecurityContext getSecurityContext();

    Storage getOffHeapStorage();

    void destroy();
}
```

The three important methods to implement are `void beforeInitialize(Node node)`, `void afterInitialize(Node node)` and `Storage getOffHeapStorage()`. These are our entry points to plug in our own implementation. The method `beforeInitialize()` is where you can initialize your off-heap memory (and probably a bunch of other preparation works), and the method `getOffHeapStorage()` is when Hazelcast wants to get a handle of your off-heap memory for reading/writing data.

The open source version of Hazelcast has a `DefaultNodeInitializer` already, so it is probably better to just extend it and override only the necessary methods, such as:

```Java
public class RenzhiNodeInitializer extends DefaultNodeInitializer
{
    @Override
    public void beforeInitialize(Node node)
    {
        // Implementation code here
    }

    @Override
    public Storage getOffHeapStorage()
    {
        // Implementation code here
    }
}
```

One last thing before we go further is that you need to create a file

```
/META-INF/services/NodeInitializer
```

and put your own node initializer class name in that file, such as:

```
ca.renzhi.hazelcast.RenzhiNodeInitializer
```

Ok, so that gets us a foot in the door. Next thing to do is to get familiar with some of the configuration parameters when we use off-heap memory in Hazelcast. From the source code in my last post, we can see the following configuration:

```Java
Config cfg = new Config();
cfg.setProperty("hazelcast.elastic.memory.enabled", "true");
cfg.setProperty("hazelcast.elastic.memory.total.size", "10G");
cfg.setProperty("hazelcast.elastic.memory.chunk.size", "1K");
cfg.setProperty("hazelcast.elastic.memory.unsafe.enabled", "true");

MapConfig mapConfig = new MapConfig();
mapConfig.setInMemoryFormat(InMemoryFormat.OFFHEAP);
mapConfig.setBackupCount(0);
mapConfig.setEvictionPolicy(EvictionPolicy.LRU);
mapConfig.setEvictionPercentage(2);
MaxSizeConfig mxSizeCfg = new MaxSizeConfig(Constants.MAX_SIZE, MaxSizeConfig.MaxSizePolicy.PER_NODE);
mapConfig.setMaxSizeConfig(mxSizeCfg);
mapConfig.setName("TestCache");

cfg.addMapConfig(mapConfig);
HazelcastInstance hz = Hazelcast.newHazelcastInstance(cfg);
Map<String, Customer> map = hz.getMap("TestCache");
```

There are quite a few configuration parameters here, but the most important to understand are `hazelcast.elastic.memory.total.size`, `hazelcast.elastic.memory.chunk.size` and `InMemoryFormat.OFFHEAP`.

By just enabling "elastic memory" model does not mean that Hazelcast will use off-heap memory, you really need to set `InMemoryFormat.OFFHEAP` to tell Hazelcast that we really want to do that.

Of the other two parameters, only the total size parameter is actually important. This is the size of memory that we want to allocate during the initialization phase at startup. Remember the method `beforeInitialize()`? That's the place where we want to initialize the memory.

The chunk size parameter is useful to the extent of your own off-heap memory implementation. Depending on how you are going to implement it, you probably don't even need this parameter. But this parameter gives you a hint on how Hazelcast assumes you are going to implement it. Now, this is where your knowledge of memory management, algorithms etc will come into play. You might want to brush up on these concepts from your CS text books.

In computer science text books, system memory is presented as a big, continuous array. But internally, it is divided into small segments of equal size, called pages. Hazelcast uses the same concept. At startup, you allocate a big portion of system memory, then you divide it into small segments of equal size, called chunks. That's why the Hazelcast documentation insists on the fact that total memory size must be a multiple of the chunk size.

Whether it's called page or chunk, they are the same concept. The basic idea is to divide the memory into small slots where you can write your data. Depending on the chunk size you set, your object might, or might not, fit into one slot. If your object is smaller than the chunk size, you waste the memory portion that is not filled with data in that chunk. If the object is larger than the chunk size, then it has to be cut into pieces that fit into the chunks, and will take more than one chunk to store the data.

So, the idea is to divide your memory into small chunks, then write your data into these chunk slots. An object might be written into more than one slot, which might not be adjacent to each other. So when you read back the object, you need to figure out where to read from, in what order, and re-assemble the chunks into the original object.

There you have it. That's the basic idea behind the implementation of an off-heap storage for Hazelcast. How you implement it is up to you now.

Now, let's look at what you need to create your off-heap storage class. From the signature of the method `getOffHeapStorage()`, you are expected to return an object of type `Storage`, which is defined as:

```Java
public interface Storage {

    REF put(int hash, Data data);

    Data get(int hash, REF ref);

    void remove(int hash, REF ref);

    void destroy();
}
```

Therefore, the off-heap storage class must implement this interface, such as:

```Java
class RenzhiOffHeapStorage implements Storage
{
    // Implementation code
}
```

The off-heap storage is, basically, a key/value store. The `put` method will be called when an object is written into the storage. And the `get` method is called when an object is read from the storage. The definition of the `Storage` interface is straightforward enough that we don't really need explanation, especially after we have explained the concept of memory management.

Now, putting all these together, the parameter `InMemoryFormat.OFFHEAP` tells Hazelcast that we really want to use off-heap memory for storage, and it will call the method `getOffHeapStorage()` to get the handle of your off-heap memory. At this time, your `NodeInitializer` has already done all the preparation work, and has already plugged in your own implementation.

That's it, you should be ready to implement your own version now. But as I said before, if you have the budget, you probably would want to get the commercial license, with technical support. However, if you need some special, custom implementation, you can contact me. For a small fee, I can handle the work ;)
