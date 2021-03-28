---
layout: post
title:  "Hive面试问题总结"
date:   2021-01-27 00:10:01
categories: 面试
---

# Hive面试问题总结


1. Hive数据仓库怎么设计的？如何分层？
- 数据贴源层ods：用于直接接收业务库的原始数据、埋点数据、日志数据等；ods层无需对数据做任何更改，也不对外开放；ods是后续的数据清洗、数据整合的基础；
- 数据仓库层dw：基于ods层对数据进行清洗、加工、转换，生产出干净、工整、优质的基础数据，一般是包含非常多的字段的宽表；
- 数据应用层dm：结合业务场景，基于dw层对数据进行聚合、统计分析，用于向业务部门输出报表、BI分析；


---
2. Hive与MySQL等传统数据库有什么区别？
- 查询语言：Hive使用Hive-QL，MySQL使用ANSI SQL；
- 存储方式：Hive的数据存储在HDFS上，MySQL数据是存储在本身的系统中；
- 数据格式：Hive数据格式用户可以自定义，MySQL有自己的系统定义格式；
- 数据更新：Hive不支持行级别的数据更新，MySQL支持行级别的数据更新；
- 索引：Hive没有索引，因此查询数据的时候，是通过MapReduce进行暴力的全表查询，从而造成了Hive查询数据延迟较大的困境，而MySQL有索引；
- 延迟性：Hive延迟性高，一般用于离线计算、批处理、OLAP，而MySQL延迟性低，一般用于业务系统、OLTP；
- 数据规模：Hive存储的数据量超级大，MySQL只是存储千万行级别的业务数据；
- 底层执行原理：Hive查询基于MapReduce展开，而MySQL是Excutor执行器；


---
3. Hive中的内部表和外部表的区别？
- 语法的区别：创建内部表使用`create table`，创建外部表使用`create external table`；
- 存储的区别：
    - 内部表的HDFS路径下存储有数据，一般遵循先建表、再插入的原则；
    - 外部表的HDFS路径下没有数据，它只是对某个位置的数据的映射，一般遵循先有数据、后建表的原则；
- 删除的区别：删除内部表时，会将其元数据、存储的数据都删除掉；删除外部表时，仅删除其元数据，存储的数据不会删除；


---
4. Hive分区表和分桶表有什么区别？
- 语法的区别：
    - 分区表使用`partitioned by (column_name String)`；
    - 分桶表使用`clustered by (column_name) into 3 buckets`；
- 原理的区别：
    - 分区表将数据按照分区字段，划分为HDFS目录下的多个文件夹；但缺点是，若分区字段不合适，会导致数据倾斜；
    - 分桶表根据分桶字段的hash值取模，从而划分数据块；它将数据分解为容易管理的若干个部分，属于划分文件；
- 功能的区别：
    - 分区表的作用是，便于分隔数据、优化查询速度；
    - 分桶表的作用是，提高抽样、join时MapReduce程序的执行效率；
- 关键字段的区别：
    - 分区表使用的是表外字段，需要在建表时额外指定，分区字段是虚拟的；
    - 分桶表使用的是表内字段，无需在建表时额外指定；
- 设置的区别：
    - 分区表使用之前，无需做任何设置；
    - 分通表使用之前，需要执行`set hive.enforce.bucketing = true`;


---
5. 假如你分析的数据有很多小文件，使用Hive-QL语句会出现什么异常情况？
- 小文件越多，在执行Hive-QL语句的时候，会产生越多的map；每一个map都会开启一个JVM进程，每一个JVM进程都会创建任务、执行任务，从而在这些流程当中造成大量的资源浪费，也增加了计算的时间；
- 小文件过多，会占用大量的内存，从而严重制约了集群的性能；
- 在Hive中，小文件的读写速度远小于大文件，进一步增加了计算的耗时；


---
6. 当Hive表的数据量很大时，如何用Order by进行全局排序？
- 单纯的使用Order by进行全局排序，会使得所有的数据都分配在一个Reducer中处理，对于数据量较大的情况，其计算速度会非常慢；
- 解决方法是，先使用sort by对其进行第一次排序，由于sort by的Reducer数量较多，因此加快了排序的速度，在每个Reducer内部进行局部排序；
- 然后，再用order by对其结果进行第二次排序，从而保证全局有序；


---
7. Hive中distinct和group by的区别
- 二者的设计意图、执行计划是不一样的，但在某些场景可以获得同样的查询结果；
- distinct
    - 用于单列、多列的去重，它将字段的内容加载在内存中，类似一个hash结构，key为字段名，最后计算hash中的key即可获得结果；
    - 缺点是对内存的开销较大，它将所有的待去重字段shuffle到一个Reducer中，从而造成数据倾斜；
