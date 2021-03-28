---
layout: post
title:  "Hadoop面试问题总结"
date:   2021-01-27 00:10:01
categories: 面试
---

# Hadoop面试问题总结


---
1. MapReduce的详细执行过程是怎么样的？（极其重要）
- 整体的大概来说，MapReduce运行的时候，会通过Mapper运行时的任务读取HDFS中的数据文件，然后调用自己的方法，处理数据，最后输出<k, v>对；Reduce任务会接收上游任务输出的数据，作为自己的输入，然后调用自己的方法，最后输出到HDFS的文件中；
- 每个Mapper任务都是一个java进程，它会读取HDFS的文件，解析成很多键值对，经过给定的map方法处理后，转换为新的键值对输出。整个Mapper任务的过程又可以分为以下几个阶段：
    - 第一步，把HDFS输入的文件按照一定的标准分片，生成InputSplit，每个输入片的大小是固定的；默认情况下，输入片InputSplit的大小与HDFS存储的数据块Block的大小是相同的。每个InputSplit对应着一个Mapper进程。如果数据块(Block)的大小是默认值64MB，输入文件有两个，一个是32MB，一个是72MB。那么小的文件是一个输入片，大文件会分为两个数据块，那么是两个输入片。一共产生三个输入片。每一个输入片由一个Mapper进程处理。这里的三个输入片，会有三个Mapper进程处理；
    - 第二步，对InputSplit中的数据按照一定的规则解析成键值对。默认规则是，把每一行文本内容解析成键值对。key是每一行的起始位置，单位是字节，value是本行的文本内容；
    - 第三步，调用Mapper类中的map方法。对第二步中解析出来的每一个键值对，都调用一次map方法。如果有1000个键值对，就会调用1000次map方法。每一次调用map方法都会输出零个或者多个键值对；
    - 第四步，按照一定的规则对第三步输出的<k, v>对进行分区。分区是基于key进行的。比如我们的键表示省份（如北京、上海、山东等），那么就可以按照不同省份进行分区，同一个省份的键值对划分到一个区中。默认是只有一个区，分区的数量就是Reducer任务运行的数量，因此默认只有一个Reducer任务；
    - 第五步，对每个分区中的<k, v>对进行排序。首先，按照key进行排序，对于key相同的键值对，按照value进行排序。比如三个键值对<2, 2>、<1, 3>、<2, 1>，键和值分别是整数。那么排序后的结果是<1, 3>、<2, 1>、<2, 2>。如果有第六阶段，那么进入第六阶段；如果没有，直接输出到本地的linux文件中；
    - 第六步，是对数据进行归约处理，也就是reduce处理，通常情况下的Combiner过程，键相等的键值对会调用一次reduce方法，经过这一阶段，数据量会减少，归约后的数据输出到本地的linxu文件中。本阶段默认是没有的，需要用户自己增加这一阶段的代码；
- 每个Reducer任务是一个java进程。Reducer任务接收Mapper任务的输出，归约处理后写入到HDFS中，可以分为如下所示的几个阶段。整个Reducer任务的过程又可以分为以下几个阶段：
    - 第一步，Reducer任务会主动从Mapper任务复制其输出的<k, v>对，Mapper任务可能会有很多，因此Reducer会复制多个Mapper的输出；
    - 第二步，是把复制到Reducer的本地数据，全部进行合并，即把分散的数据合并成一个大的数据，再对合并后的数据排序；
    - 第三步，是对排序后的<k, v>对调用reduce方法，key相等的<k, v>对会调用一次reduce方法，每次调用会产生零个或者多个<k, v>对，最后把这些输出的键值对写入到HDFS文件中；
- Hadoop的核心思想是MapReduce，但shuffle又是MapReduce的核心，shuffle主要是Map结束到Reduce开始之间的过程：
    - Map端的shuffle
        - 在map端首先接触的是InputSplit，在InputSplit中含有DataNode中的数据，每一个InputSplit都会分配一个Mapper任务，Mapper任务结束后产生<K, V>的输出，这些输出先存放在缓存中，每个map有一个环形内存缓冲区，用于存储任务的输出。默认大小100MB（io.sort.mb属性），一旦达到阀值0.8(io.sort.spill.percent)，一个后台线程就把内容写到Linux本地磁盘中的指定目录（mapred.local.dir）下的新建的一个溢出文件中；
        - 将溢出的文件写入磁盘前，要进行partition、sort和combine等操作。通过分区，将不同类型的数据分开处理，之后对不同分区的数据进行排序，如果有Combiner，还要对排序后的数据进行combine。等最后记录写完，将全部溢出文件合并为一个分区且排序的文件；
        - 最后将磁盘中的数据送到Reduce中，例如Map输出有三个分区，有一个分区数据被送到一个Reduce任务中，剩下的两个分区被送到其他Reducer任务中。而一个Reducer任务的其他的输入也可以来自其他节点的Map输出；
    - Reduce端的shuffle
        - Copy：reduce端可能从多个map的结果中获取数据，而这些map的执行速度不尽相同，当其中一个map运行结束时，reduce就会从JobTracker中获取该信息。map运行结束后TaskTracker会得到消息，进而将消息汇报给JobTracker，reduce定时从JobTracker获取该信息，reduce端默认有5个数据复制线程从map端复制数据，这种获取信息的方式是通过http开展的；
        - Merge：从map端复制来的数据首先写到reduce端的缓存中，同样缓存占用到达一定阈值后会将数据写到磁盘中，同样会进行partition、combine、sort等过程。如果形成了多个磁盘文件还会进行合并，最后一次合并的结果作为reduce的输入，而不是写入到磁盘中；
        - Reduce：最后将合并后的结果作为输入传入Reduce任务中，从而进行Reduce过程。在这个过程中产生了最终的输出结果，并将其写到HDFS上；


