# From production-ready to production: building scalable data pipelines with Kedro and Ibis

In your Kedro journey, have you ever...

* ...slurped up large amounts of data into memory, instead of pushing execution down to the source database/engine?
* ...prototyped a node in pandas, and then rewritten it in PySpark/Snowpark/some other native dataframe API?
* ...implemented a proof-of-concept solution in 3-4 months on data extracts, and then struggled massively when you needed to move to running against the production databases and scale out?
* ...insisted on using Kedro across the full data engineering/data science workflow for consistency (fair enough), although dbt would have been the much better fit for non-ML pipelines, because you essentially needed a SQL workflow?

If so, read on!

## The dev-prod dilemma

I began using Kedro over five years ago, for everything from personal projects to large client engagements. I think most Kedro users would agree with my experience—Kedro excels in the proof-of-concept/development phase. In fact, Kedro originates from QuantumBlack, the AI consulting arm of McKinsey & Company, and it's no surprise that many other data consultancies and data science teams adopt Kedro to deliver quality code from the ground up. It helps data engineers, data scientists and machine learning engineers collaboratively build end-to-end data pipelines without throwing basic software engineering principles out the window.

In the development phase, teams often work off of data extracts—CSVs, dev databases and, everyone's favorite, Excel files. Kedro's connector ecosystem, [Kedro-Datasets](https://docs.kedro.org/projects/kedro-datasets/en/kedro-datasets-2.0.0/), makes it easy. Datasets expose a unified interface to read data from and write data to a plethora of sources. For most datasets, this involves loading the requested data into memory, processing it in a node (e.g. using pandas), and saving it back to some—often the same—place.

Unfortunately, deploying the same data pipelines in production often doesn't work as well as one would hope. When teams swap out data extracts for production databases, and data volumes multiply, the solution doesn't scale. A lot of teams preemptively leverage (Py)Spark for their data engineering workloads; Kedro has offered first-class support for PySpark since its earliest days for exactly this reason. However, Spark code still frequently underperforms code executed directly on the backend engine (e.g. BigQuery), even assuming you take advantage of predicate pushdown in Spark.

In practice, a machine learning (software) engineer often ends up rewriting proof-of-concept code to be more performant and scalable, but this presents new challenges. First, it takes time to reimplement everything properly in another technology, and developers frequently underestimate the effort required. Also, rewritten logic, intentionally or not, doesn't always correspond 1:1 to the original, leading to unexpected results (read: bugs). Proper testing during development can mitigate, but not eliminate, correctness issues, but expecting all data science code to be unit tested is wishful thinking. Last but not least, business stakeholders don't always understand why productionising code takes so much time and resources—after all, it worked before! Who knows? Maybe scaling data engineering code shouldn't be so hard...

Given the ubiquity of pandas during the development phase, some execution engines started supporting pandas-compatible or similar Python dataframe APIs, such as the [pandas API on Spark](https://spark.apache.org/docs/latest/api/python/user_guide/pandas_on_spark/index.html) and [BigQuery DataFrames](https://cloud.google.com/bigquery/docs/reference/bigquery-dataframes)[^1]. Case studies, like [this one from Virgin Hyperloop
One](https://www.databricks.com/blog/2019/08/22/guest-blog-how-virgin-hyperloop-one-reduced-processing-time-from-hours-to-minutes-with-koalas.html), demonstrate how these solutions can accelerate the scaling process for data-intensive code. That said, building and maintaining a pandas-compatible API requires significant commitment from the engine developer, so no panacea exists.

## The SQL solution

Thankfully, there exists a standardized programming language that every database (and more-or-less every major computation framework) supports: SQL![^2]

_If you like SQL,_ [dbt](https://www.getdbt.com) provides a battle-tested framework for defining and deploying data transformation workflows. [As of February 2022, over 9,000 companies were using dbt in production.](https://www.getdbt.com/blog/next-layer-of-the-modern-data-stack) Here on the Kedro team, we love dbt! In fact, we frequently recommend it for the T in ELT (Extract, Load, Transform) workflows. Kedro and dbt share similar self-stated goals: [Kedro empowers data teams to write reproducible, maintainable, and modular Python code,](https://docs.kedro.org/en/stable/introduction/index.html#introduction-to-kedro) and ["dbt is a SQL-first transformation workflow that lets teams quickly and collaboratively deploy analytics code following software engineering best practices."](https://www.getdbt.com/product/what-is-dbt)

> _"When I learned about Kedro (while at dbt Labs), I commented that it was like dbt if it were created by Python data scientists instead of SQL data analysts (including both being created out of consulting companies)."_
>
> —Cody Peterson, Technical Product Manager @ Voltron Data

## "What if I don't want to use SQL?"

This post isn't going to enter the SQL vs. Python debate. Nevertheless, many teams and data scientists choose Python for their analytics workflows. Furthermore, in my experience, teams using Kedro for their machine learning pipelines prefer to use the same tool for data processing and feature engineering code, too.

Can we combine the flexibility and familiarity of Python with the scale and performance of modern SQL? Yes we can! ["Ibis is a Python library that provides a lightweight, universal interface for data wrangling."](https://github.com/ibis-project/ibis) Ibis exposes a Python dataframe API where the same code can execute against over 15 query engines, from local backends like pandas, Polars, and DuckDB to remote databases and distributed computation frameworks like BigQuery, Snowflake, and Spark. Ibis constructs a query plan, or intermediate representation (IR), that it evaluates lazily (i.e. as and when needed) on the execution engine.

## Reimagining Python-first production data pipelines

### Revisiting the Jaffle Shop

Since dbt represents the industry standard for SQL-first transformation workflows, let's use the Jaffle Shop example to inform the capabilities of a Python-first solution. For those unfamiliar with the Jaffle Shop project, it provides a playground dbt project for testing and demonstration purposes, akin to the Kedro spaceflights project. Take a few minutes to [try dbt with DuckDB yourself](https://github.com/dbt-labs/jaffle_shop_duckdb) or [watch a brief walkthrough of the same](https://www.loom.com/share/ed4a6f59957e43158837eb4ba0c5ed67).

If you want to peek at the final solution using Kedro at this point, [jump to the "Quickstart" section](#quickstart).

### Creating a custom `ibis.Table` dataset

For our Kedro-Ibis integration, we need the ability to load and save data using the backends Ibis supports. The data will be represented in memory as an [Ibis table](https://ibis-project.org/reference/expression-tables#ibis.expr.types.relations.Table), analagous to a pandas dataframe. The first step in using Ibis involves connecting to the desired backend; for example, see [how to connect to the DuckDB backend](https://ibis-project.org/backends/duckdb#connect). We abstract this step in the dataset configuration. We also maintain a mapping of established connections for reuse—a technique borrowed from existing dataset implementations, like [those of `pandas.SQLTableDataset` and `pandas.SQLQueryDataset`](https://github.com/kedro-org/kedro-plugins/blob/kedro-datasets-2.0.0/kedro-datasets/kedro_datasets/pandas/sql_dataset.py).

We also need to support different [materialisation strategies](https://docs.getdbt.com/docs/build/materializations). Depending on what the user configures, we call either [`create_table`](https://ibis-project.org/backends/duckdb#ibis.backends.duckdb.Backend.create_table) or [`create_view`](https://ibis-project.org/backends/duckdb#ibis.backends.duckdb.Backend.create_view).

[Find the complete dataset implemention on GitHub.](https://github.com/deepyaman/jaffle-shop/blob/main/src/jaffle_shop/datasets/ibis/table_dataset.py)

### Configuring backends with the `OmegaConfigLoader` using variable interpolation

Since Kedro 0.18.10, you can [leverage OmegaConf-native templating in catalog files](https://docs.kedro.org/en/latest/configuration/advanced_configuration.html#catalog). In our example:

```yaml
# conf/base/catalog_connections.yml
_duckdb:
  backend: duckdb
  # `database` and `threads` are parameters for `ibis.duckdb.connect()`.
  database: jaffle_shop.duckdb
  threads: 24

# conf/base/catalog.yml
raw_customers:
  type: jaffle_shop.datasets.ibis.TableDataset
  table_name: raw_customers
  connection: ${_duckdb}
  save_args:
    materialized: table

...
```

### Building pipelines

Develop pipelines the same way you would any other Kedro pipeline. Below, we show a node written using Ibis alongside the original dbt model. Note that Ibis solves the [parametrisation problem](https://www.youtube.com/watch?v=XdZklxTbCEA&t=514s) with vanilla Python syntax; on the other hand, the dbt counterpart uses Jinja templating.

<table>
 <tr>
  <th>Kedro node
  <th>dbt model
 <tr valign="top">
  <td>

```python
from __future__ import annotations

from typing import TYPE_CHECKING

import ibis
from ibis import _

if TYPE_CHECKING:
    import ibis.expr.types as ir


def process_orders(
    orders: ir.Table, payments: ir.Table, payment_methods: list[str]
) -> ir.Table:
    total_amount_by_payment_method = {}
    for payment_method in payment_methods:
        total_amount_by_payment_method[f"{payment_method}_amount"] = ibis.coalesce(
            payments.amount.sum(where=payments.payment_method == payment_method), 0
        )

    order_payments = payments.group_by("order_id").aggregate(
        **total_amount_by_payment_method, total_amount=payments.amount.sum()
    )

    final = orders.left_join(order_payments, "order_id")[
        orders.order_id,
        orders.customer_id,
        orders.order_date,
        orders.status,
        *[
            order_payments[f"{payment_method}_amount"]
            for payment_method in payment_methods
        ],
        order_payments.total_amount.name("amount"),
    ]
    return final
```

  <td>

```sql
{% set payment_methods = ['credit_card', 'coupon', 'bank_transfer', 'gift_card'] %}

with orders as (

    select * from {{ ref('stg_orders') }}

),

payments as (

    select * from {{ ref('stg_payments') }}

),

order_payments as (

    select
        order_id,

        {% for payment_method in payment_methods -%}
        sum(case when payment_method = '{{ payment_method }}' then amount else 0 end) as {{ payment_method }}_amount,
        {% endfor -%}

        sum(amount) as total_amount

    from payments

    group by order_id

),

final as (

    select
        orders.order_id,
        orders.customer_id,
        orders.order_date,
        orders.status,

        {% for payment_method in payment_methods -%}

        order_payments.{{ payment_method }}_amount,

        {% endfor -%}

        order_payments.total_amount as amount

    from orders


    left join order_payments
        on orders.order_id = order_payments.order_id

)

select * from final
```

</table>

### Quickstart

Clone the [deepyaman/jaffle-shop GitHub repository](https://github.com/deepyaman/jaffle-shop) to download the completed Kedro Jaffle Shop project. Run `pip install -r requirements.txt` from the cloned directory to install the dependencies, including the Ibis DuckDB backend:

```bash
git clone https://github.com/deepyaman/jaffle-shop.git
cd jaffle-shop
pip install -r requirements.txt
```

Typically, your source data already resides in a data warehouse. However, for this toy example, the project's `data` folder includes CSV files that we need to initialize the database with:

```bash
kedro run --pipeline seed
```

Finally, run the actual pipeline:

```bash
kedro run
```

Voilà! Feel free to confirm that the expected tables and views got created:

```pycon
>>> import duckdb
>>>
>>> con = duckdb.connect("jaffle_shop.duckdb")
>>> con.sql("SHOW TABLES")
┌───────────────┐
│     name      │
│    varchar    │
├───────────────┤
│ customers     │
│ orders        │
│ raw_customers │
│ raw_orders    │
│ raw_payments  │
│ stg_customers │
│ stg_orders    │
│ stg_payments  │
└───────────────┘

>>> con.sql("SELECT * FROM customers WHERE customer_id = 42")
┌─────────────┬────────────┬───────────┬─────────────┬───────────────────┬──────────────────┬─────────────────────────┐
│ customer_id │ first_name │ last_name │ first_order │ most_recent_order │ number_of_orders │ customer_lifetime_value │
│    int64    │  varchar   │  varchar  │    date     │       date        │      int64       │         double          │
├─────────────┼────────────┼───────────┼─────────────┼───────────────────┼──────────────────┼─────────────────────────┤
│          42 │ Diana      │ S.        │ 2018-02-04  │ 2018-03-12        │                2 │                    27.0 │
└─────────────┴────────────┴───────────┴─────────────┴───────────────────┴──────────────────┴─────────────────────────┘

```

As always, we can view the pipeline using Kedro-Viz:

<img width="1470" alt="image" src="https://github.com/kedro-org/kedro-devrel/assets/14007150/9fedaf00-ecf6-418f-9c1e-a98844994ba2">

## But wait... there's more!

Making pipeline productionisation easier alone may justify adopting Ibis in your Kedro workflows, but other situations from my past experience also come to mind.

Have you ever found yourself developing data pipelines against an existing data warehouse, even though you know that the data infrastructure team is currently migrating to a different database solution? You expect to rewrite some amount of your code—unless you use Ibis?

Leveraging Ibis can also help you build truly reusable pipelines. I previously led development of a Kedro-based code asset that provided a suite of reusable pipelines for customer analytics. We decided to use PySpark for data engineering, but many of our users didn't need to use PySpark. One major retailer stored all of their data in Oracle, and they ended up with a suboptimal workflow wherein they extracted data into Spark and did all the work there. Other teams looked at our pipelines for inspiration, but ended up rewriting them for their infrastructure—not our intention in delivering prebuilt pipelines. Last but not least, we spent so much time and effort setting up Spark on locked-down Windows machines, so data scientists could play with the pipelines we provided; being able to run the same logic using DuckDB or pandas locally would have been a godsend!

## What's next?

If you're familiar with dbt (or even if you examined the Jaffle Shop project discussed above), you'll notice a key functionality that we didn't implement here: [validations](https://docs.getdbt.com/docs/build/validation). Kedro supports data validation through third-party plugins such as [kedro-pandera](https://kedro-pandera.readthedocs.io/en/latest/), and I've recently started work on extending pandera to support validating Ibis tables; look for a follow-up post covering that soon.

If you have any ideas or feedback about this tutorial or more generally on the pipeline productionisation experience, we would love to hear from you!

[^1]: [BigQuery implements its Python dataframe API by leveraging Ibis's BigQuery backend under the hood.](https://voltrondata.com/resources/google-bigframes-ibis)
[^2]: How standardized is the SQL standard? [Gil Forsyth's PyData NYC 2022 talk demonstrates challenges arising from differences between SQL dialects.](https://www.youtube.com/watch?v=XdZklxTbCEA&t=300s) Even the [dbt-labs/jaffle_shop GitHub repository](https://github.com/dbt-labs/jaffle_shop) README disclaims, "If this steps fails, it might mean that you need to make small changes to the SQL in the models folder to adjust for the flavor of SQL of your target database."