- group by
    - 一般与聚合函数一同使用，它是先用sort方法将字段值排序，再分组计数；sort会开启多个Reducer，因此计算会更快；
    - 一般来说，对于大数据量的计算，group by的执行速度比distinct；


---
8. Hive中Join的MapReduce过程是什么？
- step1. Split。对数据进行split操作，将大文件切分为小的数据块；
- step2. Map。首先会对不同的表打上标记，以区分数据来源；然后，Map函数将数据块转化为<k, v>结构的键值对，其中，以连接字段作为key，以其余字段、及标记作为value；
- step3. Shuffle。将上述<k, v>对按key分组，把key值相同的键值对拉取到一个task节点；
- step4. Reduce。在每一个task节点中，针对key做笛卡尔积，然后根据hive-ql代码的业务逻辑执行求和、平均、聚合运算的操作，最后再根据where条件进行一些过滤，最后将全部结果输出；
- 具体代码如下所示：
```java
import java.io.IOException;
import java.util.HashMap;
import java.util.Iterator;
import java.util.Vector;

import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.io.Writable;
import org.apache.hadoop.mapred.FileSplit;
import org.apache.hadoop.mapred.JobConf;
import org.apache.hadoop.mapred.MapReduceBase;
import org.apache.hadoop.mapred.Mapper;
import org.apache.hadoop.mapred.OutputCollector;
import org.apache.hadoop.mapred.RecordWriter;
import org.apache.hadoop.mapred.Reducer;
import org.apache.hadoop.mapred.Reporter;


// MapReduce实现Join操作
public class MapRedJoin {
    public static final String DELIMITER = "\u0009";  // 字段分隔符


    // Map过程
    public static class MapClass extends MapReduceBase implements Mapper<LongWritable, Text, Text, Text> {
        public void configure(JobConf job) {
            super.configure(job);
        }

        public void map(LongWritable key, Text value, OutputCollector<Text, Text> output, Reporter reporter) throws IOException, ClassCastException {
            // 获取输入文件的全路径和名称
            String filePath = ((FileSplit)reporter.getInputSplit()).getPath().toString();
            // 获取记录字符串
            String line = value.toString();
            // 抛弃空记录
            if (line == null || line.equal(""))
                return;

            // 处理来自表A的记录
            if (filePath.contains("m_ys_lab_jointest_a")) {
                String[] values = line.split(DELIMITER);  // 按分隔符分割出字段
                if (values.length < 2)
                    return;

                String id = value[0];  // id
                String name = value[1];  // name

                output.collect(new Text(id), new Text("a#" + name));  // 给表A的信息打上标签"a#"
            }    

            // 处理来自表B的记录
            if (filePath.contains("m_ys_lab_jointest_b")) {
                String[] values = line.split(DELIMITER);  // 按分隔符分割出字段
                if (values.length < 3)
                    return;

                String id = values[0];  // id
                String statyear = values[1];  // statyear
                String num = values[2];  // num

                output.collect(new Text(id), new Text("#b" + statyear + DELIMITER + num));  // 给表B的信息打上标签"b#"
            }
        }
    }


    // Reduce过程
    public static class Reduce extends MapReduceBase implements Reducer<Text, Text, Text, Text> {
        public void reduce(Text key, Interator<Text> values, OutputCollector<Text, Text> output, Reporter reporter) throws IOException{
            Vector<String> vecA = new Vector<String>();  // 存放来自表A的数据
            Vector<String> vecB = new Vector<String>();  // 存放来自表B的数据

            while (values.hasNext()) {
                String value = values.next().toString();
                if (value.startsWith("a#")) {
                    vecA.add(value.substring(2));
                } else if (value.startsWith("b#")) {
                    vecB.add(value.substring(2));
                }
            }

            int sizeA = vecA.size();
            int sizeB = vecB.size();

            // 遍历两个向量，做笛卡尔积
            int i;
            int j;
            for (i = 0; i < sizeA; i++) {
                for (j = 0; j < sizeB; j++) {
                    output.collect(key, new Text(vecA.get(i) + DELIMITER + vecB.get(i)));
                }
            }
        }
    }


    protected void configJob(JobConf conf) {
        conf.setMapOutputKeyClass(Text.class);
        conf.setMapOutputValueClass(Text.class);
        conf.setOutputKeyClass(Text.class);
        conf.setOutputValueClass(Text.class);
        conf.setOutputFormat(ReportOutFormat.class);
    }
}
```