---
2. Hadoop怎么从本地上传到HDFS文件？请写出命令行？
- 在HDFS中创建一个文件夹：`hdfs dfs -mkdir /input`
- 从Linux上传文件：`hdfs dfs -put /home/yangsong/test.txt /input`
- 从HDFS下载文件：`hdfs dfs -get /input/test.txt`
- 从Linux上传文件，方法二：`hadoop fs -moveFromLocal test.txt /input`
- 从HDFS下载文件，方法二：`hadoop fs -copyToLocal /input/test.txt`


---
3. Hadoop的高可用是什么意思？阐述一下高可用的机制？
- Hadoop从2.0开始引入高可用(HA, High Availability)机制；所谓高可用，即7*24小时不中断服务，而实现高可用，最关键的点在于消除单点故障；
- Hadoop的高可用，严格来说是各个组件的高可用，即：HDFS的高可用，YARN的高可用；
    - HDFS的HA机制，是通过两个NameNode的方式来消除单点故障，具体包括：两个NameNode在内存中各自保存一份元数据，其状态分别为Active/ StandBy；只有Active状态的NameNode节点可以对日志写入操作，但两个节点都可以读日志；利用两个监控器ZookeeperFailureController分别监控NameNode节点的状态，并在一个节点坏死的时候，进行主节点和备用节点的切换；
    - YARN的HA机制，是通过提供多个ResourceManager来实现的。ResourceManager负责YARN资源的跟踪和任务调度，HA机制中存在多个RM，但同时只能有一个RM处于Active状态，其他一个或多个RM处于StandBy状态，以便于RM在故障的时候快速切换；与HDFS的HA机制类似，也是通过ZookeeperFailureController对RM状态的监控来实现的；


---
4. Hadoop的负载均衡是什么意思？对集群有什么影响？
- HDFS非常容易出现不同机器之间，磁盘利用率不均衡的情况。例如，当集群内新增、删除节点，或者某个节点机器的硬盘存储达到饱和值，这时在分布式计算的时候，会将数据下发到没有存储数据的机器上，从而导致不均衡的现象发生；
- 当HDFS的负载不均衡时，需要对HDFS进行均衡调整，即对各个节点上数据的存储分布进行调整，从而让数据尽可能均衡的分布在各个DataNode上，以防止节点过热。其过程如下所述：
    - 数据均衡服务首先要求NameNode对每个DataNode进行扫描，获取数据分布的分析报告，从而得知每个DataNode的磁盘使用情况；
    - Rebalancing Server汇总需要迁移的数据分布情况，计算具体数据块的迁移计划，确保使用网络内的最短路径，以减小带宽的压力；
    - 开始数据块迁移任务，从数据量太多的DataNode上，将其数据复制到数据量较少的DataNode上，然后在原来的DataNode上删除被复制的数据块；
    - ProxySrouceDataNode向Rebalancing Server确认本次数据迁移计划的完成情况，然后继续执行这个过程，直到整个集群达到数据均衡的状态；


---
5. Hadoop是什么？
- Hadoop是由Apache基金会开源的分布式计算框架，是以Java为开发语言、专门针对海量数据进行的计算模式。它最大的特点是在性能与成本上都有较大优势，再加上可以横向扩充、生态组件丰富、社区活跃等优点，现在已经成为全球大数据领域事实上的标准了；
- Hadoop不需要使用造价昂贵的商业服务器，在廉价的个人PC上就可以使用，用于可以利用网络构成集群；随着需要处理的数据量越来越大，只需要不断增加计算机的数量，而不需要修改应用程序的代码，就可以提高Hadoop的计算能力；
- Hadoop的主要有三个组件组成：数据存储框架HDFS、并行计算框架MapReduce、任务调度工具YARN；


