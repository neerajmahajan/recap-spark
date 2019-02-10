# recap-spark

* Cluster computing platform on top of storage layer.
* Runs in memory.
* It can  run in YARN, and access data from sources including HDFS,MapR‐FS,Base and HIVE.
  * Graph processing.
  * Sql query processing
  * Batch Processing
  * Streaming
  * Machine Learning
  
###### Spark vs MapReduce
* Spark
    * Spark tries to keep everything in memory.
    * Chanining multiple jobs is faster.
    * A combination of any number of map and reduce operations.
* MapReduce
    * Read and write from/to disk after every job.
    * One Map and Reducer per job.
    * Jobs are all bacth

###### Architecture

* Driver Machine -> Driver Program -> creates SparkContext + how & where to access cluster.
 * SC connects to cluster manager. CM allocates resources across application.
 * Spark acquires executors in the worker node.
 * Jar / Python files passed to SC is sent to the executors.
 * SC send the tasks for the executor to run. ***task is like operations on RDD*
 * worker nodes can access  data  storage sources to ingest and output data as needed.
* Worker Nodes.... -> Executor -> Tasks

###### SPARK UI
* http://localhost:4040/jobs/

###### Spark Core
* Spark core is the computational engine responsible for
  * Task Scheduling
  * Memory Management
  * Fault recovery
  * Interacting with storage systems.
  * Also contain API that is used to create RDD and manioulate them.
  
###### Spark Data Sources
* Any storage source supported by Hadoop.
* MapR-FS
* Amazon S3
* Local file system
* HDFS
* Hive
* HBase
* JDBS databases

###### Data Formats
* Hadoop supported formats
* text file **WholeDirectory**
* JSON
* CVS
* Parquet File
* SequenceFiles
* Avro

###### RDD Operations
* Transformations -> Return RDD -> Lazily evaluated
  * map
  * filter
  * flatMap
  * groupByKey  -> (k,v) ..  ->> (k,(v,v,v)) .... **CAREFULL: The single key-value pair cannot spans across multiple worker nodes and cause lot of unnecessary transfer of data ove the network**
  * reduceByKey -> (k,v) ..  ->> (k,v) **Will not shuffle data like groupByKey** **IMP The reduced value should be of same type of input value**
  * countByKey
  * lookup(key)
  * sortByKey(ascending=false)
    * .top(3)(Ordering.by(_._2)) // another way of sorting
  * distinct
* Action  -> Return value to driver program
  * count
  * reduce
  * collect
  * take
  * first
  * takeOrdered

###### Reading data
* sc.textFile ->list of lines
* sc.wholeTextFile -> list of files
* sc.sequenceFile[K,V]

###### RDD Caching
* We do caching of RDD when multiple actions are required on the same transformed RDD.
* When there is branching in lineage
* when cached and not enough memory then it goes to file.
* rdd.cache() === rdd.persist(MEMORY_ONLY) --- cache is a transformation

###### Data Frames RDD with names columns
* Constructed from
  * structure data format
  * hive
  * external database
  * RDD
