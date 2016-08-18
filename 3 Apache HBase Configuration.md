#Apache HBase Configuration
本章将扩展快速开始一张，来进一步扩展HBase的配置。仔细阅读本章，特别是Basic Prerequisites来确保你HBase的测试和部署能够正常进行，并防止数据丢失。
##3. Configuration Files
Apache HBase和Hadoop使用了同样的配置系统，所有的配置文件都在conf/目录，每个节点上要保持同步。

###HBase Configuration File Descriptions

**backup-masters**

默认不存在，使用纯文本文件（plain-text），写入哪台主机要启动备份Master进程，一行一个主机

**hadoop-metrics2-hbase.properties**

>用来连接HBase Hadoop’s Metrics2 framework。See the Hadoop Wiki entry for more information on Metrics2. Contains only commented-out examples by default.

**hbase-env.cmd and hbase-env.sh**

用来设定HBase工作环境的脚本，包含java位置、java选项和其他环境变量，文件包含许多注释（commented-out）例子。

**hbase-policy.xml**

默认给RPC服务器使用的文件配置，用来对客户端请求作出授权决定，只在HBase安全启用下使用。

_**hbase-site.xml**_

HBase主配置文件，这个文件指定配置选项可以覆盖HBase默认的配置。我们可以在docs/hbase-default.xml下看到默认配置文件，但不要修改它。你也能在Hbase Configuration tab of HBase Web UI查看集群的整个有效配置。

log4j.properties
Configuration file for HBase logging via log4j.

regionservers
纯文本文件：包含在集群中要运行RS的主机列表。默认情况下这个文件包含单个本地主机，它应该包含主机名或者ip地址，一行一个，如果集群中每个节点都将运行一个RS在启本地端，那文件应该仅仅包含本地主机


4. Basic Prerequisites
这小节列出需要的服务和一些需要的系统配置
In HBase 0.98.5 and newer, you must set JAVA_HOME on each node of your cluster. hbase-env.sh provides a handy mechanism to do this.
ssh、DNS、Loopback IP（127.0.0.1 =》 localhost）、NTP
Limits on Number of Files and Processes (ulimit)
HBase是一个数据库，需要有能够打开大量文件的能力。许多linux发布版本都限制来了单个用户最多只能打卡1024个文件
You can check this limit on your servers by running the command ulimit -n when logged in as the user which runs HBase. See the Troubleshooting section for some of the problems you may experience if the limit is too low. You may also notice errors such as the following:
2010-04-06 03:04:37,542 INFO org.apache.hadoop.hdfs.DFSClient: Exception increateBlockOutputStream java.io.EOFException
2010-04-06 03:04:37,542 INFO org.apache.hadoop.hdfs.DFSClient: Abandoning block blk_-6935524980745310745_1391901

建议将限制提升到至少10000,最好是10240.，因为值通常被表示为1024的倍数。每个ColumnFamily 至少有一个存储文件，可能超过6个存储文件如果区域欠载。The number of open files required depends upon the number of ColumnFamilies and the number of regions. 下面是一个计算
rs上打开文件可能值的公式：(StoreFiles per ColumnFamily) x (regions per RegionServer)
例如，假定图表每个region有三个CF，每个CF有3个storefiles，在RS上有100个region，所以JVM将打开900=3×3×100个文件描述符，不包含JAR files, configuration files, and others.打开一个文件不会占用太多资源，而允许一个用户打开太多文件的风险也是极小的。
另外一个限制是一个用户一次可以运行进程数，In Linux and Unix, the number of processes is set using the ulimit -u command.Under load, a ulimit -u that is too low can cause OutOfMemoryError exceptions. See Jack Levin’s major HDFS issues thread on the hbase-users mailing list, from 2011.
配置文件描述符和进程数的最大值对正在运行HBase进程的用户来说，这是一个操作系统配置而不是HBase配置。确保设置已经为运行HBase的用户做过修改也是很重要的。去HBase log看下是哪个用户启动来HBase以及用户的ulimit配置。 A useful read setting config on your hadoop cluster is Aaron Kimball’s Configuration Parameters: What can you just ignore?
4.1 Hadoop
4.1.6. dfs.datanode.max.transfer.threads
一个HDFS DN有一次能够serve的文件上界，读取文件前，确保你配置过Hadoop’s conf/hdfs-site.xml,setting the dfs.datanode.max.transfer.threads value to at least the following:

<property>
  <name>dfs.datanode.max.transfer.threads</name>
  <value>4096</value>
</property>

Be sure to restart your HDFS after making the above configuration.没有配置这个项会出现奇怪的失败
For example:

10/12/08 20:10:31 INFO hdfs.DFSClient: Could not obtain block
          blk_XXXXXXXXXXXXXXXXXXXXXX_YYYYYYYY from any node: java.io.IOException: No live nodes
          contain current block. Will get new block locations from namenode and retry...
          
  
  
4.2. ZooKeeper Requirements   
ZooKeeper 3.4.x is required as of HBase 1.0.0. HBase makes use of the multi functionality that is only available since 3.4.0 (The useMulti configuration option defaults to true in HBase 1.0.0). See HBASE-12241 (The crash of regionServer when taking deadserver’s replication queue breaks replication) and HBASE-6775 (Use ZK.multi when available for HBASE-6710 0.92/0.94 compatibility fix) for background.



5. HBase run modes: Standalone and Distributed
HBase有两种启动模式 Standalone and Distributed。out of the box（初始？）HBase以单机模式运行。无论你用哪种模式，你都要配置在HBase conf目录下的相关文件，至少，你要配置hbase-env.sh的java_home,在这个文件中设置Hbase环境变量如堆大小、
其他JVM选项、log文件位置等

5.1. Standalone HBase
默认模式，已在快速开始中描述过，这中模式Hbase不使用HDFS，而是用本地文件系统代替--运行所有HBase服务和一个本地ZK在同一个JVM中。ZK和一个已知的端口绑定在一起，这样客户端可以和HBase通信


5.2. Distributed
分布式模式可分为伪分布式和全分布式。伪分布式所有守护程序都运行在一个节点上，而全分布式守护程序分布在集群中的所有节点上。这种命名法（nomenclature）来自Hadoop
伪分布式可以在本地文件和HDFS上运行，而全分布式只能在HDFS上。See the Hadoop documentation for how to set up HDFS. A good walk-through for setting up HDFS on Hadoop 2 can be found at http://www.alexjf.net/blog/distributed-systems/hadoop-yarn-installation-definitive-guide.

5.2.1. Pseudo-distributed
A pseudo-distributed mode is simply a fully-distributed mode run on a single host. Use this configuration testing and prototyping on HBase. Do not use this configuration for production nor for evaluating HBase performance.

5.3. Fully-distributed
默认情况下HBase运行以standalone运行，standalone和pseudo-distributed可用来小规模测试，对生产环境分布式更合适In distributed mode, multiple instances of HBase daemons run on multiple servers in the cluster.
像伪分布式一样，你要设置hbase-cluster.distributed这个属性为true。hbase.rootdir要配置成可用 的HDFS文件系统。另外，集群中的节点要配置成 RegionServers, ZooKeeper QuorumPeers, and backup HMaster servers. 
分布式RegionServers
一般情况下，你的集群将包含多个RS、主备Master、ZK守护程序运行在不同服务器上，conf/regionservers 在master上的这个文件包含主机表，每个主机一行，所有表中的主机上的RS将随着master服务起停。
Example 7. Example Distributed HBase Cluster
这是一个包含基本元素（bare-bones）的分布式Hbase集群的配置文件 conf/hbase-site.xml 实际工作的集群可能包含更多自定义的配置参数。大多数HBase配置指令有默认值，通常使用这些默认值，除非被 hbase-site.xml覆盖。

Procedure: HDFS Client Configuration
1、注：如果你在你的Hadoop集群上改变HDFS客户端配置，例如HDFS客户端的配置指令（directives），不是服务端的配置，你必须使用如下的方法来启动HBase并应用这些配置改变。
  a、在hbase-env.sh中增加你的HADOOP_CONF_DIR到环境变量HBASE_CLASSPATH
  b、在 ${HBASE_HOME}/conf下复制或最好链接(symlinks)一个hdfs-site.xml,或
  c、仅仅一点HDFS客户端配置把它们加入hbase-site.xml
  如：属性dfs.replication，如果你洗哪个运行5个副本，HBase将以默认3来创建文件，除非你做上面的操作使配置对HBase可用
  
  
  6. Running and Confirming Your Installation
