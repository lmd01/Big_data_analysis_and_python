# Hadoop相关知识总结

[TOC]



## 一、Hadoop入门

#### 1、简要描述如何配置安装一个apache的一个开源hadoop

1. 准备一台服务器

   使用root账户登录
   修改IP
   修改host主机名
   配置SSH无密登录
   关闭防火墙

2. 安装JDK

3. 安装Hadoop(并配置核心文件)

   hadoop-env.sh（见到env就是指定java目录路径）
   core-site.xml
   mapred.xml
   hdfs-site.xml

4. 配置Hadoop环境变量

5. 格式化NN(hadoop namemode-format)

6. 启动节点

#### 2、Hadoop中需要哪些配置文件，其作用是什么？

*能记多少多少记多少，实在记不住也没关系*

1) core-site.xml
2) hdfs-site.xml
3) mapred-site.xml
4) yarn-site.xml

以上四个**名字必须知道**，里面的内容*能记住最好，记不住也没关系*

#### 3、请列出正常工作的Hadoop集群中Hadoop都分别需要启动哪些进程，它们的作用分别是什么？

<u>（初创的小公司特别爱问）</u>

1) **namenode**：Hadoop中的主服务器，管理文件系统名称空间和对集群中存储的文件的访问，保存有metadata
2) **secondarynamenode**：并不是NN的冗余守护进程，而是提供周期检查点和清理任务，帮助NN合并editslog，减少NN的启动时间
3) **datanode**：负责管理连接到节点的存储（一个集群中可以有多个节点），每个存储数据的节点运行一个datanode守护进程
4) **resourcemanager**（jobtracker）：负责调度DN上的工作，每个DN上有一个tasktracker，它们执行实际工作
5) **nodemanager**（tasktracker）：执行任务
6) DFSZKFailoverController：高可用时负责监控NN的状态，并及时把状态信息写入ZK，它通过一个独立线程周期性的调用NN上的一个特定接口来获取NN的健康状态，FC也有选择谁作为Active NN的权利（选择策略：先到先得）
7) journalnode：高可用情况下存放NN的editlog文件

#### 4、简述Hadoop的几个默认的端口号及其含义

1) **dfs.namenode.http-address：50070**
2) secondarynamenode辅助名称节点端口号：50090
3) dfs.datanode.address：50010
4) fs.defaultFS：8020或9000
5) **yarn.resourcemanager.webapp.address:8088**

## 二、HDFS

#### 1、HDFS的存储机制（读写流程）--$\textcolor{Red}{必须掌握}$

**写数据：**

![HDFS写数据流程](.\HDFS写数据流程.png)

1）客户端通过Distributed FileSystem模块向NameNode请求上传文件，NameNode检查目标文件是否已存在，父目录是否存在。

2）NameNode返回是否可以上传。

3）客户端请求第一个 Block上传到哪几个DataNode服务器上。

4）NameNode返回3个DataNode节点，分别为dn1、dn2、dn3。

5）客户端通过FSDataOutputStream模块请求dn1上传数据，dn1收到请求会继续调用dn2，然后dn2调用dn3，将这个通信管道建立完成。

6）dn1、dn2、dn3逐级应答客户端。

7）客户端开始往dn1上传第一个Block（先从磁盘读取数据放到一个本地内存缓存），以Packet为单位，dn1收到一个Packet就会传给dn2，dn2传给dn3；**dn1每传一个packet会放入一个应答队列等待应答。**

8）当一个Block传输完成之后，客户端再次请求NameNode上传第二个Block的服务器。（重复执行3-7步）。

**读数据：**

![HDFS读数据流程](.\HDFS读数据流程.png)

1）客户端通过Distributed FileSystem向NameNode请求下载文件，NameNode通过查询元数据，找到文件块所在的DataNode地址。

2）挑选一台DataNode（就近原则，然后随机）服务器，请求读取数据。

3）DataNode开始传输数据给客户端（从磁盘里面读取数据输入流，以Packet为单位来做校验）。

4）客户端以Packet为单位接收，先在本地缓存，然后写入目标文件。

#### 2、NN与2NN的工作机制（问的可能比较少，但是得知道）