---
9. Hive中group by的MapReduce过程是什么？
- step1. Split。对数据进行split操作，将大文件切分为小的数据块；
- step2. Map。Map函数将数据块转化为<k, v>结构的键值对，其中，以group by的若干个字段作为key，以其余字段作为value；
- step3. Shuffle。将上述多个<k, v>键值对中，把key值相同的键值对拉取到一个task节点中；
- step4. Reduce。针对每个key值分别进行聚合、平均等计算，最后输出统计结果；
- 具体代码如下所示：
```java
import java.io.IOException;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.DoubleWritable;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.io.WritableComparable;
import org.apache.hadoop.io.WritableComparator;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Partitioner;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.mapreduce.lib.output.TextOutputFormat;
import org.apache.hadoop.util.Tool;
import org.apache.hadoop.util.ToolRunner;

public class GruopCount extends Configuration implements Tool{
    private Configuration configuration;
    @Override
    public Configuration getConf() {
        return this.configuration;
    }
    enum Counter{
        TIMER
    }
    @Override
    public void setConf(Configuration arg0) {
        this.configuration=arg0;
    }
    private static class Map extends Mapper<LongWritable, Text, Text, DoubleWritable>{

        @Override
        protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, Text,DoubleWritable >.Context context) throws IOException, InterruptedException {
            try{
                //获取查询的时间，如2015-05
                String querydate=context.getConfiguration().get("querydate");
                String[] columns=value.toString().split("\t");
                String datadate=columns[4];
                //将要查询的月份的所有数据输出到reduce中
                if(datadate.startsWith(querydate)){
                    //获取销售员
                    String salesman=columns[2];
                    //获取销售额
                    String salesmoney=columns[3];
                    //将销售员作为key输出，输出结果形如{“张三1”, [100,200,300, …]}，{“张三2”, [400,500,600, …]}
                    context.write(new Text(salesman),new DoubleWritable(Double.valueOf(salesmoney)));
                }
            }catch(Exception e){
                context.getCounter(Counter.TIMER).increment(1);
                e.printStackTrace();
                return;
            }
        }
    }
    private static class Reduce extends Reducer<Text, DoubleWritable, Text, DoubleWritable>{

        @Override
        protected void reduce(Text key, Iterable<DoubleWritable> values, Reducer<Text, DoubleWritable, Text, DoubleWritable>.Context context) throws IOException, InterruptedException {
            double sum=0;
            //获取当前遍历的value
            for (DoubleWritable v : values) {
                sum+=v.get();
            }
            context.write(key,new DoubleWritable(sum));
        }
    }
    @Override
    public int run(String[] arg0) throws Exception {
        Job job=Job.getInstance(getConf(), "groupcount");
        job.setJarByClass(GruopCount.class);
        FileInputFormat.setInputPaths(job, new Path(arg0[1]));
        FileOutputFormat.setOutputPath(job, new Path(arg0[2]));

        //默认即可，若需要进行效率调优使用此代码自定义分片
        //设置要分片的calss
        //job.setCombinerClass(Reduce.class);
        //设置分片calss
        //job.setPartitionerClass(SectionPartitioner.class);
        //设置分片个数
        //job.setNumReduceTasks(3);

        job.setMapperClass(Map.class);
        job.setReducerClass(Reduce.class);
        job.setOutputFormatClass(TextOutputFormat.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(DoubleWritable.class);
        job.waitForCompletion(true);
        return job.isSuccessful()?0:1;
    }
    public static void main(String[] args) throws Exception {
        String[] args2=new String[4];
        args2[0]="ss";
        args2[1]="hdfs://192.168.1.55:9000/groupcount-in/t_product_sales.txt";
        args2[2]="hdfs://192.168.1.55:9000/groupcount-out";
        args2[3]="2015-05";
        Configuration configuration=new Configuration();
        configuration.set("querydate", args2[3]);
        System.out.println(ToolRunner.run(configuration, new GruopCount(), args2));
    }
}
```


