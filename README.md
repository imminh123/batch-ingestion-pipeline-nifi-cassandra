# Batch data ingestion pipeline with Apache Nifi & Cassandra
A batch data ingestion pipeline using Apache Nifi as ETL and Cassandra for data sink.

## Architecture
![Nifi architecture](https://github.com/imminh123/batch-ingestion-pipeline-nifi-cassandra/blob/main/assets/nifi_architecture.png?raw=true)

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
