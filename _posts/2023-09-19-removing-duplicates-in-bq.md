---
layout: post
title: Removing Duplicates in BigQuery
tags:
  - Google-Cloud-Platform(GCP)
  - SQL
  - Google-BigQuery
  - Data-Engineering
  - Data
  - Warehousing
  - Databases
---
"Don't Repeat Yourself" (DRY) is one of the most famous adages in software development. The principle advises us not to represent information in multiple places. Data warehouses present a challenge to applying the DRY, because they inherently allow for the possibility of duplicate records. How then do we tackle the problem of duplicates?
### Understanding Duplicates

First, let's begin by defining what a duplicate is. A simple definition is that a duplicate record occurs when two or more rows share identical values in the same columns. For example, suppose you have a table called `person`, named after the *subject* of the table. `person` has the following rows of data:

| id | first_name | last_name |
|----|------------|-----------|
| 1  | Joshua     | Trusty    |
| 1  | Joshua     | Trusty    |
| 2  | Misael     | Gonzalez  |

It is obvious that "Joshua Trusty" appears twice in the table, making the those records duplicates. Such *exact duplication* is straightforward to identify, but there are plenty of other cases that are more nuanced.

Consider this similar `person` table where the first two records have different `favorite_color` and `date`columns, but everything else is the same.

| id | first_name | last_name | favorite_color | date       |
|----|------------|-----------|----------------|------------|
| 1  | Joshua     | Trusty    | red            | 09-14-2023 |
| 1  | Joshua     | Trusty    | scarlet        | 01-01-2023 |
| 2  | Misael     | Gonzalez  | red            | 09-14-2023 |

This is not an exact duplication, but the first two records listed share significant commonality. To address this, let's refine our definition:

> Duplicate records are records in a table that record data about the same instance of the table's subject.

To remove these duplicate records, we need an understanding of what information constitutes uniqueness inside of the table.

### The Significance of Keys

