# A new Kedro dataset for Spark Structured Streaming

This article describes a new [Kedro dataset for Spark Structured Streaming](https://github.com/kedro-org/kedro-plugins/tree/main/kedro-datasets/kedro_datasets/spark). [Spark Structured Streaming](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html) is built on the Spark SQL engine. You can express your streaming computation the same way you would express a batch computation on static data, and the Spark SQL engine will run it incrementally and continuously and update the final result as streaming data continues to arrive. 

## Introduction

This needs to be written:
* What this post solves for the reader, and who would be interested in learning about this.
* Limitations of the dataset (if any). Proof of concept only??
* Link to source code and further information about Kedro datasets.
* What the following sections show: set up a project, configure the data catalog

## Set up a project to use the `SparkStreaming` Kedro dataset

These are the steps to follow in order to use the dataset:

The methodology for integrating Kedro with Spark streaming for real-time processing of data streams involves the following steps:

1. Create a Kedro project using the `pyspark` starter
2. Extend the template project
3. Register the necessary PySpark and streaming related Hooks
4. Configure the custom dataset in the catalog.yml file, defining the streaming sources and sinks.
5. Use Kedro’s MemoryDataSet to store intermediate dataframes generated during the Spark streaming process.

By defining the sources and sinks in the catalog.yml file and using the MemoryDataSet, Kedro creates a structured data pipeline that can read and write data streams with Spark streaming. This allows for efficient processing of data streams in real-time.


### Create a Kedro project

Create a Kedro project using the Kedro `pyspark` starter: 

```bash
kedro new --starter=pyspark`
```

WHAT VERSION OF KEDRO AND KEDRO-DATASETS??

### Extend the project

1. Create a new folder called `extras` in `src/$your_kedro_project_name/extras`. Within this folder, create a subfolder called `datasets`.
2. Add an empty file called `init.py` to both of these new folders.


### Add Hooks code
Add the following code to `src/$your_kedro_project_name/hooks.py`:


To work with multiple streaming nodes, two hooks are required for:

* Integrating Pyspark. See [Build a Kedro pipeline with PySpark](https://docs.kedro.org/en/stable/integrations/pyspark_integration.html) for details.
* Running a streaming query without termination unless an exception occurs.

```python
src/$your_kedro_project_name/hooks.py



    from kedro.framework.hooks import hook_impl
    from pyspark import SparkConf
    from pyspark.sql import SparkSession
    
    
    class SparkHooks:
        @hook_impl
        def after_context_created(self, context) -> None:
            """Initialises a SparkSession using the config
            defined in project's conf folder.
            """
    
            # Load the spark configuration in spark.yaml using the config loader
            parameters = context.config_loader.get("spark*", "spark*/**")
            spark_conf = SparkConf().setAll(parameters.items())
    
            # Initialise the spark session
            spark_session_conf = (
                SparkSession.builder.appName(context._package_name)
                .enableHiveSupport()
                .config(conf=spark_conf)
            )
            _spark_session = spark_session_conf.getOrCreate()
            _spark_session.sparkContext.setLogLevel("WARN")
    
    
    class SparkStreamsHook:
        @hook_impl
        def after_pipeline_run(self) -> None:
            """Starts a spark streaming await session
            once the pipeline reaches the last node
            """
    
            spark = SparkSession.builder.getOrCreate()
            spark.streams.awaitAnyTermination()
    


```

Register the Hooks in `src/$your_kedro_project_name/settings.py`:

```python

   """Project settings. There is no need to edit this file unless you want to change values
    from the Kedro defaults. For further information, including these default values, see
    https://kedro.readthedocs.io/en/stable/kedro_project_setup/settings.html."""
    
    from streaming.hooks import SparkHooks, SparkStreamsHook
    
    HOOKS = (SparkHooks(), SparkStreamsHook())
    
    # Instantiated project hooks.
    # from streaming.hooks import ProjectHooks
    # HOOKS = (ProjectHooks(),)
    
    # Installed plugins for which to disable hook auto-registration.
    # DISABLE_HOOKS_FOR_PLUGINS = ("kedro-viz",)
    
    # Class that manages storing KedroSession data.
    # from kedro.framework.session.shelvestore import ShelveStore
    # SESSION_STORE_CLASS = ShelveStore
    # Keyword arguments to pass to the `SESSION_STORE_CLASS` constructor.
    # SESSION_STORE_ARGS = {
    #     "path": "./sessions"
    # }
    
    # Class that manages Kedro's library components.
    # from kedro.framework.context import KedroContext
    # CONTEXT_CLASS = KedroContext
    
    # Directory that holds configuration.
    # CONF_SOURCE = "conf"
    
    # Class that manages how configuration is loaded.
    # CONFIG_LOADER_CLASS = ConfigLoader
    # Keyword arguments to pass to the `CONFIG_LOADER_CLASS` constructor.
    # CONFIG_LOADER_ARGS = {
    #       "config_patterns": {
    #           "spark" : ["spark*/"],
    #           "parameters": ["parameters*", "parameters*/**", "**/parameters*"],
    #       }
    # }
    
    # Class that manages the Data Catalog.
    # from kedro.io import DataCatalog
    # DATA_CATALOG_CLASS = DataCatalog
```


## How to read data from streaming sources

Once you have set up your project, you can use the new [Kedro Spark streaming dataset](https://github.com/kedro-org/kedro-plugins/tree/main/kedro-datasets/kedro_datasets/spark). You need to configure them in the data catalog, as the following examples illustrate:

### Example 1: Reading streaming JSON file

```yaml
raw_json:
  type: spark.SparkStreamingDataSet
  filepath: data/01_raw/stream/inventory/
  file_format: json
```

Additional options can be configured via the `load_args` key.

```yaml
int.new_inventory:
   type: spark.SparkStreamingDataSet
   filepath: data/02_intermediate/inventory/
   file_format: csv
   load_args:
     header: True
```

## How to write data to streaming sinks

### Example 1: Writing into a csv sink

All the additional arguments can be kept under the `save_args` key:

```yaml
processed.sensor:
   type: spark.SparkStreamingDataSet
   file_format: csv
   filepath: data/03_primary/processed_sensor/
   save_args:
     output_mode: append
     checkpoint: data/04_checkpoint/processed_sensor
     header: True
```

Note that when you use the Kafka format, the respective packages should be added to the `spark.yml` configuration as follows:

```yaml
spark.jars.packages: org.apache.spark:spark-sql-kafka-0-10_2.12:3.3.1 
```


    
    
## Design considerations
### Pipeline design

- In order to benefit from Spark's internal query optimisation, we recommend that any interim datasets are stored as memory datasets

- As all streams start at the same time, nodes that have a dependency on another node that writes to a file sink - i.e. the output of one node is the input to another node - will fail on the first run as there isn't any files in the file sink for the stream to process when it is started. It is recommended to either:
    - keep intermediate datasets in memory
    - split out the processing into two pipelines and start by triggering the first pipeline to build up some initial history


### Feature creation

- Windowing operations only allow windowing on time columns
- Watermarks must be defined for joins
- Only certain types of joins are allowed depending on the file types (stream-stream, stream-static) which makes joining of multiple tables a little complex at times


## Logging


When initiated the Kedro pipeline will download the JAR required for the Spark Kafka.
After the first run, it won't download the file again but simply retrieve it from where the previously downloaded file was stored.

```
    :: loading settings :: url = jar:file:/usr/local/Cellar/apache-spark/3.3.2/libexec/jars/ivy-2.5.1.jar!/org/apache/ivy/core/settings/ivysettings.xml
    Ivy Default Cache set to: /Users/.ivy2/cache
    The jars for the packages stored in: /Users/.ivy2/jars
    org.apache.spark#spark-sql-kafka-0-10_2.12 added as a dependency
    :: resolving dependencies :: org.apache.spark#spark-submit-parent-dc28fbcb-cb3b-4de5-89ef-8fb2d618237d;1.0
    confs: [default]
    found org.apache.spark#spark-sql-kafka-0-10_2.12;3.3.1 in central
    found org.apache.spark#spark-token-provider-kafka-0-10_2.12;3.3.1 in central
    found org.apache.kafka#kafka-clients;2.8.1 in central
    found org.lz4#lz4-java;1.8.0 in central
    found org.xerial.snappy#snappy-java;1.1.8.4 in central
    found org.slf4j#slf4j-api;1.7.32 in central
    found org.apache.hadoop#hadoop-client-runtime;3.3.2 in central
    found org.spark-project.spark#unused;1.0.0 in central
    found org.apache.hadoop#hadoop-client-api;3.3.2 in central
    found commons-logging#commons-logging;1.1.3 in central
    found com.google.code.findbugs#jsr305;3.0.0 in central
    found org.apache.commons#commons-pool2;2.11.1 in central
    :: resolution report :: resolve 355ms :: artifacts dl 49ms
    :: modules in use:
    com.google.code.findbugs#jsr305;3.0.0 from central in [default]
    commons-logging#commons-logging;1.1.3 from central in [default]
    org.apache.commons#commons-pool2;2.11.1 from central in [default]
    org.apache.hadoop#hadoop-client-api;3.3.2 from central in [default]
    org.apache.hadoop#hadoop-client-runtime;3.3.2 from central in [default]
    org.apache.kafka#kafka-clients;2.8.1 from central in [default]
    org.apache.spark#spark-sql-kafka-0-10_2.12;3.3.1 from central in [default]
    org.apache.spark#spark-token-provider-kafka-0-10_2.12;3.3.1 from central in [default]
    org.lz4#lz4-java;1.8.0 from central in [default]
    org.slf4j#slf4j-api;1.7.32 from central in [default]
    org.spark-project.spark#unused;1.0.0 from central in [default]
    org.xerial.snappy#snappy-java;1.1.8.4 from central in [default]
    ---------------------------------------------------------------------
    | | modules || artifacts |
    | conf | number| search|dwnlded|evicted|| number|dwnlded|
    ---------------------------------------------------------------------
    | default | 12 | 0 | 0 | 0 || 12 | 0 |
    ---------------------------------------------------------------------
    :: retrieving :: org.apache.spark#spark-submit-parent-dc28fbcb-cb3b-4de5-89ef-8fb2d618237d
    confs: [default]
    0 artifacts copied, 12 already retrieved (0kB/11ms)
```

For each node, the following logs will be shown:
- Loading data
- Running node
- Saving data
- Completed x out of y tasks

The completed log doesn't mean that the stream processing in that node has stopped. It means that the Spark plan has been created, and if the output dataset is being saved to a sink, the stream has started.

```
[03/31/23 11:23:46] INFO Loading data from 'raw.new_inventory' data_catalog.py:343
 (SparkStreamingDataSet)... 

[Stage 0:> (0 + 6) / 6]
[03/31/23 11:23:49] INFO Running node: add_processing_time([raw.new_inventory]) node.py:329
 -> [int.new_inventory] 
 INFO Saving data to 'int.new_inventory' data_catalog.py:382
 (SparkStreamingDataSet)... 
23/03/31 11:23:49 WARN ResolveWriteToStream: spark.sql.adaptive.enabled is not supported in streaming DataFrames/Datasets and will be disabled.
 INFO Completed 1 out of 5 tasks sequential_runner.py:85
```

The following log will be shown once Kedro has run through all the nodes and the full Spark execution plan has been created. This doesn't mean the stream processing has stopped as the post run hook keeps the Spark Session alive. 

```
INFO Pipeline execution completed successfully. runner.py:93
```

As new data comes in, new Spark logs will be shown, even after the "Pipeline execution completed" log.

```
[Stage 22:> (0 + 1) / 1]
[Stage 22:> (0 + 1) / 1][Stage 27:> (0 + 7) / 200]
[Stage 22:> (0 + 1) / 1][Stage 27:> (1 + 6) / 200]
[Stage 22:> (0 + 1) / 1][Stage 27:> (7 + 7) / 200]
[Stage 22:> (0 + 1) / 1][Stage 27:> (11 + 7) / 200]
[Stage 22:> (0 + 1) / 1][Stage 27:=> (14 + 7) / 200]
```

If there is an error in the input data, the Spark error logs will come through and Kedro will shut down the SparkContext and all the streams within it.

```
Caused by: java.lang.IllegalStateException: Cannot call methods on a stopped SparkContext.
This stopped SparkContext was created at:
org.apache.spark.api.java.JavaSparkContext.<init>(JavaSparkContext.scala:58)
sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
java.lang.reflect.Constructor.newInstance(Constructor.java:423)
py4j.reflection.MethodInvoker.invoke(MethodInvoker.java:247)
py4j.reflection.ReflectionEngine.invoke(ReflectionEngine.java:357)
py4j.Gateway.invoke(Gateway.java:238)
py4j.commands.ConstructorCommand.invokeConstructor(ConstructorCommand.java:80)
py4j.commands.ConstructorCommand.execute(ConstructorCommand.java:69)
py4j.ClientServerConnection.waitForCommands(ClientServerConnection.java:182)
py4j.ClientServerConnection.run(ClientServerConnection.java:106)
java.lang.Thread.run(Thread.java:750)

```




## In summary

To enable Kedro to work with Spark Structured Streaming, we developed a new Spark Streaming Dataset, as the existing Kedro Spark dataset was not compatible with Spark Streaming use cases. The new dataset was equipped with the following features to ensure seamless streaming:

* A checkpoint location specification to avoid data duplication in streaming use cases.
* The use of `.start()` at the end of the `_save` method to initiate the stream.

Separate Hooks were added to the Kedro project to enable it to function as a streaming application. 

Takeaways for the reader:

* Kedro is easily extensible and you can do cool stuff as this team demonstrates with Spark Streaming integration

However, you cannot at present viably use Kedro for your streaming workflows. Specifically for Spark, being able to deploy a streaming application is predicated on being able to deploy a job to a cluster, which is something that has been requested for a while but isn't support AFAIK. (The alternative may be to support cluster deployment.)