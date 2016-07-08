---
layout: post
title:  "Warehousing at Uninstall.io"
date:   2016-07-07 12:47:00
categories: azuresqlwarehouse luigi
author: ripusingla
---

### Some Background

In early days of Uninstall.io, we were using MongoDB as our primary data store along with Redis (to power our dashboard) and 
MySql (Auth tables and few structured tables). Most of the insights would be served by dashboard and few custom report requests 
were entertained by python scripts running over Mongo.

Over time, we have more clients signing up every week than ever with specific custom reporting requests. As data and client requests 
were growing at fast speed and Mongo scripts were taking more and more time, we decided to move to a Warehousing solution.

### The Tech Stack

We decided to go ahead with Azure SQL warehouse. Rolling out an instance 
of warehouse from Azure portal is quite trivial. The challenge was to pump massive amount of existing and incoming json data from Mongo to SQL warehouse 
while keeping the process simple and resilient to failures/outages. Failures should be rare, but with as any large systems, they do happen 
from time to time. May be the server instance goes down, network partition, RAM overflow etc. In these cases we wanted the process to 
auto recover without any data corruption.

We decided on to using Luigi framework (from Spotify) for funneling the data to warehouse. It is simple to use and has nice web interface 
inbuilt.

![alt text](/assets/images/luigi2_cropped.jpg "Luigi Dashboard Snapshot")
![alt text](/assets/images/luigi_cropped.jpg "Luigi Dashboard Snapshot")


### 1. Process

We had approx 45MM user documents and 2 Billion custom events to start with and 20MM custom events adding with each passing day. 
Broadly, we devided our data in three categories -

  1. User/Device data, 
  2. Third party data and 
  3. Custom Events

For each cateogry, we run separate pipeline, Each pipeline has 3 tasks:

  1. Data Extraction from production - Pull, Clean, Transform
  2. Data upload to Warehouse Storage - flat files
  3. Load into respective sql tables - bcp

Data is pushed to separate sql tables for each client to optimize performance and few other reasons. Sql tables are created automatically as 
the data starts flowing to warehouse. Indexes are created using cronjobs. 
All the pipelines run 24x7 - sequentially for clients and parallely for data categories. So any piece of data is reflected in warehouse in few hours.

As some of the data in production keep mutating in place over time, to keep track of the mutations, we push data to 2 versions of sql tables - 
Current and History. Current tables have the most updated values and History tables records all the mutations in location, library versions, 
disk space etc.

Whole process runs on a single 4 core, 28GB RAM machine and is engineered to scale out on multiple nodes if needed. All pipelines are run
using supervisor and auto restarted in case of failure which brings us to our next goal.


### 2. Resilience

Recently we started planning for transition from Mongo to Couchbase cluster given its ease of scalability and high read/write throughput (separate 
blog post on that). We thought of leveraging couchbase for warehousing process as well. We maintain a meta data 
for each data transfer iteration in couchbase. 


{% highlight javascript %}
{
  u'pipeline': 'events',
  u'count': 1129545,
  u'db': u'xaf87434c0c994db8badf09feb813dd9351casdf',
  u'warehouse_id': 29148775,
  u'production_timestamp': u'2016-06-25 08:43:28',
  u'luigi_timestamp': u'2016-06-30-09-28-00'
}
{% endhighlight %}


Coming to failures, below are the possible point of failures:

  1. Data extraction step
  2. Upload Step
  3. Data loading step


Each pipeline first looks at the last piece of meta data available and starts from there. After completion of data extraction, a flat file is generated and
uploaded to azure blob storage outside the server disk. And new meta data is uploaded to couch. In case of failure in any of these steps, the pipeline will
safely and automatically start over from old meta data and system admin will receive email for each failure.

Data load task is written in such a way that the same flat file can be loaded multiple times into warehouse table without duplicating the outcome.
Data load step also asserts that the number of rows added to sql table matches with the count in meta data. Taking advantage
of these properties, on restart, each pipeline starts by loading the flat file corresponding to last meta data first. Data file is expected to be found 
either on local disk or Azure blob storage. In case file is missing, meta data entry is deleted and pipeline is restarted. So in case of failure in data load step,
simply restarting the pipeline is sufficient.


### 3. Nested Documents

Custom events in Mongo are nested inside user documents to save on document overhead and indexing costs, So for any user, one document 
will store 100 custom events and then next 100 custom events will be stored in next document (buckets). This makes lot of sense for production databases but creates
a challenge for pipelines. The pipeline has to remember timestamps at user level to know which custom events have been already shipped to warehouse.
Again we use couchbase for this user level meta data as our couchbase cluster setup provides sub millisecond latencies. However this step introduces a data
corruption opportunity in case of pipeline failure at data extraction step. To avoid this, we delay all writes to couch till upload of flat file succeeds, then write
this user level meta information to couch and finally update pipeline level meta data.


### Conclusion

We fully deployed this system to production in May 2016. All past and incoming data has been successfully transfered to warehouse without a single data
corruption issue in past 2 months. Recently we have developed our reporting infrastructure using Luigi (probably a separate blog post) on top of warehouse and 
deployed 100s of custom reports to run automatically for our clients. Now we have increased our efforts in deploying more
Machine Learning solutions to understand the data better and provide valuable insights to our clients.