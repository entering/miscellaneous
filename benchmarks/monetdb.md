MonetDB - column-store pioneers
=============================

<img align="right" src="http://dev.monetdb.org/imgs/monetdb-final-500.png" alt="MonetDB Logo" width="125" height="51"></img>

Introduction
------------

On end of 2012 I was looking for a column oriented DB and I found [MonetDB the column-store pioneer](http://www.monetdb.org/Home) at the time I was evaluating several tecnhologies, some just reading anothers like InfiniDB benchmarking. So I compare InfiniDB against MonetDB, **but I will just post MonetDB results** (that overpass InfiniDB hugely).

MonetDB is not well known, I never heard about it, it is not a real commercial product but more an academic project, a great project.

Environment
------------

I used AWS Cloud, you can have a look on more detail about instance types in [here](http://aws.amazon.com/ec2/instance-types/).

<table>
  <tr>
    <th>Code</th>
    <th>Name</th>
    <th>API name</th>
    <th>CPU</th>
    <th>Memory GB</th>
    <th>I/O Performance</th>
  </tr>
  <tr>
    <td>S1</td>
    <td>M1 Extra Large Instance</td>
    <td>m1.xlarge</td>
    <td>8 EC2 Compute Units (4 virtual cores with 2 EC2 Compute Units each)</td>
    <td>15</td>
    <td>High</td>
  </tr>
  <tr>
    <td>S2</td>
    <td>High-Memory Double Extra Large Instance</td>
    <td>m2.2xlarge</td>
    <td>13 EC2 Compute Units (4 virtual cores with 3.25 EC2 Compute Units each)</td>
    <td>34.2</td>
    <td>High</td>    
  </tr>
  <tr>
    <td>S3</td>
    <td>High I/O Quadruple Extra Large Instance</td>
    <td>hi1.4xlarge</td>
    <td>35 EC2 Compute Units (16 virtual cores)</td>
    <td>60.5</td>
    <td>Very High</td>
  </tr>  
</table>

**S1 and S2 have 4 volumes EBS raid 0 and S3 have the 2 SSD raid 0.**

Data
------------

Table with 33 columns:
- with 20 columns as integers
- 8 as varchar
- 4 float
- 1 datetime

Rows: 640M

Data in MySQL:
- average row length of 203 bytes (usingshow table status)
- size: ~121GB

MonetDB: 77GB (due to compression)

Queries
------------

Q1)
```sql
SELECT count(*)
FROM summary
WHERE gender = 'M'; -- Gender have 3 options M F N (unknow)
```

Q29
```sql
-- age have low cardinality 7 segments
SELECT count(*)
FROM summary
WHERE age = 'C' AND sent > 100;
```

Q3)
```sql
-- provider have low cardinality, and is a small string, like hotmail, gmail, etc.
SELECT provider, count(*) as cnt
FROM summary
GROUP BY provider
ORDER BY cnt desc;
```

Q4)
```sql
SELECT parent_id, gender, sum(clicks) as tc
FROM summary
GROUP BY parent_id, gender
ORDER BT tc desc limit 10;
```

Results
------------

I register a cold time, run the query first time, after import data and restart MonetDB. And I register the time after warm up, usually 4-5 queries was enough to get a stable time. I restarted server and run again to verify the values.

**Instance S1**

<table>
  <tr>
    <th>Query</th>
    <th>Cold time (s)</th>
    <th>After warm up (s)</th>
  </tr>
  <tr>
    <td>Q1</td>
    <td>84</td>
    <td>32.1</td>
  </tr>
  <tr>
    <td>Q2</td>
    <td>121</td>
    <td>33.9</td>
  </tr>
  <tr>
    <td>Q3</td>
    <td>15.4</td>
    <td>10.0</td>
  </tr>
  <tr>
    <td>Q4</td>
    <td>77</td>
    <td>61</td>
  </tr> 
</table>

**Instance S2**

I was short on time and just run Q1 and Q2 on this instance. The times after warm up are better, the instance have double of memory so is expected result.

<table>
  <tr>
    <th>Query</th>
    <th>Cold time (s)</th>
    <th>After warm up (s)</th>
  </tr>
  <tr>
    <td>Q1</td>
    <td>95</td>
    <td>20.7</td>
  </tr>
  <tr>
    <td>Q2</td>
    <td>49.5</td>
    <td>26.4</td>
  </tr>
</table>

**Instance S3 (SSD)**

Improves are huge, and cold time is much closer to warm up time then before, because the 2 SSD with raid 0 have greater performance.

<table>
  <tr>
    <th>Query</th>
    <th>Cold time (s)</th>
    <th>After warm up (s)</th>
  </tr>
  <tr>
    <td>Q1</td>
    <td>21.3</td>
    <td>15.5</td>
  </tr>
  <tr>
    <td>Q2</td>
    <td>30.5</td>
    <td>21.2</td>
  </tr>
  <tr>
    <td>Q3</td>
    <td>4.2</td>
    <td>4.9</td>
  </tr>
  <tr>
    <td>Q4</td>
    <td>11.6</td>
    <td>10.1</td>
  </tr> 
</table>

Lack of popularity
------------

When I found MonetDB was so difficult to get any information on Web about it, almost no one using it (at least writting about it). I found [Analyzing air traffic performance with InfoBright and MonetDB](http://www.mysqlperformanceblog.com/2009/10/02/analyzing-air-traffic-performance-with-infobright-and-monetdb/) and [Quick comparison of MyISAM, Infobright, and MonetDB](http://www.mysqlperformanceblog.com/2009/09/29/quick-comparison-of-myisam-infobright-and-monetdb/) both from 2009. MonetDB team would win getting more involved with community, like: move code and documentation to Github and have a technical blog about it.

Conclusions
------------

MonetDB have great performance, it scales well vertically, more hardware faster queries. Is simple to install and I didn't need to make any configuration, you don't have to careful choose indexes and so on. It excells in queries with group bys like Q3 and Q4, I could see it using all the cores of S3 (16 cores).

Don't have any built-in shard neither replication (master - slave would be nice), documentation is poor but mailing list is quite active. It have all the disadvantages inherent of column oriented design, eg: single inserts and updates are expensive, don't handle well high concurrency.

The client is important too, I justed test the PHP client and it disappointed me (performance and code), I made some tests bringing a lot of rows, I run the PHP code in the same server, and running it on MonetDb CLI take almost half the time of using PHP client. I found that MonetDb had a client as module of PHP with good performance, but was deprecated becase was difficult to maintain. If you plan to use MonetDb first check if the client performance suit your needs, if you are using MonetDb for reports probably will not be a problem (result set probably will be small because will be aggregated before).

I like MonetDB, I would like to see it as a commercial product rather than an academic project, but kept as open source, with a community version and an enterprise edition with some more features on top of it.
But even the community version needs better documentation and better clients. MonetDB as a SaaS would be great too.

[![githalytics.com alpha](https://cruel-carlota.pagodabox.com/6ca5e52989fdd36e216cbbe6c0685408 "githalytics.com")](http://githalytics.com/entering/miscellaneous)
