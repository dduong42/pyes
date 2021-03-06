.. _es-guide-reference-index-modules-store:

=====
Store
=====

The store module allow you to control how index data is stored. It's important to note that the storage is considered temporal only for the shared gateway (non **local**, which is the default).


The index can either be stored in-memory (no persistence) or on-disk (the default). In-memory indices provide better performance at the cost of limiting the index size to the amount of available physical memory.


When using a local gateway (the default), file system storage with *no* in memory storage is required to maintain index consistency. This is required since the local gateway constructs its state from the local index state of each node. When using a shared gateway (like NFS or S3), the index can be safely stored in memory (with replicas).


Another important aspect of memory based storage is the fact that ElasticSearch supports storing the index in memory *outside of the JVM heap space* using the "Memory" (see below) storage type. It translates to the fact that there is no need for extra large JVM heaps (with their own consequences) for storing the index in memory.


Store Level Compression
=======================

(0.19.5 and above).


In the mapping, one can configure the **_source** field to be compressed. The problem with it is the fact that small documents don't end up compressing well, as several documents compressed in a single compression "block" will provide a considerable better compression ratio. This version introduces the ability to compress stored fieds using the **index.store.compress.stored** setting, as well as term vector using the **index.store.compress.tv** setting.


The settings can be set on the index level, and are dynamic, allowing to change them using the index update settings API. elasticsearch can handle mixed stored / non stored cases. This allows, for example, to enable compression at a later stage in the index lifecycle, and optimize the index to make use of it (generating new segmetns that use compression).


Best compression, compared to _source level compression, will mainly happen when indexing smaller documents (less than 64k). The price on the other hand is the fact that for each doc returned, a block will need to be decompressed (its fast though) in order to extract the document data.


Store Level Throttling
======================

(0.19.5 and above).


The way Lucene, the IR library elasticsearch uses under the covers, works is by creating immutable segments (up to deletes) and constantly merging them (the merge policy settings allow to control how those merges happen). The merge process happen in an asyncronous manner without affecting the indexing / search speed. The problem though, especially on systems with low IO, is that the merge process can be expensive and affect search / index operation simply by the fact that the box is now taxed with more IO happening.


The store module allows to have throttling configured for merges (or all) either on the node level, or on the index level. The node level throttling will make sure that out of all the shards allocated on that node, the merge process won't pass the specific setting bytes per second. It can be set by setting **indices.store.throttle.type** to **merge**, and setting **indices.store.throttle.max_bytes_per_sec** to something like **5mb**. The node level settings can be changed dynamically using the cluster update settings API.


If specific index level configuration is needed, regardless of the node level settings, it can be set as well using the **index.store.throttle.type**, and **index.store.throttle.max_bytes_per_sec**. The default value for the type is **node**, meaning it will throttle based on the node level settings and participate in the global throttling happening. Both settings can be set using the index udpate settings API dynamically.


The following sections lists all the different storage types supported.


File System
===========

File system based storage is the default storage used. There are different implementations or storage types. The best one for the operating environment will be automatically chosen: **mmapfs** on Solaris/Windows 64bit, **simplefs** on Windows 32bit, and **niofs** for the rest.


The following are the different file system based storage types:


Simple FS
---------

The **simplefs** type is a straightforward implementation of file system storage (maps to Lucene **SimpleFsDirectory**) using a random access file. This implementation has poor concurrent performance (multiple threads will bottleneck). Its usually better to use the **niofs** when you need index persistence.


NIO FS
------

The **niofs** type stores the shard index on the file system (maps to Lucene **NIOFSDirectory**) using NIO. It allows multiple threads to read from the same file concurrently. It is not recommended on Windows because of a bug in the SUN Java implementation.


MMap FS
-------

The **mmapfs** type stores the shard index on the file system (maps to Lucene **MMapDirectory**) by mapping a file into memory (mmap). Memory mapping uses up a portion of the virtual memory address space in your process equal to the size of the file being mapped.  Before using this class, be sure your have plenty of virtual address space.


Memory
======

The **memory** type stores the index in main memory with the following configuration options:


There are also *node* level settings that control the caching of buffers (important when using direct buffers):


====================================  ===============================================================================
 Setting                               Description                                                                   
====================================  ===============================================================================
**cache.memory.direct**               Should the memory be allocated outside of the JVM heap. Defaults to **true**.  
**cache.memory.small_buffer_size**    The small buffer size, defaults to **1kb**.                                    
**cache.memory.large_buffer_size**    The large buffer size, defaults to **1mb**.                                    
**cache.memory.small_cache_size**     The small buffer cache size, defaults to **10mb**.                             
**cache.memory.large_cache_size**     The large buffer cache size, defaults to **500mb**.                            
====================================  ===============================================================================