---
10. Hive中distinct的MapReduce过程是什么？
- step1. Split。对数据进行split操作，将大文件切分为小的数据块；
- step2. Map。Map函数将数据块转化为<k, v>结构的键值对，其中，以distinct的若干个字段作为key，以其余字段作为value；
- step3. Shuffle。将上述多个<k, v>键值对中，把key值相同的键值对拉取到一个task节点中；
- step4. Reduce。针对每个key值分别去除重复的value，最后输出统计结果；
- 具体代码如下所示：
```java
// Hive-QL: select distinct x from table;
import java.io.IOException;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.GenericOptionsParser;

public class Dedup {

    public static class RemoveDupMapper extends Mapper<Object, Text, Text, NullWritable> {
    	public void map(Object key, Text value, Context context) throws IOException, InterruptedException {
    		context.write(value, NullWritable.get());
    	}
    }

    public static class RemoveDupReducer extends Reducer<Text, NullWritable, Text, NullWritable> {
    	public void reduce(Text key, Iterable<NullWritable> values, Context context)
        throws IOException, InterruptedException {
    		context.write(key, NullWritable.get());
    	}
    }

    public static void main(String[] args) throws Exception{
        Configuration conf = new Configuration();
        conf.set("mapred.jar","Dedup.jar");   //去掉这行也能运行，目前还不知道这行有什么用
        String[] ioArgs=new String[]{"dedup_in","dedup_out"};
        String[] otherArgs = new GenericOptionsParser(conf, ioArgs).getRemainingArgs();
        if (otherArgs.length != 2) {
        	System.err.println("Usage: Data Deduplication <in> <out>");
        	System.exit(2);
        }

        Job job = new Job(conf, "Data Deduplication");
        job.setJarByClass(Dedup.class);

        //设置Map、Combine和Reduce处理类
        job.setMapperClass(RemoveDupMapper.class);
        job.setCombinerClass(RemoveDupReducer.class);
        job.setReducerClass(RemoveDupReducer.class);

        //设置输出类型
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(NullWritable.class);

        //设置输入和输出目录
        FileInputFormat.addInputPath(job, new Path(otherArgs[0]));
        FileOutputFormat.setOutputPath(job, new Path(otherArgs[1]));
        System.exit(job.waitForCompletion(true) ? 0 : 1);
	}
}

```


---
11. Hive的数据倾斜是如何造成的？（重点）
- 数据倾斜现象，本质上是各个Reducer上的数据量差异过大造成的。其具体原因如下：
- 原因一：key分布不均匀。在Shuffle阶段，会将key值相同的<k, v>对拉取到同一个task节点中进行计算。如果key分布不均匀，就会造成一些节点的数据很多，而另一些节点的数据很少，从而产生数据倾斜；
- 原因二：业务数据本身的特性，就容易产生数据倾斜。例如电商行业，在618、双11等节假日的数据会较多，而工作日的数据较少；如果按照日期来分区，那么自然就会产生数据倾斜；
- 原因三：Hive-QL语句造成的数据倾斜。例如group by时维度过小，导致某些Reducer的数据过多；join时某张表的key很集中；count distinct时null等特殊值过多等；


---
12. Hive的数据倾斜有哪些解决办法？（重点）
- 参数调节
    - `set hive.map.aggr = true;`，在map阶段聚合，相当于Combiner操作；
    - `set hive.groupby.skewindata = true;`，在数据倾斜现象发生时进行负载均衡，将数据随机分散之后并行计算，再全局合并；
- Hive-QL语句调节
    - 在join操作时，选择key分布最均匀的表作为主表；
    - 在join之前，提前做好列裁剪、过滤操作，以减小表的数据量；
    - 让较小的维度表先写入内存，在map阶段完成reduce；
    - 将倾斜的数据单独统计，将不倾斜的数据单独统计，最后将结果union；
    - 在count distinct时，将null值单独处理，给定特殊值、随机值，或者使用`nvl()`函数；
    - 用group by代替count distinct；
    - 灵活使用分区表、分桶表，灵活选择分区字段、分桶字段；
- 数据块不均匀
    - 当待查询的数据块存在大文件和小文件时，要先对小文件合并，从而减小JVM的进程，进而减小磁盘和内存的开销；


---
13. Hive中的排序关键字有哪些？
- order by: 全局排序；它会将待排序的字段都shuffle到同一个Reducer当中计算，因此当数据较大时，排序速度会很慢；
- sort by: 局部排序；它会将待排序的字段shuffle到多个Reducer中，在每个Reducer的内部进行排序，因此只是局部有序；
- distribute by: 将指定的字段作为<k, v>键值对的key，然后将key相同的数据输出到同一个Reduce当中；
- cluster by: 当sort by和distribute by字段相同时，其效果等同于cluster by；


---
14. 海量的日期数据分布在100台电脑中，如何高效统计出这批数据的TOP10的日期？
- Step1. 分别统计出100台电脑中，每台电脑的top10日期。在每台电脑内部进行数据排序的过程中，可以将`sort by`和`order by`嵌套使用的方式；
- Step2. 汇总上述数据，然后在这100台电脑 * 10条数据中，统计出top10的日期；


---
15. Hive中追加、导入数据的4种方式是什么？其简要语法是什么？
- 从本地导入
    - `load data local inpath [file_path] into table [table_name]`
    - `load data local inpath [file_path] overwrite into table [table_name]`
- 从HDFS导入
    - `load data inpath [file_path] into table [table_name]`
    - `load data inpath [file_path] overwrite into table [table_name]`
- 从Hive表中导入。`create table [table_a] as select * from [table_b]`
- 从Hive的查询结果导入。`insert into/overwrite table [table_name] [Hive-ql]`


---
16. Hive导出数据有几种方式？其简要方法是什么？
- Linux shell命令导出数据到本地
    - `hive -e [Hive-ql] > [file_name]`
    - `hive -f [Hive_file] >> [file_name]`
