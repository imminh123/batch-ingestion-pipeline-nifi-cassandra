# Batch data ingestion pipeline with Apache Nifi & Cassandra
A batch data ingestion pipeline using Apache Nifi as ETL and Cassandra for data sink.

## Technologies
- **Database**: Apache Cassandra
- **ETL (Data ingestion)**: Apache Nifi
- **Deploy**: Docker, Docker Compose

## Interactions, Architecture and Infrastructures
![Apache Nifi Architecture](https://i.ibb.co/Tvng14S/Pasted-image-20240215235735.png)
Using Apache Nifi processor, there are 3 main component groups, which responsible for 3 sequential tasks of Extract - Transform - Load respectively.
- **Extract (ListFile, FetchFile)**
Data (CSV, TSV files) will be fetched from an origin directory, read, and delivered as *flow file* to the next stage of the pipeline.
- **Transform**
   * *ReplaceText*: due to possible conflict with encapsulation token and delimiter, all quotation marks will be removed.
  * *UpdateRecord*: due to possible conflict in data type (int, string), all non-consistent number field will be converted to string format. A new field of [processed_date] is added to each record at this stage, to mark the time a record is processed.
  * *UpdateAttribute*: there will be multiple files that possibly have the same names. At this stage, *file name* will be modified with the format of `review_yyyy-MM-dd-HH:mm:ss.csv`, to ensure file uniqueness, and store pre-processed file with processed date for future use.
- **Load**
  * *PutFile:* Store pre-processed files for future use (cold storage).
  * *PutCassandraRecord*: insert pre-processed data into **mysimbdp-coredms** database.

##  3. Database Configuration
- The Cassandra cluster is setted up with 4 nodes, this help to prevent a single point of failure since the data is distributed across multiple nodes, and each node in the cluster is responsible for a specific range of data so if one fails, other nodes still hold copies of the data, ensuring data availability.
- The cluster is configured with 2 data centers (helsinki, tokyo) with nodes stay in different data centers, providing additional fault tolerance, as data is distributed across different physical locations.
- The nodes were designed to simulate separate racks within each data center, providing fault tolerance in case of rack-level failures.
## 4. Data Replication & Redundancy
- The keyspace is designed with using *NetworkTopologyStrategy*, with the number of replicas for each data center is 3 ensures that each piece of data is stored on three different nodes. This is a better choice for production environment than *SimpleStrategy* especially in scenarios where a data center might experience a failure.
- Two nodes are introduced for each data center (cassandra1 and cassandra2 for [helsinki], and cassandra3 and cassandra4 for [tokyo]).
This configuration ensures redundancy across nodes and data centers. The number of nodes are set at 4, combine with replication factor of 3 ensure that even if a whole data center goes down, and for the remaining data center, half the nodes (in this case is 1) go down, the system remains operational with replicas stored in other data center, and other nodes.

## Deployment/installation guide

### Deploy Docker Compose
The docker compose located in /code/docker-compose.yml
```
docker-compose up -d
```

### Setting up Cassandra Keyspace 
From inside container `cassandra1-1`
```
$ bin/bash
$ cqlsh
```

Create new keyspace `amazon_reviews`
```
CREATE KEYSPACE amazon_reviews
  WITH REPLICATION = {
   'class' : 'NetworkTopologyStrategy',
   'helsinki' : 2,
   'tokyo' : 1
  };
```

Create new table `product_reviews`
```
CREATE TABLE amazon_reviews.product_reviews (
   marketplace text,
   customer_id int,
   review_id text,
   product_id text,
   product_parent int,
   product_title text, 
   product_category text, 
   star_rating int,
   helpful_votes int,
   total_votes int,
   vine text,
   verified_purchase text,
   review_headline text,
   processed_date text, review_body text,
   review_date text,
PRIMARY KEY (product_id,review_id));
```

Verify setting up (~wait for 5 mins)
```
$ nodetool status
```
![Nifi architecture](https://i.ibb.co/tQrvZx5/image.png)

### Apache Nifi 
- Start Apache Nifi and import template from `/code/CassandraIngestPipeline.xml`.
- Change `Input Directory` of ListFile Processor to your source directory.
- Change `Directory` of PutFile Processor to your destination directory.
- Make sure to enable and point CassandraSessionProvider to the right host (9042)
- Start the pipeline