确保首先运行HDFS。 在HADOOP_HOME directory目录执行bin/start-hdfs.sh，同过put和get文件进行测试确保HDFS运行。HBase一般不使用MR或YARN daemons。这些不用被启动
如果你在管理你自己的ZK，启动它并确定它在运行，不然HBase会启动ZK作为其启动进程的一部分
bin/start-hbase.sh
现在我们应该又拉一个正在运行的HBase instance，日志可以在logs子目录下找到，检查它们特别是如果Hbase没有正常启动
HBase会拉起一个UI来列出关键属性。默认情况下部署在主节点端口16010上，RS在16020以及一个 informational HTTP server在16030
Prior to HBase 0.98 the master UI was deployed on port 60010, and the HBase RegionServers UI on port 60030.

To stop HBase after exiting the HBase shell enter
$ ./bin/stop-hbase.sh
关闭可能要花点时间，特别是如果你的集群由许多机器组成。如果在分布式系统上运行，一定要确保所有HBase关闭，才能结束Hadoop daemons


7. Default Configuration
7.1. hbase-site.xml and hbase-default.xml
就像在Hadoop中一样在hdfs-site.xml文件中加入特定的HDFS配置，对HBase来说在conf/hbase-site.xml文件中编辑特定的自定义配置For the list of configurable properties, see hbase default configurations below or view the raw hbase-default.xml source file in the HBase source code at src/main/resources.
不是所有的配置选项都会在hbase-default.xml中体现，那些很少有人会改的配置只在代码中，唯一的方法去改变这些配置的方法就是去阅读源代码
Currently, changes here will require a cluster restart for HBase to notice the change

7.2. HBase Default Configuration
hbase.tmp.dir
本地文件系统的临时目录，为这个设定指定一个比/tmp（the usual resolve for java.io.tmpdir, as the '/tmp' directory is cleared on machine restart.）更持久的位置
Default ${java.io.tmpdir}/hbase-${user.name}

hbase.rootdir
Description：The directory shared by region servers and into which HBase persists. （Rs共享的目录以及HBase存在的目录）URL要严格满足文件系统，例如指定hdfs://namenode.example.org:9000/hbase   这里的HDFS目录‘/hbase’，HDFS实例的NN运行在namenode.example.org on port 9000  By default, we write to whatever ${hbase.tmp.dir} is set too — usually /tmp — so change this configuration or else all data will be lost on machine restart.

hbase.fs.tmp.dir
Description：A staging directory （临时目录）in default file system (HDFS) for keeping temporary data.
Default
/user/${user.name}/hbase-staging

hbase.bulkload.staging.dir
Description：批量加载（bulk loading）的临时目录
Default
${hbase.fs.tmp.dir}

hbase.cluster.distributed
Description：集群运行模式，默认false

hbase.zookeeper.quorum
Description zk集合以逗号分割的服务器表（Comma separated list of servers in the ZooKeeper ensemble？）默认为standalone和伪分布式设置为本地主机。对一个分布式来说，要设置为完整的ZK集合服务器，如果hbase-env.sh 中HBASE_MANAGES_Z被设置，这个是哪个hbase将start/stop ZK随着集群start/stop的servers表。客户端将使用这个表中的集合成员，and put it together with the hbase.zookeeper.clientPort config. and pass it into zookeeper constructor as the connectString parameter.
Default
localhost

hbase.local.dir
在本地文件系统上的目录用于本地存储  Default   ${hbase.tmp.dir}/local/

hbase.master.port
HBase Master绑定的端口  Default 16000
hbase.master.info.port
The port for the HBase Master web UI. Set to -1 if you do not want a UI instance run.  
默认：16010

hbase.master.info.bindAddress   Description  The bind address for the HBase Master web UI  Default  0.0.0.0

hbase.master.logcleaner.plugins
#LogCleaner 定期清理.oldlogs目录下的log。它会从hbase.master.logcleaner.plugins配置里加载所有BaseLogCleanerDelegate。某个log可以被删除，只有所有delegate都同意。
逗号分割的BaseLogCleanerDelegate表，会被LogsCleaner service引用。这些WAL（预写日志） cleaner按顺序被引用，所以可以把先调用的放到前面。你也可以补充自己的BaseLogCleanerDelegate，只要加到HBase的classpath下，要写下合格的完整类名，Always add the above default log cleaners in the list.
Default    org.apache.hadoop.hbase.master.cleaner.TimeToLiveLogCleaner

hbase.master.logcleaner.ttl
WAL可以在.oldlogdir中保存的最长时间，之后会被Master线程清除 默认：600000

hbase.master.hfilecleaner.plugins
A comma-separated list of BaseHFileCleanerDelegate invoked by the HFileCleaner service. These HFiles cleaners are called in order, so put the cleaner that prunes the most files in front. To implement your own BaseHFileCleanerDelegate, just put it in HBase’s classpath and add the fully qualified class name here. Always add the above default log cleaners in the list as they will be overwritten in hbase-site.xml.
#HFileCleaner  定期清理.archive目录下的HFile。它会从hbase.master.hfilecleaner.plugins配置里加载所有BaseHFileCleanerDelegate。
Default  org.apache.hadoop.hbase.master.cleaner.TimeToLiveHFileCleaner

hbase.master.infoserver.redirect
不管Master是否listen to web ui 端口，重定向请求到 Master and RegionServer的web ui server
默认 true

hbase.regionserver.port
RS绑定的端口号 默认16020
hbase.regionserver.info.port  
The port for the HBase RegionServer web UI Set to -1 if you do not want the RegionServer UI to run.
默认16030

hbase.regionserver.info.bindAddress
Description   The address for the HBase RegionServer web UI   Default    0.0.0.0

hbase.regionserver.info.port.auto
Master或RS是否搜索端口来绑定页面，打开自动端口搜索如果hbase.regionserver.info.port被占用。Useful for testing, turned off by default.

hbase.regionserver.handler.count
RegionServers受理的RPC Server实例数量，对Master来说是Master能够受理的handler数量
默认30

hbase.ipc.server.callqueue.handler.factor
呼叫队列数量决定因素，0代表只有一个队列用于所有handler共享，1代表每个handler有自己的队列
Default   0.1

hbase.ipc.server.callqueue.read.ratio
将呼叫队列分为读、写队列。分割间隔（0.0-1.0之间）乘以call queues。0表示不分割，即读和写请求都将被推入同一队列，小于0.5表示读少于写，0.5表示read= write，大于0.5read>write，1表示1个用于写其余都用于读
Example: Given the total number of call queues being 10 a read.ratio of 0 means that: the 10 queues will contain both read/write requests. a read.ratio of 0.3 means that: 3 queues will contain only read requests and 7 queues will contain only write requests. a read.ratio of 0.5 means that: 5 queues will contain only read requests and 5 queues will contain only write requests. a read.ratio of 0.8 means that: 8 queues will contain only read requests and 2 queues will contain only write requests. a read.ratio of 1 means that: 9 queues will contain only read requests and 1 queues will contain only write requests.
默认为0

hbase.ipc.server.callqueue.scan.ratio
已知读呼叫队列数，由所有队列数×callqueue.read.ratio，属性scan.ratio将把读队列分为长、短读队列，小于0.5 less long-read 。0或1指用同一组队列来get和scan
Example: Given the total number of read call queues being 8 a scan.ratio of 0 or 1 means that: 8 queues will contain both long and short read requests. a scan.ratio of 0.3 means that: 2 queues will contain only long-read requests and 6 queues will contain only short-read requests. a scan.ratio of 0.5 means that: 4 queues will contain only long-read requests and 4 queues will contain only short-read requests. a scan.ratio of 0.8 means that: 6 queues will contain only long-read requests and 2 queues will contain only short-read requests
默认0

hbase.regionserver.msginterval
RegionServer 发消息给 Master 时间间隔，单位是毫秒 默认3000

hbase.regionserver.logroll.period
提交commit log的时间间隔，不管是否记录足够的值
默认3600000  =1h

hbase.regionserver.logroll.errors.tolerated
允许连续WAL关闭错误的次数，超过这个次数，将触发服务器中止，0将使RS中止如果在日志滚动中关闭当前WAL记录者失败。即使一个很小的值（2或3）将允许一个RS短暂越过HDFS错误（allow a region server to ride over transient HDFS errors）  默认 2

hbase.regionserver.hlog.reader.impl
WAL file reader的实现 Default    org.apache.hadoop.hbase.regionserver.wal.ProtobufLogReader

