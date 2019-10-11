# Flume 搭建高可用采集数据

### 1.配置flume-Agent

#### agent.conf  主机245

~~~
#agent1 name
agent1.channels = c1
agent1.sources = r1
agent1.sinks = k1 k2
#
##set 设置组，保证后面两台flume在一个组下
agent1.sinkgroups = g1
#
##set 设置管道
agent1.channels.c1.type = memory
agent1.channels.c1.capacity = 1000
agent1.channels.c1.transactionCapacity = 100

agent1.channels.c1.byteCapacity=800000

#监控文件
agent1.sources.r1.channels = c1
#监控文件夹
agent1.sources.r1.type = TAILDIR
#元数据保存位置
agent1.sources.r1.positionFile = /home/data/flume_log/taildir_position.json
agent1.sources.r1.filegroups = f1
agent1.sources.r1.filegroups.f1 = /home/data/yuniko_flume_text/.*
agent1.sources.r1.fileHeader = true
#
agent1.sources.r1.interceptors = i1 i2
agent1.sources.r1.interceptors.i1.type = static
agent1.sources.r1.interceptors.i1.key = Type
agent1.sources.r1.interceptors.i1.value = LOGIN
agent1.sources.r1.interceptors.i2.type = timestamp
#
## set sink1
agent1.sinks.k1.channel = c1
agent1.sinks.k1.type = avro
agent1.sinks.k1.hostname = node246
agent1.sinks.k1.port = 52022
#
## set sink2
agent1.sinks.k2.channel = c1
agent1.sinks.k2.type = avro
agent1.sinks.k2.hostname = node247
agent1.sinks.k2.port = 52022
#
##set sink group
agent1.sinkgroups.g1.sinks = k1 k2
#
##set failover
agent1.sinkgroups.g1.processor.type = failover
agent1.sinkgroups.g1.processor.priority.k1 = 10
agent1.sinkgroups.g1.processor.priority.k2 = 1
agent1.sinkgroups.g1.processor.maxpenalty = 10000

~~~

#### collect_1.conf   主机246

~~~
#set Agent name
a1.sources = r1
a1.channels = c1
a1.sinks = k1
#
##set channel
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
#
## other node,nna to nns
a1.sources.r1.type = avro
a1.sources.r1.bind = node246
a1.sources.r1.port = 52022
a1.sources.r1.interceptors = i1
a1.sources.r1.interceptors.i1.type = static
a1.sources.r1.interceptors.i1.key = Collector
a1.sources.r1.interceptors.i1.value = node246
a1.sources.r1.channels = c1
#
##set sink to hdfs
a1.sinks.k1.type=hdfs
a1.sinks.k1.hdfs.path= hdfs://node245:9000/flume/failover_yuniko/customer_release/%Y-%m-%d
a1.sinks.k1.hdfs.fileType=DataStream
a1.sinks.k1.hdfs.writeFormat=TEXT
a1.sinks.k1.hdfs.rollInterval=0
a1.sinks.k1.channel=c1
a1.sinks.k1.hdfs.rollCount=0
a1.sinks.k1.hdfs.rollSize=134217700
a1.sinks.k1.hdfs.filePrefix=%Y-%m-%d

~~~

#### collect_2.conf    主机 node247 

~~~
#set Agent name
a1.sources = r1
a1.channels = c1
a1.sinks = k1
#
##set channel
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
#
## other node,nna to nns
a1.sources.r1.type = avro
a1.sources.r1.bind = node247
a1.sources.r1.port = 52022
a1.sources.r1.interceptors = i1
a1.sources.r1.interceptors.i1.type = static
a1.sources.r1.interceptors.i1.key = Collector
a1.sources.r1.interceptors.i1.value = node247
a1.sources.r1.channels = c1
#
##set sink to hdfs
a1.sinks.k1.type=hdfs
a1.sinks.k1.hdfs.path= hdfs://node245:9000/flume/failover_yuniko/customer_release/%Y-%m-%d
a1.sinks.k1.hdfs.fileType=DataStream
a1.sinks.k1.hdfs.writeFormat=TEXT
a1.sinks.k1.hdfs.rollInterval=0
a1.sinks.k1.hdfs.rollCount=0
a1.sinks.k1.hdfs.rollSize=134217700
a1.sinks.k1.channel=c1
a1.sinks.k1.hdfs.filePrefix=%Y-%m-%d

~~~

#### 启动以及使用 

> 先启动collect 在启动agent  
>
> node247命令
> ./bin/flume-ng agent -n a1 -c conf -f conf/collect_2.conf -Dflume.root.logger=DEBUG,console
> node246命令
> ./bin/flume-ng agent -n a1 -c conf -f conf/collect_1.conf -Dflume.root.logger=DEBUG,console
> node245命令
> bin/flume-ng agent -n agent1 -c conf -f conf/agent.conf -Dflume.root.logger=DEBUG,console
>
> 参数解释：
>
> -n 指定agent名称，要与配置文件一致（等价于--name)
>
> -c 指定配置文件位置（等价于 --conf）
>
> -f 指定agent配置文件（等价 --conf-file）
>
> -Dflume.root.logger=INFO,console 设置日志等级 

#### 注意事项

> 1.启动顺序不能错误
>
> 2.注意设置sink端的 收集设置 
>
> a1.sinks.k1.hdfs.rollInterval=0   // hdfs sink间隔多长将临时文件滚动成最终目标文件
> a1.sinks.k1.hdfs.rollCount=0   //当events数据达到该数量时候，将临时文件滚动成目标文件
> a1.sinks.k1.hdfs.rollSize=134217700 //当临时文件达到该大小（单位：bytes）时，滚动成目标文件；
>
> 3.往/home/data/yuniko_flume_text/ source目录里放文件（不能是文件夹）
>
> 4.如果报错event 失败  修改参数 agent1.channels.c1.byteCapacity=800000