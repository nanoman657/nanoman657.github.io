---
layout: post
title: Data Loading with BigQuery
cover-img:
  - https://images.unsplash.com/photo-1607434472257-d9f8e57a643d?q=80&w=2672&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D: "Photo BY Mike van den Bos. Source: https://tinyurl.com/mvdb-photo License: Unsplash License"
tags:
  - Tutorial
  - python
  - ELT
  - BigQuery
---

Loading and storing data is an important part of a data engineer's work. Just like buying food from a store is the beginning, Data Extraction only unlocks the other parts of a data pipeline. This tutorial showcases the process of taking ownership of your data by storing it in #BigQuery .

### Introduction
Data Loading is the process of loading data into a Data Warehouse. A data warehouse is a platform where data is both stored and analyzed. If we return to the cookie analogy from last time, imagine that we now have some cookie batter. To make cookies, we will need to load the batter into a tray for further processing. There are many Data Warehouses we could load our data into, but for this tutorial we'll be using [BigQuery](https://cloud.google.com/bigquery?hl=en) (BQ), part of Google Cloud Platform (GCP). At the time of writing, you can get [10 GB of storage and 1 TB of querying per month](https://cloud.google.com/free?hl=en) for free. Before starting the tutorial, make sure you have your BigQuery account set-up.

#### Real-world Scenario
Suppose you have a website and want to perform analysis on all your users. The data originates from a database, but those are intended for data transactions, not data analysis. You have already extracted your user data from that database, and now need to load it into your data warehouse.

### Data Loading

