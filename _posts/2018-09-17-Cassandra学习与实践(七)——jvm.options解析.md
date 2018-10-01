---
layout: post
title:  "Cassandra学习与实践(七)——jvm.options解析"
date:   2018-09-17 12:00:00 +0800
categories: nosql
---

#### *参考资料*

- *Cassandra官方文档 [http://cassandra.apache.org/doc/latest/faq/index.html](http://cassandra.apache.org/doc/latest/faq/index.html)*

  ​



<br>

Cassandra的JVM配置可以在`jvm.options`配置文件中设置，当Cassandra启动时，`cassandra-env.sh`脚本会读取`jvm.options`并加载到`JVM_OPTS`环境变量中，最终这些参数将被传递给JVM

<br>

```shell
###########################################################################
#                             jvm.options                                 #
#                                                                         #
# - all flags defined here will be used by cassandra to startup the JVM   #
# - one flag should be specified per line                                 #
# - lines that do not start with '-' will be ignored                      #
# - only static flags are accepted (no variables or parameters)           #
# - dynamic flags will be appended to these on cassandra-env              #
###########################################################################

######################
# STARTUP PARAMETERS #
# 启动参数            #     
######################

# Uncomment any of the following properties to enable specific startup parameters
# 取消以下任何属性的注释可启用特定的启动参数

# In a multi-instance deployment, multiple Cassandra instances will independently assume that all
# 在多实例部署中，假设多个Cassandra实例会独立的部署
# CPU processors are available to it. This setting allows you to specify a smaller set of processors and perhaps have affinity.
# CPU处理器可用。此设置允许您指定较小的处理器集以有更好的兼容
#-Dcassandra.available_processors=number_of_processors

# The directory location of the cassandra.yaml file.
# 指定cassandra.yaml配置文件的位置，注释的话默认在同一个文件夹内
#-Dcassandra.config=directory

# Sets the initial partitioner token for a node the first time the node is started.
# 在第一次启动Cassandra节点时为节点设置初始的partitioner token ???
#-Dcassandra.initial_token=token

# Set to false to start Cassandra on a node but not have the node join the cluster.
# 设置为false以在节点上启动Cassandra但不让节点加入群集
#-Dcassandra.join_ring=true|false

# Set to false to clear all gossip state for the node on restart. Use when you have changed node information in cassandra.yaml (such as listen_address).
# 设置为false会在节点重新启动时更新节点的所有gossip状态。在cassandra.yaml中更改节点信息时使用（例如listen_address）
#-Dcassandra.load_ring_state=true|false

# Enable pluggable metrics reporter. See Pluggable metrics reporting in Cassandra 2.0.2.
# 启用可插入指标报告器。请参阅Cassandra 2.0.2中的可插入指标报告
#-Dcassandra.metricsReporterConfigFile=file

# Set the port on which the CQL native transport listens for clients. (Default: 9042)
# 设置CQL本机传输侦听客户端的端口。（默认：9042）
#-Dcassandra.native_transport_port=port

# Overrides the partitioner. (Default: org.apache.cassandra.dht.Murmur3Partitioner)
# 覆盖partitioner （默认：org.apache.cassandra.dht.Murmur3Partitioner）
#-Dcassandra.partitioner=partitioner

# To replace a node that has died, restart a new node in its place specifying the address of the dead node. The new node must not have any data in its data directory, that is, it must be in the same state as before bootstrapping.
# 如果要替换已宕机的节点，请在启动时指定宕机节点的ip地址。新节点的数据目录中不得包含任何数据，也就是说，它必须处于引导前的状态
#-Dcassandra.replace_address=listen_address or broadcast_address of dead node

# Allow restoring specific tables from an archived commit log.
# 允许从归档的commit log中恢复特定的表
#-Dcassandra.replayList=table

# Allows overriding of the default RING_DELAY (30000ms), which is the amount of time a node waits before joining the ring.
# 节点在加入环之前等待的时间，默认的RING_DELAY为30000ms
#-Dcassandra.ring_delay_ms=ms

# Set the port for the Thrift RPC service, which is used for client connections. (Default: 9160)
# 用于客户端连接的Thrift RPC服务端口，默认：9160
#-Dcassandra.rpc_port=port

# Set the SSL port for encrypted communication. (Default: 7001)
# 加密通信的SSL端口，默认：7001
#-Dcassandra.ssl_storage_port=port

# Enable or disable the native transport server. See start_native_transport in cassandra.yaml.
# 启用或禁用本地传输服务器。见start_native_transport在cassandra.yaml，默认：true
# 绑定的地址与cassandra.yaml中的rpc_address相同，端口是不同的，通过native_transport_port指定，默认为9042
# cassandra.start_native_transport=true|false

# Enable or disable the Thrift RPC server. (Default: true)
# 启用或禁用Thrift RPC服务，默认：true
#-Dcassandra.start_rpc=true/false

# Set the port for inter-node communication. (Default: 7000)
# 节点间通信的端口，默认：7000
#-Dcassandra.storage_port=port

# Set the default location for the trigger JARs. (Default: conf/triggers)
# 设置触发器JAR的默认位置
#-Dcassandra.triggers_dir=directory

# For testing new compaction and compression strategies. It allows you to experiment with different strategies and benchmark write performance differences without affecting the production workload. 
# 启用测试新压缩和压缩策略，允许您在不影响生产工作负载的情况下尝试不同的策略和基准写入性能差异。请参阅测试压实和压缩
#-Dcassandra.write_survey=true

# To disable configuration via JMX of auth caches (such as those for credentials, permissions and roles). This will mean those config options can only be set (persistently) in cassandra.yaml and will require a restart for new values to take effect.
# 禁用JMX身份验证缓存（例如凭据，权限和角色的配置），这意味着那些配置选项只能在cassandra.yaml中设置（持久），并且需要重新启动才能使新值生效
#-Dcassandra.disable_auth_caches_remote_configuration=true

# To disable dynamic calculation of the page size used when indexing an entire partition (during initial index build/rebuild). If set to true, the page size will be fixed to the default of 10000 rows per page.
# 禁用动态计算页面大小当索引整个分区时（在初始索引构建/重建期间）。如果设置为true，则页面大小将固定为每页10000行的默认值
#-Dcassandra.force_default_indexing_page_size=true

########################
# GENERAL JVM SETTINGS #
# 通用JVM设置           #
########################

# enable assertions. highly suggested for correct application functionality.
# 启用断言。强烈建议正确使用应用程序的功能
# 从JDK1.4开始，java可支持断言机制，用于诊断运行时问题。通常在测试阶段使断言有效，在正式运行时不需要运行断言。断言后的表达式的值是一个逻辑值，为true时断言不运行，为false时断言运行，抛出java.lang.AssertionError错误。缺省时虚拟机关闭断言机制，用-ea可打开断言机制
-ea

# enable thread priorities, primarily so we can give periodic tasks a lower priority to avoid interfering with client workload
# 启用线程优先级，主要是因为我们可以为周期性任务提供较低的优先级，以避免干扰客户端工作负载
-XX:+UseThreadPriorities

# allows lowering thread priority without being root on linux - probably
# not necessary on Windows but doesn't harm anything.
# see http://tech.stolsvik.com/2010/01/linux-java-thread-priorities-workar
# 非linux的root用户时允许降低线程优先级 - 可能在Windows上不是必需的但不会损害任何东西
-XX:ThreadPriorityPolicy=42

# Enable heap-dump if there's an OOM
# 发生内存溢出时导出dump文件
-XX:+HeapDumpOnOutOfMemoryError

# Per-thread stack size.
# 每个线程的堆栈大小
-Xss256k

# Larger interned string table, for gossip's benefit (CASSANDRA-6410)
# 更大的StringTable，对gossip协议有好处
# String.intern()被调用时会往Hashtable插入一个String（若该String不存在），这里的Table就是StringTable，此参数就是这个StringTable的大小
-XX:StringTableSize=1000003

# Make sure all memory is faulted and zeroed on startup.
# This helps prevent soft faults in containers and makes
# transparent hugepage allocation more effective.
# JAVA进程启动的时候,虽然我们可以为JVM指定合适的内存大小,但是这些内存操作系统并没有真正的分配给JVM,而是等JVM访问这些内存的时候,才真正分配,这样会造成以下问题
# 1. GC的时候,新生代的对象要晋升到老年代的时候,需要内存,这个时候操作系统才真正分配内存,这样就会加大young gc的停顿时间;
# 2. 可能存在内存碎片的问题
# 可以在JVM启动的时候,配置-XX:+AlwaysPreTouch参数,这样JVM就会先访问所有分配给它的内存,让操作系统把内存真正的分配给JVM.后续JVM就可以顺畅的访问内存了
-XX:+AlwaysPreTouch

# Disable biased locking as it does not benefit Cassandra.
# 禁用偏向锁, 因为它对Cassandra没有好处
# 偏向锁的目的是为了在无锁竞争的情况下避免在锁获取过程中执行不必要的CAS原子指令；现有的CAS原子指令虽然相对于重量级锁来说开销比较小但还是存在非常可观的本地延迟。而偏向锁则针对拥有当前锁的线程，允许其在竞争不存在的情况下，直接进入同步的代码块，无需同步操作，从而获取了相当的性能提升
# 在偏向锁的应用场景主要集中在竞争不激烈的情况下，通过使用偏向锁可以减少其在CAS操作下的同步性能消耗，从而获取性能的提升。关键点在于是否存在激烈的锁竞争，如果存在则不适合使用它。默认情况下是被打开的
-XX:-UseBiasedLocking

# Enable thread-local allocation blocks and allow the JVM to automatically
# resize them at runtime.
# 启用线程局部分配块，并允许JVM在运行时自动调整它们的大小
# TLAB Thread Local Allocation Buffer，JDK1.7默认开启TLAB一般不需要去设置，TLAB极大提高程序性能，它是Java的一个优化方案
-XX:+UseTLAB		# 开启TLAB
-XX:+ResizeTLAB		# 自调整TLABRefillWasteFraction 阀值
-XX:+UseNUMA        # 使用NUMA开启性能优化，默认不开启，该项只有在开启了-XX:+UseParallelGC后才有效

# http://www.evanjones.ca/jvm-mmap-pause.html
# 减少高IO时的JVM停顿，原理请见http://www.evanjones.ca/jvm-mmap-pause.html
-XX:+PerfDisableSharedMem

# Prefer binding to IPv4 network intefaces (when net.ipv6.bindv6only=1). See
# http://bugs.sun.com/bugdatabase/view_bug.do?bug_id=6342561 (short version:
# comment out this entry to enable IPv6 support).
# 首选绑定到IPv4网络接口（当net.ipv6.bindv6only = 1时），注释掉此条目可启用IPv6支持
# 详情请参阅http://bugs.sun.com/bugdatabase/view_bug.do?bug_id=6342561
-Djava.net.preferIPv4Stack=true

### Debug options

# uncomment to enable flight recorder
# 取消注释以启用飞行记录器（这两个选项要一起使用）
# Java飞行记录器（JFR）会收集Java应用程序以及Java VM的行为信息。JFR构建在了Java VM之中，能够为用户提供运行时的信息。使用JFR并不会影响其他的Java VM优化，它的最小开销会小于2%
#-XX:+UnlockCommercialFeatures
#-XX:+FlightRecorder

# uncomment to have Cassandra JVM listen for remote debuggers/profilers on port 1414
# 取消注释让Cassandra JVM在端口1414上侦听远程调试器/分析器 ???
# -agentlib:libname[=options] 这个命令加载指定的native agent库。理论上这条option出现后，JVM会到本地固定路径下LD_LIBRARY_PATH这里加载名字为libxxx.so的库。而这样的库理论上是JVM TI的功能，具体可以参考JVM TI规范
#-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=1414

# uncomment to have Cassandra JVM log internal method compilation (developers only)
# -XX:+LogCompilation开启编译时日志输出，在编译时会有一个hotspot.log的日志输出到当前工作目录下。可以用-XX:LogFile指定不同的目录。默认这个参数是关闭的，即编译日志不输出。这个参数需要和-XX:UnlockDiagnosticVMOptions一起使用。也可以使用-XX:+PrintCompilation选项在控制台打印编译过程信息
#-XX:+UnlockDiagnosticVMOptions
#-XX:+LogCompilation

#################
# HEAP SETTINGS #
#################

# Heap size is automatically calculated by cassandra-env based on this
# 堆大小由cassandra-env.sh 脚本根据操作系统内存自动计算，规则如下
# formula: max(min(1/2 ram, 1024MB), min(1/4 ram, 8GB))
# That is:
# - calculate 1/2 ram and cap to 1024MB
# - calculate 1/4 ram and cap to 8192MB
# - pick the max
#
# For production use you may wish to adjust this for your environment.
# If that's the case, uncomment the -Xmx and Xms options below to override the
# automatic calculation of JVM heap memory.
# 对于生产环境，希望您可以根据环境调整
# 如果是这种情况，请取消注释下面的-Xmx和Xms选项以覆盖JVM堆内存的自动计算
# 
# It is recommended to set min (-Xms) and max (-Xmx) heap sizes to
# the same value to avoid stop-the-world GC pauses during resize, and
# so that we can lock the heap in memory on startup to prevent any
# of it from being swapped out.
# 建议将 min (-Xms) 和 max (-Xmx) 堆大小设置为相同的值，以避免在调整大小期间GC暂停stop-the-world，所以我们可以在启动时就锁定内存中将来要使用到部分，防止任何内容被换出
#-Xms4G
#-Xmx4G

# Young generation size is automatically calculated by cassandra-env
# based on this formula: min(100 * num_cores, 1/4 * heap size)
# 年轻代大小由cassandra-env.sh 脚本根据以下公式自动计算：min(100 * num_cores, 1/4 * heap size)
#
# The main trade-off for the young generation is that the larger it
# is, the longer GC pause times will be. The shorter it is, the more
# expensive GC will be (usually).
#
# It is not recommended to set the young generation size if using the
# G1 GC, since that will override the target pause-time goal.
# More info: http://www.oracle.com/technetwork/articles/java/g1gc-1984535.html
#
# The example below assumes a modern 8-core+ machine for decent
# times. If in doubt, and if you do not particularly want to tweak, go
# 100 MB per physical CPU core.
#-Xmn800M

###################################
# EXPIRATION DATE OVERFLOW POLICY #
###################################

# Defines how to handle INSERT requests with TTL exceeding the maximum supported expiration date:
# * REJECT: this is the default policy and will reject any requests with expiration date timestamp after 2038-01-19T03:14:06+00:00.
# * CAP: any insert with TTL expiring after 2038-01-19T03:14:06+00:00 will expire on 2038-01-19T03:14:06+00:00 and the client will receive a warning.
# * CAP_NOWARN: same as previous, except that the client warning will not be emitted.
#
#-Dcassandra.expiration_date_overflow_policy=REJECT

#################
#  GC SETTINGS  #
#################

### CMS Settings

-XX:+UseParNewGC
-XX:+UseConcMarkSweepGC
-XX:+CMSParallelRemarkEnabled
-XX:SurvivorRatio=8
-XX:MaxTenuringThreshold=1
-XX:CMSInitiatingOccupancyFraction=75
-XX:+UseCMSInitiatingOccupancyOnly
-XX:CMSWaitDuration=10000
-XX:+CMSParallelInitialMarkEnabled
-XX:+CMSEdenChunksRecordAlways
# some JVMs will fill up their heap when accessed via JMX, see CASSANDRA-6541
-XX:+CMSClassUnloadingEnabled

### G1 Settings (experimental, comment previous section and uncomment section below to enable)

## Use the Hotspot garbage-first collector.
#-XX:+UseG1GC
#
## Have the JVM do less remembered set work during STW, instead
## preferring concurrent GC. Reduces p99.9 latency.
#-XX:G1RSetUpdatingPauseTimePercent=5
#
## Main G1GC tunable: lowering the pause target will lower throughput and vise versa.
## 200ms is the JVM default and lowest viable setting
## 1000ms increases throughput. Keep it smaller than the timeouts in cassandra.yaml.
#-XX:MaxGCPauseMillis=500

## Optional G1 Settings

# Save CPU time on large (>= 16GB) heaps by delaying region scanning
# until the heap is 70% full. The default in Hotspot 8u40 is 40%.
#-XX:InitiatingHeapOccupancyPercent=70

# For systems with > 8 cores, the default ParallelGCThreads is 5/8 the number of logical cores.
# Otherwise equal to the number of cores when 8 or less.
# Machines with > 10 cores should try setting these to <= full cores.
#-XX:ParallelGCThreads=16
# By default, ConcGCThreads is 1/4 of ParallelGCThreads.
# Setting both to the same value can reduce STW durations.
#-XX:ConcGCThreads=16

### GC logging options -- uncomment to enable

-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+PrintHeapAtGC
-XX:+PrintTenuringDistribution
-XX:+PrintGCApplicationStoppedTime
-XX:+PrintPromotionFailure
#-XX:PrintFLSStatistics=1
#-Xloggc:/var/log/cassandra/gc.log
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=10
-XX:GCLogFileSize=10M
```

