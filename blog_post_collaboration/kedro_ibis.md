# From production-ready to production: building scalable data pipelines with Kedro and Ibis

In your Kedro journey, have you ever...

* ...slurped up large amounts of data into memory, instead of pushing execution down to the source database/engine?
* ...prototyped a node in pandas, and then rewritten it in PySpark/Snowpark/some other native dataframe API?
* ...implemented a proof-of-concept solution in 3-4 months on data extracts, and then struggled massively when you needed to move to running against the production databases and scale out?
* ...insisted on using Kedro across the full data engineering/data science workflow for consistency (fair enough), although dbt would have been the much better fit for non-ML pipelines, because you essentially needed a SQL workflow?

If so, read on!

## The dev-prod dilemma

I began using Kedro over five years ago, for everything from personal projects to large client engagements. Most Kedro users would agree with my experience—Kedro excels in the proof-of-concept/development phase. In fact, Kedro originates from [QuantumBlack, AI by McKinsey](https://www.mckinsey.com/capabilities/quantumblack), and it's no surprise that many other data consultancies and data science teams adopt Kedro to deliver quality code from the ground up. It helps data engineers, data scientists and machine learning engineers collaboratively build end-to-end data pipelines without throwing basic software engineering principles out the window.

In the development phase, teams often work off of data extracts—CSVs, dev databases and, everyone's favorite, Excel files. Kedro's connector ecosystem, [Kedro-Datasets](https://docs.kedro.org/projects/kedro-datasets/en/kedro-datasets-2.0.0/), makes it easy. Datasets expose a unified interface to read data from and write data to a plethora of sources. For most datasets, this involves loading the requested data into memory, processing it in a node (e.g. using pandas), and saving it back to some—often the same—place.

Unfortunately, deploying the same data pipelines in production often doesn't work as well as one would hope. When teams swap out data extracts for production databases, and data volumes multiply, the solution doesn't scale. A lot of teams preemptively use (Py)Spark for their data engineering workloads; Kedro has offered first-class support for PySpark since its earliest days for exactly this reason. However, Spark code still frequently underperforms code executed directly on the backend engine (e.g. BigQuery), even if you take advantage of predicate pushdown in Spark.

In practice, a machine learning (software) engineer often ends up rewriting proof-of-concept code to be more performant and scalable, but this presents new challenges. First, it takes time to reimplement everything properly in another technology, and developers frequently underestimate the effort required. Also, rewritten logic, intentionally or not, doesn't always correspond 1:1 to the original, leading to unexpected results (read: bugs). Proper testing during development can mitigate, but not eliminate, correctness issues, but expecting all data science code to be unit tested is wishful thinking. Last but not least, business stakeholders don't always understand why productionising code takes so much time and resources—after all, it worked before! Who knows? Maybe scaling data engineering code shouldn't be so hard...

Given the ubiquity of pandas during the development phase, some execution engines started supporting pandas-compatible or similar Python dataframe APIs, such as the [pandas API on Spark](https://spark.apache.org/docs/latest/api/python/user_guide/pandas_on_spark/index.html) and [BigQuery DataFrames](https://cloud.google.com/bigquery/docs/reference/bigquery-dataframes)[^1]. Case studies, like [this one from Virgin Hyperloop
One](https://www.databricks.com/blog/2019/08/22/guest-blog-how-virgin-hyperloop-one-reduced-processing-time-from-hours-to-minutes-with-koalas.html), demonstrate how these solutions can accelerate the scaling process for data-intensive code. That said, building and maintaining a pandas-compatible API requires significant commitment from the engine developer, so no panacea exists.

## The SQL solution

Thankfully, there exists a standardised programming language that every database (and more-or-less every major computation framework) supports: SQL![^2]

_If you like SQL,_ [dbt](https://www.getdbt.com) provides a battle-tested framework for defining and deploying data transformation workflows. [As of February 2022, over 9,000 companies were using dbt in production.](https://www.getdbt.com/blog/next-layer-of-the-modern-data-stack) The Kedro team frequently recommends it for the T in ELT (Extract, Load, Transform) workflows. Kedro and dbt share similar self-stated goals: [Kedro empowers data teams to write reproducible, maintainable, and modular Python code,](https://docs.kedro.org/en/stable/introduction/index.html#introduction-to-kedro) and ["dbt is a SQL-first transformation workflow that lets teams quickly and collaboratively deploy analytics code following software engineering best practices."](https://www.getdbt.com/product/what-is-dbt)

> _"When I learned about Kedro (while at dbt Labs), I commented that it was like dbt if it were created by Python data scientists instead of SQL data analysts (including both being created out of consulting companies)."_
>
> —Cody Peterson, Technical Product Manager @ Voltron Data

## "What if I don't want to use SQL?"

This post isn't going to enter the SQL vs. Python debate. Nevertheless, many teams and data scientists choose Python for their analytics workflows. Furthermore, in my experience, teams using Kedro for their machine learning pipelines prefer to use the same tool for data processing and feature engineering code, too.

Can we combine the flexibility and familiarity of Python with the scale and performance of modern SQL? Yes we can! ["Ibis is a Python library that provides a lightweight, universal interface for data wrangling."](https://github.com/ibis-project/ibis) Ibis exposes a Python dataframe API where the same code can execute against over 15 query engines, from local backends like pandas, Polars, and DuckDB to remote databases and distributed computation frameworks like BigQuery, Snowflake, and Spark. Ibis constructs a query plan, or intermediate representation (IR), that it evaluates lazily (i.e. as and when needed) on the execution engine. [Of course, if you still want to write parts of your logic in SQL, Ibis has you covered.](https://ibis-project.org/how-to/extending/sql.html)

## Reimagining Python-first production data pipelines

### Revisiting the Jaffle Shop

Since dbt represents the industry standard for SQL-first transformation workflows, let's use the Jaffle Shop example to inform the capabilities of a Python-first solution. For those unfamiliar with the Jaffle Shop project, it provides a playground dbt project for testing and demonstration purposes, akin to the Kedro spaceflights project. Take a few minutes to [try dbt with DuckDB](https://github.com/dbt-labs/jaffle_shop_duckdb) or [watch a brief walkthrough of the same](https://www.loom.com/share/ed4a6f59957e43158837eb4ba0c5ed67).

If you want to peek at the final solution using Kedro at this point, [jump to the "Try it yourself" section](#try-it-yourself).

### Creating a custom `ibis.Table` dataset

Kedro-Ibis integration needs the ability to load and save data using the backends Ibis supports. The data will be represented in memory as an [Ibis table](https://ibis-project.org/reference/expression-tables#ibis.expr.types.relations.Table), analogous to a (lazy) pandas dataframe. The first step in using Ibis involves connecting to the desired backend; for example, see [how to connect to the DuckDB backend](https://ibis-project.org/backends/duckdb#connect). We abstract this step in the dataset configuration. We also maintain a mapping of established connections for reuse—a technique borrowed from existing dataset implementations, like [those of `pandas.SQLTableDataset` and `pandas.SQLQueryDataset`](https://github.com/kedro-org/kedro-plugins/blob/kedro-datasets-2.0.0/kedro-datasets/kedro_datasets/pandas/sql_dataset.py).

There's also a need to support different [materialisation strategies](https://docs.getdbt.com/docs/build/materializations). Depending on what the user configures, we call either [`create_table`](https://ibis-project.org/backends/duckdb#ibis.backends.duckdb.Backend.create_table) or [`create_view`](https://ibis-project.org/backends/duckdb#ibis.backends.duckdb.Backend.create_view).

[Find the complete dataset implemention on GitHub](https://github.com/deepyaman/jaffle-shop/blob/main/src/jaffle_shop/datasets/ibis/table_dataset.py).

### Configuring backends with the `OmegaConfigLoader` using variable interpolation

Kedro 0.18.10 introduced [OmegaConf-native templating in catalog files](https://docs.kedro.org/en/latest/configuration/advanced_configuration.html#catalog). In our example:

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

Develop pipelines in the usual way. Here's a node written using Ibis alongside the original dbt model. Note that Ibis solves the [parametrisation problem](https://www.youtube.com/watch?v=XdZklxTbCEA&t=514s) with vanilla Python syntax; on the other hand, the dbt counterpart uses Jinja templating.

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

Ibis uses deferred execution, pushing code execution to the query engine and only moving required data into memory when necessary. The output below shows how Ibis represents the above node logic. At the run time, Ibis generates SQL instructions from `final`'s intermediate representation (IR), executing it at one time; no intermediate `order_payments` gets created. Note that the below code requires that the `stg_orders` and `stg_payments` datasets be materialised already.

```python
(kedro-jaffle-shop) deepyaman@Deepyamans-MacBook-Air jaffle-shop % kedro ipython
ipython --ext kedro.ipython
Python 3.11.5 (main, Sep 11 2023, 08:31:25) [Clang 14.0.6 ]
Type 'copyright', 'credits' or 'license' for more information
IPython 8.18.1 -- An enhanced Interactive Python. Type '?' for help.
[01/25/24 08:33:22] INFO     Resolved project path as: /Users/deepyaman/github/deepyaman/jaffle-shop.                                                                                            __init__.py:146
                             To set a different path, run '%reload_kedro <project_root>'
[01/25/24 08:33:22] INFO     Kedro project Jaffle Shop                                                                                                                                           __init__.py:115
                    INFO     Defined global variable 'context', 'session', 'catalog' and 'pipelines'                                                                                             __init__.py:116
[01/25/24 08:33:23] INFO     Registered line magic 'run_viz'                                                                                                                                     __init__.py:122

In [1]: orders = catalog.load("stg_orders")
[01/25/24 08:36:19] INFO     Loading data from stg_orders (TableDataset)...                                                                                                                  data_catalog.py:482

In [2]: payments = catalog.load("stg_payments")
[01/25/24 08:36:40] INFO     Loading data from stg_payments (TableDataset)...                                                                                                                data_catalog.py:482

In [3]: payment_methods = catalog.load("params:payment_methods")
[01/25/24 08:37:26] INFO     Loading data from params:payment_methods (MemoryDataset)...                                                                                                     data_catalog.py:482

In [4]: from jaffle_shop.pipelines.data_processing.nodes import process_orders

In [5]: final = process_orders(orders, payments, payment_methods)

In [6]: final
Out[6]:
r0 := DatabaseTable: stg_orders
  order_id    int64
  customer_id int64
  order_date  date
  status      string

r1 := DatabaseTable: stg_payments
  payment_id     int64
  order_id       int64
  payment_method string
  amount         float64

r2 := Aggregation[r1]
  metrics:
    credit_card_amount:   Coalesce([Sum(r1.amount, where=r1.payment_method == 'credit_card'), 0])
    coupon_amount:        Coalesce([Sum(r1.amount, where=r1.payment_method == 'coupon'), 0])
    bank_transfer_amount: Coalesce([Sum(r1.amount, where=r1.payment_method == 'bank_transfer'), 0])
    gift_card_amount:     Coalesce([Sum(r1.amount, where=r1.payment_method == 'gift_card'), 0])
    total_amount:         Sum(r1.amount)
  by:
    order_id: r1.order_id

r3 := LeftJoin[r0, r2] r0.order_id == r2.order_id

Selection[r3]
  selections:
    order_id:             r0.order_id
    customer_id:          r0.customer_id
    order_date:           r0.order_date
    status:               r0.status
    credit_card_amount:   r2.credit_card_amount
    coupon_amount:        r2.coupon_amount
    bank_transfer_amount: r2.bank_transfer_amount
    gift_card_amount:     r2.gift_card_amount
    amount:               r2.total_amount

In [7]: final.visualize()
```

In the final step, we draw a visual representation of the expression tree. This step requires the `graphviz` Python library be installed. In practice, we leave these complexities up to Ibis and the underlying engine!

![tmpip_tmztu](https://github.com/kedro-org/kedro-devrel/assets/14007150/e149b55b-3dbb-4953-affb-0658bfba394c)

### Try it yourself

Clone the [deepyaman/jaffle-shop GitHub repository](https://github.com/deepyaman/jaffle-shop) to download the completed Kedro Jaffle Shop project. Run `pip install -r requirements.txt` from the cloned directory to install the dependencies, including the Ibis DuckDB backend:

```bash
git clone https://github.com/deepyaman/jaffle-shop.git
cd jaffle-shop
pip install -r requirements.txt
```

Typically, the source data already resides in a data warehouse. However, for this toy example, the project's `data` folder includes CSV files necessary to initialise the database:

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

View the pipeline using Kedro-Viz with `kedro viz run`:

<img width="1470" alt="image" src="https://github.com/kedro-org/kedro-devrel/assets/14007150/9fedaf00-ecf6-418f-9c1e-a98844994ba2">

## But wait... there's more!

Making pipeline productionisation easier alone may justify adopting Ibis into a Kedro workflow, but other situations from my past experience also come to mind.

Have you ever found yourself developing data pipelines against an existing data warehouse, even though you know that the data infrastructure team is currently migrating to a different database solution? You expect to rewrite some amount of your code—unless you use Ibis?

Using Ibis can also help you build truly reusable pipelines. I previously led development of a Kedro-based code asset that provided a suite of reusable pipelines for customer analytics. We decided to use PySpark for data engineering, but many of our users didn't need to use PySpark. One major retailer stored all of their data in Oracle, and they ended up with a suboptimal workflow wherein they extracted data into Spark and did all the work there. Other teams looked at our pipelines for inspiration, but ended up rewriting them for their infrastructure—not our intention in delivering prebuilt pipelines. Last but not least, we spent so much time and effort setting up Spark on locked-down Windows machines, so data scientists could play with the pipelines we provided; being able to run the same logic using DuckDB or pandas locally would have been a godsend!

## What's next?

If you're familiar with dbt (or even if you examined the Jaffle Shop project discussed above), you'll notice a key functionality that we didn't implement here: [validations](https://docs.getdbt.com/docs/build/validation). Kedro natively integrates with pytest for unit testing, which plays well for verifying the correctness of transformations developed in Ibis.  Kedro also supports data validation through third-party plugins such as [kedro-pandera](https://kedro-pandera.readthedocs.io/en/latest/), and I've recently started work on extending pandera to support validating Ibis tables; look for a follow-up post covering that soon.

Ibis supports a subset of DDL operations, which means dbt's `incremental` and `materialized view` materialisations currently don't have counterparts yet. Some Ibis backends have explored exposing materialised views. While not explicitly covered above, the `ephemeral` materialisation equates to Kedro's `MemoryDataset`.

Finally, dbt offers enhanced deployment functionality, like the ability to detect and deploy only modified models; it's less straightforward to detect such changes with Kedro.

If you have any ideas or feedback about this tutorial or more generally on the pipeline productionisation experience, we would love to hear from you!

[^1]: [BigQuery implements its Python dataframe API by leveraging Ibis's BigQuery backend under the hood](https://voltrondata.com/resources/google-bigframes-ibis).
[^2]: How standardized is the SQL standard? [Gil Forsyth's PyData NYC 2022 talk demonstrates challenges arising from differences between SQL dialects](https://www.youtube.com/watch?v=XdZklxTbCEA&t=300s). Even the [dbt-labs/jaffle_shop GitHub repository](https://github.com/dbt-labs/jaffle_shop) README disclaims, "If this steps fails, it might mean that you need to make small changes to the SQL in the models folder to adjust for the flavor of SQL of your target database."
