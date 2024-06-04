---
layout: post
title: Default Row Timestamps in BigQuery with SQLAlchemy
cover-img: ["https://upload.wikimedia.org/wikipedia/commons/e/e0/Digital_clock_5.00_AM.jpg" : "Digital clock BY PEAK99. Source: https://tinyurl.com/digital-clock-peak99 License: CC BY 3.0 DEED"]
tags:
  - Google-Cloud-Platform(GCP)
  - Tutorial
  - SQLAlchemy
  - python
---
When adding new records, data warehouses often set a field that denotes when data was ingested. This helps with tracking changes to data over time, and [removing duplicates](https://nanoman657.github.io/2023-09-29-removing-duplicates-in-bq/). In this tutorial, you'll first learn how to set a `synced_at` column in #BigQuery and beyond with #SQLAlchemy.

### Typical Use Case

To better understand a common use case, let's take a look at the below raw `person` table:

| id | first_name | last_name | favorite_color | synced_at       |
|----|------------|-----------|----------------|------------|
| 1  | Joshua     | Trusty    | red            | 09-14-2023 |
| 1  | Joshua     | Trusty    | scarlet        | 01-01-2023 |
| 2  | Misael     | Gonzalez  | red            | 09-14-2023 |

Since the table is raw, it has an obvious duplicate. The person 'Joshua Trusty' appears twice in this table. We may consider this a duplicate if the first three columns constitute the table's *primary key*.  In such a case, the `synced_at` field helps us to identify the most recent version. This could also be helpful if we want to understand how a unique `person` changes their `favorite_color` over time.  Here, we see that person 'Joshua Trusty' had a favorite color of 'scarlet' in the beginning of the year, but later changed it to 'red'.

Clearly, a `synced_at` column has multiple uses for building data pipelines and understanding your data. Fortunately, the process of incorporating it is relatively straightforward. Let's discuss a quick and easy way to add a `synced_at` timestamp column in BigQuery.

### Add `synced_at` to BigQuery
#### Via Table Creation
The easiest way to add `synced_at` is right when you are designing your table from the start. As an example, you can do this using the code example below:
```sql
CREATE OR REPLACE TABLE
  `tutorial.person` (
    id INT64,
    first_name STRING,
    last_name STRING,
    favorite_color STRING,
    synced_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
;
```
Here's a brief explanation of the code block:

1. The SQL code creates a table named `person` within the tutorial dataset.
2. Most of the columns (except `synced_at`), such as `id` being an INT64 and so on.
3. The synced_at column is configured with `DEFAULT CURRENT_TIMESTAMP`, ensuring that if its value is not explicitly set by a query, it will be automatically set to the current timestamp.

Now suppose we decided to insert a record into this table like so:
```
INSERT INTO
  `tutorial.person` (
    id,
    first_name,
    last_name,
    favorite_color
    )
VALUES
  (1, "Joshua", "Trusty", "red")
;
```

Notice that we did not set the `synced_at` column in our query. Yet, when we select the only value in the table, we see that it has been populated for us:

| id | first_name | last_name | favorite_color | synced_at       |
|----|------------|-----------|----------------|------------|
| 1  | Joshua     | Trusty    | red            | 2023-10-14 16:12:44.138938 UTC |

If you don't already have your table created, this is the cleanest way of doing so within BigQuery.

#### Via Table Alteration

Table alteration shouldn't be your first choice, but often times we work with pre-existing infrastructure.

First, we'll recreate the `person` table to omit the `synced_at` column:

```sql
CREATE OR REPLACE TABLE
  `tutorial.person` (
    id INT64,
    first_name STRING,
    last_name STRING,
    favorite_color STRING,
)
;
```

Now, we can alter the table and add the `synced_at` column:

```sql
ALTER TABLE `tutorial.person`

ADD COLUMN synced_at TIMESTAMP;

ALTER TABLE `tutorial.person` ALTER COLUMN synced_at SET DEFAULT CURRENT_TIMESTAMP;

UPDATE `tutorial.person` SET synced_at = CURRENT_TIMESTAMP WHERE TRUE;
```

Here's a brief explanation of the code block:
1. The first statement adds a new column, `synced_at`, of type TIMESTAMP to the existing `tutorial.person` table.
2. The second statement modifies the default value of the `synced_at` column to be the current timestamp.
3. The third statement updates all rows in the `tutorial.person` table, setting the `synced_at` column to the current timestamp. The `WHERE TRUE` condition ensures that all rows are updated.

At time of writing, the approach requires three different queries to accomplish. If you try in under three, you'll get an error message from BQ saying:

>Add field with default value to an existing table schema is not supported

 For this reason, having the default value set at table creation makes the most sense. Next, we'll explore a more flexible option provided by SQLAlchemy.
 
### SQLAlchemy

I'm a huge fan of SQLA because it allows you to setup your tables in a largely cloud-agnostic way. 

```python
import sqlalchemy as sa

from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()
your_project = 'edit_this'
your_dataset = 'edit_this'
credentials = json.loads(  
environ.get("GOOGLE_APPLICATION_CREDENTIALS", "{}")  
) # Set your credentials in an environment variable

class Person(Base):
    __tablename__ = 'person'

    id = sa.Column(sa.Integer, primary_key=True)
    first_name = sa.Column(sa.String)
    last_name = sa.Column(sa.String)
    favorite_color = sa.Column(sa.String)
    synced_at = sa.Column(
        sa.TIMESTAMP,
        server_default=sa.func.current_timestamp(),
    )

engine = sa.create_engine(f'bigquery://{your_project}/{your_dataset}/',  
credentials_info={credentials},  
)

Base.metadata.create_all(engine)


```
Here's a brief explanation of the code block:
1. It defines `Base` as an instance of the declarative base class. This `Base` will be used as the base class for your SQLAlchemy models.
2. The code defines a class named `Person`, which represents a table in the database.
    - `id`, `first_name`, `last_name`, `favorite_color`, and `synced_at` are class attributes representing the columns in the `person` table.   
3. An SQLAlchemy database engine is created using `sa.create_engine`. The engine specifies the database type (in this case, BigQuery), your project and dataset, and any credentials required for connecting to the database.
4. Finally, `Base.metadata.create_all(engine)` creates the table in the database based on the `Person` class. This statement generates the SQL required to create the table structure based on the class definition and executes it on the database.
    
If you'd like to get that code operational, you just need to include your specific credentials and other minor details. While it works immediately for BigQuery, you can switch the `engine` to any database you'd like. That means the code is flexible enough to adjust to your preferred data warehousing service.


### Conclusion
I hope you enjoyed this short tutorial on how to keep track of different iterations of your data. This approach is extraordinarily helpful if you want to deduplicate, or just want to perform data modeling over a time series. If you're just starting out with building your data pipelines, I recommend delving deeper into SQLAlchemy. The library can help you explore multiple data warehousing services before you commit to the one that best fits your use case. Thanks for reading.
