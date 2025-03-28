---
layout: post
title: Intelligent Querying with Generative AI
cover-img:
  - https://upload.wikimedia.org/wikipedia/commons/d/d2/Fxemoji_u2728.svg: "Image BY Mozilla. Source: https://github.com/mozilla/fxemoji License: CC BY 4.0"
tags:
  - Tutorial
  - Coming...Soon
---
I recently gave a talk called Data Transformation - Intelligent Querying with GenAI. This is a placeholder entry until I fully polish the step-by-step tutorial in the blog. Until then, you can experiment with the workshop's official [Google Colaboratory notebook](https://colab.research.google.com/drive/1B2FSXRtonrC2zmAhe5vfc9Kfpcmp_GUe?usp=drive_link) to explore making your own GenAI-powered queries. It contains everything you'll need to run the code (sans your Google API key). 

If you enjoyed the workshop or any of the material on my blog, please consider supporting me by giving me a [LinkedIn](https://www.linkedin.com/in/jatrusty/) recommendation or endorsement. Stay tuned for future content, both digital and in-person.

Generative AI allows you to generate text with simple prompt. This is great for general purpose  inquiries, but what if you want to generate text that is highly context dependent? What if you want to solve problems that are beyond your current domain knowledge? In this tutorial, I'll discuss how you can code dynamic prompts to create SQL code that incorporates domain knowledge.

### Use Case
Suppose you are working as a data analyst at a hospital. The medical field has a system called the Ambulatory Payment Classification. It allows hospitals to classify the various reasons people seek medical care, and determine the right price to charge patients. Suppose as an analyst, you want to determine where in the country it would be cheapest to treat some minor arm scrapes (the kind you might get from a skateboarding injury). The problem is that there are thousands of APCs, and determining the correct APC for an injury can be very challenging for someone without the right background. This tutorial will showcase how you can dynamically generate a SQL query to produce this data using Gemini function calling.

## Tutorial

### Getting Started
To get started, you'll want a Google AI Studio key. You can get one for free by registering here. Google provides a generous quantity of tokens per day for use, but this tutorial won't use a large quantity of your quota anyway. You'll also need an environment to run code. Google Colab is very convenient, and highly recommended for this tutorial.

### Installations and Imports
Now that we're on to writing code, install the GenAI Python library. It will contain all the functionality needed for the tutorial. Everything else can be found in python's built-in libraries.

### Writing the Instructions 
Instructions are effectively a prompt that is given to the Large Language Model (Gemini) before the user's prompt is submitted. It helps ensure that the LLM knows exactly how to treat the prompt. The right instructions will ensure that the output produced will be useful to us. Unlike many prompts you may be familiar with, this prompt must be highly structured to give a consistent output. As outlined in the microsoft link, there are three key components of the prompt:
- first component
- second
- third

Structure alone is not enough to grant us success. WE also need to ensure that the prompt is aware of all the APCs that could be present in the database. In the code snippet, two have been listed for conciseness. The APCs listed are in the exact format present in the SQL table columns. An APC like "0012" alone will not select the right data. Additionally, it is important to note that the APCs of the real world can change over time, whereas the tutorial data is static. 0012 is no longer used, but serves as an easy example for us here. We are injecting the known APCs into the prompt to ensure that it is aware of the classes we are considering.

### Classification Function
This function will perform the bulk of the classification work, and ensure that each prompt gets matched to a valid APC. The typing literal we created for our prompt is used here as well.  Gemini will not *directly* call the function, but it is aware of the function's existence, and will do its best to ensure that the process calling the function gives it a valid input. Even though Gemini will "do its best", it doesn't always succeed. The Google Colab tutorial notebook has been fine-tuned over hours of work to ensure a >90% success rate, though your results may vary. If you run into a 50x error, that is Gemini saying that it had a problem running the function. It shows up as a server error, but that doesn't mean you are off the hook. If you programmed the code  with a bug, it is still a problem that requires your intervention, rather than waiting for Google to fix. For an overview of Test Driven Development and how that can support your ability to reduce code, see my forthcoming article here.

### Function Calling Setup
Function calling can be as complex or intuitive as you want. Gemini can identify the functions it needs to call based on your request. In our case, we will select that option so Gemini will always make use of our function. If you have a lot of different functions, this could lead to the wrong functions being called without a well-structured prompt. The ____ option is good in those cases, and you can explore it on your own.

The rest of the code is fairly boilerplate. For more details on how it works, I recommend perusing the Google documentation on it here.

### Prompting Gemini
Now we are on to the fun part - asking Gemini a question! The prompt will use will be "". Gemini's output will be inserted into a SQL template designed to give us the average cost. This template can be made even further dynamic, but we'll keep it simple for the tutorial. The APC Gemini determines is appropriate will be injected in to the SQL template. That SQL code is then run in BQ, and printed out to the screen. There you have it! We have a menas of creating a dynamic SQL query beyond our domain knowledge!