hbase.regionserver.hlog.writer.impl
Description  The WAL file writer implementation.  Default    org.apache.hadoop.hbase.regionserver.wal.ProtobufLogWriter

hbase.regionserver.global.memstore.size
 在新的更新被屏蔽、被迫刷新新之前一个RS中所有memstore的最大size。默认为40%的heap，Updates are blocked and flushes are forced until size of all memstores in a region server hits hbase.regionserver.global.memstore.size.lower.limit. The default value in this configuration has been intentionally left empty in order to honor the old hbase.regionserver.global.memstore.upperLimit property if present.
 
 hbase.regionserver.global.memstore.size.lower.limit
Maximum size of all memstores in a region server before flushes are forced. Defaults to 95% of hbase.regionserver.global.memstore.size (0.95). A 100% value for this value causes the minimum possible flushing to occur when updates are blocked due to memstore limiting. The default value in this configuration has been intentionally left empty in order to honor the old hbase.regionserver.global.memstore.lowerLimit property if present.

hbase.regionserver.optionalcacheflushinterval
Description Maximum amount of time an edit lives in memory before being automatically flushed. 一个日志文件在内存中刷新之前停留的最长时间。Default 1 hour. Set it to 0 to disable automatic flushing.
Default 3600000

hbase.regionserver.dns.interface
当使用DNS的时候，RegionServer用来上报的IP地址的网络接口名字。Default   default

hbase.regionserver.dns.nameserver
当使用DNS的时候，RegionServer使用的DNS的域名或者IP 地址，RegionServer用它来确定和master用来进行通讯和展示的域名.

hbase.regionserver.region.split.policy
一个决定一个region何时被分割的分割策略The various other split policies that are available currently are BusyRegionSplitPolicy, ConstantSizeRegionSplitPolicy, DisabledRegionSplitPolicy, DelimitedKeyPrefixRegionSplitPolicy, and KeyPrefixRegionSplitPolicy. DisabledRegionSplitPolicy blocks manual region splitting.
Default    org.apache.hadoop.hbase.regionserver.IncreasingToUpperBoundRegionSplitPolicy

hbase.regionserver.regionSplitLimit
region数量限制，region分割的数量不会超过这个值，但是这不是一个硬限制，只是作为一个guideline，使regionserver停止分割 默认1000

zookeeper.session.timeout
ZK会话超时时间 毫秒  ，被用于两种不同的方式，首先备用在ZK客户端，即Hbase用来连接集合。也被HBase用于启动ZK server HBase把这个值传递给zk集群 默认90000

zookeeper.znode.parent
zooKeeper中的HBase的根ZNode。所有的HBase的ZooKeeper会用这个目录配置相对路径。默认情况下，所有的HBase的ZooKeeper文件路径是用相对路径，所以他们会都去这个目录下面。
默认：/hbase

zookeeper.znode.acl.parent
znode的访问控制表 默认acl

hbase.zookeeper.dns.interface
Description
The name of the Network Interface from which a ZooKeeper server should report its IP address.
Default
default
ZK server用来上包IP地址的网络接口名字

hbase.zookeeper.dns.nameserver
Description
The host name or IP address of the name server (DNS) which a ZooKeeper server should use to determine the host name used by the master for communication and display purposes.
Default
default

ZK server用来确定和master通讯、展示的域名或ip

hbase.zookeeper.peerport
Description
Port used by ZooKeeper peers to talk to each other. See http://hadoop.apache.org/zookeeper/docs/r3.1.1/zookeeperStarted.html#sc_RunningReplicatedZooKeeper for more information.
Default
2888
ZK之间通信的端口

hbase.zookeeper.leaderport
Description
Port used by ZooKeeper for leader election. See http://hadoop.apache.org/zookeeper/docs/r3.1.1/zookeeperStarted.html#sc_RunningReplicatedZooKeeper for more information.
Default
3888
ZK选举leader的端口

hbase.zookeeper.useMulti
Description
Instructs HBase to make use of ZooKeeper’s multi-update functionality. This allows certain ZooKeeper operations to complete more quickly and prevents some issues with rare Replication failure scenarios (see the release note of HBASE-2611 for an example). IMPORTANT: only set this to true if all ZooKeeper servers in the cluster are on version 3.4+ and will not be downgraded. ZooKeeper versions before 3.4 do not support multi-update and will not fail gracefully if multi-update is invoked (see ZOOKEEPER-1495).
Default
true
指导HBase使用ZK的multi-update功能，这个参数准许某些ZK操作更快得完成，并防止一些罕见的复制失败场景的问题发生

hbase.zookeeper.property.initLimit
Description
Property from ZooKeeper’s config zoo.cfg. The number of ticks that the initial synchronization phase can take.
Default
10
来自ZK配置zoo.cfg的属性，初始化synchronization阶段的ticks数量限制

hbase.zookeeper.property.syncLimit
Description
Property from ZooKeeper’s config zoo.cfg. The number of ticks that can pass between sending a request and getting an acknowledgment.
Default
5
ZooKeeper的zoo.conf中的配置。 发送一个请求到获得承认之间的ticks的数量限制

hbase.zookeeper.property.dataDir
Description
Property from ZooKeeper’s config zoo.cfg. The directory where the snapshot is stored.
Default
${hbase.tmp.dir}/zookeeper
快照存储的位置

hbase.zookeeper.property.clientPort
Description
Property from ZooKeeper’s config zoo.cfg. The port at which the clients will connect.
Default
2181
客户端连接端口

hbase.zookeeper.property.maxClientCnxns
Description
Property from ZooKeeper’s config zoo.cfg. Limit on number of concurrent connections (at the socket level) that a single client, identified by IP address, may make to a single member of the ZooKeeper ensemble. Set high to avoid zk connection issues running standalone and pseudo-distributed.
Default
300
单个客户端和ZK集群中单个节点同时连接的数量，这个值可以设置的高点，防止单击和伪分布式中出现连接问题。

hbase.client.write.buffer
Description
Default size of the HTable client write buffer in bytes. A bigger buffer takes more memory — on both the client and server side since server instantiates the passed write buffer to process it — but a larger buffer size reduces the number of RPCs made. For an estimate of server-side memory-used, evaluate hbase.client.write.buffer * hbase.regionserver.handler.count
Default
2097152
HTable客户端写缓存的默认大小，这个值越大，消耗内存越大，因为在客户和服务端都要实例花写缓存来处理它。但一个较大的缓冲值，可以减少RPC次数

hbase.client.pause
Description
General client pause value. Used mostly as value to wait before running a retry of a failed get, region lookup, etc. See hbase.client.retries.number for description of how we backoff from this initial pause amount and how this pause works w/ retries.
Default
100
一般客户端的暂停时间，在遇到错误需要重试时的等待时间

hbase.client.retries.number
Description
Maximum retries. Used as maximum for all retryable operations such as the getting of a cell’s value, starting a row update, etc. Retry interval is a rough function based on hbase.client.pause. At first we retry at this interval but then with backoff, we pretty quickly reach retrying every ten seconds. See HConstants#RETRY_BACKOFF for how the backup ramps up. Change this setting and hbase.client.pause to suit your workload.
Default
35
最大重试次数。用在所有需要重试的操作

hbase.client.max.total.tasks
Description
The maximum number of concurrent tasks a single HTable instance will send to the cluster.
Default
100
一个HTable实例一次送给集群最大任务量

hbase.client.max.perserver.tasks
Description
The maximum number of concurrent tasks a single HTable instance will send to a single region server.
Default
5
一个HTable实例一次送给一个RS的最大任务量

hbase.client.max.perregion.tasks
Description
The maximum number of concurrent connections the client will maintain to a single Region. That is, if there is already hbase.client.max.perregion.tasks writes in progress for this region, new puts won’t be sent to this region until some writes finishes.
Default
1
客户端和一个Region维持的最大连接数，即如果在region中已有hbase.client.max.perregion.tasks写如进程，新的输入不会被送入这个region，直到某些写入完成。

hbase.client.scanner.caching
Description
Number of rows that we try to fetch when calling next on a scanner if it is not served from (local, client) memory. This configuration works together with hbase.client.scanner.max.result.size to try and use the network efficiently. The default value is Integer.MAX_VALUE by default so that the network will fill the chunk size defined by hbase.client.scanner.max.result.size rather than be limited by a particular number of rows since the size of rows varies table to table. If you know ahead of time that you will not require more than a certain number of rows from a scan, this configuration should be set to that row limit via Scan#setCaching. Higher caching values will enable faster scanners but will eat up more memory and some calls of next may take longer and longer times when the cache is empty. Do not set this value such that the time between invocations is greater than the scanner timeout; i.e. hbase.client.scanner.timeout.period
Default
2147483647
当调用scanner的next方法时，而又不在内存中，从服务端一次获取的行数，这个配置和hbase.client.scanner.max.result.size一起试着有效使用网络。默认为整形最大值，这样网络可以用满块大小（由hbase.client.scanner.max.result.size设定）而不会被一个特定的值限制住，因为每个表的行数都不一样。值越大，scanner越快，但会占用更多的内存，当缓存被占满时，下一次调用会越来越慢。Do not set this value such that the time between invocations is greater than the scanner timeout？最终超时


