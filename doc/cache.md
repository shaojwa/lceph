### SPANNING TREE

* 如果/usr/bin/vi 在缓存里，那么/usr/bin/, /usr/, /也在缓存里，包括inode，directory对象，以及dentry。

### 权威

* 维护关于什么节点缓存什么inode的列表。
* 每一个副本会有一个nonce用来消除多个副本之间的歧义，即一个缓存项可能在多个mds上缓存，nonce用在这里。
    map<int, int> replicas; 
* cached_by 集合存储所有缓存某个对象的所有mds节点，有时候也包括之前缓存而现在不缓存的某个MDS。
    * 权威节点包括所有的缓存该对象的MDS
    * 缓存过期了之后，过期通知一定会发送到权威MDS。（非常重要，有其他节点在用会导致权威MDS节点pin主这个元数据，以避免被trim掉）
* 每一个元数据对象都有一个权威标记位，用来标记这个元数据对象是权威的，还只是一个副本。

### 副本 NONCE

* 每一个副本对象都维护一个nonce值，这个值时这个副本被创建时，权威节点分配的。
* nonce用来区别是哪个副本过期，有时候同一个对象的多个副本同时被创建。

### 子树分区

* 如果/分配给mds0，/usr分配给mds1，那么/usr的inode由mds0管理，因为这个inode信息是/目录的一部分。
* 每一个CDir对象都有一个dir_auth元数，如果这个字段的值时AUTH_PARENT表示，CDir的权威节点和这个目录的inode的权威节点相同。
* 如果dir_auth指向其他MDS，那么表示一棵新的子树。CDir是一棵子树根，当且仅当dir_auth指向一个MDS，且不是AUTH_PARENT。

一棵冻结的树的根dir会auth_pin它的inode，当且仅当这个目录不是子树的根。如果是子树的根，那么dir和inode是属于不同的mds的。
dir属于这棵子树所对应的权威mds，而inode属于父目录所在子树的权威mds。所以，如果一个目录，不是一个子树的根（也就是 dir_auth=AUTH_PARENT)，那么这个目录的parnet inode会被 auth_pin住。这避免一个父目录被同时锁住，以及一定程度上和元数据迁移相关的实现上的复杂度。

### 针对子树导出的缓存过期

MDS会因为子树的导出而收到其他MDS发来的缓存过期的消息，可能延后处理，也可能马上处理，这取决于发送者和接受者的状态。导入子树的MDS总是等到导出完成之后才进行处理，因为导入可能会失败。而对于导出的MDS来说，只有在过期的MDS不知道子树已经导出，或者导出的MDS已经不再是权威节点时，导出的MDS才进行过期处理。

因为MDS会得到导出通知，所以，这是安全的，不管是下面哪种情况：(1) 过期的MDS知道导出事件，而且也发送消息给相关的MDS。（2）过期的MDS不知道导出事件，所以只发送消息给导出的MDS（即原先的权威MDS），这隐含的意思是，导出的MDS还没有对状态进行编码，并且发松给副本MDS。

当子树导出完成后，延后的过期处理会被处理（收到消息的MDS已经是权威MDS），或者被丢弃（如果此时MDS已经不是权威MDS），因为不管是导入还是导出元数据，都可能在迁移的过程中失败，MDS并不知道自己到底变成权威的还是非权威的，直到迁移完成。

在迁移的过程中，子树首先在导出MDS以及导入MDS上都被冻结，然后所有的副本都会被通知，子树现在的权威是模糊的。这保证，所有的在迁移过程中的过期事件都会被两者知道，就算是故障发生时，也不会失去什么。

### 正常情况下的迁移

迁移开始时，会现在export_dir()接口中做一些校验，以核实导出子树是允许的。特别地，集群不能处于降级状态，子树的根可能不是正在冻结，或者已经冻结，而且路径必须已经pin住。如果这些条件蛮熟，子树更目录会临时的auth pin住。子树会先初始化为freeze，导出的MDS也会被提交到子树迁移流程中。

MExportDiscover 用来保证导出的base directory的inode在目标节点上打开。这个inode会被导入的MDS pin住，以避免被trim掉。
这发生在导出的MDS完成对子树冻结之前，是为了确保导入的mds可以备份必要的元数据。当导出的MDS收到MDiscoverAck时，导出的MDS就允许通过移除临时的auth pin后来冻结子树。（导入的MDS发送MDiscoverAck给导出的MDS）

接着导出MDS发送MExportPrep消息，在导出范围内的包括所有dir、inode、dentry的信息填充到导入MDS。这也备份元数据，但这会导出MDS push out掉，以避免死锁。

导入的MDS负责打开边界目录后，然后发送Ack。这保证导入的MDS拥有正确的dir auth信息，即权威的MDS重新代理被迁移子树的范围是什么。当导入的MDS在处理MExportPrep是，导入的MDS冻结正科子树区域，来避免新的副本或者缓存过期。

如果base subtree directory被打开的MDS既不是到导入的MDS，也不是导出的MDS，告警阶段就会发生。

*base subtree directory* 是什么意思？？？
