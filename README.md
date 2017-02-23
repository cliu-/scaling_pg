# Scaling PostgreSQL Database

___PostgreSQL___ is one of the most popular database in SaaS world. As other relational databases, PostgreSQL is designed to run on a single server in order to maintain the integrity of the dataset and support the [ACID](https://en.wikipedia.org/wiki/ACID) properties. The lack of scalability and elasticity for PostgreSQL (or other relational databases) would be a huge barrier to your SaaS.

This  artical will show you how to scale PostgreSQL database for multi-tenant SaaS applications.

## Questions Before Starting

Before scaling, you should deeply understand the requirements from database point of view. You can try to answer the questions:

- What is the expected RPS for my database?
- What is the actual RPS for the production?
- The workload of my database is read-heavy or write-heavy?
- Did you try to tune your database to obtain better performance?
- Did you try to use cache to reduce the RPS on database? 

After answering the questions, if you still think scaling database is the only way to improve peformance, let's start it.

## Vertical Scaling

Vertical scaling, as known as ``scaling up``, is a way to scaling database with adding more hardware resources for database server. It is pretty easy to operate, and there is no changes in data modeling and applications source code.

For PostgreSQL, there is one thing you should be aware of is that some pg server configurations (such as shared_buffer, max_connections, etc.) should be changed as the hardware resources (CPU, memory) increments.

## Horizontal Scaling

There are two strategies to scale out a PostgreSQL database, one is with native streaming replication, the other is with sharding.

#### Streaming Replication

The feature [streaming replication](https://www.postgresql.org/docs/9.5/static/warm-standby.html#STREAMING-REPLICATION "streaming replication") is introduced since Postgresql 9.0, which allows us to create a cluster with one or more hot standby servers (replicas) which is able to run read-only queries.

With help of a postgres queries load balancer, we can easily distribute ```SELECT``` queries among available servers, improve the system's overall throughput, reduce the load on each PostgreSQL server. But writing queries will be always sent to primary server. 

Load balancing works best in a scenario where there are many read-only queries happening at the same time.

![postgresql-ha-topology1](scaling_pg/postgresql-ha-topology1.jpg)

 If you need a ```SELECT-intensive``` database, you can add an extra layer of ```pgpool``` to your postgresql deployment as the queries load balancer, and simply add some hot standby servers into your postgresql cluster. 

To apply this scaling strategy, you do not need to change data model in current PostgreSQL database, or change source code in applications.

#### Sharding

> A database shard is a horizontal partition of data in a database or search engine. Each individual partition is referred to as a __shard__ or __database shard__. Each shard is held on a separate database server instance, to spread load. [wikipedia][1]

Sharding provides an essential horizontal scalability for relational database, and is becoming one of the most important consideration in database design.

PostgreSQL does not provide built-in tool for sharding. However, nowadays, you can find many 3rd-party sharding tools for PostgreSQL from internet. Today, we will use [_Citus_](https://www.citusdata.com/) which extends PostgreSQL capability to do sharding and replication.

##### Citus Architecture

![citus-basic-arch](scaling_pg/citus-basic-arch.png)

At a high level, Citus distributes the data across a cluster of commodity servers. Incoming SQL queries are then parallel processed across these servers.

##### Our Multi-Tenant Data

Let's look at a very basic SaaS schema:

```SQL
CREATE TABLE tenants
(
  id character varying(50) NOT NULL,
  name character varying(255) NOT NULL,
  CONSTRAINT tenants_pkey PRIMARY KEY (id)
);

CREATE TABLE acl
(
  id character varying(50) NOT NULL,
  tenantid character varying(50) NOT NULL,
  name character varying(256),
  owner character varying(50),
  CONSTRAINT acl_pkey PRIMARY KEY (id),
  CONSTRAINT acl_name_key UNIQUE (name)
);

CREATE TABLE item
(
  id character varying(50) NOT NULL,
  tenantid character varying(50) NOT NULL,
  name character varying(1024),
  acl character varying(50),
  CONSTRAINT item_pkey PRIMARY KEY (id),
  CONSTRAINT item_acl_fkey FOREIGN KEY (acl) REFERENCES acl (id),
);
```

The above schema shows a simplified multi-tenant website. Every tenant has some ```acl```s and many ```item```s, each ```item``` is with reference to one ```acl```.

Now, let's say we have to handle more than 500 writing-intensive queries per second on this database. We tried to [scale up](#sharding) PostgreSQL server, and tune SQL queries and PostgreSQL configuration, but still can not acheive the target. We have to scale out the PostgreSQL database with sharding strategy.

Because of multi-tenant model, we can know that most of queries to postgres are merely for a particular tenant; queries joining between two or more tenants would seldom happen. 

The easiest level to do sharding at is the tenant level. The largetst tables over time are likely to be ```acl``` and ```item```, we could shard on both of them. The difficulty lies in the fact that we want to make tuples with the same ```tenantid``` in these two tables could be put into the same shard, so that we will not lose some database power including joins, indexing and more.

> Citus' **co-location** is the practice of dividing data tactically, where one keeps related information on the same machines to enable efficient relational opertions, but takes advantage of the horizontal scalability for the whole dataset. The principle of data co-location is that all tables in the database have a common distribution column and are sharded across machines in the same way, such that rows with the same distribution column value are always on the same machine, even across different tables. As long as the distribution column provides a meaningful grouping of data, relational operations can be performed within the groups. [citus co-location](https://docs.citusdata.com/en/v6.1/sharding/colocation.html#colocation)

The key that makes co-location happen is including the ```tenantid``` on all tables. By doing this we can easily shard out all our data so it's located on the same shard. In the above data model we coincidentally had ```tenantid``` on all the tables. 

Before sharding, our data for each table looks like below.

![multi-tenant dataset before sharding](scaling_pg/multi-tenant dataset before sharding.jpg)

Now let's try sharding our tenant by executing below [functions](https://docs.citusdata.com/en/v6.1/reference/user_defined_functions.html). 

```sql
SELECT master_create_distributed_table('tenants', 'id', 'hash');
SELECT master_create_distributed_table('acl', 'tenantid', 'hash');
SELECT master_create_distributed_table('item', 'tenantid', 'hash');
```

Citus will divide each of three tables for us, and re-group them based on ``tenantid``. The layout of our data should looks like below after sharding.

![multi-tenant dataset after sharding](scaling_pg/multi-tenant dataset after sharding.jpg)

SQL query is started from client side, and sent to the Citus master node initially. The master node parses the SQL statement and finds out which tenant it's querying and which worker the query should be forwarded to. The master does not deal with the SQL statements, but forward them different workers.

Return to our case, we need to deal with more than 500 write-intensive queries per second on our postgres database. If we have a Citus cluster with some workers, the queries will be forwarded to different workers based on its ```tenantid``` for parallel processing. Will the master node be the new bottleneck? Negative. Master node plays a coordinator role in Citus cluster, it just does some slightly analysis for incoming SQL statements and then forward them to workers. If you worry about that, as well as high availability, you can easily set up multiple hot standby master nodes for load balancing by leveraging [streaming replication](#streaming-replication).

## In Conclusion

Today, in this artical, I showed some technics to achieve scalability goal on PostgreSQL database, but remember there is no silver bullet, how to choose them is the most important skill you should have. Do **NOT** scale your database without any analytics from production, do NOT scale out your database before tuning and scaling up.





[1]: https://en.wikipedia.org/wiki/Shard_(database_architecture)