hbase.client.keyvalue.maxsize
Description
Specifies the combined maximum allowed size of a KeyValue instance. This is to set an upper boundary for a single entry saved in a storage file. Since they cannot be split it helps avoiding that a region cannot be split any further because the data is too large. It seems wise to set this to a fraction of the maximum region size. Setting it to zero or less disables the check.
Default
10485760
一个KeyValue实例的最大值，这个是用来设置存储文件中单个entry的大小上界。因为KeyValue不能被分割，所以可以避免因为数据过大导致region不可分割。明智的做法是把它设为可以被最大region size整除的数。

hbase.client.scanner.timeout.period
Description
Client scanner lease period in milliseconds.
Default
60000
scanner客户端租赁期限

hbase.client.localityCheck.threadPoolSize
Default
2

hbase.bulkload.retries.number
Description
Maximum retries. This is maximum number of iterations to atomic bulk loads are attempted in the face of splitting operations 0 means never give up.
Default
10
分割操作时迭代去原子批量加载尝试的最大次数

hbase.balancer.period
Description
Period at which the region balancer runs in the Master.
Default
300000
Master上运行region balancer的时间间隔

hbase.normalizer.period
Description
Period at which the region normalizer runs in the Master.
Default
1800000
Master执行region normalizer的间隔

hbase.regions.slop
Description
Rebalance if any regionserver has average + (average * slop) regions. The default value of this parameter is 0.001 in StochasticLoadBalancer (the default load balancer), while the default is 0.2 in other load balancers (i.e., SimpleLoadBalancer).
Default
0.001
如果任意RS有 average + (average * slop)个分区，将执行重新均衡

hbase.server.thread.wakefrequency
Description
Time to sleep in between searches for work (in milliseconds). Used as sleep interval by service threads such as log roller.
Default
10000


hbase.server.versionfile.writeattempts
Description
How many times to retry attempting to write a version file before just aborting. Each attempt is separated by the hbase.server.thread.wakefrequency milliseconds.
Default
3
退出前尝试写版本文件的次数

hbase.hregion.memstore.flush.size
Description
Memstore will be flushed to disk if size of the memstore exceeds this number of bytes. Value is checked by a thread that runs every hbase.server.thread.wakefrequency.
Default
134217728
当memstore的大小超过这个值时，将flush到磁盘。这个值被线程每隔--检查一次

hbase.hregion.percolumnfamilyflush.size.lower.bound.min
Description
If FlushLargeStoresPolicy is used and there are multiple column families, then every time that we hit the total memstore limit, we find out all the column families whose memstores exceed a "lower bound" and only flush them while retaining the others in memory. The "lower bound" will be "hbase.hregion.memstore.flush.size / column_family_number" by default unless value of this property is larger than that. If none of the families have their memstore size more than lower bound, all the memstores will be flushed (just as usual).
Default
16777216


hbase.hregion.preclose.flush.size
Description
If the memstores in a region are this size or larger when we go to close, run a "pre-flush" to clear out memstores before we put up the region closed flag and take the region offline. On close, a flush is run under the close flag to empty memory. During this time the region is offline and we are not taking on any writes. If the memstore content is large, this flush could take a long time to complete. The preflush is meant to clean out the bulk of the memstore before putting up the close flag and taking the region offline so the flush that runs under the close flag has little to do.

Default
5242880

hbase.hregion.memstore.block.multiplier
Description
Block updates if memstore has hbase.hregion.memstore.block.multiplier times hbase.hregion.memstore.flush.size bytes. Useful preventing runaway memstore during spikes in update traffic. Without an upper-bound, memstore fills such that when it flushes the resultant flush files take a long time to compact or split, or worse, we OOME.

Default
4

hbase.hregion.memstore.mslab.enabled
Description
Enables the MemStore-Local Allocation Buffer, a feature which works to prevent heap fragmentation under heavy write loads. This can reduce the frequency of stop-the-world GC pauses on large heaps.

Default
true

hbase.hregion.max.filesize
Description
Maximum HFile size. If the sum of the sizes of a region’s HFiles has grown to exceed this value, the region is split in two.

Default
10737418240

hbase.hregion.majorcompaction
Description
Time between major compactions, expressed in milliseconds. Set to 0 to disable time-based automatic major compactions. User-requested and size-based major compactions will still run. This value is multiplied by hbase.hregion.majorcompaction.jitter to cause compaction to start at a somewhat-random time during a given window of time. The default value is 7 days, expressed in milliseconds. If major compactions are causing disruption in your environment, you can configure them to run at off-peak times for your deployment, or disable time-based major compactions by setting this parameter to 0, and run major compactions in a cron job or by another external mechanism.

Default
604800000

hbase.hregion.majorcompaction.jitter
Description
A multiplier applied to hbase.hregion.majorcompaction to cause compaction to occur a given amount of time either side of hbase.hregion.majorcompaction. The smaller the number, the closer the compactions will happen to the hbase.hregion.majorcompaction interval.

Default
0.50

hbase.hstore.compactionThreshold
Description
If more than this number of StoreFiles exist in any one Store (one StoreFile is written per flush of MemStore), a compaction is run to rewrite all StoreFiles into a single StoreFile. Larger values delay compaction, but when compaction does occur, it takes longer to complete.

Default
3

hbase.hstore.flusher.count
Description
The number of flush threads. With fewer threads, the MemStore flushes will be queued. With more threads, the flushes will be executed in parallel, increasing the load on HDFS, and potentially causing more compactions.

Default
2

hbase.hstore.blockingStoreFiles
Description
If more than this number of StoreFiles exist in any one Store (one StoreFile is written per flush of MemStore), updates are blocked for this region until a compaction is completed, or until hbase.hstore.blockingWaitTime has been exceeded.

Default
10

hbase.hstore.blockingWaitTime
Description
The time for which a region will block updates after reaching the StoreFile limit defined by hbase.hstore.blockingStoreFiles. After this time has elapsed, the region will stop blocking updates even if a compaction has not been completed.

Default
90000

hbase.hstore.compaction.min
Description
The minimum number of StoreFiles which must be eligible for compaction before compaction can run. The goal of tuning hbase.hstore.compaction.min is to avoid ending up with too many tiny StoreFiles to compact. Setting this value to 2 would cause a minor compaction each time you have two StoreFiles in a Store, and this is probably not appropriate. If you set this value too high, all the other values will need to be adjusted accordingly. For most cases, the default value is appropriate. In previous versions of HBase, the parameter hbase.hstore.compaction.min was named hbase.hstore.compactionThreshold.

Default
3

hbase.hstore.compaction.max
Description
The maximum number of StoreFiles which will be selected for a single minor compaction, regardless of the number of eligible StoreFiles. Effectively, the value of hbase.hstore.compaction.max controls the length of time it takes a single compaction to complete. Setting it larger means that more StoreFiles are included in a compaction. For most cases, the default value is appropriate.

Default
10

hbase.hstore.compaction.min.size
Description
A StoreFile (or a selection of StoreFiles, when using ExploringCompactionPolicy) smaller than this size will always be eligible for minor compaction. HFiles this size or larger are evaluated by hbase.hstore.compaction.ratio to determine if they are eligible. Because this limit represents the "automatic include" limit for all StoreFiles smaller than this value, this value may need to be reduced in write-heavy environments where many StoreFiles in the 1-2 MB range are being flushed, because every StoreFile will be targeted for compaction and the resulting StoreFiles may still be under the minimum size and require further compaction. If this parameter is lowered, the ratio check is triggered more quickly. This addressed some issues seen in earlier versions of HBase but changing this parameter is no longer necessary in most situations. Default: 128 MB expressed in bytes.

Default
134217728

hbase.hstore.compaction.max.size
Description
A StoreFile (or a selection of StoreFiles, when using ExploringCompactionPolicy) larger than this size will be excluded from compaction. The effect of raising hbase.hstore.compaction.max.size is fewer, larger StoreFiles that do not get compacted often. If you feel that compaction is happening too often without much benefit, you can try raising this value. Default: the value of LONG.MAX_VALUE, expressed in bytes.