We will use the [google-cloud-bigquery](https://github.com/googleapis/python-bigquery) library to load the data into BigQuery. You need to authenticate with BigQuery to successfully load data into a table, but the process is pretty simple. Querying the API can be trivially done using Python's builtin libraries. I recommend running the code on a [Google Colab](https://colab.google/) Jupyter notebook to easily segment the program and identify the outputs. You can access [the previous tutorial's notebook](https://colab.research.google.com/drive/1eHgSxAaJZ5Mm1BjGyjnxvMUR754Z6NHc?usp=sharing) and copy it to see the foundation we're building on here. 

#### Install Libraries

`google-cloud-bigquery` must be installed before we can use it in the code. At time of writing, Google Colab already has `pydantic` pre-installed. If the library isn't already installed in your environment, you can install the BigQuery library using pip:
```
!pip install google-cloud-bigquery
```

Add the below import statement to the list of imports in the second cell. 

```python
from google.cloud.bigquery import SchemaField, Client, Table
```

Once you're done, your import cell will look like the below:

```python
from datetime import datetime

import requests
from google.cloud.bigquery import SchemaField, Client, Table
from pydantic import BaseModel, Field
from os import environ
```

#### Authentication
Since we're going to load data into BQ, it's necessary to authenticate with GCP. There are various ways of authenticating with GCP. Google's own [Laurent Picard](https://cloud.google.com/developers/advocates/laurent-picard) wrote an [article](https://medium.com/google-colab/a-better-way-to-use-google-cloud-from-colab-bb93f88b5021) recommending this simple solution making use of Google Colab's built-in library:

```python
from google.colab import auth

PROJECT_ID = "YOUR_PROJECT_ID"
auth.authenticate_user(project_id=PROJECT_ID)
```

I recommend adding the import statement to the second cell with the others, and creating a new third cell to define the `PROJECT_ID` and authenticate.

You can skip to the "Data Ingestion" section on authentication if you use the code above. If a 3-liner doesn't sound appealing to you, then buckle up for the more involved approach below.

##### Authentication Without Google Colab
*Note: This approach uses json-stored credentials which is more involved and less secure. It is included for comparison purposes. Read the official Google documentation on Identities for Workloads [here](https://cloud.google.com/iam/docs/workload-identities).*

We'll be using the GCP console to type commands needed to create the credentials. This is an interface that is more likely to be stable across time vs the user interface. 

First, set the `GOOGLE_APPLICATION_CREDENTIALS` environment variable. You'll need a service account for this this. If you don't already have one, head to the [GCP console](https://console.cloud.google.com/welcome?cloudshell=true) and open up the shell. The shell code below will create a service account for you with the necessary role. Copy the shell codes below, and replace [PROJECT_ID] and [SERVICE_ACCOUNT_NAME] with your project id. The project id should be shown in parenthesis in your shell at the bottom.

**Create a new service account**:

```bash
gcloud iam service-accounts create [SERVICE_ACCOUNT_NAME] --display-name="service account"

```

**Add permissions to service account**:
```bash
gcloud projects add-iam-policy-binding [PROJECT_ID] --member="serviceAccount:[SERVICE_ACCOUNT_NAME]@[PROJECT_ID].iam.gserviceaccount.com" --role="roles/bigquery.user"
```

```bash
gcloud iam service-accounts keys create GOOGLE_APPLICATION_CREDENTIALS.json --iam-account [SERVICE_ACCOUNT_NAME]@PROJECT_ID.iam.gserviceaccount.com
```

**Download the credentials**:
```
cloudshell download GOOGLE_APPLICATION_CREDENTIALS.json
```

Now, let's upload the json credentials to our Google Colab, and set the value of the `GOOGLE_APPLICATION_CREDENTIALS` environment variable.

```python
environ["GOOGLE_APPLICATION_CREDENTIALS"] = "/content/project/google_application_credentials.json"
```

### Data Ingestion

Once we're authenticated, the code interacts with BigQuery using the `Client` class from the `google-cloud-bigquery` library. The `Client()` constructor is called to establish a connection with BigQuery.

We need to create a BigQuery dataset to store the table that will have our data. At the end of your notebook, make a new code cell that uses the Client to create a new dataset. 
```python
client = Client()
dataset_name = "tutorial"
dataset = client.create_dataset(dataset_name, exists_ok=True)
```

Set `exists_ok` to `True`. This ensures that if you run the cell again, it will automatically recognize that the dataset exists, and not throw an error. This makes the dataset creation *idempotent*. 

> [!NOTE]
> [**Idempotence**](https://en.wikipedia.org/wiki/Idempotence)is the property of certain operations in mathematics and computer science whereby they can be applied multiple times without changing the result beyond the initial application.
> 
   
Now we'll define our BigQuery schema. The schema creates the structure of your data in BigQuery, specifying the column names and their data types. Here, we'll use the [`SchemaField`](https://cloud.google.com/python/docs/reference/bigquery/latest/google.cloud.bigquery.schema.SchemaField) class to represent each column in the table. Make sure the data types in the schema (`STRING` and `DATETIME` in this example) match the data types of the corresponding fields in your `FakeUser` class. It should look like the below:

```python
schema = [
    SchemaField("id", "INTEGER"),
    SchemaField("firstname", "STRING"),
    SchemaField("lastname", "STRING"),
    SchemaField("email", "STRING"),
    SchemaField("phone", "STRING"),
    SchemaField("birthday", "DATE"),
    SchemaField("gender", "STRING"),
    SchemaField("address", "RECORD", fields=[SchemaField("zipcode", "STRING")]),
]
```

See how `zipcode` is nested inside `address`? This mirrors what we did in our use of Pydantic. We're effectively telling BigQuery to match this structure in the data warehouse.

Next we'll create the table to store the data. This will be done in an idempotent manner, similar to how how we created the dataset. Include the schema to define the table. 

```python
table_name = 'user'
table = dataset.create_table(table_name, schema=schema, exists_ok=True)
```

Finally, the `load_table_from_json` method is called on the BigQuery `client` object. This method takes a list of JSON objects, where each object represents a row of data to be inserted. Each `FakeUser` from Pydantic has a json method to return its data in stringified JSON format. 

```python
json_rows = [user.model_dump(mode="json") for user in fake_users]
client.load_table_from_json(json_rows, table)])
```

That's it! We have successfully loaded the data into BQ. The next few steps will be used to confirm that the data is there. We simply need to define the dataset, table, and project names as variables. Then we'll use an f-string to format a SQL select statement to pull the data. The `client` is used to query the table. You'll get a print out of the data to confirm that it's there.

```python
sql_code = f" SELECT * FROM `{PROJECT_ID}.{dataset_name}.{table_name}`"
output_dataframe = client.query(sql_code).to_dataframe()

print(output_dataframe)
```

### Conclusion
Data Loading is one of the most important parts of the ELT workflow. It often requires authentication with your data warehouse for security. Once the data is finally in hand, you can perform transformations on it. In the next tutorial, we'll discuss how to perform transformations to your data to increase its usefulness.

