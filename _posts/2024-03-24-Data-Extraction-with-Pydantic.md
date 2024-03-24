---
layout: post
title: Data Extraction with Pydantic
cover-img: ["https://images.unsplash.com/photo-1501526029524-a8ea952b15be?q=80&w=2670&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D" : "Photo BY Hunter Harritt. Source: https://tinyurl.com/hh-photo License: Unsplash License"]
tags:
  - Tutorial
  - Pydantic
  - python
  - ELT
---

Extract Load Transform is one of the most popular types of data pipelines. While the term combines multiple concepts, today we’ll be zooming in on the first concept of Data Extraction. This tutorial offers a simple and scalable way to extract large volumes of data in real-time via #Pydantic .

### Introduction
Data extraction simply means pulling data from a source. As an analogy, if you were trying to make cookies, you need to 'extract' the necessary ingredients from the grocery store. In data engineering, extraction can be done by opening a file, reading from a sensor for real-time data, or connecting to an Application Programming Interface (API). The latter is what we'll be doing for this tutorial.

Suppose you have a website that needs to register new users. Each registration has data associated with it, such as name, address, email, et cetera. Users register on their own time, so you need to accommodate each sign-up in real-time.

### Data Extraction

For the data source, we'll use [Faker API](https://fakerapi.it/en) to get simulated Person data to extract. Querying the API can be done straightforwardly using Python's builtin libraries. Extracting and validating the data is quite simple when using the [Pydantic](https://github.com/pydantic/pydantic) library.  I recommend running the code on a [Google Colab](https://colab.google/) Jupyter notebook to easily segment the program and identify the outputs. The rest of the tutorial will reference such a notebook, but it isn't required to learn.


#### Reviewing Fake User Data

Before we start coding, let's take a look at the kinds of data we might get. Visit the [Faker API](https://fakerapi.it/api/v1/persons) page and copy all the text. Paste it into a [jsonformatter](https://jsonformatter.org/) and view it. While it will be randomized, the output will look something like the below. 

```json
{
  "status": "OK",
  "code": 200,
  "total": 10,
  "data": [
    {
      "id": 1,
      "firstname": "Walker",
      "lastname": "Kemmer",
      "email": "shad.bauch@gmail.com",
      "phone": "+8371180037431",
      "birthday": "1962-12-03",
      "gender": "male",
      "address": {
        "id": 0,
        "street": "98656 Bud Land Suite 654",
        "streetName": "Sherwood Street",
        "buildingNumber": "6665",
        "city": "Riceborough",
        "zipcode": "06823",
        "country": "Lao People's Democratic Republic",
        "county_code": "GQ",
        "latitude": 1.028953,
        "longitude": 114.201811
      },
      "website": "http://cremin.net",
      "image": "http://placeimg.com/640/480/people"
    }
```


#### Install Libraries

If you're using a Google Colab, copy and paste the below into a new notebook's first cell. This will install the Pydantic library we will need.
```
!pip install pydantic
```

#### Import Libraries
Let's begin by copying all of the imports we'll need later. You can just copy and paste these inputs into the next cell.  
```python
from datetime import datetime

import requests
from pydantic import BaseModel, Field
```

#### Accessing The API
In a new cell,  set a variable for the API url. Then collect the raw data from the url and save it to a variable. View the data to make sure it came in correctly. Your code will look like the below.

```python
# Define Faker instance and API endpoint
url = "https://fakerapi.it/api/v1/persons"

# Generate and collect fake user data
response = requests.get(url)
data = response.json()
```

#### Parsing Raw Data

 Part of data engineering is ensuring that data is reliable. When you view the data, you'll see that it truly is raw. It's just a string. Since it hasn't been processed yet, the data is *unvalidated*. Whereas you might expect that birthdates are going to be correct, user errors can abound. What if someone submits simply '2000' for their birthday? Remember, we're just extracting the data here. We do not have control over how the data is generated. Fortunately, we can validate that the data is correct with Pydantic.

In a new cell, let's define an object to contain our user data. The ':' will type a variable, and the `Field` function will declare that property to be a object field. Python's types are generally optional, but they take on a crucial role for Pydantic. Ensure that each property is typed according to your expectations about the kind of data you're extracting. For example, `firstname` should be a string, and a birthday should be a `date`. The example code below selects the properties that I like from the Faker API, but you can extract more.

```python
class FakeUser(BaseModel):
    id: int = Field()
    firstname: str = Field()
    lastname: str = Field()
    email: str = Field()
    phone: str = Field()
    birthday: datetime.date = Field()
    gender: str = Field()
```

The `FakeUser` object inherits a method from `BaseModel` that allows us to parse the raw data. In a new cell,  run the below code in a new cell to parse all of the raw data.
```python
fake_users = [FakeUser.parse_raw(user) for user in data]
```

In your final cell, write the below code. The `fake_user` variable is an object that is aware of the types of data it contains. You can easily access the data by name, thanks to your `FakeUser` model. Congratulations! You have now successfully performed data extraction.

```python
fake_user = fake_users[0]
fake_user.birthday
```

### Conclusion
Data Extraction can be very simple, and it lays the ground work for data quality and the rest of an ELT pipeline. In this example we just connected to our source, pulled the data out, and validated it according to our preferred structure. Validation is where we detect if something is wrong with the data, and possibly discard bad data. Try making your own raw data with invalid values and discard invalid FakeUsers as you form your list of users. In the next article, we'll discuss how to load the data into a #DataWarehouse in preparation for future analysis.