---
6. 请列出正常工作的Hadoop集群中，都需要启动哪些进程，它们的作用分别是什么？
- Hadoop由三部分组成：HDFS/ MapReduce/ YARN；
- HDFS
    - HDFS遵循master-slave架构，其中，NameNode是master，DataNode是slave；
    - NameNode: 它是整个HDFS的管理节点，存储着整个文件系统的元数据，包括文件名称、大小、存储位置、创建时间等；维护整个HDFS的文件目录；同时还接收用户的读写请求，然后下发个DataNode执行。在Hadoop启动的时候，NameNode会对整个HDFS进行一次快照，保存在`fs image`中；在Hadoop启动后的运行过程当中，读写操作会对HDSF的元数据进行改变，这种改变会写入`edit logs`当中；
    - SecondaryNameNode: 冗余的守护进程，属于NameNode的备份，且内存需求均较大，因此一般将其运行在与NameNode不同的机器上；随着Hadoop的持续运行，`edit logs`会变得越来越大，而SecondaryNameNode会定期将`edit logs`更新到`fs image`中，因此在下次重启Hadoop时，会直接使用更新后的`fs image`，从而大大减小重启的时间；
    - DataNode: 它是HDFS的工作节点，负责数据的存储，多个DataNode共同实现了海量数据的本地存储，文件会按照block定义的大小切分成若干块，分布式的存储在多个DataNode上面；而每一个文件又有多个副本，也是分布式的存储在不同的DataNode上面；在执行HDFS的读写操作时，NameNode与DataNode通信，告诉后者需要哪些数据，最后由DataNode进行数据的检索、读写操作；
- MapReduce
    - MapReduce遵循master-slave架构，其中JobTracker是master，TaskTracker是slave；
    - JobTracker：它是整个集群唯一的全局管理者，功能包括任务调度、作业管理、状态监控。JobTracker是一个后台服务进程，启动之后，会一直监听各个TaskTracker的心跳信息，包括资源使用情况、任务运行情况等。JobTracker对应NameNode，负责实际调度DataNode的工作，NameNode/DataNode是针对数据的存储而言的，JobTracker/ TaskTracker是针对MapReduce的任务调度而言的；
    - Tasktracker：它负责任务执行、结果反馈等。执行JobTracker分配分发的任务，并向JobTracker汇报任务的执行情况；
- YARN
    - YARN遵循master-slave架构，ResourceManager是master，NodeManager是slave，前者对后者的资源进行统一的管理和调度；
    - ResourceManager: yarn的守护进程，负责集群资源的分配与调度，监控NodeManager；它由调度器、应用程序管理器两部分组成，调度器负责单纯的调度，程序管理器负责作业的提交、重启、kill等操作；
    - NodeManager: 执行来自ResourceManager的具体命令，管理单个节点的资源；


---
6. 简述HDFS数据写入机制
- HDFS的设计初衷，是一次写入、多次读取，因此，当用户请求向HDFS写入文件时，会按照以下流程展开；
- 与NameNode通信请求上传文件，NameNode检查目标文件、及其文件目录是否存在；然后，NameNode会返回是否可以上传的消息，已确认可以将文件上传到HDFS，并得知接收文件block的DataNode；
- 客户端会对带上传的文件进行切分，切分为HDFS中定义的文件大小。例如，一个block块默认是128M，文件有300M，那么就会被切分成3个块，一个128M、一个128M、一个44M；
- 客户端向NameNode发起请求，以获得接收第一个block的DataNode的位置，而随后NameNode返回该DataNode的信息，一般会有三个DataNode冗余的接收同一个block；
- 客户端通过RPC向第一台DataNode上传数据，第一个DataNode收到请求后，会写入该block；然后会继续调用第二个DataNode，继续写入数据；以此类推，以这种管道pipeline的形式写入数据，直到多个DataNode都写入该block。整个写入过程完成后，将完成信息返回客户端；
- 当第一个block传输完成之后，客户端会再次请求NameNode，以上传第二个block到指定的DataNode中；以此类推，直到该文件的所有block都写入HDFS为止；


---
7. 简述HDFS数据读取机制
使用HDFS提供的客户端开发库Client，向远程的NameNode发起RPC请求；NameNode会视情况返回文件的部分或全部block列表，对于每个block，NameNode都会返回有该block拷贝的DataNode地址；客户端开发库Client会选取离客户端最接近的DataNode来读取block；如果客户端本身就是DataNode,那么将从本地直接获取数据.读取完当前block的数据后，关闭与当前的DataNode连接，并为读取下一个block寻找最佳的DataNode；当读完列表的block后，且文件读取还没有结束，客户端开发库会继续向NameNode获取下一批的block列表。读取完一个block都会进行 checksum 验证，如果读取 DataNode 时出现错误，客户端会通知 NameNode，然后再从下一个拥有该 block 拷贝的 DataNode 继续读。


