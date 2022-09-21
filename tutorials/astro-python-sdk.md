---
title: 'Write a DAG with the Astro Python SDK'
sidebar_label: 'Use the Astro Python SDK'
id: astro-python-sdk
description: 'Write an ETL pipeline for Amazon S3 and Snowflake with the Astro SDK.'
---

This tutorial demonstrates how to write an Extract, Transform, Load (ETL) pipeline on your local machine with the Astro Python SDK. The Astro SDK is maintained by Astronomer and simplifies the pipeline authoring process with native Python functions for common data orchestration use cases.

The pipeline you build in this tutorial will:

- **Extract** a file into a Snowflake relational table.
- **Transform** that table.
- **Load** the transformed table into a reporting table.

The example DAG in this tutorial uses Amazon S3 and Snowflake, but you can replace these with any supported data sources and tables simply by changing connection information.

## Assumed knowledge

To get the most out of this tutorial, make sure you have a knowledge of:

- Basic Python and SQL.
- Basic knowledge of [Amazon S3](https://aws.amazon.com/s3/getting-started/) and [Snowflake](https://docs.snowflake.com/en/user-guide-getting-started.html).
- Airflow fundamentals, such as writing DAGs and defining tasks. See [Get started with Apache Airflow](get-started-with-airflow.md).

## Prerequisites

To complete this tutorial, you need:

- An AWS S3 account with a [storage bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/GetStartedWithS3.html). If you don't already have an account, Amazon offers 5GB of free storage in S3 for 12 months. This should be more than enough for this tutorial.
- A Snowflake Enterprise account. If you don't already have an account, Snowflake has a [free Snowflake trial](https://signup.snowflake.com/) for 30 days.
- The [Astro CLI](https://docs.astronomer.io/astro/cli/get-started).

## Step 1: Set up your data stores

For this example, you first need to load a file into S3 and create a destination for the data in Snowflake. In this case, the data has three example orders with distinct customers, purchase dates, and amounts.

1. On your local machine create a file named `orders_data_header.csv` with the following data:

    ```text
    order_id,customer_id,purchase_date,amount
    ORDER1,CUST1,1/1/2021,100
    ORDER2,CUST2,2/2/2022,200
    ORDER3,CUST3,3/3/2023,300
    ```

2. [Upload `orders_data_header.csv`](https://docs.aws.amazon.com/AmazonS3/latest/userguide/upload-objects.html) to your S3 bucket.
3. In Snowflake, create a new [worksheet](https://docs.snowflake.com/en/user-guide/ui-worksheet.html) and run the following SQL commands:

    ```sql
    CREATE DATABASE ASTRO_SDK_DB;
    CREATE WAREHOUSE ASTRO_SDK_DW;
    CREATE SCHEMA ASTRO_SDK_SCHEMA;
    ```

    These commands create all of the required data stores for the load step of your pipeline. Ensure that you have `ACCOUNTADMIN` permissions for your newly created database.

## Step 2: Set up your Airflow environment

Now that you have a staging table in Snowflake and some example data ready to load, you need to set up a local Airflow environment for your DAG to run in. This is easy with the Astro CLI.

1. Create a new Astro project:

    ```sh
    $ mkdir astro-sdk-tutorial && cd astro-sdk-tutorial
    $ astro dev init
    ```

2. Add the following line to the `requirements.txt` file of your Astro project:

    ```text
    astro-sdk-python[amazon,snowflake]>=0.11
    ```

    This installs the SDK package as well as the Snowflake and Amazon providers, which are required for any DAG that loads data to or from Amazon S3 and Snowflake. If you build this pipeline with different database services, you'll have to modify this line.

3. Add the following environment variables to the `.env` file of your project:

    ```text
    AIRFLOW__CORE__ENABLE_XCOM_PICKLING=True
    AIRFLOW__ASTRO_SDK__SQL_SCHEMA=ASTRO_SDK_SCHEMA
    ```

4. Run the following command to start your project in a local environment:

    ```sh
    astro dev start
    ```

## Step 3: Create Airflow connections to S3 and Snowflake

1. Open the Airflow UI at `http://localhost:8080/`
2. Go to **Admin** > **Connections**.
3. Create a new connection to AWS S3 with the following values:

    - Connection ID: `aws_default`
    - Connection type: `S3`
    - Extra: `{"aws_access_key_id": "<your_access_key>", "aws_secret_access_key": "<you_secret_access_key>"}`

4. Create a new connection to Snowflake with the following values

    - Connection Id: `snowflake_default`
    - Connection Type: `Snowflake`
    - Host: `https://<account>.<region>.snowflakecomputing.com/`. This is the URL where you can log into your Snowflake account.
    - Schema: `ASTRO_SDK_SCHEMA`
    - Login:
    - Password:
    - Account:
    - Database: `ASTRO_SDK_DB`
    - Region: (something like `us-east-1` or `us-central1.gcp`)
    - Role: `ACCOUNTADMIN`
    - Warehouse: `ASTRO_SDK_DW`

## Step 4: Create and populate some tables in Snowflake

Create some auxiliary tables in Snowflake and populate them with a small amount of data for this ETL example.

1. In your Snowflake worksheet, create and populate a `customers_table`. You'll join this table with an orders table that you create with the Astro Python SDK:

    ```sql
    CREATE OR REPLACE TABLE customers_table (customer_id CHAR(10), customer_name VARCHAR(100), type VARCHAR(10) );

    INSERT INTO customers_table (CUSTOMER_ID, CUSTOMER_NAME,TYPE) VALUES     ('CUST1','NAME1','TYPE1'),('CUST2','NAME2','TYPE1'),('CUST3','NAME3','TYPE2');
    ```

2. Create and populate a reporting table. This is where you'll merge your transformed data:

    ```sql

    CREATE OR REPLACE TABLE reporting_table (
        CUSTOMER_ID CHAR(30), CUSTOMER_NAME VARCHAR(100), ORDER_ID CHAR(10), PURCHASE_DATE DATE, AMOUNT FLOAT, TYPE CHAR(10));

    INSERT INTO reporting_table (CUSTOMER_ID, CUSTOMER_NAME, ORDER_ID, PURCHASE_DATE, AMOUNT, TYPE) VALUES
    ('INCORRECT_CUSTOMER_ID','INCORRECT_CUSTOMER_NAME','ORDER2','2/2/2022',200,'TYPE1'),
    ('CUST3','NAME3','ORDER3','3/3/2023',300,'TYPE2'),
    ('CUST4','NAME4','ORDER4','4/4/2022',400,'TYPE2');
    ```

## Step 5: Write a DAG for a simple ETL workflow

Use your favorite code editor or text editor to copy-paste the following code into a `.py` file in your project's `dags` directory:

```python
from datetime import datetime

from airflow.models import DAG
from pandas import DataFrame

# Import decorators and classes from the SDK
from astro import sql as aql
from astro.files import File
from astro.sql.table import Table

# Define constants for interacting with external systems
S3_FILE_PATH = "s3://<aws-bucket-name>"
S3_CONN_ID = "aws_default"
SNOWFLAKE_CONN_ID = "snowflake_default"
SNOWFLAKE_ORDERS = "orders_table"
SNOWFLAKE_CUSTOMERS = "customers_table"
SNOWFLAKE_REPORTING = "reporting_table"

# Define an SQL query for our transform step as a Python function using the SDK.
# This function filters out all rows with an amount value less than 150.
@aql.transform
def filter_orders(input_table: Table):
    return "SELECT * FROM {{input_table}} WHERE amount > 150"

# Define an SQL query for our transform step as a Python function using the SDK.
# This function joins two tables into a new table.
@aql.transform
def join_orders_customers(filtered_orders_table: Table, customers_table: Table):
    return """SELECT c.customer_id, customer_name, order_id, purchase_date, amount, type
    FROM {{filtered_orders_table}} f JOIN {{customers_table}} c
    ON f.customer_id = c.customer_id"""

# Define a function for transforming tables to dataframes
@aql.dataframe
def transform_dataframe(df: DataFrame):
    purchase_dates = df.loc[:, "purchase_date"]
    print("purchase dates:", purchase_dates)
    return purchase_dates

# Basic DAG definition. Run the DAG starting January 1st, 2019 on a daily schedule.
dag = DAG(
    dag_id="astro_orders",
    start_date=datetime(2019, 1, 1),
    schedule_interval="@daily",
    catchup=False,
)

with dag:
    # Extract a file with a header from S3 into a temporary Table, referenced by the
    # variable `orders_data`
    orders_data = aql.load_file(
        # Data file needs to have a header row. The input and output table can be replaced with any
        # valid file and connection ID.
        input_file=File(
            path=S3_FILE_PATH + "/orders_data_header.csv", conn_id=S3_CONN_ID
        ),
        output_table=Table(conn_id=SNOWFLAKE_CONN_ID),
    )

    # Create a Table object for customer data in the Snowflake database
    customers_table = Table(
        name=SNOWFLAKE_CUSTOMERS,
        conn_id=SNOWFLAKE_CONN_ID,
    )

    # Filter the orders data and then join with the customer table,
    # saving the output into a temporary table referenced by the Table instance `joined_data`
    joined_data = join_orders_customers(filter_orders(orders_data), customers_table)

    # Merge the joined data into the reporting table based on the order_id.
    # If there's a conflict in the customer_id or customer_name, then use the ones from
    # the joined data
    reporting_table = aql.merge(
        target_table=Table(
            name=SNOWFLAKE_REPORTING,
            conn_id=SNOWFLAKE_CONN_ID,
        ),
        source_table=joined_data,
        target_conflict_columns=["order_id"],
        columns=["customer_id", "customer_name"],
        if_conflicts="update",
    )

    # Transform the reporting table into a dataframe
    purchase_dates = transform_dataframe(reporting_table)

    # Delete temporary and unnamed tables created by `load_file` and `transform`, in this example
    # both `orders_data` and `joined_data`
    aql.cleanup()
```

This DAG extracts the data you loaded into S3 and runs a few simple SQL statements to clean the data, load it into a reporting table on Snowflake, and transform it into a dataframe so that you can print various table details to Airflow logs using Python.

Much of this DAG's functionality comes from the Python functions that use task decorators from the Python SDK. See [How it works](#how-it-works) for more information about these decorators and the benefits of using the SDK for this implementation.  

## Step 6: Run the code

1. In the Airflow UI, you should see a DAG called `astro_orders`. Make it active by clicking the slider next to its name:

    ![toggle](/img/guides/unpause-dag.png)

2. Click the play button next to the DAG's name to run the DAG:

    ![trigger](/img/guides/trigger-dag.png)

3. Click the DAG's name to see how it ran in the **Grid** view:

    ![gridview](/img/guides/select-dag-grid-view.png)

## How it works

The example DAG uses the TaskFlow API and decorators to define dependencies between tasks. If you're used to defining dependencies with bitshift operators, this might not look familiar. Essentially, the TaskFlow API abstracts dependencies, XComs, and other boilerplate DAG code so that you can define task dependencies with function invocations.

The Astro SDK takes this abstraction a step further while providing more flexibility to your code. The most important details are:

- Using `aql` decorators, you can run both SQL and Python within a Pythonic context. This example DAG uses decorators to run both SQL queries and Python code
- The Astro SDK includes a `Table` object which contains all of the metadata that's necessary for handling SQL table creation between Airflow tasks. When a `Table` is passed into a function, the Astro SDK automatically passes all connection, XCom, and metadata configuration to the task.

    The example DAG demonstrates one of the key powers of the `Table` object. When the DAG ran `join_orders_customers`, it joined two tables that had different connections and schema. The Astro SDK automatically creates a temporary table and handles joining the tables. This also means that you can replace the S3 and Snowflake configurations with any valid configurations for other supported data stores and the code will still work. The Astro SDK handles all of the translation between services and database types in the background.

- The Astro SDK can automatically convert to SQL tables to pandas DataFrames using the `aql.dataframe`, meaning you can run complex ML models and SQL queries on the same data in the same DAG without any additional configuration.

Now that you understand the core qualities of the Astro SDK, let's look at it in the context of the example DAG by walking through each step in your ETL pipeline.

### Extract

To extract from S3 into a SQL Table, you only need to specify the location of the data on S3 and an Airflow connection for the destination SQL table in Snowflake.

```python
# Extract a file with a header from S3 into a temporary Table, referenced by the
# variable `orders_data`
orders_data = aql.load_file(
    # data file needs to have a header row
    input_file=File(path=S3_FILE_PATH + "/orders_data_header.csv", conn_id=S3_CONN_ID),
    output_table=Table(conn_id=SNOWFLAKE_CONN_ID),
)
```

Because the content of `orders_data` isn't needed after the DAG is completed, it's specified without a name. When you define a `Table` object without a preexisting name, that table is considered a temporary table.

The Astro SDK deletes all temporary tables after you run `aql.cleanup` in your DAG.

### Transform

You can filter your loaded table from S3 and join it to a Snowflake table with single line of code. The result of this function is a temporary table called `joined_data`:

```python
@aql.transform
def filter_orders(input_table: Table):
    return "SELECT * FROM {{input_table}} WHERE amount > 150"


@aql.transform
def join_orders_customers(filtered_orders_table: Table, customers_table: Table):
    return """SELECT c.customer_id, customer_name, order_id, purchase_date, amount, type
    FROM {{filtered_orders_table}} f JOIN {{customers_table}} c
    ON f.customer_id = c.customer_id"""


# Create a Table object for customer data in our Snowflake database
customers_table = Table(
    name=SNOWFLAKE_CUSTOMERS,
    conn_id=SNOWFLAKE_CONN_ID,
)


# Filter the orders data and then join with the customer table,
# saving the output into a temporary table referenced by the Table instance `joined_data`
joined_data = join_orders_customers(filter_orders(orders_data), customers_table)
```

Because you defined `customers_table`, it is not considered temporary and will not be deleted after running `aql.cleanup`.

### Merge

To merge your processed data into a reporting table, you can call the `aql.merge` function:

```python
# Merge the joined data into our reporting table, based on the order_id .
# If there's a conflict in the customer_id or customer_name then use the ones from
# the joined data
reporting_table = aql.merge(
    target_table=Table(
        name=SNOWFLAKE_REPORTING,
        conn_id=SNOWFLAKE_CONN_ID,
    ),
    source_table=joined_data,
    target_conflict_columns=["order_id"],
    columns=["customer_id", "customer_name"],
    if_conflicts="update",
)
```

`aql.merge` is database agnostic. It automatically handles all background XComs and configuration that are required when working with tables from separate sources.

### Dataframe transformation

To illustrate the power of the `@aql.dataframe` decorator, the DAG simply converts your reporting table to a simple dataframe operation:

```python
@aql.dataframe
def transform_dataframe(df: DataFrame):
    purchase_dates = df.loc[:, "purchase_date"]
    print("purchase dates:", purchase_dates)
    return purchase_dates
```


You can find the output of this function in the logs of the final task:

![log](/img/guides/task-logs.png)

### Clean up temporary tables

Temporary tables can be created by setting `table.temp=True` when defining a Table object, or by simply not defining a `Table` outside of a function. The example DAG creates two temporary tables: `joined_data` and `orders_data`.

To remove these tables from your database once the DAG completes, you can delete them using the `aql.cleanup()` function:

```python
# Delete all temporary tables
aql.cleanup()
```

## Conclusion

You now know how to use the Astro SDK to write a common ETL pipeline! More specifically, you can now:

- Pass connection and database information to downstream Airflow tasks.
- Merge tables from different schemas and platforms without using XComs.
- Run SQL in an entirely Pythonic context.

As a next step, read the [Astro Python SDK documentation](https://astro-sdk-python.readthedocs.io/) to learn more about its functionality and task decorators.