Default
9223372036854775807

hbase.hstore.compaction.ratio
Description
For minor compaction, this ratio is used to determine whether a given StoreFile which is larger than hbase.hstore.compaction.min.size is eligible for compaction. Its effect is to limit compaction of large StoreFiles. The value of hbase.hstore.compaction.ratio is expressed as a floating-point decimal. A large ratio, such as 10, will produce a single giant StoreFile. Conversely, a low value, such as .25, will produce behavior similar to the BigTable compaction algorithm, producing four StoreFiles. A moderate value of between 1.0 and 1.4 is recommended. When tuning this value, you are balancing write costs with read costs. Raising the value (to something like 1.4) will have more write costs, because you will compact larger StoreFiles. However, during reads, HBase will need to seek through fewer StoreFiles to accomplish the read. Consider this approach if you cannot take advantage of Bloom filters. Otherwise, you can lower this value to something like 1.0 to reduce the background cost of writes, and use Bloom filters to control the number of StoreFiles touched during reads. For most cases, the default value is appropriate.

Default
1.2F

hbase.hstore.compaction.ratio.offpeak
Description
Allows you to set a different (by default, more aggressive) ratio for determining whether larger StoreFiles are included in compactions during off-peak hours. Works in the same way as hbase.hstore.compaction.ratio. Only applies if hbase.offpeak.start.hour and hbase.offpeak.end.hour are also enabled.

Default
5.0F

hbase.hstore.time.to.purge.deletes
Description
The amount of time to delay purging of delete markers with future timestamps. If unset, or set to 0, all delete markers, including those with future timestamps, are purged during the next major compaction. Otherwise, a delete marker is kept until the major compaction which occurs after the marker’s timestamp plus the value of this setting, in milliseconds.

Default
0

hbase.offpeak.start.hour
Description
The start of off-peak hours, expressed as an integer between 0 and 23, inclusive. Set to -1 to disable off-peak.

Default
-1

hbase.offpeak.end.hour
Description
The end of off-peak hours, expressed as an integer between 0 and 23, inclusive. Set to -1 to disable off-peak.

Default
-1

hbase.regionserver.thread.compaction.throttle
Description
There are two different thread pools for compactions, one for large compactions and the other for small compactions. This helps to keep compaction of lean tables (such as hbase:meta) fast. If a compaction is larger than this threshold, it goes into the large compaction pool. In most cases, the default value is appropriate. Default: 2 x hbase.hstore.compaction.max x hbase.hregion.memstore.flush.size (which defaults to 128MB). The value field assumes that the value of hbase.hregion.memstore.flush.size is unchanged from the default.

Default
2684354560

hbase.hstore.compaction.kv.max
Description
The maximum number of KeyValues to read and then write in a batch when flushing or compacting. Set this lower if you have big KeyValues and problems with Out Of Memory Exceptions Set this higher if you have wide, small rows.

Default
10

hbase.storescanner.parallel.seek.enable
Description
Enables StoreFileScanner parallel-seeking in StoreScanner, a feature which can reduce response latency under special conditions.

Default
false

hbase.storescanner.parallel.seek.threads
Description
The default thread pool size if parallel-seeking feature enabled.

Default
10

hfile.block.cache.size
Description
Percentage of maximum heap (-Xmx setting) to allocate to block cache used by a StoreFile. Default of 0.4 means allocate 40%. Set to 0 to disable but it’s not recommended; you need at least enough cache to hold the storefile indices.

Default
0.4

hfile.block.index.cacheonwrite
Description
This allows to put non-root multi-level index blocks into the block cache at the time the index is being written.

Default
false

hfile.index.block.max.size
Description
When the size of a leaf-level, intermediate-level, or root-level index block in a multi-level block index grows to this size, the block is written out and a new block is started.

Default
131072

hbase.bucketcache.ioengine
Description
Where to store the contents of the bucketcache. One of: heap, offheap, or file. If a file, set it to file:PATH_TO_FILE. See http://hbase.apache.org/book.html#offheap.blockcache for more information.

Default
none

hbase.bucketcache.combinedcache.enabled
Description
Whether or not the bucketcache is used in league with the LRU on-heap block cache. In this mode, indices and blooms are kept in the LRU blockcache and the data blocks are kept in the bucketcache.

Default
true

hbase.bucketcache.size
Description
A float that EITHER represents a percentage of total heap memory size to give to the cache (if < 1.0) OR, it is the total capacity in megabytes of BucketCache. Default: 0.0

Default
none