---
6. HDFS在上传文件的时候，如果其中一个块突然损坏了怎么办？


---



---
8. Hadoop的作业提交流程


---
9. Hadoop怎么分片


---
10. 如何减少Hadoop Map端到Reduce端的数据传输量


---
11.








1.


---
2. 请列出你所知道的Hadoop调度器，并简要说明其工作方法？
- 调度器的作用在于，将系统中空闲的资源，按一定策略分配给每个作业；
- 先进先出调度器（FIFO）：Hadoop 中默认的调度器，也是一种批处理调度器。它先按照作业的优先级高低，再按照到达时间的先后选择被执行的作业；
- 容量调度器（Capacity Scheduler)：支持多个队列，每个队列可配置一定的资源量，每个队列采用FIFO调度策略，为了防止同一个用户的作业独占队列中的资源，该调度器会对同一用户提交的作业所占资源量进行限定。调度时，首先按以下策略选择一个合适队列：计算每个队列中正在运行的任务数与其应该分得的计算资源之间的比值，选择一个该比值最小的队列；然后按以下策略选择该队列中一个作业：按照作业优先级和提交时间顺序选择，同时考虑用户资源量限制和内存限制；
- 公平调度器（Fair Scheduler）：公平调度是一种赋予作业（job）资源的方法，它的目的是让所有的作业随着时间的推移，都能平均的获取等同的共享资源。所有的job 具有相同的资源,当单独一个作业在运行时，它将使用整个集群。当有其它作业被提交上来时，系统会将任务（task）空闲资源（container）赋给这些新的作业，以使得每一个作业都大概获取到等量的CPU时间。与Hadoop默认调度器维护一个作业队列不同，这个特性让小作业在合理的时间内完成的同时又不"饿"到消耗较长时间的大作业。公平调度可以和作业优先权搭配使用——优先权像权重一样用作为决定每个作业所能获取的整体计算时间的比例。同计算能力调度器类似，支持多队列多用户，每个队列中的资源量可以配置，同一队列中的作业公平共享队列中所有资源；


---
3. 当前日志采样格式为如下，请编写MapReduce计算第四列每个元素出现的个数？
<br>a,b,c,d</br>
<br>a,s,d,f</br>
<br>d,f,g,c</br>

```java
package make.Hadoop.com.four_column;

import java.io.IOException;

import org.apache.Hadoop.conf.Configuration;
import org.apache.Hadoop.conf.Configured;
import org.apache.Hadoop.fs.FileSystem;
import org.apache.Hadoop.fs.Path;
import org.apache.Hadoop.io.IntWritable;
import org.apache.Hadoop.io.LongWritable;
import org.apache.Hadoop.io.Text;
import org.apache.Hadoop.MapReduce.Job;
import org.apache.Hadoop.MapReduce.Mapper;
import org.apache.Hadoop.MapReduce.Reducer;
import org.apache.Hadoop.MapReduce.lib.input.FileInputFormat;
import org.apache.Hadoop.MapReduce.lib.output.FileOutputFormat;
import org.apache.Hadoop.util.Tool;
import org.apache.Hadoop.util.ToolRunner;

public class four_column extends Configured implements Tool {
	// 1、自己的map类
	// 2、继承mapper类，<LongWritable, Text, Text,
	// IntWritable>输入的key,输入的value，输出的key,输出的value
	public static class MyMapper extends
			Mapper<LongWritable, Text, Text, IntWritable> {
		private IntWritable MapOutputkey = new IntWritable(1);
		private Text MapOutputValue = new Text();

		@Override
		protected void map(LongWritable key, Text value, Context context)
				throws IOException, InterruptedException {

			String strs = value.toString();
			// 分割数据
			String str_four = strs.split(",")[3];

			MapOutputValue.set(str_four);
			System.out.println(str_four);
			context.write(MapOutputValue, MapOutputkey);

		}
	}
	// 2、自己的reduce类，这里的输入就是map方法的输出
	public static class MyReduce extends
			Reducer<Text, IntWritable, Text, IntWritable> {

		IntWritable countvalue = new IntWritable(1);

		@Override
		// map类的map方法的数据输入到reduce类的group方法中，得到<text,it(1,1)>,再将这个数据输入到reduce方法中
		protected void reduce(Text inputkey, Iterable<IntWritable> inputvalue,
				Context context) throws IOException, InterruptedException {

			int sum = 0;

			for (IntWritable i : inputvalue) {
				System.out.println(i.get());
				sum = sum + i.get();
			}
			// System.out.println("key: "+inputkey + "...."+sum);
			countvalue.set(sum);
			context.write(inputkey, countvalue);
		}
	}
	// 3运行类，run方法，在测试的时候使用main函数，调用这个类的run方法来运行

	/**
	 * param args 参数是接受main方得到的参数，在run中使用
	 */
	public int run(String[] args) throws Exception {

		Configuration conf = new Configuration();

		Job job = Job.getInstance(this.getConf(), "four_column");

		// set mainclass
		job.setJarByClass(four_column.class);

		// set mapper
		job.setMapperClass(MyMapper.class);
		job.setMapOutputKeyClass(Text.class);
		job.setMapOutputValueClass(IntWritable.class);

		// set reducer
		job.setReducerClass(MyReduce.class);
		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(IntWritable.class);

		// set path
		Path inpath = new Path(args[0]);
		FileInputFormat.setInputPaths(job, inpath);
		Path outpath = new Path(args[1]);
		FileOutputFormat.setOutputPath(job, outpath);
		FileSystem fs = FileSystem.get(conf);
		// 存在路径就删除
		if (fs.exists(outpath)) {
			fs.delete(outpath, true);
		}
		job.setNumReduceTasks(1);

		boolean status = job.waitForCompletion(true);

		if (!status) {
			System.err.println("the job is error!!");
		}

		return status ? 0 : 1;

	}
	public static void main(String[] args) throws IOException,
			ClassNotFoundException, InterruptedException {

		Configuration conf = new Configuration();

		int atatus;
		try {
			atatus = ToolRunner.run(conf, new four_column(), args);
			System.exit(atatus);
		} catch (Exception e) {
			e.printStackTrace();
		}

	}
}
```