- overwrite导出
    - 导出到本地：`insert overwrite local directory [file_path] row format delimited fields terminated by '\t' [Hive-ql]`
    - 导出到HDFS：`insert overwrite directory [file_path] row format delimited fields terminated by '\t' [Hive-ql]`
- Sqoop导出到关系型数据库。`sqoop export`命令行配置导出到关系型数据库；


---
17. 简述Hive的特点
- Hive是由Facebook开源、基于Hadoop的一种数据仓库工具，可以将存储在HDFS中的结构化的数据文件，映射成一张张数据库的表，并提供完备的Hive-ql用于查询；
- 还可以将SQL语句，转化为MapReduce任务去执行，从而避免了开发MR程序；
- 同时，其底层的计算引擎，可更换为Spark-SQL、Tez；
- 其优点在于，学习成本低，适用于海量数据的离线处理、批处理，适用于OLAP系统；
- 其缺点在于，不支持行级别的数据更新，查询数据较慢，不能用于实时数据分析，不适用于OLTP系统；


---
18. Hive保存元数据的方式有哪些？
- 本地模式下，使用MySQL保存元数据，这是最常见的方式；
- 远程模式下，Hive客户端会连接远程服务器，采用Thrift协议进行通信，在该服务器上查询元数据；
- 单元测试模式下，默认采用Derby数据库存储元数据，且每次只能有一个进程连接；


---
19. 写出Hive中split、coalesce及collect_list函数的用法
- split：将字符串转化为数组，即：split('a,b,c,d', ',') ==> ["a", "b", "c", "d"]；
- coalesce(T v1, T v2, …)：返回参数中的第一个非空值；如果所有值都为 NULL，那么返回NULL；
- collect_list：列出该字段所有的值，不去重，例如`select collect_list(id) from table`；


---
20. Hive如何进行权限控制？
- 基于HDFS的权限管理模型。Hive本质上是操作HDFS的数据，因此Hive的权限可以与HDFS的权限保持一致；
- Hive默认的授权模式。仅仅是为了防止用户误操作，而不是防止恶意用户访问未经授权的数据；
- 基于SQL标准的Hive授权，推荐使用；可以创建不同的角色，然后给不同的用户赋予不同的角色，再给不同的角色配置不同的操作权限即可；
`set hive.security.authorization.enabled = true;`
`set hive.server2.enable.doAs = false;`
`set hive.users.in.admin.role = root;`


---
21. Hive中的文件存储格式有哪些？各有什么特点和区别？
- TextFile
    - 默认格式，存储方式为行存储，数据不做压缩，磁盘开销大，数据解析开销大；
    - 可结合Gzip、Bzip2使用(系统自动检查，执行查询时自动解压)；
    - 但使用这种方式，压缩后的文件不支持split，Hive不会对数据进行切分，从而无法对数据进行并行操作；
    - 并且在反序列化过程中，必须逐个字符判断是不是分隔符和行结束符，因此反序列化开销会比SequenceFile高几十倍；
- SequenceFile
    - SequenceFile是Hadoop API提供的一种二进制文件支持，存储方式为行存储，其具有使用方便、可分割、可压缩的特点；
    - SequenceFile支持三种压缩选择：NONE/ RECORD/ BLOCK；Record压缩率低，一般建议使用BLOCK压缩；
    - 优势是文件和hadoop api中的MapFile是相互兼容的；
- Parquet
    - 列式存储结构；
    - 有良好的压缩性能和查询性能，但是写入数据的速度一般比较慢，这种格式主要应用在Impala中；
- RCfile
    - 存储方式：数据按行分块，每块按列存储；结合了行存储和列存储的优点；
    - RCFile保证同一行的数据位于同一节点，因此元组重构的开销很低；
    - 其次，像列存储一样，RCFile能够利用列维度的数据压缩，并且能跳过不必要的列读取；
    - 相比TEXTFILE和SEQUENCEFILE，RCFILE由于列式存储方式，数据加载时性能消耗较大，但是具有较好的压缩比和查询响应；
    - 数据仓库的特点是一次写入、多次读取，因此整体来看，RCFILE相比其余两种格式具有较明显的优势；
- ORCfile
    - 存储方式：数据按行分块，每块按照列存储；
    - 默认采用ZLIB的压缩格式；
    - 压缩快，快速列存取；效率比RCFile高，是RCFile的改良版本；



---
22. Hive join过程中大表小表的放置顺序是什么样的？
- 在编写带有join操作的Hive-QL语句时，应该将记录数少的表、子查询放在join操作符的左边；
- 因为在reduce阶段，位于join操作符左边的表的内容会被加载进内存；载入条目较少的表可以有效减少oom(out of memory)，即内存溢出；
- 所以对于同一个key来说，对应的数据量小的表放前，数据量大的表放后；