* STEPS
  * create SQLContext
  * import sqlContext.implicits._
  * Define schema using case class
  * read RDD
     * Reflective --- mapping rdd to DF using case classes. can't create RDD using case class with more than 22 fields
     * Programtic --- Use to construct DataFrames when column & thier types not known until runtime.
  * map rdd data to case class
  * call toDf()
  * call .registerTempTable("tableName") - Now SQL queries can be executed against this table.
  
  * Using Reflection
  ```
  val sqlContext = new org.apache.spark.sql.SQLContext(sc)
  import sqlContext.implicits._
  case class Employee(firstName:String, lastName:String, age:Int)
  val empRDD = sc.textFile("employee.csv").map(_.split(","))
  val employees = empRDD.map(e => Employee(e(0),e(1),e(2).toInt))
  val employeeDF = employees.toDF()
  employeeDF.registerTempTable("employeeDF")
  employeeDF.printSchema() // print column names and thier type
  employeeDF.columns  // return array of column names
  employeeDF.count
  employeeDF.show
  employeeDF.collect // return Array[row], where each row contains values derived from Employee case class
  employeeDF.first   // first row in the dataframe
  employeeDF.head    // first row in the dataframe
  employeeDF.take(10) // first 10 rows of the data frame
  employeeDF.explain // print physical plan
  employeeDF.toDF    // returns new data frame
  employeeDF.cache   // Caches data frame in memory
  employeeDF.distinct // returns a new DF with unique rows
  employeeDF.select("firstName","lastName").show // select specific columns from data frame
  employeeDF.groupBy(col1,col2)  // group rows based on the specified column/s values
  employeeDF.groupBy("age").avg().show
  
  employeeDF.filter(employeeDF("age") > 25).show
  OR
  employeeDF.filter("age > 25").show
  
  employeeDF.agg(expr,expr)  //Aggregate on entire dataframe without groups
  employeeDF.describe("age").show  // Compute statistics for numeric columns inluding count, mean, stddev, min, max

  DataFrame department = sqlContext.read().parquet("...");
  employeeDF.join(department,employeeDF.col("dept_id").equalTo(department("id"))    // Joining DataFrames
  
  employeeDF.agg(avg("age") ,max("age"), mean("age"))   // Aggregates on the entire data frame without groups
  employeeDF.except(otherEmployeeDF)  // employeeDF - otherEmployeeDF
  employeeDF.sort($"count".desc).show(5)
  
  ```
  * Programmatically
  ```
  import sqlContext.implicits._
  import org.apache.spark.sql._
  import org.apache.spark.sql.types._
  val rowRDD = sc.textFile("/user/user01/data/test.txt”).map(x=>x.split(" ")).map(p=>Row(p(0),p(2),p(4)))
  val testsch = StructType(Array(StructField("IncNum",StringType,true),  StructField("Date",StringType,true),StructField("District",StringType,true)))
  val testDF = sqlContext.createDataFrame(rowRDD,schema)
  testDF.registerTempTable("test")
  val incs = sql(“SELECT * FROM test”)
  ```
  
  ###### Creating data frames from other data sources
  
  * sqlContext.load("xyz.parquet"); // default format is parquet // can be overridden by changing property spark.sql.sources.default
  * sqlContext.load("/path/to/json/data","json")
    * Depriciated methods
    * sqlContext.jdbc  
    * sqlContext.jsonFile
    * sqlContext.jsonRDD // RDD containing JSON data
    * sqlContext.parquetFile
    
  ###### Saving data frame
  * save
  ```
  val usersDF = spark.read.load("examples/src/main/resources/users.parquet")
  usersDF.select("name", "favorite_color").write.save("namesAndFavColors.parquet")
  ```
  ```
  val peopleDF = spark.read.format("json").load("examples/src/main/resources/people.json")
  peopleDF.select("name", "age").write.format("parquet").save("namesAndAges.parquet")
  ```
  ```
  val peopleDFCsv = spark.read.format("csv")
  .option("sep", ";")
  .option("inferSchema", "true")
  .option("header", "true")
  .load("examples/src/main/resources/people.csv")
  ```
  * Running SQL on files directly
  ```
  val sqlDF = spark.sql("SELECT * FROM parquet.`examples/src/main/resources/users.parquet`")
  ```
  * Bucketing data while storing
  ```
  peopleDF.write.bucketBy(42, "name").sortBy("age").saveAsTable("people_bucketed")
  ```
  * PArtioning while saving
  ```
  usersDF.write.partitionBy("favorite_color").format("parquet").save("namesPartByColor.parquet")
  ```
  * insertIntoJDBC
  
  ##### Spark Program
  * Create SparkContext -- This tell spark **how and where** to access cluster
    * SparkContext connects to cluster manager.
    * Cluster Manager allocate resources across applications.
    * Once connected, Spark acquires executors in worker nodes.
    * Jar or python files passed to SC are sent to the executors.
    * SC will send the tasks for the executors to run.
    * Executors runs computations and store data for application.
  * When the program is submitted through spark-submit, driver runs in its own process and each executor in its own
  * Thr driver together with its executors are referred to as a Spark Application. 
    
  ```
  import org.apache.spark.SparkContext
  import org.apache.spark.SparkContext._
  import org.apache.spark.SparkConf
  
  object EmployeeApp{      // This file name should be EmployeeApp.scala
    def main(args:Array[String]){
        val conf = new SparkConf().setAppName("ProductDataMergerApp")
        val sc = new SparkContext(conf)
        val empRDD = sc.textFile("employee.csv").map(_.split(","))
    }
  }
  ```
