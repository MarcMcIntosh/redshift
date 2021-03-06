NBS Data Warehousing (via Redshift)
============

This repository contains example queries and usage info for Next Big Sound data warehouses hosted within the Amazon Redshift service.

What is [Redshift](http://aws.amazon.com/redshift/) exactly?

"Amazon Redshift is a fast, fully managed, petabyte-scale data warehouse solution that makes it simple and cost-effective to efficiently analyze all your data using your existing business intelligence tools."

At NBS, we've extracted large chunks of our dataset and made those available to our customers (at no charge) as a way to provide nearly instant access to answers for questions our other products can't support.


## Getting Started

First, if you're interested in using our data warehouses you'll have to contact us so that we can get one setup for you.

We will need the following:
  1. IP addresses or subnets from hosts needing access
  2. (Optional) Public ssh keys for users wanting access

Once we've approved the request and finished building the warehouse, you'll have acess to it one of two ways:
  1. [Shell Access](http://docs.aws.amazon.com/redshift/latest/mgmt/connecting-from-psql.html) - We've taken the liberty of installing __psql__ on an Amazon EC2 instance that resides in the highly isolated [Virtual Private Cloud](http://aws.amazon.com/vpc/) that also contains your data warehouse.  Using this client is as easy as ssh'ing to the instance, provided you've given us a public key, and then running the __psql__ client.

Overall, the connection process generally looks like this:
```
# The ip here will be provided by NBS
my-computer> ssh ubuntu@ip-of-public-instance

# The following cluster_leader_host_name, db_name, user_name, and user_password will also be provided by NBS
ubuntu@ip-of-public-instance> psql -h cluster_leader_host_name -U user_name -d db_name -p 5439
```

  2. [Programmatic Access](http://docs.aws.amazon.com/redshift/latest/mgmt/connecting-in-code.html) - Redshift is a fully ODBC and JDBC client data store so it may be accessed through any programming language.

## Data Model

Generally, there will be at least 3 tables provided in your warehouse containing the following:

####Table 'entity_data'
    Column    |       Type       | Description
--------------|------------------|-------------
 entity_name  | varchar(1024)    | NBS artist/entity name
 *entity_id*    | integer    | NBS artist/entity identifier

This table contains over 1 million different entities currently being tracked by NBS

####Table 'metric_data'

    Column    |       Type       | Description
--------------|------------------|-------------
 network_name  | varchar(128)    | Name of network associated with metric (e.g. 'Facebook')
 metric_name  | varchar(128)    | Name of metric (e.g. 'Page Likes')
 *metric_id*    | integer    | Metric integer identifier (same as timeseries\_data.metric\_id)

 There are over 200 metrics available spanning data sources such as:
  - _Social_ - Facebook, Instagram, Twitter, YouTube, etc.
  - _Web_ - Google Analytics, Sitecatalyst
  - _Sales_ - iTunes, Amazon, etc.
  - _Radio_ - Mediabase (terrestrial), Spotify, Rdio, etc.

Here is a random sample:

 ```
snapshot20140818=# select * from metric_data order by random() limit 10;
     network_name     |     metric_name     | metric_id
----------------------+---------------------+-----------
 WeeklyBookSalesUnits | Barnes & Noble      |       323
 iTunes               | Video Units (Net)   |       137
 Mediabase            | Pop Audience        |       116
 Direct POS           | Total Units         |       179
 Last.fm              | Listeners           |         9
 Facebook Insights    | Consumptions        |       193
 Wikipedia            | Pageviews           |        41
 YouTube Analytics    | Viewer Percentage   |       274
 iTunes               | Track Units (Promo) |       338
 BarnesDailySales     | All Sales Units     |       304
```


####Table 'timeseries_data'

    Column    |       Type       | Description
--------------|------------------|-------------
 *entity_id*    | integer          | NBS artist/entity identifier
 endpoint_id  | integer          | Identifier for network to NBS artist relationship (only helpful for support)
 metric_id    | integer          | Integer identifier associated with metric names (e.g. 'Facebook Page Likes')
 count_type   | character(1)     | One of either 'd' (for delta) or 't' (for total) indicating whether or not this value was accrued over 1 day or if it is a cumulative, lifetime sum
 is_private   | boolean          | Flag indicating whether or not this data is public or if it was provided through some proprietary agreement
 unix_seconds | bigint           | Number of seconds since 1970-01-01 used to encode date associated with this metric value
 value        | double precision | Metric value

This table contains over 2 billion data points and consists of all timeseries data tracked by NBS between Jan. 1st, 2014 and Aug. 15th, 2014.

## Example Operations

Once connected to a psql shell, here are some common operations that can be done:

- Showing the tables available:
```
snapshot20140818=# \dt
              List of relations
  schema   |      name       | type  | owner
-----------+-----------------+-------+-------
 pg_temp_6 | dow             | table | sony
 public    | entity_data     | table | sony
 public    | metric_data     | table | sony
 public    | timeseries_data | table | sony
(4 rows)
```

- Showing the fields within a single table:
```
snapshot20140818=# \d+ timeseries_data
                           Table "public.timeseries_data"
    Column    |       Type       | Modifiers | Storage  | Stats target | Description
--------------+------------------+-----------+----------+--------------+-------------
 entity_id    | integer          |           | plain    |              |
 endpoint_id  | integer          |           | plain    |              |
 metric_id    | integer          |           | plain    |              |
 count_type   | character(1)     |           | extended |              |
 is_private   | boolean          |           | plain    |              |
 unix_seconds | bigint           |           | plain    |              |
 value        | double precision |           | plain    |              |
Has OIDs: yes
```

- Selecting data for a single artist -- in this case the query is for Charli XCX's SoundCloud play numbers over some recent week:

```sql
SELECT
TIMESTAMP WITH Time Zone 'epoch' + unix_seconds * INTERVAL '1 second' date,
value
FROM timeseries_data
WHERE entity_id = (
    SELECT entity_id FROM entity_data
    WHERE entity_name = 'Charli XCX'
)
AND metric_id = (
    SELECT metric_id FROM metric_data
    WHERE network_name = 'SoundCloud'
    AND metric_name = 'Plays'
)
AND count_type = 'd'
ORDER BY unix_seconds DESC
LIMIT 7

        date         | value
---------------------+-------
 2014-08-15 00:00:00 | 19762
 2014-08-14 00:00:00 | 20432
 2014-08-13 00:00:00 | 20110
 2014-08-12 00:00:00 | 20125
 2014-08-11 00:00:00 | 19814
 2014-08-10 00:00:00 | 18849
 2014-08-09 00:00:00 | 19110
```

- Selecting the 10 artists with the most Twitter followers so far in 2014

```sql
SELECT
B.entity_id,                      -- NBS id for artist
B.entity_name,                    -- NBS artist name
round(A.value) AS total_followers -- Total followers gained between 2014-06-01 and 2014-06-30
FROM (
    SELECT entity_id, sum(value) AS value
    FROM timeseries_data

    WHERE metric_id = (
        -- Select metric id for Twitter Followers
        SELECT metric_id FROM metric_data
        WHERE network_name = 'Twitter' AND metric_name = 'Followers'
    )
    GROUP BY entity_id
    ORDER BY sum(value) DESC
    LIMIT 10
) A
INNER JOIN entity_data B
ON A.entity_id = B.entity_id
ORDER BY A.value DESC;

 entity_id |    entity_name    | total_followers
-----------+-------------------+-----------------
       652 | Katy Perry        |     11888000556
     13455 | Justin Bieber     |     11600234088
     25167 | Barack Obama      |      9693752032
      8859 | YouTube           |      9386136472
       659 | Lady Gaga         |      9378098318
       143 | Taylor Swift      |      9164097823
      5774 | Britney Spears    |      8370927576
      2104 | Rihanna           |      7961606092
    339704 | Instagram         |      7307233594
       451 | Justin Timberlake |      7219574293
(10 rows)
```


There are many, many different types of queries possible beyond just those above, and the [Query Examples](example_queries) in this repository should help to give a sense of some other possibilities.


## Getting Help

The SQL dialect used by Redshift is derived from PostgreSQL and is [fully documented here](http://docs.aws.amazon.com/redshift/latest/dg/c_redshift-and-postgres-sql.html).  In our experience though, very little DML syntax that works for Postgres doesn't work with redshift so looking for help in the [PostgreSQL Documentation](http://www.postgresql.org/docs/9.3/interactive/tutorial-sql.html) is generally just as effective.


