＃Broadcast 广播变量介绍
## Broadcast是什么
广播变量是spark中实现数据共享的一种方式，其将只读的变量广播到其他的执行节点上去，满足整个执行节点能够访问这一变量的目标。同时又能满足每一个执行节点上只维护一份变量，不同的task共享这一份变量，避免每次task执行的时候都要序列化。其通常用于将一份稍大的数据广播到执行执行节点上，比如将字典数据广播出去.

##广播变量的如何使用
｀｀｀
var sc = new SparkContext(sparkConf)
var dict = Map(1->1, 2->2, 3->3)
var broadcastDict = sc.broadcast(dict)
var rdd = sc.parallelize(Array(1, 2, 3))
rdd.foreachPartition(x => broadcastDict.value.get(x))
broadcastDict.onDestory()
｀｀｀

##广播变量的机制
广播变量在序列化的时候，会将本地的数据储存到本地的blockmanager中，存储级别是MEMORY_AND_DISK,有的也会直接序列化完毕后存储在本地磁盘上(如HttpBroadcast),当其随着task序列化为字节流传递到执行节点的时候，在executor进行反序列化的过程中，会调用｀broadcast.readObject｀方法，其主要的逻辑是首先从本地的blockmanager中获取broadcastID对应的block，没有的话再去远端获取，不同的广播变量类型获取的方式不一样。

##广播变量的种类
当前spark1.6.1主要有两种广播变量类型，分别为`HttpBroadcast`和`TorrentBroadcast`，他们的实现机制不同，http的方式是在driver端开启一个用于广播变量的http服务，广播变量会序列化后作为文件存储到本地的磁盘上，task任务反序列化的时候会通过http的方式来fetch相应的数据信息；由于所有的数据都需要从driver端获取，如果executor数目过多的话，会造成driver端的网络拥塞。TorrentBroadcast参看了bit的方式，随着广播变量在executor上存储，其余的executor也会从已经有的executor上fetch相应的数据信息。二者的类图如下所示
![broadcast](img/Broadcast.png)