###### Running Spark application
* First, user submit an application using spark-submit.
* spark-submit launches the Driver program, which invokes the main method. The main method creates SparkContext which tells the driver the location of the cluster manager.
* The driver contacts the cluster manager for resources, and to launch executors.
* Cluster manager will launch executors for the driver program.
* Driver runs through the program instructions (RDD, transformation and Actions) sending work to executors in the form of tasks.

###### Spark running mode
* Local mode
* Standalone deploy mode
   * Place compiled version of Spark on each cluster node.
   * Start master and workers by hand or use launch scripts provided by Apache Spark.
   * To run on spark cluster pass **spark://IP:PORT** url of the master to SparkContext constructor.
   * Running modes
      * Client - 1) Driver launches in the client process that submitted the job. 2) Need to wait for result when job finishes(sync)
      * Cluster -1) Driver program gets launched on one of the cluster node. 2) Can quit without waiting for job results (async)
* Hadoop Yarn
   * It is advantageous to run Spark on YARN if there is an existing Haddop cluster.
   * Don't need to maintain separate cluster.
   * We can take advantage of YARN scheduler for categorizing, isolating and prioritzing workloads.
   * Cluster Mode
      * Driver program is launched in Application Master
      * Can quit without waiting for job results.
      * Suitable for productions deployments.
   * Client Mode
      * Driver launched in the client process that submitted the job.
      * Need to wait for results until job finishes.
      * Useful for spark Interactive shell or debugging.
* Mesos
* spark-submit syntax
```
spark-submit \
--class       <main-class>  \
--master      <master-url>  \ 
--deploy-mode <deploy-mode> \
--conf        <key>=<value> \
...   #other options
<application-jar>  <path to bundled jar including all dependencies>
[application-arguments]  <arguments passed to the main method>
```
###### Examples
* To run in local mode
```
spark-submit --class <fully-qualified-path> \
--master local[n] \      //n is the number of cores/executors
/path/to/application-jar
```
* To run standalone **client** mode
```
spark-submit --class <fully-qualified-path> \
--master spark:<master url> \      
/path/to/application-jar
```
* To run on yarn **cluster**
* In YARN mode the ResourceManager’s address is picked up from the Hadoop configuration and not given with --master

```
spark-submit --class <fully-qualified-path> \
--master yarn-cluster \      
/path/to/application-jar

```

* To run on yarn **client**
* In YARN mode the ResourceManager’s address is picked up from the Hadoop configuration and not given with --master
```
spark-submit --class <fully-qualified-path> \
--master yarn-client \      
/path/to/application-jar

```
* Package you application in a jar file using maven or sbt and use it in spark submit

```spark-submit --class com.xyz.ProductDataApp --master local target/product-single-view.jar```
* For Python application pass .py

###### PairRDD - 2 field tuple
* When working with distributed data, it is useful to oraganise data into key-value pairs as it allows us to **aggreagte** data or **regroup** data across the network.

```
val textFile = spark.textFile("hdfs://....")
val wordCount =  textFile.flatMap(line => line.split(","))
                         .map(word => (word,1))
                         .reduceByKey(_+_)
wordCount.saveAsTextFile("hdfs://....")
```

###### Partioninig

* Range Partinioning  eg new RangePartitioner(5,pariRDD)
* Hash Partinoning eg new HashPartitioner(100)
* Customizing Partitioning is only avaialbel on Pair RDD.
* Partitioning will send data of same keys to same worker.
* **Always do persist or cache after partioninig if you are going to reuse it**
* Some operation automatically  result in RDD with known partioner
  * sortByKey - RangePartioner
  * groupByKey - HashPartioner
* We can also specify partitionType while doing transformation.
* .partitioner tells the current Partitioner.
* rdd.partition.size gives current number of partitioner.
* repartition - Shuffle data across the network to create new set of partitions.
* coalesce    - Decreases the number of partition
* ***When we use multiple RDDs in our application and for operations that act on two pair RDDs, when we pre-partition there will be no shuffling across the network if both RDD have the same partitioner*

###### UDF

```
val func1 = udf((arguments) => {function-definition})
```

