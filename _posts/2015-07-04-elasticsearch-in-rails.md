---
layout: post
title: "ElasticSearch in Rails"
description: "The reason why I  selected <code>ElasticSearch</code> for a recent project and that the journey that got us there. </p> <p><strong>Spoiler alert </strong>it wasn’t my  first choice nor second, but such a huge relief when it was tried, and it performed splendidly."
date: 2015-06-04
read: "10 min read"
category: software
permalink: /blog/elasticsearch-in-rails/
---

## ElasticSearch in Rails

I have become an unlikely  fanboy of ElasticSearch.

ElasticSearch has been in my periphery for a  long time. I resisted learning it as it seemed that  Postgres database had  similar functionality. I had used Postgres for a number of years, and  I  couldn’t  justify the time  to learn another  platform.

Needless to say that I was wrong. What follows is my rationale in using ElasticSearch and how it solved a problem that couldn’t be easily solved using a  traditional database like Postgres.

### The Journey
I started working on a project that required fast full- text search over 100,000s of records. Great, I thought I can do that with Postgres!

However, the requirements kept on coming, with additional text fields to include and exclude, it wasn’t long before I found that it was  going to be to slow for the number of records.

> Necessity is the mother of invention
> ######an English-language proverb

I needed to consider my options, as  Postgres was simply not  fast enough.

### The Journey of discovery
The application is very similar to a library catalogue with a large number of records. These records need to be quickly searched against with the matching records being send to the web client, using the Json format.

### The initial issue
Initially a prototype was built to test the overall concept. This worked by sending all records to the web client. The web application would then filter these assets with the resulting matched records being displayed.

After the initial request, this was super responsive with a few hundred records. However, it was not going to scale when we increased the number of records.

The other problem was that users were usually only interested in a subset of the records but was burdened with **all** of the records.

###Attempt (1)
My first attempt was to move the records querying to server from the web application.

### The Server Steps
The steps that the server went through to select the records to send to the web client:

-  Identify the user
-  Identify those records the user can view
-  Search against each of these records for a matche
-  Paginate these matching records so that only  100 records are returned at any one time.

This sped up the initial request as I was only ever sending 100 records at a time. However the time taken to generate the JSON response for these 100 records was too slow.  The culprit was the time taken to identify the records and to generate the JSON response.  To identifying the records required a number of database queries and to generating the JSON response required additional queries to retrieve additional information.

###Attempt 2
The second solution considered was to cache the  request. The cache used the search terms and user's role as the identifying key.  This definitely sped up the request when it was in the cache, otherwise it was just as slow.

I also allowing free text searches so  it wasn’t possible to **pre-cache** all possible search combinations.  It was obvious that I needed a better solution.

###Attempt 3 - Enter Left ElasticSearch
Using ElasticSearch dramatically sped up the server response. It achieved this in two ways:

### 1) Record Selection
When documents are stored in ElasticSearch, the fields used in search are tokenized into an index. This works similarly to the index at the back of the book. In a similar way the list of words used are in the search fields with the corresponding records that they appear in.

This makes for fast searches as by simply looking up an index which then identifies the matching records.


###JSON Response
The records that are matched above are stored in a JSON format. They are combined together into a JSON document and sent to the client as the response to their query.

The JSON used for each record is determined from what was initially used in the creation of the record in ElasticSearch. In other words, what you **put in** you  **get out**.  This sped up the JSON response as I did not require  additional queries, for missing information, as this was pre-populated when the record was added to ElasticSearch.

The big advantage of ElasticSearch is that it does one thing really well - search. Indexing the document,  when the record is added to ElasticSearch,  gives it an edge when it come to  full text search.

## Elasticsearch Rails
There are a number of Ruby Gems to interact with Elasticsearch. If you are looking for a good option take a look at Elasticsearch Rails.  They provide a really helpful working example that can be used as a template for most projects.

### In summary
Consider using Elastic Search:

-  Full-text search functionality
-  JSON responses from your queries
-  Scale horizontal, allows the very large datasets
-  Speed

## Inspiration
-  <a href ="https://www.elastic.co/">elastic</a>
-  <a href= "https://github.com/elastic/elasticsearch-rails"> elasticsearch-rails</a>