---
23. Hive自定义UDF函数的流程是怎样的？继承的是哪个类？
- 使用Java新建一个类，继承`org.apache.hadoop.Hive.ql.exec.UDF`的UDF类；
- 重写`evaluate`方法，写入业务逻辑的函数代码；
- 打成jar包，并放置在Linux的路径下；
- 在Hive中添加jar包到类的路径下，`add jar [jar_path]`；
- 在Hive中新建函数，`create function [func_name] as '[class_name]' using jar [jar_path]`


---
24. Hive与HBase的区别和联系是什么？
- Hive
    - 是一种基于Hadoop的数据仓库工具，能将HDFS中存储的数据块映射成一张数据库的表，从而使用Hive-QL语句进行查询；
    - 它能将Hive-QL语句转化为MapReduce程序，从而减少了数据开发的工作量；
    - 主要适用于离线处理、批处理；
- HBase
    - 是一种Hadoop生态的No-SQL数据库，用于存储<k, v>键值对；
    - 适用于实时计算；
    - 数据可以从Hive转移到HBase，也可以从HBase转移到Hive；


---
25. Hive-QL是如何转化为MapReduce程序的？
- Step1. 语法解析：Hive内的SQL Parser解释器定义Hive-QL的语法规则，完成SQL词法、语法解析，将SQL转化为抽象语法树AST Tree；
- Step2. 抽象查询：Physical Plan编译器遍历AST Tree，抽象出查询的基本组成单元QueryBlock；
- Step3. 遍历单元：遍历QueryBlock，翻译为执行操作树OperatorTree；
- Step4. 优化数据：Query Optimizer逻辑层优化器进行OperatorTree变换，合并不必要的ReduceSinkOperator，减少shuffle数据量；
- Step5. 翻译任务：遍历OperatorTree，翻译为MapReduce任务；
- Step6. 提交执行：物理层优化器进行MapReduce任务的变换，生成最终的执行计划，并由Execution执行器提交给SQL执行；


---
26. Hive视图有什么特点？
- Hive的视图是逻辑概念，不存在物理上的视图的存储；
- Hive中视图的查询语法与表类似；
- Hive视图只能读，不能insert/ overwrite；


---
27. Hive参数配置的方法是什么？请列举8个常用的参数配置
- 修改配置文件。Hive的默认配置文件在`${Hive_HOME}/conf/Hive-default.xml`路径下，如果要修改参数配置，可以新建一个`${Hive_HOME}/conf/Hive-site.xml`文件；
- 命令行参数。在从linux进入Hive CLI界面时，通过给定`--hiveconf [param] = [value]`的方式配置；
- Hive命令。在进入Hive CLI内部，通过`set [param] = [value]`的方式配置；
- 动态分区：`set hive.exec.dynamic.partition = true`；
- MR模式，严格模式不允许笛卡尔积：`set hive.mapred.mode = strict`；
- 合并小文件：`set hive.merge.mapredfiles = true`；


---
28. Hive有哪些优化策略？（Hive调优，及其重要）
- 配置优化
    - `set hive.exec.parallel = true`，开启并发执行，使得Hive-QL中并行的stage可以并行计算；
    - `set hive.fetch.task.conversion = more`，默认为minimal，进行简单查询时会开启MapReduce程序；改为more后，简单查询不用MapReduce；
    - `set mapred.max.split.size = 256000000;`
    - `set mapred.min.split.size.per.node = 100000000;`
    - `set mapred.min.split.size.per.rack = 100000000;`
    - `set hive.input.format = org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;`
    - `set hive.auto.convert.join = true;`，join时将小表写入内存，而无需下发到其他节点，从而提升计算效率；
    - `set hive.mapjoin.smalltable.filesize = 25000000;`，设置表小于25000000字节，即可认为是小表；
    - `set hive.exec.reducers.bytes.per.reducer = 1000000000`，调整reducer的个数，调大使得Reducer数据量增大，从而减少了Reducer的数量；调小使得其数量增大，开启Reducer的开销变多；
    - `set hive.exec.reducers.max = 999`，防止一个job消耗资源过大，导致其他job无法计算的情况；
    - `set mapred.job.reuse.jvm.num.tasks = 10`，开启jvm重用，使得一个job中可以不必开启过多的jvm；
    - `set hive.multigroupby.singlereducer = true;`，将过个group by组装到一个MapReduce当中执行；
    - `set hive.exec.mode.local.auto = true;`，开启本地模式的运行开关；
    - `set hive.exec.mode.local.auto.input.files.max = 4;`，当task数量小于等于4个才启用本地模式；
    - `set hive.exec.mode.local.auto.inputbytes.max = 50000000`，当输入数据量小于等于50000000时才启用本地模式；
    - `set hive.exec.compress.output = true;`，对Hive的输出结果进行压缩；
    - `set hive.exec.compress.intermediate = true;`，对Hive的中间结果进行压缩；
