# A new Kedro dataset for Spark Structured Streaming

This article guides data practitioners on how to set up a Kedro project to integrate Spark Structured Streaming and process data streams in realtime. 

It provides details on the setup of the project to use the new `SparkStreaming` Kedro dataset, example use cases, and a deep-dive on some design considerations.
It's meant for practitioners familiar with Kedro so we'll not be covering the basics of a project, but you can familiarise yourself with them in the [Kedro documentation](https://docs.kedro.org/en/stable/get_started/install.html).

This post was created by [Tingting Wan](https://github.com/tingtingQB), [Tom Kurian](https://github.com/kuriantom369), and [Haris Michailidis](https://github.com/harisqb), Data Engineers in the London office of QuantumBlack, AI by McKinsey.

## What is Kedro?

[Kedro](https://kedro.org) is a toolbox for production-ready data science. It's an open-source Python framework that enables the development of clean data science code, borrowing concepts from software engineering and applying them to machine-learning projects. A Kedro project provides scaffolding for complex data and machine-learning pipelines. It enables developers to spend less time on tedious "plumbing" and focus on solving new problems.

### What are Kedro datasets?

[Kedro datasets](https://docs.kedro.org/en/stable/data/data_catalog.html) are abstractions for reading and loading data, designed to decouple these operations from your business logic. These datasets manage reading and writing data from a variety of sources, while also ensuring consistency, tracking, and versioning. They allow users to maintain focus on core data processing, leaving data I/O tasks to Kedro.

## What is Spark Structured Streaming?
[Spark Structured Streaming](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html) is built on the Spark SQL engine. You can express your streaming computation the same way you would express a batch computation on static data, and the Spark SQL engine will run it incrementally and continuously and update the final result as streaming data continues to arrive. 

## Integrating Kedro and Spark Structured Streaming

To enable Kedro to work with Spark Structured Streaming, a team inside [QuantumBlack Labs](https://www.mckinsey.com/capabilities/quantumblack/labs) developed a new [Spark Streaming Dataset](https://docs.kedro.org/en/stable/kedro_datasets.spark.SparkStreamingDataSet.html), as the existing Kedro Spark dataset was not compatible with Spark Streaming use cases. To ensure seamless streaming, the new dataset has a checkpoint location specification to avoid data duplication in streaming use cases and it uses `.start()` at the end of the `_save` method to initiate the stream.


## Set up a project to integrate Kedro with Spark Structured streaming

The project uses a Kedro dataset to build a structured data pipeline that can read and write data streams with Spark Structured Streaming and process data streams in realtime. You need to add two separate Hooks to the Kedro project to enable it to function as a streaming application. 

Integration involves the following steps:

1. Create a Kedro project.
2. Register the necessary PySpark and streaming related Hooks.
3. Configure the custom dataset in the `catalog.yml` file, defining the streaming sources and sinks. 
4. Use Kedro’s new [dataset for Spark Structured Streaming](https://github.com/kedro-org/kedro-plugins/tree/main/kedro-datasets/kedro_datasets/spark) to store intermediate dataframes generated during the Spark streaming process.

### Create a Kedro project

Ensure you have installed a version of Kedro greater than 0.18.9 and `kedro-datasets` greater than 1.4.0.

```bash
pip install kedro==0.18 kedro-datasets~=1.4.0
```

Create a new Kedro project using the Kedro `pyspark` starter: 

```bash
kedro new --starter=pyspark`
```

### Register the necessary PySpark and streaming related Hooks

To work with multiple streaming nodes, two hooks are required. The first is for integrating Pyspark: see [Build a Kedro pipeline with PySpark](https://docs.kedro.org/en/stable/integrations/pyspark_integration.html) for details. You will also need a Hook for running a streaming query without termination unless an exception occurs.

Add the following code to `src/$your_kedro_project_name/hooks.py`:


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


## How to set up your Kedro project to read data from streaming sources

Once you have set up your project, you can use the new [Kedro Spark streaming dataset](https://docs.kedro.org/en/stable/kedro_datasets.spark.SparkStreamingDataSet.html). You need to configure the data catalog, in `conf/base/catalog.yml` as follows:

### Reading a streaming JSON file

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

## How to set up your Kedro project to write data to streaming sinks

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

In order to benefit from Spark's internal query optimisation, we recommend that any interim datasets are stored as memory datasets.

All streams start at the same time, so any nodes that have a dependency on another node that writes to a file sink (i.e. the input to that node is the output of another node) will fail on the first run. This is because there are no files in the file sink for the stream to process when it starts. 

We recommended that you either keep intermediate datasets in memory or split out the processing into two pipelines and start by triggering the first pipeline to build up some initial history.

### Feature creation

Be aware that windowing operations only allow windowing on time columns.

Watermarks must be defined for joins. Only certain types of joins are allowed, and these depend on the file types (stream-stream, stream-static) which makes joining of multiple tables a little complex at times. For further information or advice......TO DO.


## Logging

When initiated, the Kedro pipeline will download the JAR required for the Spark Kafka. After the first run, it won't download the file again but simply retrieve it from where the previously downloaded file was stored.

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

For each node, the logs for the following will be shown: Loading data, Running nodes, Saving data, Completed x out of y tasks.

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

In this article, we explained how to use a new Kedro dataset to create streaming pipelines. We created a new Kedro project using the Kedro `pyspark` starter and added two separate Hooks to the Kedro project to enable it to function as a streaming application, then configured the Kedro data catalog to use the new dataset, defining the streaming sources and sinks.

There are currently some limitations because it is not yet ready for use with a service broker, e.g. Kafka, as an additional JAR package is required.

