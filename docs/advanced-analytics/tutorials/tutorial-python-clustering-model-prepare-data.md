---
title: "Tutorial: Prepare data to perform clustering in Python"
description: In part two of this four-part tutorial series, you'll prepare the data from a SQL Server database to perform clustering in Python with SQL Server Machine Learning Services.
ms.prod: sql
ms.technology: machine-learning
ms.devlang: python
ms.date: 08/30/2019
ms.topic: tutorial
author: garyericson
ms.author: garye
ms.reviewer: davidph
monikerRange: ">=sql-server-2017||>=sql-server-linux-ver15||=sqlallproducts-allversions"
---

# Tutorial: Prepare data to perform clustering in Python with SQL Server Machine Learning Services

In part two of this four-part tutorial series, you'll import and prepare the data from a SQL database using Python. Later in this series, you'll use this data to train and deploy a clustering model in Python with SQL Server Machine Learning Services.

In this article, you'll learn how to:

> [!div class="checklist"]
> * Separate customers along different dimensions using Python
> * Load the data from the SQL database into a Python data frame

In [part one](tutorial-python-clustering-model.md), you installed the prerequisites and imported the sample database.

In [part three](tutorial-python-clustering-model-build.md), you'll learn how to create and train a K-Means clustering model in Python.

In [part four](tutorial-python-clustering-model-deploy.md), you'll learn how to create a stored procedure in a SQL database that can perform clustering in Python based on new data.

## Prerequisites

* Part two of this tutorial assumes you have fulfilled the prerequisites of [**part one**](tutorial-python-clustering-model.md).

## Separate customers

To prepare for clustering customers, you'll first separate customers along the following dimensions:

* **orderRatio** = return order ratio (total number of orders partially or fully returned versus the total number of orders)
* **itemsRatio** = return item ratio (total number of items returned versus the number of items purchased)
* **monetaryRatio** = return amount ratio (total monetary amount of items returned versus the amount purchased)
* **frequency** = return frequency

Open a new notebook in Azure Data Studio and enter the following script.

In the connection string, replace connection details as needed.

```python
# Load packages.
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd
import revoscalepy as revoscale
from scipy.spatial import distance as sci_distance
from sklearn import cluster as sk_cluster

################################################################################################

## Connect to DB and select data

################################################################################################

# Connection string to connect to SQL Server named instance.
conn_str = 'Driver=SQL Server;Server=localhost;Database=tpcxbb_1gb;Trusted_Connection=True;'

input_query = '''SELECT
ss_customer_sk AS customer,
ROUND(COALESCE(returns_count / NULLIF(1.0*orders_count, 0), 0), 7) AS orderRatio,
ROUND(COALESCE(returns_items / NULLIF(1.0*orders_items, 0), 0), 7) AS itemsRatio,
ROUND(COALESCE(returns_money / NULLIF(1.0*orders_money, 0), 0), 7) AS monetaryRatio,
COALESCE(returns_count, 0) AS frequency
FROM
(
  SELECT
    ss_customer_sk,
    -- return order ratio
    COUNT(distinct(ss_ticket_number)) AS orders_count,
    -- return ss_item_sk ratio
    COUNT(ss_item_sk) AS orders_items,
    -- return monetary amount ratio
    SUM( ss_net_paid ) AS orders_money
  FROM store_sales s
  GROUP BY ss_customer_sk
) orders
LEFT OUTER JOIN
(
  SELECT
    sr_customer_sk,
    -- return order ratio
    count(distinct(sr_ticket_number)) as returns_count,
    -- return ss_item_sk ratio
    COUNT(sr_item_sk) as returns_items,
    -- return monetary amount ratio
    SUM( sr_return_amt ) AS returns_money
FROM store_returns
GROUP BY sr_customer_sk ) returned ON ss_customer_sk=sr_customer_sk'''


# Define the columns we wish to import.
column_info = {
    "customer": {"type": "integer"},
    "orderRatio": {"type": "integer"},
    "itemsRatio": {"type": "integer"},
    "frequency": {"type": "integer"}
}
```

## Load the data into a data frame

Results from the query are returned to Python using the revoscalepy **RxSqlServerData** function. As part of the process, you'll use the column information you defined in the previous script.

```python
data_source = revoscale.RxSqlServerData(sql_query=input_query, column_Info=column_info,
                                        connection_string=conn_str)
revoscale.RxInSqlServer(connection_string=conn_str, num_tasks=1, auto_cleanup=False)
# import data source and convert to pandas dataframe.
customer_data = pd.DataFrame(revoscale.rx_import(data_source))
```

Now display the beginning of the data frame to verify it looks correct.

```python
print("Data frame:", customer_data.head(n=5))
```

```results
Rows Read: 37336, Total Rows Processed: 37336, Total Chunk Time: 0.172 seconds
Data frame:     customer  orderRatio  itemsRatio  monetaryRatio  frequency
0    29727.0    0.000000    0.000000       0.000000          0
1    97643.0    0.068182    0.078176       0.037034          3
2    57247.0    0.000000    0.000000       0.000000          0
3    32549.0    0.086957    0.068657       0.031281          4
4     2040.0    0.000000    0.000000       0.000000          0
```

## Clean up resources

If you're not going to continue with this tutorial, delete the tpcxbb_1gb database from SQL Server.

## Next steps

In part two of this tutorial series, you completed these steps:

* Separate customers along different dimensions using Python
* Load the data from the SQL database into a Python data frame

To create a machine learning model that uses this customer data, follow part three of this tutorial series:

> [!div class="nextstepaction"]
> [Tutorial: Create a predictive model in Python with SQL Server Machine Learning Services](tutorial-python-clustering-model-build.md)
