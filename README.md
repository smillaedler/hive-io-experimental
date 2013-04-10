== Overview ==

Hive Input/Output Library in Java.

The Hive project is built to be used through command line HQL and as such does
not have a good interface to connect to it through code directly. This project
adds support for input/output with Hive in code.

For starters this means a Java Hadoop compatible Input/OutputFormat that can
interact with Hive tables. In the future we will add other hooks for dealing
with Hive as demand grows.

== Design ==
The input and output classes implemented here have a notion of profiles. A profile is tagged by a name and describes the input or output you wish to connect to. Profiles are serialized into the Hadoop Configuration so that they can be passed around easily. MapReduce takes classes you configure it with and instantiates them using reflection on many hosts. This sort of model makes it hard to also configure the classes. The approach we have chosen is to have the user create input / output descriptions, and set them with a profile. This writes it to the configuration which gets shipped to every worker by MapReduce. See example code mentioned below for more information.

== Input ==
The Hive IO library conforms to Hadoop's [http://hadoop.apache.org/docs/r0.23.6/api/org/apache/hadoop/mapreduce/InputFormat.html](OutputFormat) API.
The class that implements this API is [hive-io-exp-core/src/main/java/com/facebook/giraph/hive/input/HiveApiInputFormat.java](HiveApiInputFormat).
MapReduce creates the InputFormat using reflection. It will call getSplits() to generate splits on the hive tables to read. Each split is then sent to some worker, which then calls createRecordReader(split). This creates a [[hive-io-exp-core/src/main/java/com/facebook/giraph/hive/input/RecordReaderImpl.java](RecordReaderImpl) which is used to read each record.

=== Benchmark ===
Using Hive IO we were able to read from Hive at 140MB/s. For comparison, using the hive command line on the same partition reads at around 35 MB/s. This benchmark is an evolving piece of work and we still have performance tweaks to make.

== Output ==
The Hive IO library conforms to Hadoop's [http://hadoop.apache.org/docs/r0.23.6/api/org/apache/hadoop/mapreduce/OutputFormat.html](OutputFormat) API.
The class that implements this API is [hive-io-exp-core/src/main/java/com/facebook/giraph/hive/output/HiveApiOutputFormat.java](HiveApiOutputFormat).
MapReduce creates the OutputFormat using reflection. It will call checkOutputSpecs() to verify that the output can be written. Then it will call createRecordWriter() on each map task host. These record writers will get passed each record to write. Finally it will call getOutputCommitter() and finalize the output.
Because of its use of reflection, to make use of this library you will need to extend HiveApiOutputFormat (and potentially HiveApiRecordWriter).

The best way to get started with output is to have a look at the example mapreduce [hive-io-exp-cmdline/src/main/java/com/facebook/giraph/hive/cmdline](code). This is a simple example of using the library that should be used as a starting point for projects. 

== References ==
[http://giraph.apache.org/](Giraph) uses Hive IO to read and write graphs from Hive. It is another good example that can be used for starter code. It is a more complicated use case than the examples mentioned above as it converts records to graph vertex and edge objects. It has scaled to terabytes of data and is running in production at large companies (e.g. Facebook).
The [http://giraph.apache.org/](Giraph) module code is [https://github.com/apache/giraph/tree/trunk/giraph-hive/src/main/java/org/apache/giraph/hive](here).
Specifically you should look at [https://github.com/apache/giraph/blob/trunk/giraph-hive/src/main/java/org/apache/giraph/hive/input/vertex/HiveVertexInputFormat.java](vertex input), [https://github.com/apache/giraph/blob/trunk/giraph-hive/src/main/java/org/apache/giraph/hive/input/edge/HiveEdgeInputFormat.java](edge input), and [https://github.com/apache/giraph/blob/trunk/giraph-hive/src/main/java/org/apache/giraph/hive/output/HiveVertexOutputFormat.java](output).
Note especially how the vertex and edge inputs use different profiles to allow reading from multiple Hive tables.