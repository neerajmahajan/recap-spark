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
  employeeDF.filter(employeeDF("age") > 25).show
  employeeDF.agg(expr,expr)  //Aggregate on entire dataframe without groups
  employeeDF.describe("age").show  // Compute statistics for numeric columns inluding count, mean, stddev, min, max
  ```