![NN_2NN工作机制](.\NN_2NN工作机制.png)

 **第一阶段：NameNode启动**

​    （1）第一次启动NameNode格式化后，创建Fsimage和Edits文件。如果不是第一次启动，直接加载编辑日志和镜像文件到内存。

​    （2）客户端对元数据进行增删改的请求。

​    （3）NameNode记录操作日志，更新滚动日志。

​    （4）NameNode在内存中对数据进行增删改。

 **第二阶段：Secondary NameNode工作**

​    （1）Secondary NameNode询问NameNode是否需要CheckPoint。直接带回NameNode是否检查结果。

​    （2）Secondary NameNode请求执行CheckPoint。

​    （3）NameNode滚动正在写的Edits日志。

​    （4）将滚动前的编辑日志和镜像文件拷贝到Secondary NameNode。

​    （5）Secondary NameNode加载编辑日志和镜像文件到内存，并合并。

​    （6）生成新的镜像文件fsimage.chkpoint。

​    （7）拷贝fsimage.chkpoint到NameNode。

​    （8）NameNode将fsimage.chkpoint重新命名成fsimage。

#### 3、NN与2NN的区别与联系？

(1)流程机制与上题一样

(2)区别：

- NN负责管理整个系统文件的元数据，以及每个路径（文件）所对应的数据块信息

- 2NN辅助NN，定期合并镜像文件和编辑日志

(3)联系：

-  NN故障时可以从2NN拷贝数据以恢复NN，但不是无损的恢复

#### 4、服役新节点和退役旧节点步骤

新节点直接启动就行，注意要添加白名单，在白名单上的节点才可以启动（如果设置了的话）

退役旧节点时其实是先把它备份了

#### 5、NN挂了怎么办？

**方法一：**将2NN的数据拷贝到NN存储数据的目录

**方法二：**使用`importCheckpoint`选项启动NN的守护进程，从而将2NN的数据拷贝到NN中

## 三、MapReduce

#### 1、谈谈Hadoop序列化和反序列化以及自定义bean对象实现序列化（关键在自定义bean对象--七步走）

具体实现bean对象序列化步骤如下7步。

（1）必须实现Writable接口

（2）反序列化时，需要反射调用空参构造函数，所以必须有**空参构造**

  ```javascript
public FlowBean() { 
    super(); 
}  
  ```

（3）重写序列化方法

```javascript
@Override  
public void write(DataOutput out) throws IOException { 
    out.writeLong(upFlow);    
    out.writeLong(downFlow); 
    out.writeLong(sumFlow); 
}   
```

（4）重写反序列化方法

```javascript
@Override  
public void readFields(DataInput in) throws IOException {
                upFlow = in.readLong();
				downFlow = in.readLong();
				sumFlow = in.readLong();
}
```

（5）$\textcolor{Red}{注意反序列化的顺序和序列化的顺序完全一致}$

（6）要想把结果显示在文件中，需要重写toString()，可用”\t”分开，方便后续用。

（7）如果需要将自定义的bean放在key中传输，则还需要实现Comparable接口，因为MapReduce框中的Shuffle过程要求对key必须能排序。详见后面排序案例。
    ```javascript
@Override  public int compareTo(FlowBean o) { 
	// 倒序排列，从大到小    
	return this.sumFlow >  o.getSumFlow() ? -1 : 1;  
}   
    ```

#### 2.FileInputFormat切片机制

提交的三个文件：

1. xml
2. jar包
3. split

切片怎么切的：

1. **默认大小是按照block的大小切的**

2. **1.1倍**

#### 3.自定义InputFormat流程(三步走)--问的比较少

1. 自定义一个类继承FileInpuFormat
2. 改写RecordReader，实现一次读取一个完整文件封装为KV
3. 在输出时使用SequenceFileOutPutFormat输出合并文件

#### 4.如何决定一个job的map和reduce数量？

1）map的数量（其实就是切片的数量）

```javascript
splitSize = max{minSize,min{maxSize,blockSize}}
```

map的数量由处理的数据分成的block的数量决定`default_num = total_size/split_size`

2）reduce的数量

`job.setNumReduceTasks(x)`中的x决定，默认值为1

#### 5.Maptask的个数由什么决定？

