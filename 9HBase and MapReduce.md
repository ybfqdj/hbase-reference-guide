# HBase and MapReduce #
Apache MapReduce是一种用来分析大量数据的软件框架，是Hadoop最常使用的框架。MapReduce本身超出本文范围。 A good place to get started with MapReduce is [http://hadoop.apache.org/docs/r2.6.0/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html](http://hadoop.apache.org/docs/r2.6.0/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html)。 MapReduce version 2 (MR2)is now part of [YARN](http://hadoop.apache.org/docs/r2.4.1/hadoop-yarn/hadoop-yarn-site/).

本章讨论下要在HBase中使用MapReduce需要指定的配置步骤。另外，讨论下其他HBase和MapReduce工作之间的相互作用和问题。 Finally, it discusses Cascading, an alternative API for MapReduce。

## 46.HBase, Mapreduce, and the CLASSPATH ##
默认情况下，MapReduce job部署在MR集群上，不会与$HBASE_CONF_DIR下的HBase配置或HBase类产生关联。

要使MapReduce工作获得它们需要的，你要将**hbase-site.xml**放入**$HADOOP_HOME/conf**目录，并将HBase jar放入**$HADOOP_HOME/lib**目录，每个集群都要如此配置。或者你可以编辑***$HADOOP_HOME/conf/hadoop-env.sh***，并把它加入到**HADOOP_CLASSPATH **变量中，但并不推荐你这种方法，因为这会影响你根据Hbase referenc的Hadoop安装。这也需要你重启hadoop集群以使Hadoop能够使用Hbase data。

建议的方法是使HBase自己增加他的依赖jars并使用**HADOOP_CLASSPATH** or **-libjars**

从0.90.x起，HBase可自动将依赖jars加入到job配置中，只需保证相关jars存在于本地的**CLASSPATH**。下面的事例运行了绑定的HBase RowCounter MR工作，其中表名叫**usertable**。如果你没有设置命令中（该命令以$为前缀，并被花括号包围）预期的环境变量，可以使用真实系统路径替代。确保为你的系统使用正确的HBase jar。单引号会使shell执行子命令setting the output of **hbase classpath** (the command to dump HBase CLASSPATH) to **HADOOP_CLASSPATH**.This example assumes you use a BASH-compatible shell.

    $ HADOOP_CLASSPATH=`${HBASE_HOME}/bin/hbase classpath` ${HADOOP_HOME}/bin/hadoop jar ${HBASE_HOME}/lib/hbase-server-VERSION.jar rowcounter usertable

当这个命令运行时，内部地， HBase jar会去找它需要的依赖并将它们加入到MR工作配置中。See the source at ***TableMapReduceUtil#addDependencyJars(org.apache.hadoop.mapreduce.Job)*** for how this is done.

命令 ***hbase mapredcp*** 能帮你转储MR需要的CLASSPATH条目，同样jars ***TableMapReduceUtil#addDependencyJars***会加入。你可以将它们一起放入HBase配置目录 **HADOOP_CLASSPATH** 中。那些不打包依赖或调用***TableMapReduceUtil#addDependencyJars***的工作，下面的命令结构式必须的：

    $ HADOOP_CLASSPATH=`${HBASE_HOME}/bin/hbase mapredcp`:${HBASE_HOME}/conf hadoop jar MyApp.jar MyJobMainClass -libjars $(${HBASE_HOME}/bin/hbase mapredcp | tr ':' ',') ...

> 这个案例可能会出错如果你在HBase的build 目录而不是安装目录运行它。可能会有如下错误：
> `java.lang.RuntimeException: java.lang.ClassNotFoundException: org.apache.hadoop.hbase.mapreduce.RowCounter$RowCounterMapper`
> 如果这个发生了，试着将命令改成如下，使它可以使用build环境中***target/directory***下的HBase jars
> `$ HADOOP_CLASSPATH=${HBASE_BUILD_HOME}/hbase-server/target/hbase-server-VERSION-SNAPSHOT.jar:`${HBASE_BUILD_HOME}/bin/hbase classpath` ${HADOOP_HOME}/bin/hadoop jar ${HBASE_BUILD_HOME}/hbase-server/target/hbase-server-VERSION-SNAPSHOT.jar rowcounter usertable`

***0.96.1至0.98.4的Hbase MapReduce使用者请注意***
某些使用Hbase的MR工作可能无法启动。The symptom is an exception similar to the following:

> Exception in thread "main" java.lang.IllegalAccessError: class
    com.google.protobuf.ZeroCopyLiteralByteString cannot access its superclass
    com.google.protobuf.LiteralByteString
    at java.lang.ClassLoader.defineClass1(Native Method)
    at java.lang.ClassLoader.defineClass(ClassLoader.java:792)
    at java.security.SecureClassLoader.defineClass(SecureClassLoader.java:142)
    at java.net.URLClassLoader.defineClass(URLClassLoader.java:449)
    at java.net.URLClassLoader.access$100(URLClassLoader.java:71)
    at java.net.URLClassLoader$1.run(URLClassLoader.java:361)
    at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
    at java.security.AccessController.doPrivileged(Native Method)
    at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
    at
    org.apache.hadoop.hbase.protobuf.ProtobufUtil.toScan(ProtobufUtil.java:818)
    at
    org.apache.hadoop.hbase.mapreduce.TableMapReduceUtil.convertScanToString(TableMapReduceUtil.java:433)
    at
    org.apache.hadoop.hbase.mapreduce.TableMapReduceUtil.initTableMapperJob(TableMapReduceUtil.java:186)
    at
    org.apache.hadoop.hbase.mapreduce.TableMapReduceUtil.initTableMapperJob(TableMapReduceUtil.java:147)
    at
    org.apache.hadoop.hbase.mapreduce.TableMapReduceUtil.initTableMapperJob(TableMapReduceUtil.java:270)
    at
    org.apache.hadoop.hbase.mapreduce.TableMapReduceUtil.initTableMapperJob(TableMapReduceUtil.java:100)
...

这是由HBASE-9867版本引入的优化引起的，即引进了类加载依赖。

This affects both jobs using the -libjars option and "fat jar," those which package their runtime dependencies in a nested lib folder.

要满足新类加载器的要求，hadoop的类路径必须包含hbase-protocol.jar。See 46.HBase, MapReduce, and the CLASSPATH for current recommendations for resolving classpath errors.

这个可以在系统范围内被解决通过将hbase-protocol.jar的索引添加到Hadoop的lib目录，通过符号链接或者将jar直接拷贝到目录中。

也可以每次job启动时通过将它包含进HADOOP_CLASSPATH环境变量中解决。当启动job时要打包依赖，下面所有三个job启动命令都可以：

    $ HADOOP_CLASSPATH=/path/to/hbase-protocol.jar:/path/to/hbase/conf hadoop jar MyJob.jar MyJobMainClass
    $ HADOOP_CLASSPATH=$(hbase mapredcp):/path/to/hbase/conf hadoop jar MyJob.jar MyJobMainClass
    $ HADOOP_CLASSPATH=$(hbase classpath) hadoop jar MyJob.jar MyJobMainClass

For jars that do not package their dependencies, the following command structure is necessary:

    $ HADOOP_CLASSPATH=$(hbase mapredcp):/etc/hbase/conf hadoop jar MyApp.jar MyJobMainClass -libjars $(hbase mapredcp | tr ':' ',') ...

## 47.MapReduce Scan Caching MR扫描缓存 ##

TableMapReduceUtil恢复了设置对传入扫描对象的缓存选项（返回客户端结果之前可以缓存的行数量），这个功能曾因为0.95版中的bug而被放弃，这个bug已经在0.98.5和0.96.3中解决了选择扫描缓存的优先级顺序如下：
	
1. 扫描对象的缓存设置。
2. 配置选项 hbase.client.scanner.caching中指定的缓存设置，可以在hbase-site.xml中手动设置也可以通过辅助方法TableMapReduceUtil.setScannerCaching()设置。
3. HConstants.DEFAULT_HBASE_CLIENT_SCANNER_CACHING的默认值，100.

对缓存设置的优化是对客户端请求结果的等待时间和客户端需要获得结果数量之间的一个平衡。如果设置太大，客户端可能会因超时而结束。如果设置太小，扫描需要返回多块结果。如果把扫描想成一把铲子，大的缓存相当于一个大铲子，小铲子意味着为了填满bucket需要铲更多次。

上面提到的优先级列表允许您设置一个合理的默认值，并为特定的操作重写它。

## 48. Bundled HBase MapReduce Jobs##
The HBase JAR also serves as a Driver for some bundled MapReduce jobs. To learn about the bundled MapReduce jobs, run the following command.

    $ ${HADOOP_HOME}/bin/hadoop jar ${HBASE_HOME}/hbase-server-VERSION.jar
    An example program must be given as the first argument.
	Valid program names are:
	  copytable: Export a table from local cluster to peer cluster
      completebulkload: Complete a bulk data load.
      export: Write table data to HDFS.
      import: Import data written by Export.
      importtsv: Import data in TSV format.
      rowcounter: Count rows in HBase table

Each of the valid program names are bundled MapReduce jobs. To run one of the jobs, model your command after the following example.

    $ ${HADOOP_HOME}/bin/hadoop jar ${HBASE_HOME}/hbase-server-VERSION.jar rowcounter myTable

## 49.HBase作为一个MapReduce作业的数据源和数据接收器 ##
HBase可以被用作mapreduce的数据源，TableInputFormat 和数据接收器，TableOutputFormat或MultiTableOutputFormat。写Mr作业，读或写HBase,最好是子类TableMapper and/or tableReducer。See the do-nothing pass-through classes IdentityTableMapper and IdentityTableReducer for basic usage. For a more involved example, see RowCounter or review the org.apache.hadoop.hbase.mapreduce.TestTableMapReduce unit test.

如果运行MR job时使用HBase作为数据源或数据接收器，需要在配置中指定源和结果表及列名。

当你从HBase中读时， TableInputFormat从HBase中请求regions表并制成map,或者是map-per-region或是mapreduce.job.maps map，无论哪个是更小的。如果job只有两个map，提高mapreduce.job.maps使其大于regions数量。如果你在每个regionServer节点上都运行了TaskTracker/NodeManager，Maps将运行在相邻的TaskTracker/NodeManager上。当向HBase写入时，可以避免Reduce步骤并从map中写回HBase。当任务不需要分类和整理，这种map发出数据的方法会管用。插入时, Hbase 'sorts'中没有double-sorting（以及对你的MR集群shuffling data），除非你需要这个。如果不需要Reduce，map可能会发出处理的记录数来报告任务结束，或将Reduce数设为0，并使用TableOutputFormat。如果在你的案例中，有必要启动Reduce，通常你应该使用多个reducers，这样负载可以分布到整个HBase集群中。

HRegionPartitioner，HBase新的分割，可启动与regions同样数量的reducers.如果你的表格很大而上传不会大大改变现有region的数量，HRegionPartitioner很合适。 否则使用默认的分割方法。


## 50. 批量导入时直接写入HFiles ##
如果你在导入一张新的表格，可以忽略HBase API直接将内容写入文件系统，格式化成HBase数据文件（HFiles.）导入将运行得更快，也许是一个数量级的更快。

## 51. RowCounter Example ##
包括RowCounter的MR job使用TableInputFormat并记录指定表中行的数量。 To run it, use the following command:

    $ ./bin/hadoop jar hbase-X.X.X.jar

这将引用HBase MapReduce Driver类。在提供的选择中选择rowcounter。会将rowcounter使用建议打印到标准输出中。指定表名、计数的列以及输出目录。

## 52.map任务分解  ##
### 52.1. 默认的HBase MapReduce分离器 ###
在MapReduce job中使用TableInputFormat从HBase表中请求数据时，分离器将为表的每个region生成一个map任务。因此， 如果表中有100个regions这有100个map任务，不管扫描操作中选择了多少列族。

### 52.2. 自定义分离器 ###
For those interested in implementing custom splitters, see the method getSplits in TableInputFormatBase. That is where the logic for map-task assignment resides.

## 53. HBase MapReduce Examples ##
### 53.1. HBase MapReduce Read Example###
下面是以只读方式使用HBase作为MR数据源的例子。明确地，这里只有Mapper实例而没有Reducer，不会从Mapper中发出任何数据。

    Configuration config = HBaseConfiguration.create();
	Job job = new Job(config, "ExampleRead");
	job.setJarByClass(MyReadJob.class);     // class that contains mapper

    Scan scan = new Scan();
	scan.setCaching(500);        // 1 is the default in Scan, which will be bad for MapReduce jobs
	scan.setCacheBlocks(false);  // don't set to true for MR jobs
	// set other scan attrs
    ...
	
    TableMapReduceUtil.initTableMapperJob(
	 tableName,        // input HBase table name
	 scan,             // Scan instance to control CF and attribute selection
	 MyMapper.class,   // mapper
	 null,             // mapper output key
	 null,             // mapper output value
	 job);
	job.setOutputFormatClass(NullOutputFormat.class);   // because we aren't emitting anything from mapper

    boolean b = job.waitForCompletion(true);
	if (!b) {
		throw new IOException("error with job!");
	}

...mapper实例将extend TableMapper...

    public static class MyMapper extends TableMapper<Text, Text> {
		 public void map(ImmutableBytesWritable row, Result value, Context context) throws InterruptedException, IOException {
			//process data for the row from the Result instance.
		}
	}

### 53.2. HBase MapReduce Read/Write Example ###
下面是个使用HBase即作为MR的数据源又作为数据接收端。这个例子只是将数据从一张表拷贝到另一张表。

    Configuration config = HBaseConfiguration.create();
		Job job = new Job(config, "ExampleReadWrite");
		job.setJarByClass(MyReadWriteJob.class);   //包含 mapper的类
		
		Scan scan = new Scan();
		scan.setCashing(500);  //Scan中默认为1，对MR job来说不太好
		scan.setCasheBlocks(false);  //对MR JOB不要设置为true
		//设置其他scan属性
		
		TableMapReduceUtil.intTableMapperJob(
				sourceTable,	//input table
				scan,			//Scan instance to control CF and attribute selection
				MyMapper.class, //mapper class
				null,			//mapper output key
				null,			//mapper output value
				job);
		TableMapReduceUtil.initTableReducerJob(
				targetTable,	//output table
				null,			//reducer class
				job);
		job.setNumReduceTasks(0);
		
		boolean b = job.waitForCompletion(true);
		if(!b){
			throw new IOException("error with job!");
		}

这里要解释下TableMapReduceUtil的工作内容，特别是对于reducer.TableOutputFormat，被用作outputFormat类，许多参数被在config上设置（如，TableOutputFormat.OUTPUT_TABLE ），同样设置reducer输出值为ImmutableBytesWritable ,reducer值为Writable.这些可以被programmer设置在job和conf上，但是TableMapReduceUtil试着使事情简单。

下面的mapper例子，将创建一个put操作，并匹配输入结果并发送它。注意这是CopyTable utility所完成的。

    public static class MyMapper extends TableMapper<ImmutableBytesWritable, Put>  {

  	public void map(ImmutableBytesWritable row, Result value, Context context) throws IOException, InterruptedException {
    // this example is just copying the data from the source table...
      	context.write(row, resultToPut(row,value));
    	}

    private static Put resultToPut(ImmutableBytesWritable key, Result result) throws IOException {
      Put put = new Put(key.get());
      for (KeyValue kv : result.raw()) {
        put.add(kv);
     	 }
      return put;
  	  }
	}
