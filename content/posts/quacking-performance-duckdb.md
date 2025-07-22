---
title: "Quacking Performance: DuckDB"
date: 2025-07-22T12:44:37-07:00
draft: true
---

# Intro-duck-tion

If you've not came across DuckDB previously I'd highly recommend looking into it, it's fast, open source and rapidly gaining traction within the Data & Analytics space. DuckDB is designed for fast SQL queries on analytical workloads, namely it's design choices favour aggregation and grouping, particularly in instances where only a subset of columns are used. This greatly reduces memory usage and runtime in applicable instances.

# Warehouse Storage: Rows vs Columns

To highlight the differences between row-based and column-based data stores, I'll use the analogy that we're a grocery store storing customer orders.

Imagine a massive warehouse storing customer orders. In a row-based system, each order is packed into its own shopping cart or box. For instance, cart 12345 contains everything for John's order: 2 bottles of detergent, some soft drinks, some food, and the order date. When you need to retrieve John's complete order, you simply grab his cart and have everything in one place which is extremely efficient when retrieving full orders. Retrieving multiple full orders, or all orders for a specific person is also very efficient due to the row-based storage and indices. Row-based relational stores are extremely popular for use in applications because of this.

```goat
ROW-BASED STORAGE (Shopping Carts)
┌───────────────────────────────────────────────────────────────────┐
│                         Warehouse Floor                           │
│                                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  │  Cart 12345 │  │  Cart 12346 │  │  Cart 12347 │  │  Cart 12348 │
│  │             │  │             │  │             │  │             │
│  │ Detergent   │  │ Bread       │  │ Detergent   │  │ Bread       │
│  │ Bread       │  │ Drinks      │  │ Bread       │  │ Detergent   │
│  │ Drinks      │  │ Detergent   │  │ Date        │  │ Drinks      │
│  │ Date        │  │ Date        │  │             │  │ Date        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```
In a column-based system like DuckDB, the warehouse is organized into aisles by product type. All detergent from every order goes into the detergent aisle, all bread goes into the bread aisle, and so on.  To reconstruct John's complete order, you'd need to walk through multiple aisles collecting his items which would be notably slower than just retrieving his shopping cart.

However, should you need to know how many detergents were sold in the last three months, column based storage excels. You simply walk down the detergent aisle, scanning quantities and dates without touching or seeing any food, soft drinks or any other products. In the row-based warehouse, you'd have to inspect lots of individual shopping carts, search inside and count detergent bottles which is less efficient.


```goat
COLUMN-BASED STORAGE (Aisles)
┌───────────────────────────────────────────────────────────────────┐
│                         Warehouse Floor                          │
│                                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  │ Detergent   │  │    Bread    │  │   Drinks    │  │    Dates    │
│  │    Aisle    │  │    Aisle    │  │    Aisle    │  │    Aisle    │
│  │             │  │             │  │             │  │             │
│  │ Item 1      │  │ Item 1      │  │ Item 1      │  │ 2023-01-01  │
│  │ Item 2      │  │ Item 2      │  │ Item 2      │  │ 2023-01-15  │
│  │ Item 3      │  │ Item 3      │  │ Item 3      │  │ 2023-02-01  │
│  │ Item 4      │  │ Item 4      │  │ Item 4      │  │ 2023-02-15  │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

When doing large analytical queries where columns are likely to be omitted, and data sets are massive (usually being the result of joining large business tables) DuckDB will outperform row based data sources. This will utilise less memory (as it won't have to visit every aisle) and take less time. 

# How should I use it?

The biggest gain you'll see from this technology is avoiding directly querying data in relational data stores by utilizing an analytics data store or cache. Cloud providers have various options for storing large flat files but to start an on-premise file share works too (I always recommend starting simple and scaling when necessary). Use this to store datasets which are pre-joined as relational database will outperform column stores when executing joins. Data can be stored in a number of formats but Parquet is a sensible open standard. 

```goat
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ PostgreSQL  │    │    MySQL    │    │   Oracle    │    │ SQL Server  │
│     DB      │    │     DB      │    │     DB      │    │     DB      │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
       │                   │                   │                   │
       │ ETL/Export        │ ETL/Export        │ ETL/Export        │ ETL/Export
       │                   │                   │                   │
       v                   v                   v                   v
┌─────────────────────────────────────────────────────────────────────┐
│                     Analytics Datastore (NFS)                       │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────┐  │
│  │ sales.parquet│  │ users.csv    │  │ events.json  │  │ logs.csv│  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └─────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   │ Read Files
                                   │
                                   v
                         ┌─────────────────┐
                         │     DuckDB      │
                         │  🦆 In-Process  │
                         │   Analytics     │
                         └─────────────────┘
                                   │
                                   │ Generate
                                   │
                                   v
┌─────────────────────────────────────────────────────────────────────┐
│                         Aggregated Datasets                         │
│                                                                     │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│   │ Monthly Sales   │  │ User Analytics  │  │ Event Metrics   │     │
│   │   Summary       │  │   Dashboard     │  │   Report        │     │
│   └─────────────────┘  └─────────────────┘  └─────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

The goal here is to reduce push & pull from your relational data stores and to store the data in a column format (you'll get less juice from the squeeze if DuckDB needs to translate from CSV or another row based format). DuckDB can pull directly from relational databases and improve performance, this architecture will mean repeat queries are more performant as the data is stored natively in a column storage. It has the added benefit of reducing load on databases and transfering it to the data store.  


# That's quacking, what do I do next?

Super simple, if you're using SAS the 2025.07 release includes [SAS/ACCESS for DuckDB](https://communities.sas.com/t5/SAS-Communities-Library/The-Quack-is-Back-SAS-ACCESS-Meets-DuckDB/ta-p/969374) and will work once you upgrade - simply use a LIBNAME:
{{< highlight SAS>}}
LIBNAME POND DUCKDB;
{{< / highlight >}}

This libname can create local DuckDB storage or connect to NFS or Cloud storage like Azure Blob Storage.

DuckDB also runs as an in-process database so you can integrate it into your [application](https://duckdb.org/docs/stable/index#client-apis) or run queries as [command line tools](https://duckdb.org/docs/stable/clients/cli/overview). I'll avoid trying to reinvent the wheel with a getting started guide, but grab your favourite dataset and run some queries (time will show you how long the query took):
{{< highlight bash>}}
time duckdb < query.sql
{{< / highlight >}}
