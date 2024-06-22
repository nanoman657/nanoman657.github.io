---
layout: post
title: Data Transformation with BigQuery
cover-img:
  - https://images.unsplash.com/photo-1618172193763-c511deb635ca?q=80&w=2564&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D: "Photo BY Milad Fakurian. Source: https://tinyurl.com/mf-photo1 License: Unsplash License"
tags:
  - Tutorial
  - ELT
  - BigQuery
  - SQL
---
Data Transformation is one of the most important operations in Data Engineering. Just like mixing and baking the raw materials of flour, butter, and sugar provides delicious shortbread, transforming data generates new *business value*. This tutorial shows how data can be joined and aggregated in #SQL to generate new insights.

### Introduction
Data Transformation is the last component of an ELT, and potentially the most involved stage for Data Engineering. With the first two stages, there's a straightforward technical goal: get the data out of its source and into its destination. The transformations you design will depend on the *business use cases*, so there is a wide variety of possible transformations you could perform. Data must often be combined with other data, aggregated, and subjected to business logic to extract as much value as possible. This tutorial will take place entirely within [BigQuery](https://cloud.google.com/bigquery?hl=en) (BQ), part of Google Cloud Platform (GCP). At the time of writing, you can get [10 GB of storage and 1 TB of querying per month](https://cloud.google.com/free?hl=en) for free. Before starting the tutorial, make sure you have your BigQuery account set-up. You can also review the previous articles to see how we got the data into BQ, or just skip to the [latest notebook](https://colab.research.google.com/drive/1DwdkM9fkQfSoB6-VfD6D_YrDMRX_-Kip?usp=sharing).

#### Real-world Scenario
Suppose you have a website that has registered users. You have already extracted and ingested the user data into BigQuery. Now, you want to connect the user data to healthcare data to understand healthcare availability.

### Data Transformation

We will use the Google SQL dialect on BigQuery to perform our transformations. Performing the transformations will be done in 3 stages:
1. Extracting the zip code.
2. Connect their zip code to outpatient charge data.
	1. `bigquery-public-data.cms_medicare.outpatient_charges_2015`
3. Calculate the quantity of nearby healthcare options.



#### Setup
 Use the mini-series' [latest notebook](https://colab.research.google.com/drive/1DwdkM9fkQfSoB6-VfD6D_YrDMRX_-Kip?usp=sharing) in the series as a baseline for the code you write. No further setup is needed. While the mini-series has previously used random data in to the database, this tutorial will use a pre-made table to ensure deterministic results. It is publicly readable.

**BigQuery table**: `tutorial-398619.tutorial.user`

### Selecting User Data
Let's begin right where we left off with data transformation. We'll convert the sql query to one that uses a [Common Table Expression](https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax#cte_name).  Whereas the original `sql_code` variable was programmatically defined, we'll use simple text to make our query. Within the CTE, exclude `address`, and include `zipcode`.

```sql
With user as (
  SELECT * EXCEPT(address), address.zipcode FROM `tutorial-398619.tutorial.user`
)
SELECT * from user
```

The CTE will act as shorthand for complex code, and allow us to create send the output to a single table. It'll also allow us to program in SQL in a *slightly* more similar fashion to object-oriented programming. Now, we want to connect (join) user data to the medical data. 

### Connecting User and Healthcare Data
Here's a sample of the data we want to join.

**Table: outpatient_charges_2015**

| **provider_id** | **provider_name**                | **provider_street_address** | **provider_city** | **provider_state** | **provider_zipcode** | **apc**                                         | **hospital_referral_region** | **outpatient_services** | **average_estimated_submitted_charges** | **average_total_payments** |
| --------------- | -------------------------------- | --------------------------- | ----------------- | ------------------ | -------------------- | ----------------------------------------------- | ---------------------------- | ----------------------- | --------------------------------------- | -------------------------- |
| 020001          | Providence Alaska Medical Center | Box 196604                  | Anchorage         | AK                 | 99508                | 0012 - Level I Debridement & Destruction        | AK - Anchorage               | 105                     | 288.94342857                            | 113.67761905               |
| 020001          | Providence Alaska Medical Center | Box 196604                  | Anchorage         | AK                 | 99508                | 0015 - Level II Debridement & Destruction       | AK - Anchorage               | 430                     | 576.48853488                            | 166.18195349               |
| 020001          | Providence Alaska Medical Center | Box 196604                  | Anchorage         | AK                 | 99508                | 0020 - Level II Excision/ Biopsy                | AK - Anchorage               | 19                      | 5410.0194737                            | 962.93                     |
| 020001          | Providence Alaska Medical Center | Box 196604                  | Anchorage         | AK                 | 99508                | 0096 - Level II Noninvasive Physiologic Studies | AK - Anchorage               | 19                      | 2350.4873684                            | 368.05105263               |

The `provider_zipcode` is of particular interest here, because we can use it to identify healthcare providers in the same zipcode as our users. Let's update our SQL query to include another CTE with the healthcare providers and SQL code. We'll have to cast one of the zipcodes to a STRING.

```sql
WITH user as (
  SELECT * EXCEPT(address), address.zipcode FROM `tutorial-398619.tutorial.user`
),
outpatient_charge as (
	SELECT * FROM `bigquery-public-data.cms_medicare.outpatient_charges_2015`
),
user_provider as (
  SELECT * from user JOIN outpatient_charge on user.zipcode = CAST(outpatient_charge.provider_zipcode as STRING)  
)
SELECT * from user_provider
```

### Counting Healthcare Options
Data transformation often requires *aggregation* of data. A simple form of this is counting. Let's conclude this exercise by counting how many healthcare options are available to our users.
#### Query
```sql
WITH user as (
  SELECT * EXCEPT(address), address.zipcode FROM `tutorial-398619.tutorial.user`
),
outpatient_charge as (
	SELECT * FROM `bigquery-public-data.cms_medicare.outpatient_charges_2015`
),
user_provider as (
  SELECT * from user JOIN outpatient_charge on user.zipcode = CAST(outpatient_charge.provider_zipcode as STRING)  
)
SELECT 
  CONCAT(firstname, " ", lastname) as name, 
  count(*) as nearby_healthcare_providers 
FROM user_provider 
GROUP BY name 
ORDER BY nearby_healthcare_providers DESC
```


### Conclusion
Just like transforming raw ingredients creates delicious food, data transformation unlocks valuable insights from raw data. This tutorial demonstrated how to join and aggregate data in BigQuery using SQL to uncover healthcare options near users. You can go beyond this simple analysis to understand how healthcare costs vary by location and service. By understanding how to transform data, you can extract valuable business insights that can inform better decision-making. If you've enjoyed this free ELT introduction series, please consider supporting me by giving me a [LinkedIn](https://www.linkedin.com/in/jatrusty/) recommendation or endorsement.
