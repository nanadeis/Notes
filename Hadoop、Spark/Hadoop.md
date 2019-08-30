# Hadoop和关系型数据库的比较     
1. 因为磁盘的寻址时间大于传输时间（传输速率取决于硬盘的带宽）   
MapReduce比较适合以批处理方式处理需要分析整个数据集的问题，尤其是动态分析。MapReduce适合一次写入，多次读取数据的应用    
关系型数据库适合点查询和更新，数据集创建索引之后，数据库系统能够提供低延迟的数据检索和快速的少量数据更新。持续更新的数据集     
2. mapreduce适合处理非结构化数据  

# HDFS block、packet、chunk
## block  
如果块设置得足够大，从硬盘传输数据的时间会明显大于定位这个块开始位置所需的时间，可以节约寻址开销  
一个map任务一次处理一个数据块，块太大会导致任务数太少   
## packet  
client向DataNode，或DataNode的pipeline之间传数据的基本单位，默认64KB  
## chunk  
client向datanode或datanode的pipeline之间进行数据校验的基本单位，默认512Byte  

# HDFS特性  
1. 流式数据访问，一次只能读一个块，不能只读块里某行，而且没有索引，只能顺序读；只能在后面追加写，不支持随机读写  
2. 简单一致性模型，假定文件是一次写入，多次读取  
3. 不支持低延迟数据访问  
4. 不支持并发写入，一个文件只能有一个写入者  
5. 不适合大量小文件存储，因为每个元数据占用空间是一定的  

# HDFS
## namenode  
维护着文件系统树和整棵树内所有的文件和目录。这些信息以两个文件形式用久保存在本地磁盘上：命名空间镜像文件（fsImage）和编辑日志文件（edits）    

命名空间镜像保存着某一特定时刻HDFS的目录树、元信息和数据块索引（每个文件对应的数据块列表）等信息，后续对这些信息的改动，则保存在编辑日志中，它们一起提供了一个完整的namenode第一关系  

【文件的元信息】  
replication level、修改时间、访问时间、访问许可、块大小、组成该文件的块等  
【目录的元信息】  
修改时间、访问许可、配额元数据  

文件系统客户端执行写操作时，这些操作首先为记录到编辑日志中。namenode在聂村中维护文件系统的元数据，当编辑日志被修改时，内存中的相关元数据也同步更新。如果namenode发生故障，可以先把fsImage加载到内存重构新进的元数据，再执行编辑日志中记录的各项操作  

namenode也记录着每个文件中各个块所在的数据节点信息，但它并不永久保存块的位置信息，因为这些信息会在系统启动时由数据节点重建，各个块所在的数据节点信息称为namenode第二关系   
namenode还能获取hdfs整体运行状态的一些信息，如系统的可用空间、已经使用的空间、各数据节点的当前状态等     
## secondary namenode  
用于定期合并命名空间镜像和镜像编辑日志的辅助守护进程     
与namenode的区别在于它不接收和记录HDFS的任何实时变化，而只是根据集群配置的时间间隔或编辑日志大小达到64MB，获取HDFS某一时刻的命名空间镜像和镜像的编辑日志，合并得到一个新的命名空间镜像，该新镜像会上传到namenode，替换原有的命名空间镜像，并清空上述日志。 
1. secondary NameNode 请求NameNode停止使用edits文件，暂时将新的写操作记录到一个新文件中  
2. secondary NameNode从主那么node获取fsImage和edits文件（采用HTTP GET）  
3. secondary NameNode将fsImage文件载入内存，逐一执行edits文件中的操作，创建新的fsimage文件  
4. secondary NameNode将新的fsimage文件发送回主namenode（使用HTTP POST）  
5. NameNode用从secondary NameNode接收的fsImage文件替换旧的fsImage文件，用步骤1所产生的edits文件替换旧的edits文件。同时，还更新fsImage文件来记录检查点执行的时间  

secondary namenode为NameNode的第一关系提供了一个checkpoint机制，并避免出现编辑日志过大，导致namenode启动时间过长的问题。secondary NameNode有一份NameNode的备份，但不是最新的     

## DataNode  
存储并检索数据块，定期向namenode发送它们所存储的块的列表   

# HDFS读流程
1. client访问namenode，查询元数据信息、获得这个文件的数据块位置列表，返回输入流对象  
2. 就近挑选一台datanode服务器，请求建立输入流。读到块的末端时，关闭与该datanode的连接，寻找下一个块的最佳datanode  
3. 关闭输入流

# HDFS写流程  
1. 客户端向namenode发出写文件请求  
2. namenode检查是否已存在文件，检查权限。若通过检查，直接先将操作写入编辑日志，并返回输出流对象  
3. client块切分文件  
4. client将namenode返回的分配的可写的datanode列表和data数据一同发送给最近的第一个datanode节点，此后client端和namenode分配的多个datanode构成pipeline（client->datanode1->datanode2->datanode3,副本数是3就有3个datanode构成pipeline），client端向输出流对象写数据。client向datanode1发送packet，datanode1存储并转发给datanode2，datanode2存储并转发给datanode3  
5. 每个datanode写完一个块后，会给客户端返回确认信息  
6. 写完数据，关闭输出数据流 
7. 发送完成信号给NameNode  

# HDFS的高可用性  
配置一对active-standby NameNode  
namenode之间通过高可用的共享存储实现编辑日志的共享  
当备用namenode接管工作之后，它将逐一执行编辑日志中的操作，以实现与活动namenode的状态同步  
datanode需要同时向两个namenode发送数据块处理报告，因为数据块的映射信息存储在内存中，而不是磁盘  


## 安全模式  
namenode的文件系统对于客户端是只读的

# Hadoop根据什么划分task


  


# Zookeeper在Hadoop中的作用  
主要用于HDFS的Namenode和YARN的resourceManager的单点问题  

                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                