- 分区表、分桶表、分库分表
    - 合理使用分区表，可以加快查询速度；
    - 合理使用分桶表，获得比分区表更细粒度的划分，从而加快join、抽样的效率；
    - 拆分表，将极大的表按行或列拆分为多个子表，将大表中不常使用的部分做冷表处理；
- Hive-QL优化
    - join时，将where写在子查询中，从而使得where在map阶段执行；
    - join时，将小表写在左边，大表写在右边，从而使小表加载到内存当中；
    - join时，若表中null值过多，则需要对null值单独处理，再对剩余数据单独处理，最后将结果union；
    - 用cluster by/ sort by代替大表的order by；
    - 使用insert into代替union all；
    - 使用group by代替distinct实现去重；
    - 使用count() + group by代替count(distinct column)；
- 数据的压缩与存储
    - 创建表时，尽量使用orc/ parquet等列式存储格式，其每一列数据在物理上是存储在一起的，Hive查询只会遍历需要的列数据，从而减少了处理的数据量；
    - 对数据进行压缩，可以减少数据量，从而减小磁盘IO和网络IO的开销；考虑压缩比率、解压速度综合选择压缩方式；
- 数据倾斜解决方案
    - 参数调节
        - `set hive.map.aggr = true;`，在map阶段聚合，相当于Combiner操作；
        - `set hive.groupby.skewindata = true;`，在数据倾斜现象发生时进行负载均衡，将数据随机分散之后并行计算，再全局合并；
    - Hive-QL语句调节
        - 在join操作时，选择key分布最均匀的表作为主表；
        - 在join之前，提前做好列裁剪、过滤操作，以减小表的数据量；
        - 让较小的维度表先写入内存，在map阶段完成reduce；
        - 将倾斜的数据单独统计，将不倾斜的数据单独统计，最后将结果union；
        - 在count distinct时，将null值单独处理，给定特殊值、随机值，或者使用`nvl()`函数；
        - 用group by代替count distinct；
        - 灵活使用分区表、分桶表，灵活选择分区字段、分桶字段；
    - 数据块不均匀
        - 当待查询的数据块存在大文件和小文件时，要先对小文件合并，从而减小JVM的进程，进而减小磁盘和内存的开销；
- 执行计划
    - `explain [Hive-QL]`查看执行计划，优化业务逻辑，减少job的数量；


---
29. 向Hive-QL传递变量的方式有哪些？
- Hive命令参数。在启动Hive CLI时传入参数，语法为`hive --hiveconf [param] = [value] --hivevar p_date = [var]`；
- 系统环境变量。直接获取系统的参数，例如获取Linux配置的参数；


---
30. Hive的动态分区如何实现？（参数配置的命令要牢记）
- 在向Hive分区表中插入数据时，如果想插入数据到多个分区，那么可以采用动态分区的方式，基于参数自动的判断分区字段，一次性插入从而减少工作量；
- `set hive.exec.dynamic.partition = true`，表示开启动态分区功能；
- `set hive.exec.dynamic.partition.mode = nonstrict`，表示允许所有的分区都是动态的；


---
31. Hive-QL手写题目
- https://www.jianshu.com/p/4e5645c1b403
- https://www.jianshu.com/p/1c99ba078d4f
- https://blog.csdn.net/qq_41568597/article/details/84309503


---
32. 简要描述数据库中的null，说出null在Hive底层如何存储？
- null在Hive底层默认是用'\N'来存储的；
- `set hive.null.format = ''`，可以改变null值的存储方式，例如这里将null转化为空字符串存储；
- null与任何值运算的结果都是null，可以使用`where col is null`/ `where col is not null`/ `nvl()`的筛选条件或函数，指定在其值为null情况下的取值；


---
33. Hive的join有几种方式？如何实现join？
- 在reduce端进行join，是最常用的join方式
    - reduce端的主要工作：在reduce端获取分组完成的<k, v>键值对，只需要在每个Reducer中将不同来源的<k, v>对进行笛卡尔积即可；
- 在map端进行join
    - Map端的主要工作：为来自不同表的<k, v>键值对打标签，以区别不同来源的记录；
    - 然后用连接字段作为key，其余部分和新加的标志作为value，最后进行输出；
    - 使用场景：一张表很小、一张表很大，在提交作业的时候先将小表文件放到该作业的DistributedCache中，即集群的内存中；
    - 然后扫描大表，看大表对应的<k, v>值是否能够在内存中找到相同key，如果有则直接输出结果；