---
4. 请简述MapReduce中，Combiner/ Partition分别是什么？其作用是什么？
- Combiner：
    - Combiner是MR程序中，Mapper和Reducer之外的一种组件；其父类是Reducer；
    - Combiner和Reducer的区别在于运行的位置；
    - Reducer是每一个接收全局的Map Task所输出的结果，Combiner是在Map Task的节点中运行；
    - 每个map都会产生大量的本地输出，Combiner的作用是，对map输出的结果先做一次合并，以减少map节点和reduce节点之间的数据传输量；
    - Combiner的存在就是提高当前网络、IO传输的效率，也是MapReduce的一种优化手段；

- Partitioner：
    - Partitioner的作用是对Mapper产生的结果进行分片，以便将同一个分组的数据交给同一个Reducer处理，它将直接影响Reduce阶段的负载均衡；
    - Partitioner有两个具体的功能：均衡负载，尽量将工作均匀的分配给不同的Reduce；效率，分配速度一定要快；

- 在MapReduce整个过程中，combiner是可有可无的，需要是自己的情况而定，如果只是单纯的对map输出的key-value进行一个统计，则不需要进行combiner，它相当于提前做了一个reduce的工作，减轻了reduce端的压力；Combiner只应该适用于那种Reduce的输入（key：value与输出（key：value）类型完全一致，且不影响最终结果的场景。比如累加、最大值等，也可以用于过滤数据，在map端将无效的数据过滤掉。在这些需求场景下，输出的数据是可以根据key值来作合并的，合并的目的是减少输出的数据量，减少IO的读写，减少网络传输,以提高MR的作业效率。

    1.combiner的作用就是在map端对输出先做一次合并，以减少传输到reducer的数据量；
    2.combiner最基本是实现本地key的归并,具有类似本地reduce，那么所有的结果都是reduce完成，效率会相对降低；
    3.使用combiner，先完成的map会在本地聚合，提升速度；

- partition意思为分开，分区。它分割map每个节点的结果，按照key分别映射给不同的reduce，也是可以自定义的。其实可以理解归类。也可以理解为根据key或value及reduce的数量来决定当前的这对输出数据最终应该交由哪个reduce task处理。partition的作用就是把这些数据归类。每个map任务会针对输出进行分区，及对每一个reduce任务建立一个分区。划分分区由用户定义的partition函数控制，默认使用哈希函数来划分分区。HashPartitioner是MapReduce的默认partitioner。计算方法是which reducer=(key.hashCode() & Integer.MAX_VALUE) % numReduceTasks，得到当前的目的reducer


---
简述MapReduce计算的基本流程
- 1. split: 将文件分割为小的数据块，传输到mapper节点当中；
- 2. map: 将上述小文件开发为<key, value>形式的数据；
- 3. shuffle: Shuffle先进行HashPartition或者自定义的partition，会有数据倾斜和reduce的负载均衡问题;再进行排序，默认按字典排序;为减少mapper输出数据，再根据key进行合并，相同key的数据value会被合并;最后分组形成(key,value{})形式的数据，输出到下一阶段；
- 4. reduce: 按计算逻辑生成最终数据；


