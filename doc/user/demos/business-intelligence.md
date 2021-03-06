---
title: "Business Intelligence Demo"
description: "Find out how Materialize can help improve your team's business intelligence tools."
menu:
    main:
        parent: "demos"
        weight: 4
---

**tl;dr** Materialize can enable real-time monitoring within business
intelligence tools, and we have a [demo](#run-the-demo) showing you that it's
feasible.

Traditionally, business intelligence tools rely on reading day-old answers from
data warehouses. Given the substantial size of the data they're required to
process, it makes sense that you wouldn't want to recompute the answer every
time someone wants to view it.

Materialize, though, offers a totally different paradigm for powering BI tools
that offers complex, real-time analysis over massive datasets. The fundamental
idea is that Materialize persists the results of your queries, and then
incrementally updates them as new data comes in. Unlike the traditional RDBMS
model, which doesn't rely on previously completed work, Materialize minimizes
the time required to provide fresh answers.

## Overview

In this demo, we'll take the role of a business that has a transactional
workload running against our servers. To support the business' goals, we'll
stand up a BI tool to perform analytic-style queries over that transactional
data.

BI tools often sit on top of complex infrastructure, which is unreasonable to
spin up for a demo. To let you get a feel of what Materialize is like to use
without an enormous investment of time, we've built out a Docker Compose file
that can take care of the entire deployment for you.

In this section, we'll cover the "what" and "why" of our proposed deployment
using Materialize to provide real-time analytics within Metabase, an open source
business intelligence tool.

### Dataset & load generator

For this demo, Materialize uses a custom benchmark called chBench that is really
just a concatenation of two well known database benchmarking tools, TPC-C and
TPC-H.

[TPC-C](http://www.tpc.org/tpcc/detail.asp) is an industry-standard benchmark
for transactional workloads, meant to test a database's scalability.

[TPC-H](http://www.tpc.org/tpch/) is from the same group that developed TPC-C,
and is meant to test the capabilities of online analytic platforms by performing
complex analytic style queries. This includes large aggregations, many
groupings, and complex multi-way joins.

[CH-benCHmark](https://db.in.tum.de/research/projects/CHbenCHmark/?lang=en)
brings together the a TPC-C-like dataset, with TPC-H's analytical queries. This
is a great approximation for how many businesses perform OLAP queries over OLTP
data.

### Database (MySQL)

This demo relies on MySQL, which is a stable, well supported platform with good
performance.

### Change Data Capture & Streaming (Debezium & Kafka)

As the chBench load generator writes data to your database, we need to propagate
those changes to Materialize so it can update the answers to your queries.

With MySQL, the easiest way to do this is through a change data capture (CDC)
tool, which can describe changes to your data. This demo relies on Debezium as
its CDC tool.

To ferry the CDC data to Materialize, we stream the data using Kafka. This
requires a suite of tools from Confluent, e.g. ZooKeeper, Confluent Schema
Registry, etc, which we've also included in our demo's Docker deployment.

### Materialize

Materialize ingests the CDC data from MySQL, representing the chBench load
generator's continued activity.

With that data available, you can perform complex queries, and Materialize
maintains their results, even as the underlying data changes. This means that
you can get answers to your queries in real time, rather than relying on day-old
answers from your data warehouse.

### BI Tool (Metabase)

Metabase is an open source tool to create visualizations of SQL queries'
results, and then group them into dashboards. For instance, teams might use
Metabase to monitor purchasing geographic purchasing patterns from their stores.

In our demo, we'll use Metabase to visualize the results of some TPC-H-like
queries, and watch the visualizations update as quickly as Metabase allows.

In thinking of this deployment in terms of client-server relationships, this BI
tool represents a client, and Materialize represents a server.

### Diagram

Putting this all together, our deployment looks like this:

![Materialize deployment diagram with
Metabase](/images/demos/bi_architecture_diagram.png)

## Run the demo

### Preparing the environment

1. Start the Docker daemon for your machine, and [follow our Docker integration
   guide](../../third-party/docker).

1. Clone the Materialize repo:

    ```shell
    git clone git@github.com:MaterializeInc/materialize.git
    ```

1. Move to the `demo/chbench` dir:

    ```shell
    cd <path to materialize>/demo/chbench
    ```

    You can also find the demo's code on
    [GitHub](https://github.com/MaterializeInc/materialize/tree/{{< version >}}/demo/chbench).

1. Deploy and start all of the components we've listed above.

    Note that pulling down all of the Docker images necessary for the demo takes
    quite a bit of time (upwards of 10 minutes, even on very fast connections).

    ```shell
    # Deploy MySQL, Debezium, Kafka et. al., Metabase, and Materialize.
    # Initialize MySQL with some chBench data.
    ./dc.sh up :init:
    ./dc.sh up :demo:
    # Run the chBench workload generator.
    ./dc.sh demo-load
    ```

### Define sources & views

Now that our deployment is running (and looks like the diagram shown above), we
can get Materialize to read data from Kafka, and define the views we want
Materialize to maintain for us.

1. Launch a new terminal window and `cd <path to materialize>/demo/chbench`.

1. Launch the Materialize CLI (`mzcli`) by running:

    ```shell
    ./mzcompose run cli
    ```

1. Within `mzcli`, ensure you have all of the necessary sources, which represent
   all of the tables from MySQL:

    ```sql
    SHOW SOURCES;
    ```

    ```nofmt
    customer
    district
    item
    nation
    neworder
    order
    orderline
    region
    stock
    supplier
    warehouse
    ```

1. Create a straightforward view of the underlying data.
   <a name="define-query01"></a>

    ```sql
    CREATE MATERIALIZED VIEW query01 as SELECT
        ol_number,
        sum(ol_quantity) as sum_qty,
        sum(ol_amount) as sum_amount,
        avg(ol_quantity) as avg_qty,
        avg(ol_amount) as avg_amount,
        count(*) as count_order
    FROM
        orderline
    WHERE
        ol_delivery_d > date '1998-12-01'
    GROUP BY
        ol_number;
    ```

    This is used to repesent "Query 01" in chBench, which tracks statistics
    about the TPC-C `orderline` table.

1. Check the results of the view:

    ```sql
    SELECT * FROM query01;
    ```

    If you run this query multiple times, you should see the results change, and
    the answers should come back pretty quickly. (How quickly depends on the
    speed of your computer, but it should be some small fraction of a second.)

1. Define another view for "Query 07", which involves a complex 11-way `JOIN`
   across 6 tables:
   <a name="define-query07"></a>

    ```sql
    CREATE MATERIALIZED VIEW query07 AS
    SELECT
        su_nationkey AS supp_nation,
        substr(c_state, 1, 1) AS cust_nation,
        extract('year' FROM o_entry_d) AS l_year,
        sum(ol_amount) AS revenue
    FROM
        supplier,
        stock,
        orderline,
        order,
        customer,
        nation AS n1,
        nation AS n2
    WHERE
        ol_supply_w_id = s_w_id
        AND ol_i_id = s_i_id
        AND s_su_suppkey = su_suppkey
        AND ol_w_id = o_w_id
        AND ol_d_id = o_d_id
        AND ol_o_id = o_id
        AND c_id = o_c_id
        AND c_w_id = o_w_id
        AND c_d_id = o_d_id
        AND su_nationkey = n1.n_nationkey
        AND c_n_nationkey = n2.n_nationkey
    GROUP BY
        su_nationkey,
        substr(c_state, 1, 1),
        extract('year' FROM o_entry_d)
    ORDER BY
        su_nationkey, cust_nation, l_year;
    ```

1. Check the results of this query:

    ```sql
    SELECT * FROM query07;
    ```

    This query can take a few minutes to begin producing answers. Before answers
    are produced, you will see the following error:

    ```nofmt
    mz>  SELECT * FROM query07;
    At least one input has no complete timestamps yet.
    ```

    If you receive an empty set of results, either wait or disconnect from `mzcli`
    and relaunch it. Just like the prior reads from the materialized view, you
    should see these results update, and the response times should be quick!

### Set up Metabase

1. In a browser, go to <localhost:3030>.

1. Click **Let's get started**.

1. Complete first set of fields asking for your email address. This information
   isn't crucial for anything but does have to be filled in.

1. On the **Add your data** page, fill in the following information:

    Field             | Enter...
    ----------------- | ----------------
    Database          | **Materialize**
    Name              | **tpcch**
    Host              | **materialized**
    Port              | **6875**
    Database name     | **materialize**
    Database username | **root**
    Database password | Leave empty.

1. Proceed past the screens until you reach your primary dashboard.

## Create dashboards

1. Click **Ask a question**.

1. Click **Native query**.

1. From **Select a database**, select **tpcch**.

1. In the query editor, enter:

    ```sql
    SELECT * FROM query01;
    ```

    Materialize relies on you already having created a materialized view with
    this name, which you did the Materialize CLI [a few steps
    back](#define-query01).

    In a production setting, you might want to let users find views' underlying
    queries. For example, you might store the underlying queries in a GitHub
    repository, and provide a link to the repository as a comment. Those with
    direct access to the `materialized` process through a SQL client can find
    the underlying queries using [`SHOW CREATE
    VIEW`](../../sql/show-create-view).

1. Click the large **>** button.

    If you receive an error stating `One or more sources has no closed timestamps`, wait a few seconds, press enter in the query editor, and then
    repeat this process.

1. Once you see the results similar to those below, click **Save**, enter
   **query01** as the query's name, and then click **Save**.

1. When asked if you want to save this to a dashboard, click **Yes please!**.

1. Click **My personal collection**, and then click **Create a new dashboard**.

1. Enter **tpcch** as the **Name**, and then click **Create**.

1. Modify the size of the **query01** card, and then click **Save**.

1. Click **Auto-refresh**, and then select **1 minute** (which is the fastest
   refresh rate that the tool offers; Materialize can reasonably support 1
   second refreshes for many query types).

At this point, Metabase will now automatically refresh this analysis for you
every 1 minute.

If you want to see more chBench queries, you can repeat these steps for the view
`query07` or any of the queries listed in our [chBench query index](https://github.com/MaterializeInc/materialize/blob/master/demo/chbench/chbench/mz-default.cfg).

## Recap

In this demo, we saw:

-   How to define sources and views within Materialize
-   How to query defined views in a BI tool
-   Materialize's ability to serve results for views with the fastest refresh
    rates the tool offers

## Details

### Typical operation

In this demo, you "played" the role of both infrastructure engineer and
business analyst; that is to say, you both deployed Materialize, as well
as ran queries within your BI tool, Metabase.

To give you a sense of what this experience would look like if you were two
different people, let's highlight what the expected workflow is.

-   **Business analysts** perform _ad hoc_ queries in Materialize to determine
    what queries are meaningful to them.
-   **Business analysts** then provide those queries to **DBAs**.
-   **DBAs** turn those queries into materialized views, and then provide the
    **business analysts** with the views' names, e.g. `some_view`. **Business
    analysts** then query those views from within their BI tools, e.g. `SELECT * FROM some_view`.

Of course, it's possible to change this workflow to make it more self-service,
etc. but this is a reasonable workflow.

## Related pages

-   [Microservice demo](../microservice)
-   [Log parsing demo](../log-parsing)
-   [`CREATE SOURCE`](../../sql/create-source)
