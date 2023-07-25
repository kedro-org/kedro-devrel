The ways of integrating Kedro with Databricks
 
The official documentation shows 3 different ways of integrating with Databricks (link: https://docs.kedro.org/en/stable/deployment/databricks/index.html). You can choose a workflow based on Databricks jobs to deploy a project that finished development, but for fast iteration on changes it’s preferrable to use either notebooks or an IDE. 
More experienced developers will prefer an IDE for the extra features they provide, but the experience is not ideal – you must sync your work inside Databricks with dbx and run the pipeline inside a notebook. Debugging has a long setup for each change and less flexibility than inside an IDE. 
If the user wants an experience completely inside an IDE, we can use Databricks Connect. 
 
0.	What is Databricks Connect?
Databricks Connect is the official method of interacting with a remote Databricks instance while using a local environment (link: https://docs.databricks.com/dev-tools/databricks-connect-ref.html). To configure your Connect for later usage with Kedro, follow the official setup to create a databrickscfg file containing your access token.
It can be installed with a `pip install databricks-connect`, and it will substitute your local SparkSession:
```
from databricks.connect import DatabricksSession
 
spark = DatabricksSession.builder.getOrCreate()
```
Spark commands are sent and executed on the cluster, and results are returned to the local environment as needed. 
In the context of Kedro, this has an amazing effect: As long as you don’t explicitly ask for the data to be collected in your local env, operations will be executed only when saving the outputs of your node. If you use datasets saved to a Databricks path, there will be no performance hit for transferring data between environments. 
This tool was recently made available as a thin client for Spark Connect, one of the highlights of Spark 3.4, and configuration was made easier than earlier versions. If your cluster doesn’t support the current Connect, please refer to the documentation - Previous versions had different limitations.
 
 
 
1.	What is possible with this workflow?
DB Connect enables us to have a completely local development flow, while all artifacts can be remote objects. Using Delta tables for all our datasets and MLFlow for model objects and tracking, nothing needs to be saved locally.
This allows developers to take full advantage of the Databricks stack while maintaining their full IDE usage.
 
2.	How to use Databricks as your PySpark engine
Kedro supports PySpark through the use of hooks (link: https://docs.kedro.org/en/stable/integrations/pyspark_integration.html).
To configure and enable your Databricks Session, simply setup your SPARK_REMOTE environment variable and substitute the common Spark hook for the DatabricksSession. Here is an example implementation:
 
import os
from pathlib import Path
 
from databricks.connect import DatabricksSession
from kedro.framework.hooks import hook_impl
 
class SparkHooks:
    @hook_impl
    def after_context_created(self) -> None:
        """Initialises a SparkSession using the config
        from Databricks.
        """
        set_databricks_creds()
        _spark_session = DatabricksSession.builder.getOrCreate()
 
def set_databricks_creds():
    """
    Pass databricks credentials as OS variables. Reading the DEFAULT in databrickscfg.
    """
    with open(Path.home() / ".databrickscfg") as f:
        lines = f.readlines()
 
    idx = 0
    default_idx = 0
    default_found = False
    for line in lines:
        idx = idx + 1
        if "[DEFAULT]" in line:
            default_idx = idx
            default_found = True
        elif "[" in line:
            if default_found == False:
                continue
            break
 
    out = "".join(lines[default_idx:idx]).split("\n")
    for line in out:
        if "host" in line:
            host = line.split("=", 1)[1].split("//", 1)[1].strip()[:-1]
        if "cluster_id" in line:
            cluster_id = line.split("=", 1)[1].strip()
        if "token" in line:
            token = line.split("=", 1)[1].strip()
 
    os.environ[
        "SPARK_REMOTE"
    ] = f"sc://{host}:443/;token={token};x-databricks-cluster-id={cluster_id}"
 
 
 
Notice that DB Connect (and Spark Connect) doesn’t support SparkContext, so we don’t set it up. We also don’t need to setup a spark.yml file as is common in other pyspark templates – we’re not passing any configuration, just using the cluster that is in Databricks. We also don’t need to load any extra Spark files (e.g. jars), as the tool is a thin Spark Connect client.
 
For best usage of Databricks, use ` kedro_datasets.databricks.ManagedTableDataSet` as your dataset type in the catalog. 
 
 
3.	Enabling MLFlow on Databricks
Using MLFlow to save all your artifacts directly to Databricks leads to a powerful workflow. For this we can use kedro-mlflow (link: https://github.com/Galileo-Galilei/kedro-mlflow). 
After doing the basic setup of the library in your project, you should see a mlflow.yml configuration file. In this file, change the following:
 
Setup your URI:
server:
  mlflow_tracking_uri: databricks # if null, will use mlflow.get_tracking_uri() as a default
  mlflow_registry_uri: databricks # if null, mlflow_tracking_uri will be used as mlflow default
 
Setup your experiment name (this should be a valid Databricks path):
  experiment:
    name: /Shared/your_experiment_name
 
kedro-flow is built on top of the mlflow library – although the databricks config cannot be found in its documentation, you can read more about it on the documentation from mlflow directly.
 
By default, all your parameters will be logged, and objects such as models and metrics can be saved as MLflow objects referenced in the catalog.
 
4. Limitations
The new Databricks Connect tool (built on top of Spark Connect) supports only recent versions of Spark. We recommend looking at the detailed limitations on the official website (link: https://docs.databricks.com/dev-tools/databricks-connect-ref.html), but in common usage users need to be careful about the upload limit of only 128MB for dataframes. 
 
Users also need to be conscious that .toPandas() will move the data to your local pandas environment.  Saving results back as MLflow objects is the preferred way to avoid local objects.