---
8. 请简述Hadoop怎么样实现二级排序？
- 在MapReduce中本身就会对我们key进行排序，所以我们要对value进行排序，主要思想为将key和部分value拼接成一个组合key（实现WritableComparable接口或者调用 setSortComparatorClass函数），这样reduce获取的结果便是先按key排序，后按value排序的结果，在这个方法中，用户需 要自己实现Paritioner，继承Partitioner<>,以便只按照key进行数据划分。Hadoop显式的支持二次排序，在Configuration类中有个 setGroupingComparatorClass()方法，可用于设置排序group的key值。
第一种方法是，Reducer将给定key的所有值都缓存起来，然后对它们再做一个Reducer内排序。但是，由于Reducer需要保存给定key的所有值，可能会导致出现内存耗尽的错误。
第二种方法是，将值的一部分或整个值加入原始key，生成一个组合key。这两种方法各有优势，第一种方法编写简单，但并发度小，数据量大的情况下速度慢(有内存耗尽的危险)。
第二种方法则是将排序的任务交给MapReduce框架shuffle，更符合Hadoop/Reduce的设计思想。这篇文章里选择的是第二种。我们将编写一个Partitioner，确保拥有相同key(原始key，不包括添加的部分)的所有数据被发往同一个Reducer，还将编写一个Comparator，以便数据到达Reducer后即按原始key分组。


---
9. 把数据仓库从传统关系数据库转到Hadoop有什么优势？
- RDBMS存储的空间有限，而Hadoop可以通过扩展集群的方式极大的提升存储空间；
- RDBMS存储方式昂贵，而Hadoop可以建设在比较便宜的商品PC机集群中；
- RDBMS基本只能存储结构化数据，而Hadoop支持结构化（RDBMS），非结构化（例如images/ PDF/ docs）和半结构化（例如logs，XMLs）数据的存储；


---
10. 为什么JobTraker通常与NameNode在一个节点启动？
- Hadoop的集群是基于Master/Slave模式展开的，NameNode和JobTracker属于Master，DataNode和Tasktracker属于Slave，Master只有一个，而Slave有多个


---
11. NameNode和DataNode作用是什么？
- NameNode管理文件系统的元数据，并在启动集群的时候将其存储在内存中，例如文件系统的命名空间、集群配置信息和存储块的信息；
- DataNode是HDFS中文件存储的基本单元，它将数据块存储在本地文件系统中，存储实际的数据；
- 客户端通过同NameNode和DataNodes的交互访问文件系统。客户端联系NameNode以获取文件的元数据，而真正的文件I/O操作是直接和DataNode进行交互的；


---
12. Client客户端读取HDFS文件的顺序流是什么？
- Client向NameNode发起文件读取的请求；
- NameNode返回存储该文件的DataNode的信息；
- Client读取DataNode上的文件信息；


---
13. Client客户端将数据写入HDFS文件的顺序流是什么？
- Client向NameNode发起文件写入的请求；
- NameNode根据文件大小和文件块配置情况，返回给Client相应的DataNode的信息；
- Client将文件划分为多个Block，根据DataNode的地址信息，按顺序写入到每一个DataNode块中；


---
14. Hadoop主要的端口都有哪些？
- 9000 NameNode
- 8020 NameNode
- 8021 JT RPC
- 50030 mapred.job.tracker.http.address JobTracker administrative web GUI
- 50070 dfs.http.address NameNode administrative web GUI
- 50010 dfs.DataNode.address DataNode control port
- 50020 dfs.DataNode.ipc.address DataNode IPC port, used for block transfer
- 50060 mapred.task.tracker.http.address Per Tasktracker web interface
- 50075 dfs.DataNode.http.address Per DataNode web interface
- 50090 dfs.secondary.http.address Per secondary NameNode web interface


---
15. HDFS系统的常见命令有哪些？
- 查看文件：Hadoop fs -cat /user/output/outfile
- 把文件系统中的文件存入HDFS文件系统：Hadoop fs -put /home/file.txt /user/input
- 把HDFS文件系统的文件目录写到本地文件系统目录：Hadoop fs -get /user/localfile/ /home/Hadoop_dir/
- 在HDFS文件系统中复制文件：Hadoop fs -cp /user/source /user/target


---
16. Hadoop的核心配置文件是什么？
- Hadoop-default.xml
- Hadoop-site.xml


---
18. Hadoop的性能调优有哪些手段？
调优可以通过系统配置、程序编写和作业调度算法来进行。
HDFS的block.size可以调到128/256（网络很好的情况下，默认为64）
调优的大头：
mapred.map.tasks、mapred.reduce.tasks设置mr任务数（默认都是1）
mapred.Tasktracker.map.tasks.maximum每台机器上的最大map任务数
mapred.Tasktracker.reduce.tasks.maximum每台机器上的最大reduce任务数
mapred.reduce.slowstart.completed.maps配置reduce任务在map任务完成到百分之几的时候开始进入
这个几个参数要看实际节点的情况进行配置，reduce任务是在33%的时候完成copy，要在这之前完成map任务，（map可以提前完成）
mapred.compress.map.output,mapred.output.compress配置压缩项，消耗cpu提升网络和磁盘io
合理利用combiner
注意重用writable对象


