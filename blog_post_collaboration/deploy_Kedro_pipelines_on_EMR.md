# Title TBD. Maybe "Seven steps to deploy Kedro pipelines on AWS EMR" ??

## Introduction


[Amazon EMR](https://aws.amazon.com/emr/) (previously called Amazon Elastic MapReduce) is a managed cluster platform that simplifies running big data frameworks, such as Apache Hadoop and Apache Spark, to process and analyze vast amounts of data with AWS.

> TO DO 
This section needs another sentence or two to set the scene on a typical use case of getting Kedro working with EMR.

## 1. Set up the cluster environment

One way to install Python libraries onto EMR is to package a virtual environment and deploy it to EMR. To do this, the cluster needs to have the [same Amazon Linux 2 environment as used by EMR](https://docs.aws.amazon.com/emr/latest/EMR-Serverless-UserGuide).

We used this [example Dockerfile](https://github.com/aws-samples/emr-serverless-samples/tree/main/examples/pyspark/dependencies) to package our dependencies on an Amazon Linux 2 base. Our example Dockerfile is as below: 

```text
FROM --platform=linux/amd64 amazonlinux:2 AS base 

RUN yum install -y python3 

ENV VIRTUAL_ENV=/opt/venv 
RUN python3 -m venv $VIRTUAL_ENV 
ENV PATH="$VIRTUAL_ENV/bin:$PATH" 

COPY requirements.txt /tmp/requirements.txt 

RUN python3 -m pip install --upgrade pip && \
    python3 -m pip install venv-pack==0.2.0 && \ 
    python3 -m pip install -r /tmp/requirements.txt 
    
RUN mkdir /output && venv-pack -o /output/pyspark_deps.tar.gz 

FROM scratch AS export 
COPY --from=base /output/pyspark_deps.tar.gz /
```


> Note: A `DOCKER_BUILDKIT` backend is necessary to run this Dockerfile (make sure you have it installed).

Run the Dockerfile  using the following command. This will generate a `pyspark_deps.tar.gz`  file at the  `<output-path>`  specified in the command above. 

```bash
DOCKER_BUILDKIT=1 docker build --output . <output-path> 
```

Use this command if your Dockerfile has a different name 

```bash
DOCKER_BUILDKIT=1 docker build -f Dockerfile-emr-venv --output . <output-path>
```

## 2. Set up `CONF_ROOT`

By default, Kedro looks at the root `conf` folder for all its configurations (catalog, parameters, globals, credentials, logging) to run the pipelines. However, [this can be customized](https://docs.kedro.org/en/stable/kedro_project_setup/configuration.html#configuration-root) by changing `CONF_ROOT`  in `settings.py`. 

As `conf` directory is not packaged together with `kedro package`, the `conf` directory needs to be available to Kedro separately and that location can be controlled by setting `CONF_ROOT`. 

### For Kedro versions < 0.18.5

-   Change `CONF_ROOT in settings.py` to the location where `conf` directory will be deployed. It could be anything. e.g. `./conf`  or `/mnt1/kedro/conf`. 

### For Kedro versions >= 0.18.5

-   Use the `--conf-source` CLI parameter directly with `kedro run` to specify the path directly. `CONF_ROOT` need not be changed in `settings.py`.

## 3. Package the Kedro project

Package the project using `kedro package`  command. This will create a `.whl` in the `dist`  folder. 

This will be used when doing `spark-submit` to the EMR cluster for specifying the `--py-files` to refer to the source code. 

```bash
# run at the root directory 
kedro package
```

## 4. Create `.tar` for `conf`

The `kedro package` command only packs the source code and yet the `conf/` directory is essential for running any Kedro project.

Therefore it needs to be deployed separately as a `tar.gz`  file. It is important to note that the contents inside the `conf` directory needs
to be zipped and **not** the `conf`  folder entirely. Use the following command to zip the contents inside the conf directory and generate a `conf.tar.gz` file containing all the `catalog.yml`, `parameters.yml` and others for running the Kedro pipeline. It will be used with `spark-submit` for the `--archives` option for unpacking the contents into a `conf` directory .

```bash
tar -czvf conf.tar.gz --exclude="local" conf/*
```

## 5. Create entrypoint For Spark application 

Create an `entrypoint.py` file that the Spark Application will use to start the job. 

Example below:

> Note: This file can be modified to take arguments and can be run only
using `main(sys.argv)` after removing the `params` array. 

Example:

```bash
python entrypoint.py --pipeline my_new_pipeline --params run_date:2023-02-05,runtime:cloud
```
This would mimic the exact `kedro run` behaviour.

```python
import sys 
from proj_name.__main__ import main: 

if __name__ == "__main__":
	"""
	These params could be used as *args to 
	test pipelines locally. The example below 
	will run `my_new_pipeline` using `ThreadRunner`
	applying a set of params
	params = [ 
		"--pipeline", 
		"my_new_pipeline", 
		"--runner", 
		"ThreadRunner", 
		"--params", 
		"run_date:2023-02-05,runtime:cloud", 
	] 
	main(params) 
	"""

	main(sys.argv)
```

# 6. Upload relevant files to S3

Upload the relevant files to an S3 bucket (EMR should have access to this bucket), in order to run the Spark Job. The following artifacts should be uploaded to S3:

-   .whl [file created in step #3]
-   Virtual Environment `.tar.gz` created in step 1 (e.g. `pyspark_deps.tar.gz`)
-   `tar` file for `conf` folder created in step #4 (e.g. `conf.tar.gz`)
-   `entrypoint.py` file created in step #5.

# 7. Spark submit to EMR cluster

Use the following `spark-submit` command as a `step` on EMR running in **cluster** mode. Few points to note:

-   `pyspark_deps.tar.gz`  is unpacked into a folder named `environment`
-   Environment variables are set referring to libraries unpacked in the `environment` directory above. e.g. `PYSPARK_PYTHON=environment/bin/python`
-   `conf`  directory is unpacked to a folder `conf`  specified here
    (`s3://{S3_BUCKET}/conf.tar.gz#conf)` after the # symbol. Note the following:
    -   Kedro versions < 0.18.5. The folder location/name after the `#` symbol should match with `CONF_ROOT` in `settings.py` 
    -   Kedro versions >= 0.18.5. You could follow the same approach as earlier. However, Kedro versions >= 0.18.5 provides flexibility to provide the `CONF_ROOT` through the CLI parameters using `--conf-source` instead of setting `CONF_ROOT` in settings.py. Therefore `--conf-root` configuration could be directly specified in the CLI parameters and step 2 (Setup `CONF_ROOT`) can be skipped completely.
        
```bash
spark-submit 
    --deploy-mode cluster 
    --master yarn 
    --conf spark.submit.pyFiles=s3://{S3_BUCKET}/<whl-file>.whl
    --archives=s3://{S3_BUCKET}/pyspark_deps.tar.gz#environment,s3://{S3_BUCKET}/conf.tar.gz#conf
    --conf spark.yarn.appMasterEnv.PYSPARK_PYTHON=environment/bin/python
    --conf spark.executorEnv.PYSPARK_PYTHON=environment/bin/python 
    --conf spark.yarn.appMasterEnv.<env-var-here>={ENV} 
    --conf spark.executorEnv.<env-var-here>={ENV} 
    
    s3://{S3_BUCKET}/run.py --env base --pipeline my_new_pipeline --params run_date:2023-03-07,runtime:cloud
```

## Summary

> TO DO
Need to close off this post with a summary of what the steps were and the result, and confirm how you know it's working perhaps??

Also add a note on who to contact for more help (probably get on the Slack organisation)
