# V2.3.0 Upgrade Feature: opentenbase_subscription User Guide

## 1. Modify wal_level

Since the multi-active architecture is implemented with logical replication, `wal_level` must be set to `logical` on all CN/DN nodes of every cluster involved in the multi-active deployment.

```
vim postgresql.conf
wal_level = logical
:wq
```

## 2. Install the opentenbase_subscription extension

Connect to a CN of each of the two clusters configured for multi-active (for any given cluster, running this on any single CN is sufficient, but both clusters need the extension installed) and execute:

```
CREATE EXTENSION opentenbase_subscription;
```

## 3. One-way replication setup

This example uses a publisher with 1 CN + 2 DNs and a subscriber with 1 CN + 2 DNs; other cluster topologies follow similar steps.

### 3.1 Connect to any CN of the publisher and create a table

```
postgres=# create table users(id int,name varchar(30),f3 int,PRIMARY KEY(id));
postgres=# insert into users select i, 'a', i+1 from generate_series(1, 399999) i;
INSERT 0 399999
```

### 3.2 Publish data on the publisher DNs

Note: if the publisher cluster has multiple DNs, publication must be configured on every DN.

Publisher DN1:

```
CREATE PUBLICATION mypub_dn001 FOR ALL TABLES;
```

Publisher DN2:

```
CREATE PUBLICATION mypub_dn002 FOR ALL TABLES;
```


### 3.3 Subscribe to data on the subscriber CN
Subscriber CN:

```
create table users(id int,name varchar(30),f3 int,PRIMARY KEY(id)) ;

CREATE OPENTENBASE SUBSCRIPTION sub_dn001 CONNECTION 'host=100.105.39.157 port=20008 user=jennyer dbname=postgres' PUBLICATION mypub_dn001 WITH (connect=true, enabled=true, create_slot=true, copy_data=true, synchronous_commit=on, ignore_pk_conflict = true, parallel_number=4);

CREATE OPENTENBASE SUBSCRIPTION sub_dn002 CONNECTION 'host=100.105.39.157 port=20009 user=jennyer dbname=postgres' PUBLICATION mypub_dn002 WITH (connect=true, enabled=true, create_slot=true, copy_data=true, synchronous_commit=on, ignore_pk_conflict = true, parallel_number=4);

```

Verify the initial data replicated from the publisher.

Subscriber CN:

```
select count(*) from users;
 count  
--------
 399999
(1 row)

```

### 3.4 Verify incremental data replication
Connect to the publisher CN and insert incremental data:

```
insert into users select i, 'a', i+1 from generate_series(400000, 500000) i;
INSERT 0 100001
```

Verify the incremental data replicated from the publisher.  
Subscriber CN:

```
select count(*) from users;
 count  
--------
 500000
(1 row)
```


## 4. Two-way replication setup

### 4.1 Create the table on both active clusters
Cluster 1 CN:

```
create table test(id int,name varchar(30),PRIMARY KEY(id));
```

Cluster 2 CN: 
 
```
create table test(id int,name varchar(30),PRIMARY KEY(id));
```


### 4.2 Cluster 1 acts as publisher: publish data on all DNs
Cluster 1 DN1:

```
CREATE PUBLICATION cluster1_dn001 FOR ALL TABLES;
```

Cluster 1 DN2：

```
CREATE PUBLICATION cluster1_dn002 FOR ALL TABLES;
```	

### 4.3 Cluster 2 acts as publisher: publish data on all DNs
Cluster 2 DN1:

```
CREATE PUBLICATION cluster2_dn001 FOR ALL TABLES;
```

Cluster 2 DN2：

```
CREATE PUBLICATION cluster2_dn002 FOR ALL TABLES;
```	

### 4.4 Cluster 1 acts as subscriber: subscribe on any CN
Cluster 1 CN：

```
CREATE OPENTENBASE SUBSCRIPTION sub_cluster2_dn001 CONNECTION 'host=100.105.50.198 port=20008 user=jennyer dbname=postgres' PUBLICATION cluster2_dn001 WITH (connect=true, enabled=true, create_slot=true, copy_data=true, synchronous_commit=on, ignore_pk_conflict = true, parallel_number=4);
CREATE OPENTENBASE SUBSCRIPTION sub_cluster2_dn002 CONNECTION 'host=100.105.50.198 port=20009 user=jennyer dbname=postgres' PUBLICATION cluster2_dn002 WITH (connect=true, enabled=true, create_slot=true, copy_data=true, synchronous_commit=on, ignore_pk_conflict = true, parallel_number=4);
```

### 4.5 Cluster 2 acts as subscriber: subscribe on any CN

Cluster 2 CN：

```
CREATE OPENTENBASE SUBSCRIPTION sub_cluster1_dn001 CONNECTION 'host=100.105.39.157 port=20008 user=jennyer dbname=postgres' PUBLICATION cluster1_dn001 WITH (connect=true, enabled=true, create_slot=true, copy_data=true, synchronous_commit=on, ignore_pk_conflict = true, parallel_number=4);
CREATE OPENTENBASE SUBSCRIPTION sub_cluster1_dn002 CONNECTION 'host=100.105.39.157 port=20009 user=jennyer dbname=postgres' PUBLICATION cluster1_dn002 WITH (connect=true, enabled=true, create_slot=true, copy_data=true, synchronous_commit=on, ignore_pk_conflict = true, parallel_number=4);
```

### 4.6 Write data on cluster 1
Cluster 1 CN：

```
insert into test values(1,'a'),(3,'a'),(5,'a'),(7,'a'),(9,'a'),(11,'a'),(13,'a'),(15,'a');
```

### 4.7 Write data on cluster 2
Cluster 2 CN：	

```
insert into test values(2,'b'),(4,'b'),(6,'b'),(8,'b'),(10,'b'),(12,'b'),(14,'b'),(16,'b');
```

### 4.8 Verify the replicated result on cluster 1 (local writes + remote replication)
Cluster 1 CN：

```
select  * from test;
postgres=# select  * from test order by 1;
 id | name 
----+------
  1 | a
  2 | b
  3 | a
  4 | b
  5 | a
  6 | b
  7 | a
  8 | b
  9 | a
 10 | b
 11 | a
 12 | b
 13 | a
 14 | b
 15 | a
 16 | b
(16 rows)
```

### 4.9 Verify the full replicated result on cluster 2 (local writes + remote replication)
Cluster 2 CN：

```
postgres=# select  * from test order by 1;
 id | name 
----+------
  1 | a
  2 | b
  3 | a
  4 | b
  5 | a
  6 | b
  7 | a
  8 | b
  9 | a
 10 | b
 11 | a
 12 | b
 13 | a
 14 | b
 15 | a
 16 | b
(16 rows)
```

You can write from either cluster 1 or cluster 2; eventually both clusters hold the complete data set.
 