---
20. 请写出以下的Hadoop命令：杀死一个job、删除HDFS上的 /tmp/aaa目录、加入一个新的存储节点、删除一个节点
- `Hadoop job –list`，得到job的id；然后执行`Hadoop job -kill job_id`，即可杀死一个指定jobId的job工作；
- `Hadoop fs -rmr /tmp/aaa`
- 增加一个新的节点，需要在新增的节点上执行`Hadoop daemon.sh start DataNode`、`Hadooop daemon.sh start Tasktracker/nodemanager`
- 删除一个节点的时候，只需要在主节点执行`Hadoop mradmin -refreshnodes`


---
21. 你认为用Java/ Streaming/ Pipe方式开发MapReduce，各有哪些优点
- Java写MapReduce可以实现复杂的逻辑，如果需求简单，则显得繁琐；
- HiveQL基本都是针对Hive中的表数据进行编写，但对复杂的逻辑很难实现，写起来简单；


---
23. 用MapReduce怎么处理数据倾斜问题？
- 本质：让各分区的数据分布均匀；
- 可以根据业务特点，设置合适的partition策略；如果事先根本不知道数据的分布规律，利用随机抽样器抽样后生成partition策略再处理


---
24. 我们在开发分布式计算job的时候，是否可以去掉Reduce阶段
- 可以。如果只是为了存储文件，而不涉及到数据的计算，就可以将MapReduce都省掉；


---
25. Hadoop常用的压缩算法？
- Lzo
- Gzip
- Default
- Snapyy


---
26. Flush的过程？
- flush是在内存的基础上进行的，首先写入文件的时候，会先将文件写到内存中，当内存写满的时候，一次性的将文件全部都写到硬盘中去保存，并清空缓存中的文件


---
27. List与set的区别
- List和Set都是接口。他们各自有自己的实现类，有无顺序的实现类，也有有顺序的实现类。
最大的不同就是List是可以重复的。而Set是不能重复的。
List适合经常追加数据，插入，删除数据。但随即取数效率比较低。
Set适合经常地随即储存，插入，删除。但是在遍历时效率比较低。


---
28. 数据的三范式分别是什么？
- 第一范式（）无重复的列
- 第二范式（2NF）属性完全依赖于主键 [消除部分子函数依赖]
- 第三范式（3NF）属性不依赖于其它非主属性 [消除传递依赖]


---
29. 三个DataNode中，当有一个DataNode出现错误时会怎样？
- NameNode会通过心跳机制感知到DataNode下线；
- 会将这个DataNode上的block块在集群中重新复制一份，恢复文件的副本数量；
- 会引发运维团队快速响应，派出同事对下线DataNode进行检测和修复，然后重新上线；


---
30. MapReduce有哪些优化经验？
- 设置合理的map和reduce的个数
- 合理设置blocksize
- 避免出现数据倾斜
- combine函数
- 对数据进行压缩
- 小文件处理优化：事先合并成大文件，combineTextInputformat，在HDFS上用MapReduce将小文件合并成SequenceFile大文件（key:文件名，value：文件内容）
- 参数优化


---
31. 现有1亿个整数均匀分布，如果要得到前1K个最大的数，最优的算法是什么？
- 《海量数据算法面试大全》


---
32. 用MapReduce实现sql语句：select count(x) from a group by b;
```
import
```


---
33. Hadoop参数调优


---
34. 通过WordCount的例子，说明MapReduce是怎么运行的？


---
35. 如何确认Hadoop集群的健康状况？
- 有完善的集群监控体系（ganglia，nagios）
- HDFS dfsadmin –report
- HDFS haadmin –getServiceState nn1


---
36. MapReduce作业，不让reduce输出，用什么代替reduce的功能？


---
37. 假设公司要建一个数据中心，你会如何处理？
- 先进行需求调查分析，设计功能划分，吞吐量的估算，采用的技术类型，软硬件选型，成本效益的分析，项目管理，扩展性，安全性，稳定性


