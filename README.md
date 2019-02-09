# recap-spark

* Cluster computing platform on top of storage layer.
* Runs in memory.
* It can  run in YARN, and access data from sources including HDFS,MapR‐FS,Base and HIVE.
  * Graph processing.
  * Sql query processing
  * Batch Processing
  * Streaming
  * Machine Learning
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
  * groupByKey  -> (k,v) ..  ->> (k,(v,v,v)) ....
  * reduceByKey -> (k,v) ..  ->> (k,v)
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
     * Reflective
     * Programtic
  * map rdd data to case class
  * call toDf()
  * call .registerTempTable("tableName") - Now SQL queries can be executed against this table.
  
  
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
  
  ```
  
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
