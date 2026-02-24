- title: From prototype to production: building data pipelines with Kedro

# Abstract

Everybody loves Jupyter notebooks… until they don’t. At some point, we all forget which cell to run first to make sure our local machine doesn’t crash, or whether Untitled1.ipynb or Copy of Untitled1.ipynb has the code for the right experiment. As we realize that we forgot to restart the kernel before running all (yet again), we recall hearing something about software engineering best practices and decide it’s finally time to figure out what that’s all about.

There’s a lot to learn, from configuration management and data versioning to pipeline development and deployment. Thankfully, you don’t need to reinvent the wheel. Kedro provides an open-source Python framework for creating reproducible, maintainable, and modular data engineering and data science code.

In this tutorial, you’ll build a data pipeline to forecast outcomes of the March Madness basketball tournament. We’ll focus on the process of moving from Jupyter notebooks to production code, not on the actual logic, though the solution is adapted from one of the most successful algorithms in the recurring March Machine Learning Mania Kaggle competition and may be of interest to many attendees.

# Description

## Agenda

### Introduction (30 minutes)

- Data science in production: the good, the bad, and the ugly
- What is Kedro?
- Kedro and data orchestrators
- How does Kedro fit into the data science ecosystem?

### Get started with Kedro (60 minutes)

- Create a Kedro project from scratch
- Create a Kedro project from a starter
- Use Kedro from a Jupyter notebook
- Set up the Kedro Data Catalog
- Explore the data
- Refactor your data processing code into functions
- Create your first data pipeline with Kedro
- Assemble your nodes into a Kedro pipeline
- Run your Kedro pipeline
- Visualise your data pipeline with Kedro-Viz

### Break (15 minutes)

### Build complex Kedro pipelines (60 minutes)

- Refactor your data science code into functions
- Work with parameters in Kedro
- Create a Kedro pipeline with parameters
- Reuse your Kedro pipeline using namespaces
- Kedro pipeline runners
- Create Kedro datasets dynamically using factories

### Deploy your Kedro project in production (30 minutes)

- Define your own Kedro environments
- Package your Kedro project
- Deploy your Kedro project using Apache Airflow

### Break (15 minutes)

### Next steps (15 minutes)

- Work with databases
- Validate your data
- Run AI workflows
- Track your experiments

## Learning goals

By the end of this tutorial, you should:

- Understand and appreciate the importance of software engineering principles as they apply to data engineering and data science code
- Be able to build and deploy your own real-world data pipelines using Kedro

# Notes

I already have a [working solution](https://github.com/deepyaman/dagster-dunks) that I will to clean up to create the tutorial. The code is based on https://www.kaggle.com/code/rustyb/paris-madness-2023/notebook, which was originally adapted from https://github.com/fakyras/ncaa_women_2018/blob/master/win_ncaa.R.

## Prerequisites

We expect attendees to have basic knowledge of Python and scientific computing. Exposure to basic machine learning/statistical learning with tabular data (not necessarily in Python) will be helpful, but not necessary, for the second half of the tutorial. Overall, we’re striving to make this tutorial as accessible as possible.

## Installation instructions

We will provide a pre-built environment using GitHub Workspaces, so all you need is a laptop that can connect to the internet. You can also use your own local environment, but we will support this workflow on a best-effort basis during the tutorial.
