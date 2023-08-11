# The ways of integrating Kedro with Databricks
 
In recent months we've updated Kedro documentation to illustrate [three different ways of integrating Kedro with Databricks](https://docs.kedro.org/en/stable/deployment/databricks/index.html). 

* You can choose a [workflow based on Databricks jobs](https://docs.kedro.org/en/stable/deployment/databricks/databricks_deployment_workflow.html) to deploy a project that finished development
* For faster iteration on changes, the workflow documented in ["Use a Databricks workspace to develop a Kedro project"](https://docs.kedro.org/en/stable/deployment/databricks/databricks_notebooks_development_workflow.html) is for those who prefer to develop and test their projects directly within Databricks notebooks, to avoid the overhead of setting up and syncing a local development environment with Databricks. 
* Alternatively, you can work locally in an IDE as described by the workflow documented in ["Use an IDE, dbx and Databricks Repos to develop a Kedro project"](https://docs.kedro.org/en/stable/deployment/databricks/databricks_ide_development_workflow.html). This is ideal if you’re in the early stages of learning Kedro, or your project requires constant testing and adjustments since you can use your IDE’s capabilities for faster, error-free development, while testing on Databricks. However, the experience is still not ideal: you must sync your work inside Databricks with dbx and run the pipeline inside a notebook. Debugging has a long setup for each change and there is less flexibility than inside an IDE. 

If you need a Databricks development experience that works completely inside an IDE, we recommend Databricks Connect. This setup is recommended if the data-heavy parts of your pipelines are in PySpark, to minimize performance drops due to the data traveling between remote and local environments.
 
## What is Databricks Connect?
[Databricks Connect](https://docs.databricks.com/dev-tools/databricks-connect-ref.html) is Databricks' official method of interacting with a remote Databricks instance while using a local environment.

To configure Databricks Connect for use with Kedro, follow the official setup to create a `databrickscfg` file containing your access token.

It can be installed with a `pip install databricks-connect`, and it will substitute your local SparkSession:

```
from databricks.connect import DatabricksSession
 
spark = DatabricksSession.builder.getOrCreate()
```

Spark commands are sent and executed on the cluster, and results are returned to the local environment as needed. 

In the context of Kedro, this has an amazing effect: As long as you don’t explicitly ask for the data to be collected in your local environment, operations will be executed only when saving the outputs of your node. If you use datasets saved to a Databricks path, there will be no performance hit for transferring data between environments. 

This tool was recently made available as a thin client for [Spark Connect](https://spark.apache.org/docs/latest/spark-connect-overview.html), one of the highlights of Spark 3.4, and configuration was made easier than earlier versions. If your cluster doesn’t support the current Connect, please refer to the [documentation](https://docs.databricks.com/en/dev-tools/databricks-connect-legacy.html) - Previous versions had different limitations.

 [Check this video from the Databricks team for a thorough introduction to Spark Connect](https://www.youtube.com/watch?v=p9IRFSjuLBE)

 
## How can I use a Databricks Connect workflow with Kedro?
Databricks Connect (and Spark Connect) enables us to have a completely local development flow, while all artifacts can be remote objects. Using Delta tables for all our datasets and MLflow for model objects and tracking, nothing needs to be saved locally.
Developers can take full advantage of the Databricks stack while maintaining their full IDE usage.


## How to use Databricks as your PySpark engine
[Kedro supports integration with PySpark](https://docs.kedro.org/en/stable/integrations/pyspark_integration.html) through the use of Hooks. To configure and enable your Databricks session through Spark Connect, simply set up your `SPARK_REMOTE` environment variable with your Databricks configuration. Here is an example implementation:

 
``` 
import configparser
import os
from pathlib import Path

from kedro.framework.hooks import hook_impl
from pyspark.sql import SparkSession


class SparkHooks:
    @hook_impl
    def after_context_created(self) -> None:
        """Initialises a SparkSession using the config
        from Databricks.
        """
        set_databricks_creds()
        _spark_session = SparkSession.Builder().getOrCreate()


def set_databricks_creds():
    """
    Pass databricks credentials as OS variables if using the local machine.
    If you set DATABRICKS_PROFILE env variable, it will choose the desired profile on .databrickscfg,
    otherwise it will use the DEFAULT profile in databrickscfg.
    """
    DEFAULT = os.getenv("DATABRICKS_PROFILE", "DEFAULT")
    if os.getenv("SPARK_HOME") != "/databricks/spark":
        config = configparser.ConfigParser()
        config.read(Path.home() / ".databrickscfg")

        host = (
            config[DEFAULT]["host"].split("//", 1)[1].strip()[:-1]
        )  # remove "https://" and final "/" from path
        cluster_id = config[DEFAULT]["cluster_id"]
        token = config[DEFAULT]["token"]

        os.environ[
            "SPARK_REMOTE"
        ] = f"sc://{host}:443/;token={token};x-databricks-cluster-id={cluster_id}"
``` 

This example will populate `SPARK_REMOTE` with your local `databrickscfg` file. We do't setup the remote connection if the project is being run from inside Databricks (if `SPARK_HOME` points to Databricks), so you're still able to run it in the usual [hybrid development flow](https://docs.kedro.org/en/stable/deployment/databricks/databricks_ide_development_workflow.html).

Notice that we don’t need to setup a `spark.yml` file as is common in other PySpark templates; we’re not passing any configuration, just using the cluster that is in Databricks. We also don’t need to load any extra Spark files (e.g. JARs), as we are using a thin Spark Connect client.
 
Now all your Spark calls in your pipelines will automatically use the remote cluster. There's no need to change anything in your code. However, notebooks might be part of the project. To use your remote cluster without needing to use environment variables, you can use the DatabricksSession:
```
from databricks.connect import DatabricksSession
 
spark = DatabricksSession.builder.getOrCreate()
```
This will read your configuration directly from the `databrickscfg` file, bypassing the setup needed for pure Spark Connect.

When using the remote cluster, it's preferred to avoid data transfers between the environments, with all catalog entries referencing remote locations. Using ` kedro_datasets.databricks.ManagedTableDataSet` as your dataset type in the catalog also allows you use Delta table features.


 
## Enabling MLFlow on Databricks
Using [MLflow](https://mlflow.org/) to save all your artifacts directly to Databricks leads to a powerful workflow. For this we can use [`kedro-mlflow`](https://github.com/Galileo-Galilei/kedro-mlflow). Note that `kedro-mlflow` is built on top of the mlflow library and although the databricks config cannot be found in its documentation, you can read more about it in the [documentation from mlflow directly](https://mlflow.org/docs/latest/index.html).

After doing the [basic setup of the library](https://kedro-mlflow.readthedocs.io/en/stable/source/02_installation/02_setup.html#activate-kedro-mlflow-in-your-kedro-project) in your project, you should see a `mlflow.yml` configuration file. In this file, change the following to set up your URI:

```
server:
  mlflow_tracking_uri: databricks # if null, will use mlflow.get_tracking_uri() as a default
  mlflow_registry_uri: databricks # if null, mlflow_tracking_uri will be used as mlflow default
```
 
Setup your experiment name (this should be a valid Databricks path):
```  
  experiment:
    name: /Shared/your_experiment_name
``` 
 
By default, all your parameters will be logged, and objects such as models and metrics can be saved as MLflow objects referenced in the catalog.
 
## Limitations of the workflow
The new Databricks Connect tool (built on top of Spark Connect) supports only recent versions of Spark. We recommend looking at the [detailed limitations on the official website](https://docs.databricks.com/dev-tools/databricks-connect-ref.html), and in common usage you need to be careful about the upload limit of only 128MB for dataframes. 
 
Users also need to be conscious that `.toPandas()` will move the data to your local pandas environment.  Saving results back as MLflow objects is the preferred way to avoid local objects. Examples can be seen in the [kedro-mlflow documentation](https://kedro-mlflow.readthedocs.io/en/stable/source/04_experimentation_tracking/index.html) for all types of supported objects.