Keys play a pivotal role in defining uniqueness within a table. An excellent overview of keys can be found in *[Database Design for Mere Mortals](https://ptgmedia.pearsoncmg.com/images/9780321884497/samplepages/0321884493.pdf)*, where author Michael J. Hernandez writes:

>The first type of key you establish for a table is the candidate key, which is a field or set of fields that uniquely identifies a single instance of the table’s subject. Each table must have at least one candidate key. You’ll eventually examine the table’s pool of available candidate keys and designate one of them as the official primary key for the table.

In our example, the combination of `first_name` and `last_name` serves as the primary key. This means that even if someone changes their favorite color, they remain the same person. 

One interesting facet about Data Warehouses is that they privilege data ingestion over uniqueness. In other words, they focus on getting data *into* tables more than they do preventing duplicates. If you read the Google BigQuery blog, you'll find that although you can specify record keys, [Google BigQuery does not enforce them](https://cloud.google.com/blog/products/data-analytics/join-optimizations-with-bigquery-primary-and-foreign-keys/#:~:text=The%20user%20must%20use%20the%20NOT%20ENFORCED%20qualifier%20when%20defining%20constraints%20as%20enforcement%20is%20not%20supported%20by%20BigQuery%20at%20this%20time.). This is clearly a violation of Hernandez's rule regarding keys. If we want to impose keys on our data warehouse tables, we'll have to do it ourselves.
### Removing Duplicates with BigQuery

#### Design

You can use a data pipeline to ensure that your table values will be unique. Let's make a simple data pipeline consisting of two tables and a query to transform data from one to the other:

> `raw_person` - > `person`

1. **`raw_person` Table**: This will serve as the raw table for all new records. It will explicitly allow duplicates for data ingestion speed.
2. **`person` Table**: This table is for data analysis, so only unique person records should be present.

#### Implementation

To implement this design, we need a SQL query that selects all meaningfully unique data from the `raw_person` table and inserts it into the `person` table. Here's a general framework for creating one table from another:

```sql
CREATE OR REPLACE TABLE `person` AS 
WITH person AS (
    -- Your query here
)
SELECT * FROM person;
```
Here is a brief explanation of the code block:
1. `CREATE OR REPLACE TABLE`: This statement creates a new table named `person` or replaces it if it already exists. If the table exists, the data and structure of the existing table will be replaced with the new data and structure defined in the subsequent part of the query. 
2. `WITH person as (...)` This is a Common Table Expression (CTE) named `person`, which is similar to the table we will be creating. It appears that you are selecting all the data from the `raw_person` table and using it in the CTE.

Now we need to remove the duplicates. You may have seen some SQL queries online that suggest using DISTINCT for deduplication. If you have, you might anticipate replacing the "..." with a statement like:
 ```sql 
SELECT DISTINCT * FROM `your_table`
```
That would 'work' if we have exact duplicates as in the first example. However, it would effectively mean that the *primary key* is just a unique combination of every column. We have already specified that unique combinations of `first_name` and `last_name` are our keys. Instead, we'll use a window function called `ROW_NUMBER`to calculate unique subjects of our table:

```sql
SELECT *, ROW_NUMBER() OVER(PARTITION BY raw_person.first_name, raw_person.last_name ORDER BY raw_person.date DESC) as row_num
FROM raw_person
```
Here's an explanation of the code block:
1. `ROW_NUMBER() OVER(...) as row_num`: This part of the code is creating a new column called `row_num`. This column will contain a unique number for each row in the result set.
2. `PARTITION BY raw_person.first_name, raw_person.last_name`: This part of the code tells the database to partition or group the rows based on the values in the `first_name` and `last_name` columns. In simpler terms, it groups rows with the same first and last names together.
3. `ORDER BY raw_person.date DESC`: This part of the code orders the rows within each partition (group) based on the `date` column in descending order. This means that the rows with the most recent dates will have lower row numbers within each partition.
   
We can now select all of the records where the `row_num` is equal to 1, and exclude `row_num` from the `person` table:

```sql
CREATE OR REPLACE TABLE `person` AS 
WITH person AS (
    SELECT *, ROW_NUMBER() OVER(PARTITION BY raw_person.first_name, raw_person.last_name ORDER BY raw_person.date DESC) as row_num
FROM raw_person
)
SELECT * EXCEPT(row_num) 
FROM person
WHERE row_num = 1
;
```

Now, we get a `person` table that is meaningful to us!

| id | first_name | last_name | favorite_color| date|
|----------|----------|----------|----------|----------|
| 1   | Joshua   | Trusty   | red   | 09-14-2023 |
| 2   | Misael   | Gonzelez | red   | 09-014-2023 |

Congratulations! You now know have a general background on what duplicates and keys are, and how to remove duplicates as part of a data pipeline. For more information on the concepts discussed, see the resources section. I have also included an "Extra Credit" section just below if you're looking to extend your skills even further. Thanks for reading. 

#### Extra Credit
Throughout the post, you may have noticed the `id` column/field in the tables. Normally, a field like that would be your primary key, and named something like 'guid' or 'uuid'. In our final `person` table, `id` *coincidentally* identifies unique instances of a person. If we had a third person named "Ethan Hunt" in the table with an id also of 1, this would not be true. Your mission, should you choose to accept it, is to replace `id` with a key based on unique combinations of `first_name` and `last_name` (the primary key). As a hint, try using [GENERATE_UUID](https://cloud.google.com/bigquery/docs/reference/standard-sql/utility-functions#:~:text=GENERATE_UUID,-GENERATE_UUID()&text=Returns%20a%20random%20universally%20unique,with%20RFC%204122%20section%204.4.).

### Resources
* [Database design for Mere Mortals](https://www.amazon.com/Database-Design-Mere-Mortals-Hands/dp/0201752840)
* [Extra Credit Solution](https://gist.github.com/nanoman657/c2d7db0210df198190e14ec60e49fe3d)