- Semi Join
    - Semi Join就是左边连接，是reduce join的一种变种，在map端过滤掉一些数据，在网络传输过程中，只传输参与连接的数据，减少了shuffle的网络传输量；
    - 其他和reduce的思想是一样的
    - 将小表中参与join的key单独抽取出来通过DistributeCache分发到相关节点，在map阶段扫描连接表，将joinkey不在内存hashset的纪录过滤掉，让参与join的纪录通过shuffle传输到reduce端进行join，其他和reduce join一样；


---
34. Hive是如何实现分区的？
- 建表时创建分区：`create table [table_name] ( [Hive-ql] ) partitioned (p_date string comment '')`
- 增加分区：`alter table [table_name] add partition (p_date = '2020-01-01')`
- 删除分区：`alter table [table_name] drop partition (p_date = '2020-01-01')`


---
35. Hive有哪些复合数据类型？
- Map
- Struct
- Array
- Uniontype


---
36. Hive有哪些窗口函数？
- `count(order_id) over()`
- `count(order_id) over(partition by product_type)`
- `count(order_id) over(partition by product_type order by product_type)`
- `sum(order_id) over()`
- `sum(order_id) over(partition by product_type)`
- `sum(order_id) over(partition by product_type order by product_type)`
- `first_value(order) over(partition by product_type order by product_type)`
- `last_value(order) over(partition by product_type order by product_type)`
- `rank() over(partition by product_type order by product_type)`
- `dense_rank() over(partition by product_type order by product_type)`
- `row_number() over(partition by product_type order by product_type)`


---
37.


---
38. Hive如何实现高可用？
- 这个一般是大数据运维实现的，通过一些配置文件，hiveserver2，zookeeper设置的形式开展；


---
39. Hive的`select 1`如何转化为MapReduce程序运行？
- 可以通过`explain select 1`来观察执行计划；
- Fetch阶段
    - TableScan。表名为空，即dummy_table，limit无限制，即-1；
    - Split。由于表为空，因此小文件个数为1；
- Select阶段
    - 按表达式直接输出`1`，字段别名为默认值`_col0`


---
40. Hive的`select name from user_info limit 10`如何转化为MapReduce程序运行？
- 可以通过`explain select name from user_info`来观察执行计划；
- Fetch阶段
    - TableScan。表名为user_info，limit为10；
    - Split。表文件很小，因此小文件个数为1；
- Select阶段
    - 按表达式直接输出`name`，输出行数为10；


---
41. Hive一般用于大表的查询，当处理数据量较小的表关联查询时，如何加快查询速度？
- 使用Hive on Spark，更换底层查询引擎为Spark，从而在内存中进行计算，也不会有OOM的情况发生，以加快查询速度；
- 使用MapJoin
    - 首先，在客户端本地开启一个Task，读取小表的数据，将其转换为一个HashTable的数据结构，并写入本地文件当中，之后再将其写入分布式缓存中；
    - 然后，开启另外一个Task去扫描大表，在map阶段，根据大表的每一条记录，去和分布式缓存中的小表的HashTable关联，并输出结果；
    - 由于MapJoin没有Reduce，所以Map直接输出结果文件，MapTask的数量和结果文件一致；


---
42. Hive-ql中出现null值如何处理？
- Hive中的null值在底层都存储为文本`\N`的形式，null参与的任何运算结果都为null；
- 实际计算中可用函数`nvl(col, 1)`来对含有null的字段进行转换；
- `set serialization.null.format = ''`，设置null的字符格式为空字符串，还可以节省磁盘空间；
- 对null值使用`hash()`函数转化；


---
43. Hive的UDF函数有哪几类？
- UDF(user defined function): 用户自定义函数，接收单个数据行作为输入，产生一个新数据行作为输出，例如nvl()/ regexp_replace()；
- UDAF(user defined aggregation function): 用户自定义聚集函数，接收多个数据行作为输入，产生一个数据行作为输出，例如count()/ max()；
- UDTF(user defined table function): 用户自定义表格函数，接收单个数据行作为输入，输出爆炸数据，例如explode()/ collect_list()/ alteral explode()；


---
44. 对于一张表中某个字段值的sum/count等操作，会有几个Reducer？
- mapper的数量是由输入文件的分片InputSplit决定的；但是像sum/count这种全局统计，只存在1个Reducer进行计算；


---
45. Hive-QL中explode和lateral view有什么作用？
- `explode`用于将一行复杂的array或map结构拆分为多行，属于UDTF函数；例如`select explode(split('a,b,c,d,e', ','))`，输出五行数据；
- `lateral view`用于与UDTF结合使用，从而生成多列数据，例如与`explode`结合使用；例如`select a, b from test lateval view explode(b) as b`；