是由切片个数决定的

#### 6.MapTask工作机制$\textcolor{Red}{（重点）}$

![MapTask工作机制](.\MapTask工作机制.png)

1. Read阶段：MapTask通过用户编写的RecordReader，从输入InputSplit中解析出一个个key/value。

2. Map阶段：该节点主要是将解析出的key/value交给用户编写map()函数处理，并产生一系列新的key/value。
3. Collect收集阶段：在用户编写map()函数中，当数据处理完成后，一般会调用OutputCollector.collect()输出结果。在该函数内部，它会将生成的key/value分区（调用Partitioner），并写入一个环形内存缓冲区中。
4. Spill阶段：即“溢写”，当环形缓冲区满后，MapReduce会将数据写到本地磁盘上，生成一个临时文件。需要注意的是，将数据写入本地磁盘之前，先要对数据进行一次本地排序，并在必要时对数据进行合并、压缩等操作。

- 溢写阶段详情：

  - 步骤1：利用快速排序算法对缓存区内的数据进行排序，排序方式是，先按照分区编号Partition进行排序，然后按照key进行排序。这样，经过排序后，数据以分区为单位聚集在一起，且同一分区内所有数据按照key有序。
  - 步骤2：按照分区编号由小到大依次将每个分区中的数据写入任务工作目录下的临时文件output/spillN.out（N表示当前溢写次数）中。如果用户设置了Combiner，则写入文件之前，对每个分区中的数据进行一次聚集操作。
  - 步骤3：将分区数据的元信息写到内存索引数据结构SpillRecord中，其中每个分区的元信息包括在临时文件中的偏移量、压缩前数据大小和压缩后数据大小。如果当前内存索引大小超过1MB，则将内存索引写到文件output/spillN.out.index中。

5. Combine阶段：当所有数据处理完成后，MapTask对所有临时文件进行一次合并，以确保最终只会生成一个数据文件。

​    当所有数据处理完后，MapTask会将所有临时文件合并成一个大文件，并保存到文件output/file.out中，同时生成相应的索引文件output/file.out.index。

​    在进行文件合并过程中，MapTask以分区为单位进行合并。对于某个分区，它将采用多轮递归合并的方式。每轮合并io.sort.factor（默认10）个文件，并将产生的文件重新加入待合并列表中，对文件排序后，重复以上过程，直到最终得到一个大文件。

​    让每个MapTask最终只生成一个数据文件，可避免同时打开大量文件和同时读取大量小文件产生的随机读取带来的开销。

#### 7.ReduceTask工作机制$\textcolor{Red}{（重点）}$

![ReduceTask工作机制](.\ReduceTask工作机制.png)

1. Copy阶段：ReduceTask从各个MapTask上远程拷贝一片数据，并针对某一片数据，如果其大小超过一定阈值，则写到磁盘上，否则直接放到内存中。

2. Merge阶段：在远程拷贝数据的同时，ReduceTask启动了两个后台线程对内存和磁盘上的文件进行合并，以防止内存使用过多或磁盘上文件过多。

3. Sort阶段：按照MapReduce语义，用户编写reduce()函数输入数据是按key进行聚集的一组数据。为了将key相同的数据聚在一起，Hadoop采用了基于排序的策略。由于各个MapTask已经实现对自己的处理结果进行了局部排序，因此，ReduceTask只需对所有数据进行一次归并排序即可。

4. Reduce阶段：reduce()函数将计算结果写到HDFS上。

#### 8.请描述mapreduce有几种排序以排序发生的阶段

1）种类：

- 部分排序

- 全排序

- 辅助排序GroupingComparator

- 二次排序

- 自定义排序WritableCompare--重写`compareTo`方法

2）发生阶段

部分排序、二次排序、全排序发生在map阶段

辅助排序发生在reduce阶段

#### 9.mapreduce中shuffle的工作流程，如何优化shuffle阶段？

**$\textcolor{Red}{工作流程--基本必问}$**

![shuffle机制](.\shuffle机制.png)

优化：

#### 10.请描述mapreduce中combiner的作用是什么，一般使用情景，哪些情况下不需要，以及和reduce的区别



## 四、Yarn

## 五、优化