##### Performance Tuning
* Optimizing partioning improves performance. by defeault any shuffle operation creates 200 partitions, which can be repartition.
* reduceByKey instead of groupBy
```
If we have a large cluster with 100 nodes and 10 slots in each Executor, then we want the
DataFrame to have 1,000 partitions to use all of the Executor slots to process it
simultaneously. In this case, it's also fine to have a DataFrame with thousands of partitions,
and a 1,000 par66ons will be processed at a time.```
```
###### Showing lineage for an RDD
* employeeRdd.toDebugString
* spark initially create RDD lineage(**logical plan**) for a set of tranformations/action required to do a job. It captures it into a DAG(Direct Acyclic graph). RDD uses pointers to trace it ancestors.
* When spark see's an action, the **spark scheduler** creates a **physical plan** to compute the RDD needed for the computation.
* When an RDD can be computed from it's parent without movement of data, multiple RDD are collapsed into a single stage. The collapsing of RDD into one stage is called **pipelining**.
* Physical plan is composed of stages and when a shuffle happens then a new stage is created.
###### Scenarios when scheduler truncate the lineage of RDD graph
* **Pipelining** : when there is no movement of data from the parent RDD, the scheduler will pipeline the RDD graph collpsing multiple RDD into single stage.
* **RDD Persistence** : when a RDD is persisted to cluster memeory or disk, the spark scheduler will truncate the lineage of RDD graph. it will begin computation based on persisted RDD.
* If the RDD is already materialize due to an earlier shuffle. This optimization is built into spark.

###### DAG to physical plan
* DAG is the logical graph of RDD operations based on user code.
* When an action is encountered, the DAG is translated into a physical plan to compute the RDD needed for performing the action.
* The spark scheduler submits a job to compute all necessary RDDs.
* Each job is made of one or more stages.
* Each stage is composed of tasks.
* Stages are processed in order and individual tasks are scheduled and executed on the cluster.

###### Debugging
* Sometime some tasks on executors are taking long than same tasks executing on other executors. This problem is called skew. It may be dues to uneven data into partition. so consider appply range/hash partitoning or repartion.
* **Common issues leading to slow performance**
    * The level of parallelism.
    * The serialization format used during shuffle operations. Java built in serializer is default, use **Kyro** serialization--oftem more efficeint.
    * Managing memeory to optimize your application.
       * By default Spark use
        * 60% for RDD storage
        * 20% for shuffle
        * 20% for user programs
        * when we cache/ by default persist(MEMORY_ONLY). if there is not enough space to cache new RDD partitions then old ones are deleted and recomputed wheen needed. It is better to use persist(MEMORY_AND_DISK) 
* **LOG FILES**
  * the location of log files depend on the deployment mode
     * In standalone : work/ directory of the Spark distribution on each worker
     * In Mesos work/ directory of slave, but is accessible  from Mesos master.
     * To access the logs in YARN, us the YARN log collection tool.
###### Best Practices
* Avoid shuffling large amounts of data.
* Do not copy all elements of RDD to driver.
* Filter sooner than later.
* If you have many idle tasks, coalesce()
* If not using all slots in cluster repartition

#### SPARK DATA PIPELINES
* using multiple components of spark in pipeline - streaming -> sql processing -> graph processing
### Spark Streaming
* Streaming data is continuos, but to process the data stream, it needs to be batched.
* Spark streaming divides the data stream into batches of X milliseconds called discretized streams or DStreams.
* A DSTREAM is a sequence of mini batches, where each mini-batch is represented as Spark RDD.
* Each RDD in the stream will contain the records that are recieved by Spark during the batch interval.
* Transformations are applied
* Processed data is output in bacthes..

```
val ssc = new StreamingContext(sc,Seconds(5))
or val ssc = new StreamingContext(sparkconf,Seconds(5))  // 5 is the batch interval
val linesDstream = ssc.textFileStream("/mapr/stream")
val sensorDstream = lineStream.map(parseIntoSensorObject)
sensorDstream.forEachRDD { rdd => 

}

ssc.start()
ssc.awaitTermination()

```
* Window operation on streaming
  * batch interval
  * window length // should be multiple of bacth interval
  * sliding interval // should be multiple of bacth interval
