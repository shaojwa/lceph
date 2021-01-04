#### 线程结构
1. 一个DSE进程拥有的线程数和这个节点上对应的pool数量有关系。
1. 每一个pool会在一个节点上关联9个instance，也就会会一个DSE进程中创建8个与这个pool相关的线程。
1. 每个engine线程对应一个engine实例，一个engine实例中包含一个DCache实例，一个ROW实例，和一个dedup实例。
1. DCache的实例中包括pool_id和engine_id，两者组合成一个实例id。

```
1. 一个节点上一个ceph-dse进程
2. ceph-dse进程中，一个ceph-dse会处理多个pool，为每个pool分配8个engine实例。
3. 每个engine实例，会有一个dcache实例，一个row实例，一个dedup实例。
4. 每个engine实例，dcache模块会映射到一个协程processor(对应一个操作系统线程)
5. 一个dcache实例对应，多个task，这些task可以在一个processor上处理，也可以在做个processor上处理。
```
现在的问题是，原来的hash是一个dcache-instance会对应一个processor，所以映射是：
```
object_id->bucket_id->engine_id->processor_id
```
如果一个dcache-instance可以在多个processor上跑，那么如何hash?   
少掉中间的engine_id的分层，所有的engine_id都对应多个processor。  
```
object_id->bucket_id->processor_id
```
问题是，一个object_id会对应一个bucket_id，然后对应到一个processor_id，一个对象最终在同一个processor_id上处理就可以。

资源分配：
```
1. 一个节点64个核，给dse32个核，32个核分割dcache，row，dedup
```