---
38. Hadoop1和Hadoop2的区别是什么？
- Hadoop1的主要结构是由HDFS和MapReduce组成的，HDFS主要是用来存储数据，MapReduce主要是用来计算的，那么HDFS的数据是由NameNode来存储元数据信息，DataNode来存储数据的。JobTracker接收用户的操作请求之后去分配资源执行task任务。
- Hadoop2中，首先避免了NameNode单点故障的问题，使用两个NameNode来组成NameNode feduration的机构，两个NameNode使用相同的命名空间，一个是standby状态，一个是active状态。用户访问的时候，访问standby状态，并且，使用journalnode来存储数据的原信息，一个NameNode负责读取journalnode中的数据，一个NameNode负责写入journalnode中的数据，这个平台组成了Hadoop的HA就是high availableAbility高可靠
- 然后在Hadoop2中没有了JobTracker的概念了，统一的使用yarn平台来管理和调度资源，yarn平台是由resourceManager和NodeManager来共同组成的，ResourceManager来接收用户的操作请求之后，去NodeManager上面启动一个主线程负责资源分配的工作，然后分配好了资源之后告知ResourceManager，然后ResourceManager去对应的机器上面执行task任务。


---
39. 文件大小默认为64M，改为128M有啥影响？
- 更改文件的block块大小，需要根据我们的实际生产中来更改block的大小，如果block定义的太小，大的文件都会被切分成太多的小文件，减慢用户上传效率，如果block定义的太大，那么太多的小文件可能都会存到一个block块中，虽然不浪费硬盘资源，可是还是会增加NameNode的管理内存压力。


---
40. RPC的原理是什么？
1.调用客户端句柄；执行传送参数

2.调用本地系统内核发送网络消息

3. 消息传送到远程主机

4. 服务器句柄得到消息并取得参数

5. 执行远程过程

6. 执行的过程将结果返回服务器句柄

7. 服务器句柄返回结果，调用远程系统内核

8. 消息传回本地主机

9. 客户句柄由内核接收消息

10. 客户接收句柄返回的数据


---
41. 以你的实际经验，说下怎样预防全表扫描？
- 应尽量避免在where 子句中对字段进行null 值判断，否则将导致引擎放弃使用索引而进行全表扫描
- 应尽量避免在 where 子句中使用!=或<>操作符，否则将引擎放弃使用索引而进行全表扫
- 描应尽量避免在 where 子句中使用or 来连接条件，否则将导致引擎放弃使用索引而进行
- in 和 not in，用具体的字段列表代替，不要返回用不到的任何字段。in 也要慎用，否则会导致全表扫描
- 避免使用模糊查询
- 任何地方都不要使用select* from t


---
42. zookeeper优点是什么？用在什么场合？
- 极大方便分布式应用的开发；（轻量，成本低，性能好，稳定性和可靠性高）


---
43. 你们提交的Job任务大概有多少个？这些job执行完大概用多少时间？(面试了三家，都问这个问题)
- 没统计过，加上测试的，会有很多
- Sca阶段，一小时运行一个job，处理时间约12分钟
- Etl阶段，有2千多个job，从凌晨12:00开始次第执行，到早上5点左右全部跑完


---
44. Hadoop中RecorderReader的作用是什么？


---
45. 在Hadoop中定义的主要公用InputFormat中，哪个是默认值？


---
46. 两个类TextInputFormat和KeyValueInputFormat的区别是什么？
- TextInputFormat主要是用来格式化输入的文本文件的，KeyValueInputFormat则主要是用来指定输入输出的key/ value类型的


---
47. 在一个运行的Hadoop任务中，什么是InputSplit？
- InputSplit是InputFormat中的一个方法，主要是用来切割输入文件的，将输入文件切分成多个小文件，然后每个小文件对应一个map任务


---
48. Hadoop框架中文件拆分是怎么调用的？
- InputFormat --> TextInputFormat -->RecordReader --> LineRecordReader --> LineReader


---
49. 如果没有自定义partitioner，那数据在被送达reducer前是如何被分区的？


---
50. 分别举例什么情况要使用 combiner，什么情况不使用？
- 求平均数的时候就不需要用combiner，因为不会减少reduce执行数量。在其他的时候，可以依据情况，使用combiner，来减少map的输出数量，减少拷贝到reduce的文件，从而减轻reduce的压力，节省网络开销，提升执行效率


---
51. Hadoop中job和tasks之间的区别是什么？
- Job是我们对一个完整的MapReduce程序的抽象封装
- Task是job运行时，每一个处理阶段的具体实例，如map task，reduce task，maptask和reduce task都会有多个并发运行的实例


---
52. Hadoop中通过拆分任务到多个节点运行来实现并行计算，但某些节点运行较慢会拖慢整个任务的运行，Hadoop采用全程机制应对这个情况？
- Speculate 推测执行


---
53. 有可能使Hadoop任务输出到多个目录中吗？如果可以，怎么做？
- 自定义outputformat或者用multioutputs工具


---
54. 如何为一个Hadoop任务设置mappers的数量？


---
55. Split机制是什么？


---
56. 如何为一个Hadoop任务设置要创建reduder的数量？
