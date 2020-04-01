---
title:  Unscientific comparison of Hazelcast memory model - on-heap and off-heap
author: xp
---
Recently, I was working on a project that requires to cache a lot of data for fast access, including images and thumbnails, as well as pure Java objects. Therefore, we set out to search for a system that can handle the workload, in a distributed manner. As the project is developed using Java, a Java solution is prefered.

One of the options we looked at is [Hazelcast](http://www.hazelcast.org). Hazelcast labels itself as the in-memory data grid. Whatever that means, it means an in-memory cache pool to us. What is nice about Hazelcast is that it does not have any external dependency (besides Log4J), and clustering feature works nicely out of the box. The API is extremely simple. It took us a total of five minutes to get a cluster of three machines up for testing.

Anyone who had worked with large data set with Java will run into the Java garbage collection problem sooner or later, in which the Java virtual machine will stop the world to clean up and recollect some unused memory. When this happens, it is extremely annoying. Each GC cycle could take tens of seconds, or in some cases, as when we worked with Cassandra, it even took up to two minutes. To get around this problem, vendors are offering solutions with different names, be it big memory, elastic memory, or what not, but the end result is to make use of off-heap memory to minimize the impact of GC cycle on the system.

Hazelcast, the company that develops and supports Hazelcast, the open source framework, also has a commercial offering for off-heap memory storage, called elastic memory storage, in their enterprise version. As the project did not have a big budget, yet we still need a solution, I set out to put together an off-heap storage implementation for the open source version of Hazelcast.

In this post, I'll show some test results, comparing the heap storage model and off-heap storage model of Hazelcast. Benchmark is a bitch, when you try to get a fair comparison of apple to apple. At first, it looks like a simple comparison. You just swap in a new memory management model, and that should be it. But when you get down to the details, comparison becomes quite tricky.

In this test, I tried to compare caching of data in the form of byte array (e.g. images), and Java objects, both for on-heap and off-heap memory storage.

Note that this benchmark is going to be very unscientific. This is just for me to get a rough idea of the performance of my own implementation of off-heap storage. Therefore, you should take any data presented in this test result with a big grain of salt.

Let's get down to the Java source code first. Here is the main class to start up the test:

```Java
package ca.renzhi.hazelcast.test;

public class HazelcastMemTest
{
    public static void printHelp()
    {
        System.out.println("");
        System.out.println("ca.renzhi.hazelcast.test.HazelcastMemTest count [maxMem]");
        System.out.println("");
    }

    public static void testObjectCache(String[] args)
    {
        ObjectCacheTest t = null;
        int count = 0; 
        int maxMem = 0;
        if (args.length == 2)
        {
            count = Integer.parseInt(args[0]);
            maxMem = Integer.parseInt(args[1]);
            t = new ObjectCacheTest(maxMem, count);
        }
        else
        {
            count = Integer.parseInt(args[0]);
            t = new ObjectCacheTest(count);
        }

        t.runTest();
        t.shutdown();        
    }

    public static void testByteArrayCache(String[] args)
    {
        ByteArrayCacheTest t = null;
        int count = 0; 
        int maxMem = 0;
        if (args.length == 2)
        {
            count = Integer.parseInt(args[0]);
            maxMem = Integer.parseInt(args[1]);
            t = new ByteArrayCacheTest(maxMem, count);
        }
        else
        {
            count = Integer.parseInt(args[0]);
            t = new ByteArrayCacheTest(count);
        }

        t.runTest();
        t.shutdown();        
    }
    public static void main(String[] args)
    {
        if (args.length < 1)
        {
            printHelp();
            System.exit(0);
        }

        //testObjectCache(args);
        testByteArrayCache(args);
    }

}
```

The main test driver is pretty simple, it takes a number of objects to generate for testing from the command line. And for testing off-heap memory, a second argument for the size of off-heap memory is provided.

And now, the main test for byte array data cache:

```Java
package ca.renzhi.hazelcast.test;

import java.util.Iterator;
import java.util.Map;
import java.util.Random;

import com.hazelcast.config.Config;
import com.hazelcast.config.InMemoryFormat;
import com.hazelcast.config.MapConfig;
import com.hazelcast.config.MaxSizeConfig;
import com.hazelcast.config.MapConfig.EvictionPolicy;
import com.hazelcast.core.Hazelcast;
import com.hazelcast.core.HazelcastInstance;

public class ByteArrayCacheTest
{
    private Random rand = new Random();
    private int count = 0;
    private int maxMem = 0;
    private HazelcastInstance hz = null;
    private Map&lt;String, byte[]&gt; map = null;

    public ByteArrayCacheTest(int maxMem, int count)
    {
        this.maxMem = maxMem;
        this.count = count;

        System.out.println("Unsafe mode");

        Config cfg = new Config();
        cfg.setProperty("hazelcast.elastic.memory.enabled", "true");
        cfg.setProperty("hazelcast.elastic.memory.total.size", this.maxMem + "G");
        cfg.setProperty("hazelcast.elastic.memory.chunk.size", "1K"); // Default is 1K
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

        hz = Hazelcast.newHazelcastInstance(cfg);
        map = hz.getMap("TestCache");
    }

    public ByteArrayCacheTest(int count)
    {
        this.count = count;

        System.out.println("Heap mode");

        Config cfg = new Config();
        MapConfig mapConfig = new MapConfig();
        mapConfig.setBackupCount(0);
        mapConfig.setEvictionPolicy(EvictionPolicy.LRU);
        mapConfig.setEvictionPercentage(2);
        MaxSizeConfig mxSizeCfg = new MaxSizeConfig(Constants.MAX_SIZE, MaxSizeConfig.MaxSizePolicy.PER_NODE);
        mapConfig.setMaxSizeConfig(mxSizeCfg);
        mapConfig.setName("TestCache");

        cfg.addMapConfig(mapConfig);

        hz = Hazelcast.newHazelcastInstance(cfg);
        map = hz.getMap("TestCache");
    }

    private long initDataForHeap()
    {
        long start = System.currentTimeMillis();
        for (int i = 0; i < count; i++)
        {
            byte[] ba = new byte[10240];
            rand.nextBytes(ba);
            map.put(Integer.toString(i), ba);
        }
        long insertDuration = System.currentTimeMillis() - start;
        return insertDuration;
    }

    private long initDataForUnsafe()
    {
        long start = System.currentTimeMillis();
        byte[] ba = new byte[10240];
        for (int i = 0; < count; i++)         
        {             
            rand.nextBytes(ba); 
            map.put(Integer.toString(i), ba); 
        }         
        long insertDuration = System.currentTimeMillis() - start;
         return insertDuration;         
    }
     public void runTest()     
     {
         System.out.println("Running test....");
         long insertDuration;
         if (maxMem > 0)
            insertDuration = initDataForUnsafe();
        else
            insertDuration = initDataForHeap();
        int size = map.size();
        long start = System.currentTimeMillis();
        Iterator it = map.keySet().iterator();
        while (it.hasNext())
        {
            String key = it.next();
            byte[] ba = map.get(key);
        }
        long readDuration = System.currentTimeMillis() - start;

        System.out.println("count=" + count + "; size=" + size + "; write=" + insertDuration + "; read=" + readDuration);
    }

    public void shutdown()
    {
        hz.shutdown();
    }

}
```

The code is very simple. It initializes a hash map for on-heap or off-heap memory storage, depending on the case, then generate byte arrays of 10KB each, fill it with random data, and store it in the hash map. Then it tries to read them back. During inserting and retrieval, we just log how much time it takes to perform the operation.

The max size of the map is set to `Constants.MAX_SIZE`, which is 5 millions entries.

At first look, it seems that the off-heap test code is unfair, as the on-heap test code will allocate byte array memory for each object, while the off-heap test code just re-uses the same memory buffer. This is actually not really the case, as for on-heap storage, you will need to allocate memory for the byte array once, and rely on the cleverness of the Java VM to manage the memory properly. But in the off-heap case, the data needs to be copied from the on-heap memory buffer to the off-heap memory. Therefore, in real production code, you would have to do the same thing anyway. And in production code, with some little trick, you can pre-allocate, and re-use it every time, a memory buffer to store the intermediate data before putting it into the off-heap storage.

And now, the main test for Java object cache:

```Java
package ca.renzhi.hazelcast.test;

import java.util.Iterator;
import java.util.Map;

import com.hazelcast.config.Config;
import com.hazelcast.config.InMemoryFormat;
import com.hazelcast.config.MapConfig;
import com.hazelcast.config.MaxSizeConfig;
import com.hazelcast.config.MapConfig.EvictionPolicy;
import com.hazelcast.core.Hazelcast;
import com.hazelcast.core.HazelcastInstance;

public class ObjectCacheTest
{
    private int count = 0;
    private int maxMem = 0;
    private HazelcastInstance hz = null;
    private Map<String, Customer> map = null;

    public ObjectCacheTest(int maxMem, int count)
    {
        this.maxMem = maxMem;
        this.count = count;

        System.out.println("Unsafe mode");

        Config cfg = new Config();
        cfg.setProperty("hazelcast.elastic.memory.enabled", "true");
        cfg.setProperty("hazelcast.elastic.memory.total.size", this.maxMem + "G");
        cfg.setProperty("hazelcast.elastic.memory.chunk.size", "1K"); // Default is 1K
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

        hz = Hazelcast.newHazelcastInstance(cfg);
        map = hz.getMap("TestCache");
    }

    public ObjectCacheTest(int count)
    {
        this.count = count;

        System.out.println("Heap mode");

        Config cfg = new Config();
        MapConfig mapConfig = new MapConfig();
        mapConfig.setBackupCount(0);
        mapConfig.setEvictionPolicy(EvictionPolicy.LRU);
        mapConfig.setEvictionPercentage(2);
        MaxSizeConfig mxSizeCfg = new MaxSizeConfig(Constants.MAX_SIZE, MaxSizeConfig.MaxSizePolicy.PER_NODE);
        mapConfig.setMaxSizeConfig(mxSizeCfg);
        mapConfig.setName("TestCache");

        cfg.addMapConfig(mapConfig);

        hz = Hazelcast.newHazelcastInstance(cfg);
        map = hz.getMap("TestCache");
    }

    public void runTest()
    {
        System.out.println("Running test....");
        long start = System.currentTimeMillis();
        for (int i = 0; i < count; i++)
        {
            String name = "Customer " + i;
            String addr = i + " Park Avenue, #" + i;
            String phone = "123456789";
            Customer cust = new Customer(i, name, addr, phone);
            map.put(Integer.toString(i), cust);
        }
        long insertDuration = System.currentTimeMillis() - start;
        int size = map.size();
        start = System.currentTimeMillis();
        Iterator it = map.keySet().iterator();
        while (it.hasNext())
        {
            String key = it.next();
            Customer cust = map.get(key);
        }
        long readDuration = System.currentTimeMillis() - start;

        System.out.println("count=" + count + "; size=" + size + "; write=" + insertDuration + "; read=" + readDuration);
    }

    public void shutdown()
    {
        hz.shutdown();
    }

}
```

This test code is very similar to the byte array cache test code, the difference is to store a Java object in the hash map instead of byte array. And here is the simple Java class used for testing.

```Java
package ca.renzhi.hazelcast.test;

import java.io.Serializable;

public class Customer implements Serializable
{
    private static final long serialVersionUID = -665013352562905224L;
    private int id;
    private String name;
    private String address;
    private String phone;

    public Customer(int id, String name, String addr, String phone)
    {
        this.id = id;
        this.name = name;
        this.address = addr;
        this.phone = phone;
    }

    public int getId()
    {
        return id;
    }

    public String getName()
    {
        return name;
    }

    public String getAddress()
    {
        return address;
    }

    public String getPhone()
    {
        return phone;
    }
}
```

A little bit more patience, and we will see the test results. Here is a last piece of bash script to start up the test:

```
#!/bin/bash

function show_help {
    echo ""
    echo "test1.sh count mode"
    echo ""
    echo "where"
    echo "   count         Integer, number of objects to generate"
    echo "   mode          Memory mode, unsafe or heap"
    echo ""
}

if [ $# -ne 2 ]
then
    show_help
    exit 0 
fi

count=$1
mode=$2

curr_dir=`pwd`
CLASSPATH=$CLASSPATH:$curr_dir/libs/hazelcast/hazelcast-3.2.jar:$curr_dir/libs/log4j/log4j-core-2.0-beta9.jar:$curr_dir/libs/log4j/log4j-api-2.0-beta9.jar:$curr_dir/bin:

if [ $mode == "unsafe" ]
then
    java -Xms256m -Xmx4096m -XX:MaxPermSize=256m -XX:MaxDirectMemorySize=10G -cp $CLASSPATH ca.renzhi.hazelcast.test.HazelcastMemTest $count 10
else
    java -Xms256m -Xmx14336m -XX:MaxPermSize=256m -cp $CLASSPATH ca.renzhi.hazelcast.test.HazelcastMemTest $count
fi
```

This script needs a bit of explanation. For the test of different memory models, we try to allocate the same amount of memory for both cases. For off-heap storage, the memory is divided into heap memory, which is managed by the JVM, and the direct access off-heap memory, which is managed by my implementation of the "elastic memory" storage.

You might be asking, since you are managing your own off-heap memory storage already, why do you still allocate so much memory for Java heap? Why don't you just put more into the off-heap pool?

Well, first of all, when we generate test data, there will be quite a bit of intermediate Java objects that live in the heap memory, before we get a chance to put them into the off-heap storage. Same when we read them back.

Secondly, you really need to understand the data structure management model inside Hazelcast. Hazelcast provides a set of built-in data structures, such as map, distributed map, queue, list, etc. When we are dealing with these data structures, such as a map, which is a key/value store, the logic of the implementation is managed by Hazelcast. And the management of the implementation logic includes the management of the key set, eviction policy, etc. The off-heap storage is used only for storing data contents. Therefore, for the key/value pair, the key part is managed in heap memory, while the value part is stored in off-heap memory. If you have a lot of entries, you need to allocate enough heap memory for JVM to handle to workload.

Therefore, depending on your use case, even if you use off-heap storage, such as the enterprise version of Hazelcast, you still need to figure out how much heap memory you need to allocate for your application.

Ok, enough of rambling, let's get to the test results. For byte array caching, I run tests for 100K, 250K, 500K, 750K, 800K and one million entries. And here are the results for off-heap storage model:

```
count=100000; size=100000; write=9600; read=2733
count=250000; size=250000; write=21497; read=6527
count=500000; size=500000; write=43982; read=12600
count=750000; size=750000; write=64904; read=19569
count=800000; size=800000; write=70220; read=20088
count=1000000; size=1000000; write=89807; read=25993
```

The first column is the count value provided on the command line, the second column is the size of the map after we entered the test data. The third column is the time to write that amount of test data into map, and the fourth column is the time to read them all back. The unit of both times is in milliseconds.

And now, let's see the results for on-heap storage model:

```
count=100000; size=100000; write=8281; read=2459
count=250000; size=250000; write=22448; read=6296
count=500000; size=500000; write=47521; read=10346
count=750000; size=750000; write=64818; read=16513
count=800000; size=800000; write=77322; read=25430
```

From the results above, we can see that when the number of entries is small, there's no advantage in using off-heap storage. The advantage starts to show when we reach 500K entries. And for on-heap storage model, we didn't get a chance to complete the test for one million entries before we ran into this error:

```
Running test....
Apr 22, 2014 2:59:05 PM com.hazelcast.partition.InternalPartitionService
INFO: [192.168.122.1]:5701 [dev] [3.2] Initializing cluster partition table first arrangement...
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
    at ca.renzhi.hazelcast.test.ByteArrayCacheTest.initDataForHeap(ByteArrayCacheTest.java:84)
    at ca.renzhi.hazelcast.test.ByteArrayCacheTest.runTest(ByteArrayCacheTest.java:112)
    at ca.renzhi.hazelcast.test.HazelcastMemTest.testByteArrayCache(HazelcastMemTest.java:58)
    at ca.renzhi.hazelcast.test.HazelcastMemTest.main(HazelcastMemTest.java:70)
Exception in thread "hz._hzInstance_1_dev.HealthMonitor" java.lang.OutOfMemoryError: Java heap space
```

Looking at the trend, we can, kind of, extrapolate what it would be like when the number of entries get even larger, and when JVM hits worse GC cycles. I don't have machines with 64GB of RAM to test with, but obviously in this case, it seems that off-heap storage can manage memory usage better for large number of entries.

Before we go further, let's look at the test results for caching Java objects now. For Java objects caching, I ran tests for 100K, 500K, 1 million, 2 million, and 5 million entries, since each Java object is much smaller than the 10KB data we used in previous test. The following results are for off-heap storage:

```
count=100000; size=100000; write=4336; read=4475
count=500000; size=500000; write=16060; read=19291
count=1000000; size=1000000; write=29587; read=36663
count=2000000; size=2000000; write=57830; read=73778
count=5000000; size=4777740; write=147139; read=178364
```

Ok, what's wrong with last line? We set to create 5 million entries, and after we entered 5 million entries into the map, the map size showed only 4,777,740 entries. Well, in the code above, we have set the max size for the hash map to be 5 million entries, and when we hit the limit, the eviction policy kicked in. Using the LRU (least recently used) algorithm, some entries (2% as set in our code) have been evicted. Nevertheless, I was a bit surprised how early the eviction process kicked in.

Therefore, the write time in the last line is for writing 5 million entries, and the read time is for reading only 4,777,740 entries.

In the test above, there's another thing that we want to plan carefully, when we use off-heap memory for caching Java objects. We set the chunk size to 1KB in our configuration, but the Java objects that we put in the cache is significantly smaller than 1KB. Therefore, in this case, we have wasted quite a bit of memory. We can certainly set the chunk size to a smaller value, but in the cost of more management overhead.

And here are the results for on-heap storage:

```
count=100000; size=100000; write=4084; read=3998
count=500000; size=500000; write=14763; read=19074
count=1000000; size=1000000; write=29877; read=36636
count=2000000; size=2000000; write=59254; read=73251
count=5000000; size=4712401; write=141923; read=167874
```

Ah, as we can see, storing Java objects in off-heap memory is slower. Well, this is, somewhat, not a surprise, as the Java object must be serialized before storing into off-heap memory, and must be deserialized when it is read back. We used the default serialization mechanism provided by Java in this test, which might not be the most efficient way. It would be interesting to compare with other serialization mechanisms.

It would be also interesting to see how Hazelcast behaves when we reach tens of millions, or even hundreds of millions, of entries. Again, I don't have the hardware necessary to perform the test.

Like I said from the beginning, this test is far from scientific. I did the test to get an idea of how well my own implementation of off-heap storage for Hazelcast performs. The implementation in the Hazelcast enterprise version might have a more efficient algorithm, and the test results might be completely different. Besides, the enterprise version also comes with a very nice management console (ok, that is just my impression from the published documentation and screenshots, I have no first-hand experience).

Another lesson learned is, using off-heap storage is better for storing objects that are larger than 1KB, or whatever chunk size you decide to use. Obviously, you need to find a balance between management overhead and memory wasting.

At the end, Hazelcast is a very nice framework to use in your project. Simple, elegant.
