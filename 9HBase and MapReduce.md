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