hbase.bucketcache.bucket.sizes
Description
A comma-separated list of sizes for buckets for the bucketcache. Can be multiple sizes. List block sizes in order from smallest to largest. The sizes you use will depend on your data access patterns. Must be a multiple of 1024 else you will run into 'java.io.IOException: Invalid HFile block magic' when you go to read from cache. If you specify no values here, then you pick up the default bucketsizes set in code (See BucketAllocator#DEFAULT_BUCKET_SIZES).

Default
none

hfile.format.version
Description
The HFile format version to use for new files. Version 3 adds support for tags in hfiles (See http://hbase.apache.org/book.html#hbase.tags). Distributed Log Replay requires that tags are enabled. Also see the configuration 'hbase.replication.rpc.codec'.

Default
3

hfile.block.bloom.cacheonwrite
Description
Enables cache-on-write for inline blocks of a compound Bloom filter.

Default
false

io.storefile.bloom.block.size
Description
The size in bytes of a single block ("chunk") of a compound Bloom filter. This size is approximate, because Bloom blocks can only be inserted at data block boundaries, and the number of keys per data block varies.

Default
131072

hbase.rs.cacheblocksonwrite
Description
Whether an HFile block should be added to the block cache when the block is finished.

Default
false

hbase.rpc.timeout
Description
This is for the RPC layer to define how long (millisecond) HBase client applications take for a remote call to time out. It uses pings to check connections but will eventually throw a TimeoutException.

Default
60000

hbase.client.operation.timeout
Description
Operation timeout is a top-level restriction (millisecond) that makes sure a blocking operation in Table will not be blocked more than this. In each operation, if rpc request fails because of timeout or other reason, it will retry until success or throw RetriesExhaustedException. But if the total time being blocking reach the operation timeout before retries exhausted, it will break early and throw SocketTimeoutException.

Default
1200000

hbase.cells.scanned.per.heartbeat.check
Description
The number of cells scanned in between heartbeat checks. Heartbeat checks occur during the processing of scans to determine whether or not the server should stop scanning in order to send back a heartbeat message to the client. Heartbeat messages are used to keep the client-server connection alive during long running scans. Small values mean that the heartbeat checks will occur more often and thus will provide a tighter bound on the execution time of the scan. Larger values mean that the heartbeat checks occur less frequently

Default
10000

hbase.rpc.shortoperation.timeout
Description
This is another version of "hbase.rpc.timeout". For those RPC operation within cluster, we rely on this configuration to set a short timeout limitation for short operation. For example, short rpc timeout for region server’s trying to report to active master can benefit quicker master failover process.

Default
10000

hbase.ipc.client.tcpnodelay
Description
Set no delay on rpc socket connections. See http://docs.oracle.com/javase/1.5.0/docs/api/java/net/Socket.html#getTcpNoDelay()

Default
true

hbase.regionserver.hostname
Description
This config is for experts: don’t set its value unless you really know what you are doing. When set to a non-empty value, this represents the (external facing) hostname for the underlying server. See https://issues.apache.org/jira/browse/HBASE-12954 for details.

Default
none

hbase.master.keytab.file
Description
Full path to the kerberos keytab file to use for logging in the configured HMaster server principal.

Default
none

hbase.master.kerberos.principal
Description
Ex. "hbase/_HOST@EXAMPLE.COM". The kerberos principal name that should be used to run the HMaster process. The principal name should be in the form: user/hostname@DOMAIN. If "_HOST" is used as the hostname portion, it will be replaced with the actual hostname of the running instance.

Default
none

hbase.regionserver.keytab.file
Description
Full path to the kerberos keytab file to use for logging in the configured HRegionServer server principal.

Default
none

hbase.regionserver.kerberos.principal
Description
Ex. "hbase/_HOST@EXAMPLE.COM". The kerberos principal name that should be used to run the HRegionServer process. The principal name should be in the form: user/hostname@DOMAIN. If "_HOST" is used as the hostname portion, it will be replaced with the actual hostname of the running instance. An entry for this principal must exist in the file specified in hbase.regionserver.keytab.file

Default
none

hadoop.policy.file
Description
The policy configuration file used by RPC servers to make authorization decisions on client requests. Only used when HBase security is enabled.

Default
hbase-policy.xml

hbase.superuser
Description
List of users or groups (comma-separated), who are allowed full privileges, regardless of stored ACLs, across the cluster. Only used when HBase security is enabled.

Default
none

hbase.auth.key.update.interval
Description
The update interval for master key for authentication tokens in servers in milliseconds. Only used when HBase security is enabled.

Default
86400000

hbase.auth.token.max.lifetime
Description
The maximum lifetime in milliseconds after which an authentication token expires. Only used when HBase security is enabled.

Default
604800000

hbase.ipc.client.fallback-to-simple-auth-allowed
Description
When a client is configured to attempt a secure connection, but attempts to connect to an insecure server, that server may instruct the client to switch to SASL SIMPLE (unsecure) authentication. This setting controls whether or not the client will accept this instruction from the server. When false (the default), the client will not allow the fallback to SIMPLE authentication, and will abort the connection.

Default
false

hbase.ipc.server.fallback-to-simple-auth-allowed
Description
When a server is configured to require secure connections, it will reject connection attempts from clients using SASL SIMPLE (unsecure) authentication. This setting allows secure servers to accept SASL SIMPLE connections from clients when the client requests. When false (the default), the server will not allow the fallback to SIMPLE authentication, and will reject the connection. WARNING: This setting should ONLY be used as a temporary measure while converting clients over to secure authentication. It MUST BE DISABLED for secure operation.

Default
false

hbase.display.keys
Description
When this is set to true the webUI and such will display all start/end keys as part of the table details, region names, etc. When this is set to false, the keys are hidden.

Default
true

hbase.coprocessor.enabled
Description
Enables or disables coprocessor loading. If 'false' (disabled), any other coprocessor related configuration will be ignored.

Default
true

hbase.coprocessor.user.enabled
Description
Enables or disables user (aka. table) coprocessor loading. If 'false' (disabled), any table coprocessor attributes in table descriptors will be ignored. If "hbase.coprocessor.enabled" is 'false' this setting has no effect.

Default
true

hbase.coprocessor.region.classes
Description
A comma-separated list of Coprocessors that are loaded by default on all tables. For any override coprocessor method, these classes will be called in order. After implementing your own Coprocessor, just put it in HBase’s classpath and add the fully qualified class name here. A coprocessor can also be loaded on demand by setting HTableDescriptor.

Default
none

hbase.rest.port
Description
The port for the HBase REST server.

Default
8080

hbase.rest.readonly
Description
Defines the mode the REST server will be started in. Possible values are: false: All HTTP methods are permitted - GET/PUT/POST/DELETE. true: Only the GET method is permitted.

Default
false

hbase.rest.threads.max
Description
The maximum number of threads of the REST server thread pool. Threads in the pool are reused to process REST requests. This controls the maximum number of requests processed concurrently. It may help to control the memory used by the REST server to avoid OOM issues. If the thread pool is full, incoming requests will be queued up and wait for some free threads.

Default
100

hbase.rest.threads.min
Description
The minimum number of threads of the REST server thread pool. The thread pool always has at least these number of threads so the REST server is ready to serve incoming requests.

Default
2

hbase.rest.support.proxyuser
Description
Enables running the REST server to support proxy-user mode.

Default
false

hbase.defaults.for.version.skip
Description
Set to true to skip the 'hbase.defaults.for.version' check. Setting this to true can be useful in contexts other than the other side of a maven generation; i.e. running in an IDE. You’ll want to set this boolean to true to avoid seeing the RuntimeException complaint: "hbase-default.xml file seems to be for and old version of HBase (\${hbase.version}), this version is X.X.X-SNAPSHOT"

Default
false

hbase.coprocessor.master.classes
Description
A comma-separated list of org.apache.hadoop.hbase.coprocessor.MasterObserver coprocessors that are loaded by default on the active HMaster process. For any implemented coprocessor methods, the listed classes will be called in order. After implementing your own MasterObserver, just put it in HBase’s classpath and add the fully qualified class name here.

Default
none

hbase.coprocessor.abortonerror
Description
Set to true to cause the hosting server (master or regionserver) to abort if a coprocessor fails to load, fails to initialize, or throws an unexpected Throwable object. Setting this to false will allow the server to continue execution but the system wide state of the coprocessor in question will become inconsistent as it will be properly executing in only a subset of servers, so this is most useful for debugging only.

Default
true

hbase.table.lock.enable
Description
Set to true to enable locking the table in zookeeper for schema change operations. Table locking from master prevents concurrent schema modifications to corrupt table state.

Default
true

hbase.table.max.rowsize
Description
Maximum size of single row in bytes (default is 1 Gb) for Get’ting or Scan’ning without in-row scan flag set. If row size exceeds this limit RowTooBigException is thrown to client.

Default
1073741824

hbase.thrift.minWorkerThreads
Description
The "core size" of the thread pool. New threads are created on every connection until this many threads are created.

Default
16

hbase.thrift.maxWorkerThreads
Description
The maximum size of the thread pool. When the pending request queue overflows, new threads are created until their number reaches this number. After that, the server starts dropping connections.

Default
1000

hbase.thrift.maxQueuedRequests
Description
The maximum number of pending Thrift connections waiting in the queue. If there are no idle threads in the pool, the server queues requests. Only when the queue overflows, new threads are added, up to hbase.thrift.maxQueuedRequests threads.

Default
1000

hbase.regionserver.thrift.framed
Description
Use Thrift TFramedTransport on the server side. This is the recommended transport for thrift servers and requires a similar setting on the client side. Changing this to false will select the default transport, vulnerable to DoS when malformed requests are issued due to THRIFT-601.

Default
false

hbase.regionserver.thrift.framed.max_frame_size_in_mb
Description
Default frame size when using framed transport, in MB

Default
2

hbase.regionserver.thrift.compact
Description
Use Thrift TCompactProtocol binary serialization protocol.

Default
false

hbase.rootdir.perms
Description
FS Permissions for the root directory in a secure (kerberos) setup. When master starts, it creates the rootdir with this permissions or sets the permissions if it does not match.

Default
700

hbase.data.umask.enable
Description
Enable, if true, that file permissions should be assigned to the files written by the regionserver

Default
false

hbase.data.umask
Description
File permissions that should be used to write data files when hbase.data.umask.enable is true

Default
000

hbase.snapshot.enabled
Description
Set to true to allow snapshots to be taken / restored / cloned.

Default
true

hbase.snapshot.restore.take.failsafe.snapshot
Description
Set to true to take a snapshot before the restore operation. The snapshot taken will be used in case of failure, to restore the previous state. At the end of the restore operation this snapshot will be deleted

Default
true

hbase.snapshot.restore.failsafe.name
Description
Name of the failsafe snapshot taken by the restore operation. You can use the {snapshot.name}, {table.name} and {restore.timestamp} variables to create a name based on what you are restoring.

Default
hbase-failsafe-{snapshot.name}-{restore.timestamp}

hbase.server.compactchecker.interval.multiplier
Description
The number that determines how often we scan to see if compaction is necessary. Normally, compactions are done after some events (such as memstore flush), but if region didn’t receive a lot of writes for some time, or due to different compaction policies, it may be necessary to check it periodically. The interval between checks is hbase.server.compactchecker.interval.multiplier multiplied by hbase.server.thread.wakefrequency.

Default
1000

hbase.lease.recovery.timeout
Description
How long we wait on dfs lease recovery in total before giving up.

Default
900000

hbase.lease.recovery.dfs.timeout
Description
How long between dfs recover lease invocations. Should be larger than the sum of the time it takes for the namenode to issue a block recovery command as part of datanode; dfs.heartbeat.interval and the time it takes for the primary datanode, performing block recovery to timeout on a dead datanode; usually dfs.client.socket-timeout. See the end of HBASE-8389 for more.

Default
64000

hbase.column.max.version
Description
New column family descriptors will use this value as the default number of versions to keep.

Default
1

dfs.client.read.shortcircuit
Description
If set to true, this configuration parameter enables short-circuit local reads.

Default
false

dfs.domain.socket.path
Description
This is a path to a UNIX domain socket that will be used for communication between the DataNode and local HDFS clients, if dfs.client.read.shortcircuit is set to true. If the string "_PORT" is present in this path, it will be replaced by the TCP port of the DataNode. Be careful about permissions for the directory that hosts the shared domain socket; dfsclient will complain if open to other users than the HBase user.

Default
none

hbase.dfs.client.read.shortcircuit.buffer.size
Description
If the DFSClient configuration dfs.client.read.shortcircuit.buffer.size is unset, we will use what is configured here as the short circuit read default direct byte buffer size. DFSClient native default is 1MB; HBase keeps its HDFS files open so number of file blocks * 1MB soon starts to add up and threaten OOME because of a shortage of direct memory. So, we set it down from the default. Make it > the default hbase block size set in the HColumnDescriptor which is usually 64k.

Default
131072

hbase.regionserver.checksum.verify
Description
If set to true (the default), HBase verifies the checksums for hfile blocks. HBase writes checksums inline with the data when it writes out hfiles. HDFS (as of this writing) writes checksums to a separate file than the data file necessitating extra seeks. Setting this flag saves some on i/o. Checksum verification by HDFS will be internally disabled on hfile streams when this flag is set. If the hbase-checksum verification fails, we will switch back to using HDFS checksums (so do not disable HDFS checksums! And besides this feature applies to hfiles only, not to WALs). If this parameter is set to false, then hbase will not verify any checksums, instead it will depend on checksum verification being done in the HDFS client.

Default
true

hbase.hstore.bytes.per.checksum
Description
Number of bytes in a newly created checksum chunk for HBase-level checksums in hfile blocks.

Default
16384

hbase.hstore.checksum.algorithm
Description
Name of an algorithm that is used to compute checksums. Possible values are NULL, CRC32, CRC32C.

Default
CRC32C

hbase.client.scanner.max.result.size
Description
Maximum number of bytes returned when calling a scanner’s next method. Note that when a single row is larger than this limit the row is still returned completely. The default value is 2MB, which is good for 1ge networks. With faster and/or high latency networks this value should be increased.

Default
2097152

hbase.server.scanner.max.result.size
Description
Maximum number of bytes returned when calling a scanner’s next method. Note that when a single row is larger than this limit the row is still returned completely. The default value is 100MB. This is a safety setting to protect the server from OOM situations.

Default
104857600

hbase.status.published
Description
This setting activates the publication by the master of the status of the region server. When a region server dies and its recovery starts, the master will push this information to the client application, to let them cut the connection immediately instead of waiting for a timeout.

Default
false

hbase.status.publisher.class
Description
Implementation of the status publication with a multicast message.

Default
org.apache.hadoop.hbase.master.ClusterStatusPublisher$MulticastPublisher

hbase.status.listener.class
Description
Implementation of the status listener with a multicast message.

Default
org.apache.hadoop.hbase.client.ClusterStatusListener$MulticastListener

hbase.status.multicast.address.ip
Description
Multicast address to use for the status publication by multicast.

Default
226.1.1.3

hbase.status.multicast.address.port
Description
Multicast port to use for the status publication by multicast.

Default
16100

hbase.dynamic.jars.dir
Description
The directory from which the custom filter JARs can be loaded dynamically by the region server without the need to restart. However, an already loaded filter/co-processor class would not be un-loaded. See HBASE-1936 for more details. Does not apply to coprocessors.

Default
${hbase.rootdir}/lib

hbase.security.authentication
Description
Controls whether or not secure authentication is enabled for HBase. Possible values are 'simple' (no authentication), and 'kerberos'.

Default
simple

hbase.rest.filter.classes
Description
Servlet filters for REST service.

Default
org.apache.hadoop.hbase.rest.filter.GzipFilter

hbase.master.loadbalancer.class
Description
Class used to execute the regions balancing when the period occurs. See the class comment for more on how it works http://hbase.apache.org/devapidocs/org/apache/hadoop/hbase/master/balancer/StochasticLoadBalancer.html It replaces the DefaultLoadBalancer as the default (since renamed as the SimpleLoadBalancer).

Default
org.apache.hadoop.hbase.master.balancer.StochasticLoadBalancer

hbase.master.normalizer.class
Description
Class used to execute the region normalization when the period occurs. See the class comment for more on how it works http://hbase.apache.org/devapidocs/org/apache/hadoop/hbase/master/normalizer/SimpleRegionNormalizer.html

Default
org.apache.hadoop.hbase.master.normalizer.SimpleRegionNormalizer

hbase.rest.csrf.enabled
Description
Set to true to enable protection against cross-site request forgery (CSRF)

Default
false

hbase.rest-csrf.browser-useragents-regex
Description
A comma-separated list of regular expressions used to match against an HTTP request’s User-Agent header when protection against cross-site request forgery (CSRF) is enabled for REST server by setting hbase.rest.csrf.enabled to true. If the incoming User-Agent matches any of these regular expressions, then the request is considered to be sent by a browser, and therefore CSRF prevention is enforced. If the request’s User-Agent does not match any of these regular expressions, then the request is considered to be sent by something other than a browser, such as scripted automation. In this case, CSRF is not a potential attack vector, so the prevention is not enforced. This helps achieve backwards-compatibility with existing automation that has not been updated to send the CSRF prevention header.

Default
Mozilla.,Opera.

hbase.security.exec.permission.checks
Description
If this setting is enabled and ACL based access control is active (the AccessController coprocessor is installed either as a system coprocessor or on a table as a table coprocessor) then you must grant all relevant users EXEC privilege if they require the ability to execute coprocessor endpoint calls. EXEC privilege, like any other permission, can be granted globally to a user, or to a user on a per table or per namespace basis. For more information on coprocessor endpoints, see the coprocessor section of the HBase online manual. For more information on granting or revoking permissions using the AccessController, see the security section of the HBase online manual.

Default
false

hbase.procedure.regionserver.classes
Description
A comma-separated list of org.apache.hadoop.hbase.procedure.RegionServerProcedureManager procedure managers that are loaded by default on the active HRegionServer process. The lifecycle methods (init/start/stop) will be called by the active HRegionServer process to perform the specific globally barriered procedure. After implementing your own RegionServerProcedureManager, just put it in HBase’s classpath and add the fully qualified class name here.

Default
none

hbase.procedure.master.classes
Description
A comma-separated list of org.apache.hadoop.hbase.procedure.MasterProcedureManager procedure managers that are loaded by default on the active HMaster process. A procedure is identified by its signature and users can use the signature and an instant name to trigger an execution of a globally barriered procedure. After implementing your own MasterProcedureManager, just put it in HBase’s classpath and add the fully qualified class name here.

Default
none

hbase.coordinated.state.manager.class
Description
Fully qualified name of class implementing coordinated state manager.

Default
org.apache.hadoop.hbase.coordination.ZkCoordinatedStateManager

hbase.regionserver.storefile.refresh.period
Description
The period (in milliseconds) for refreshing the store files for the secondary regions. 0 means this feature is disabled. Secondary regions sees new files (from flushes and compactions) from primary once the secondary region refreshes the list of files in the region (there is no notification mechanism). But too frequent refreshes might cause extra Namenode pressure. If the files cannot be refreshed for longer than HFile TTL (hbase.master.hfilecleaner.ttl) the requests are rejected. Configuring HFile TTL to a larger value is also recommended with this setting.

Default
0

hbase.region.replica.replication.enabled
Description
Whether asynchronous WAL replication to the secondary region replicas is enabled or not. If this is enabled, a replication peer named "region_replica_replication" will be created which will tail the logs and replicate the mutations to region replicas for tables that have region replication > 1. If this is enabled once, disabling this replication also requires disabling the replication peer using shell or ReplicationAdmin java class. Replication to secondary region replicas works over standard inter-cluster replication.

Default
false

hbase.http.filter.initializers
Description
A comma separated list of class names. Each class in the list must extend org.apache.hadoop.hbase.http.FilterInitializer. The corresponding Filter will be initialized. Then, the Filter will be applied to all user facing jsp and servlet web pages. The ordering of the list defines the ordering of the filters. The default StaticUserWebFilter add a user principal as defined by the hbase.http.staticuser.user property.

Default
org.apache.hadoop.hbase.http.lib.StaticUserWebFilter

hbase.security.visibility.mutations.checkauths
Description
This property if enabled, will check whether the labels in the visibility expression are associated with the user issuing the mutation

Default
false

hbase.http.max.threads
Description
The maximum number of threads that the HTTP Server will create in its ThreadPool.

Default
10

hbase.replication.rpc.codec
Description
The codec that is to be used when replication is enabled so that the tags are also replicated. This is used along with HFileV3 which supports tags in them. If tags are not used or if the hfile version used is HFileV2 then KeyValueCodec can be used as the replication codec. Note that using KeyValueCodecWithTags for replication when there are no tags causes no harm.

Default
org.apache.hadoop.hbase.codec.KeyValueCodecWithTags

hbase.replication.source.maxthreads
Description
The maximum number of threads any replication source will use for shipping edits to the sinks in parallel. This also limits the number of chunks each replication batch is broken into. Larger values can improve the replication throughput between the master and slave clusters. The default of 10 will rarely need to be changed.

Default
10

hbase.http.staticuser.user
Description
The user name to filter as, on static web filters while rendering content. An example use is the HDFS web UI (user to be used for browsing files).

Default
dr.stack

hbase.regionserver.handler.abort.on.error.percent
Description
The percent of region server RPC threads failed to abort RS. -1 Disable aborting; 0 Abort if even a single handler has died; 0.x Abort only when this percent of handlers have died; 1 Abort only all of the handers have died.

Default
0.5

hbase.mob.file.cache.size
Description
Number of opened file handlers to cache. A larger value will benefit reads by providing more file handlers per mob file cache and would reduce frequent file opening and closing. However, if this is set too high, this could lead to a "too many opened file handlers" The default value is 1000.

Default
1000

hbase.mob.cache.evict.period
Description
The amount of time in seconds before the mob cache evicts cached mob files. The default value is 3600 seconds.

Default
3600

hbase.mob.cache.evict.remain.ratio
Description
The ratio (between 0.0 and 1.0) of files that remains cached after an eviction is triggered when the number of cached mob files exceeds the hbase.mob.file.cache.size. The default value is 0.5f.

Default
0.5f

hbase.mob.sweep.tool.compaction.ratio
Description
If there’re too many cells deleted in a mob file, it’s regarded as an invalid file and needs to be merged. If existingCellsSize/mobFileSize is less than ratio, it’s regarded as an invalid file. The default value is 0.5f.

Default
0.5f

hbase.mob.sweep.tool.compaction.mergeable.size
Description
If the size of a mob file is less than this value, it’s regarded as a small file and needs to be merged. The default value is 128MB.

Default
134217728

hbase.mob.sweep.tool.compaction.memstore.flush.size
Description
The flush size for the memstore used by sweep job. Each sweep reducer owns such a memstore. The default value is 128MB.

Default
134217728

hbase.master.mob.ttl.cleaner.period
Description
The period that ExpiredMobFileCleanerChore runs. The unit is second. The default value is one day. The MOB file name uses only the date part of the file creation time in it. We use this time for deciding TTL expiry of the files. So the removal of TTL expired files might be delayed. The max delay might be 24 hrs.

Default
86400

hbase.mob.compaction.mergeable.threshold
Description
If the size of a mob file is less than this value, it’s regarded as a small file and needs to be merged in mob compaction. The default value is 192MB.

Default
201326592

hbase.mob.delfile.max.count
Description
The max number of del files that is allowed in the mob compaction. In the mob compaction, when the number of existing del files is larger than this value, they are merged until number of del files is not larger this value. The default value is 3.

Default
3

hbase.mob.compaction.batch.size
Description
The max number of the mob files that is allowed in a batch of the mob compaction. The mob compaction merges the small mob files to bigger ones. If the number of the small files is very large, it could lead to a "too many opened file handlers" in the merge. And the merge has to be split into batches. This value limits the number of mob files that are selected in a batch of the mob compaction. The default value is 100.

Default
100

hbase.mob.compaction.chore.period
Description
The period that MobCompactionChore runs. The unit is second. The default value is one week.

Default
604800

hbase.mob.compactor.class
Description
Implementation of mob compactor, the default one is PartitionedMobCompactor.

Default
org.apache.hadoop.hbase.mob.compactions.PartitionedMobCompactor

hbase.mob.compaction.threads.max
Description
The max number of threads used in MobCompactor.

Default
1

hbase.snapshot.master.timeout.millis
Description
Timeout for master for the snapshot procedure execution

Default
300000

hbase.snapshot.region.timeout
Description
Timeout for regionservers to keep threads in snapshot request pool waiting

Default
300000



8、配置举例
8.1 基本分布式HBase 安装
这里是一个由10个节点组成的分布式集群的基本配置举例：节点example0、example1等
The HBase Master and the HDFS NameNode are running on the node example0. * RegionServers run on nodes example1-example9. * A 3-node ZooKeeper ensemble runs on example1, example2, and example3 on the default ports. * ZooKeeper data is persisted to the directory /export/zookeeper.
主要配置文件hbase-site.xml, regionservers, and hbase-env.sh
8.1.1. hbase-site.xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property>
    <name>hbase.zookeeper.quorum</name>
    <value>example1,example2,example3</value>
    <description>The directory shared by RegionServers.
    </description>
  </property>
  <property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>/export/zookeeper</value>
    <description>Property from ZooKeeper config zoo.cfg.
    The directory where the snapshot is stored.
    </description>
  </property>
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs://example0:8020/hbase</value>
    <description>The directory shared by RegionServers.
    </description>
  </property>
  <property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
    <description>The mode the cluster will be in. Possible values are
      false: standalone and pseudo-distributed setups with managed ZooKeeper
      true: fully-distributed with unmanaged ZooKeeper Quorum (see hbase-env.sh)
    </description>
  </property>
</configuration>

8.1.2. regionservers
In this file you list the nodes that will run RegionServers. In our case, these nodes are example1-example9.
example1
example2
example3
example4
example5
example6
example7
example8
example9

9、重要配置
下面我们列举来一些重要的配置，这小结分为要求的配置和值得一看建议的配置
9.1 必须的配置
9.1.1 大集群配置
如果你有一个具有很多region的集群，一个RS在master刚刚启动后启动（check in）而其他RS还没有是很有可能的，第一个登记的server将被分配所有区域，但这可能不是最优的。要阻止这件事发生up thehbase.master.wait.on.regionservers.mintostart property from its default value of 1.
9.2建议的配置
9.2.1 ZK 配置
zookeeper.session.timeout
默认的超时时间是3mins，意味着如果一个server宕机了，master要花3mins来察觉并恢复它，你或许想将这个时间调低到1mins或更少，这样master可以更快发现错误。但在改变之前，你要确认你的JVM的garbage collection配置否则一个稍长时间的garbage collection就可能超过ZK session timeout。
编辑hbase-site.xml并复制到集群中并重启来改变这个配置

9.2.2 HDFS配置
dfs.datanode.failed.volumes.tolerated
这个是一个DN正常服务允许坏掉的卷的数量，默认情况下任意卷出错会导致DN关闭 ，你可能回想设置这个到你硬盘的一半

9.2.3  hbase.regionserver.handler.count
这个设定决定了响应用户请求的线程数量。经验做法是当请求很大时保持这个值较低，当请求内容较小时，把这个值放大。如果客户端请求很小，把这个值设置的和最大客户端数量一样大是安全的。一个典型的例子是给网站服务的集群，因为上传一般不会缓冲，而大部分操作都是get
将这个值设置的很大很危险，原因是当前在一个RS上发生的过大的输入值会给内存带来太大压力，或甚至触发一个OutOfMemoryError。一个RS运行在低内存下，可能会频繁触发JVM的garbage collector的运行，渐渐会感到停顿（因为所有请求内容所占用的内存不会被释放，无论GC执行多少遍）。一段时间后，整个集群会收到影响，因为每个给RS的请求都会变慢，最终问题会被加剧
9.2.4大内存机器配置
HBase有一个合理的保守的配置，这样可以运作在所有的机器上。如果你有台大内存的集群-HBase有8G或者更大的heap,接下来的配置可能会帮助你. TODO.
9.2.5 压缩
应该考虑启用ColumnFamily压缩，有好几个选项，通过减少存储文件大小来减少I/O，最终提高性能
9.2.6配置WAL文件的大小和数量
HBase使用wal来恢复没有及时写入硬盘中的memstore数据，当发生RS错误时。一般wal略小于HDFS块(by default a HDFS block is 64Mb and a WAL file is ~60Mb).
Hbase 也对wal数量有限制，这样可以保证系统恢复时不需要太多的数据恢复。这个设置预memstore的配置有关，这样所有必要数据都能适用。It is recommended to allocate enough WAL files to store at least that much data (when all memstores are close to full). For example, with 16Gb RS heap, default memstore settings (0.4), and default WAL file size (~60Mb), 16Gb*0.4/60, the starting point for WAL file count is ~109. However, as all memstores are not expected to be full all the time, less WAL files can be allocated.


