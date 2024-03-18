# Part 1 - Design

## 1. Application Domain, Generic Types of Data & Technologies

### Domain
**mysimbdp-coredms** is a data storage system designed to capture and manage Amazon product reviews.
### Data Types
The system take input as TSV/CSV file, with structured data, following the schema

```
marketplace
customer_id
review_id
product_id
product_parent
product_title
product_category
star_rating
helpful_votes
total_votes
vine
verified_purchase
review_headline
processed_date
review_body
review_date
```
### Technologies
- **Database**: Apache Cassandra
- **ETL (Data ingestion)**: Apache Nifi
- **Deploy**: Docker, Docker Compose
### Tenant Data Sources
Data sources can come from user-uploaded documents directly to the source directory (Local file system, Google Drive, etc) or from API intergration. In the scope of this assignment, the dataset come from [Amazon US Customer Reviews Dataset](https://www.kaggle.com/datasets/cynthiarempel/amazon-us-customer-reviews-dataset)
The platform will be useful for an enterprise working in Procurement Analytics ([Sievo](https://sievo.com/)), or companies working in retail that need analytics to optimize product offerings, and customer experiences. These enterprises rely on data, and the amount of data on *product review* generated on Amazon per day is gigantic (some suggest around 200,000 reviews per day in the US alone). Thus, there is a need for a big data platform to handle the storage of those information.

## 2. Interactions, Architecture and Infrastructures
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

## 5. Deployment Pros & Cons
Assume that the majority of our customer resides in the Nordic and East Asia, no matter where our data center is located, `mysimbdp-dataingest` will be prioritized close to tenant data sources to minimize the latency during the ingestion process. The reason is that the intra network communication of infrastructure provider is often much faster than the customer's commercial internet.

In this case, I will deploy `mysimbdp-dataingest` at the Google datacenter in Helsinki, and in Tokyo as they are the closest to our customers. It will be ideal if `mysimbdp-coredms` are also located in the same datacenter, that will reduce latency and network bottleneck even more.

If `mysimbdp-coredms` is in the same data center with `mysimbdp-dataingest`, our system will gain the best latency and network throughput, easier to scale vertically, and manage. The down side is centrally deployment will pose the risk of single-point-of-failure when the whole data center goes down.
If `mysimbdp-coredms` is in different data center with `mysimbdp-dataingest`, all the low latency, vertical scalability, and easy management can be traded with higher availability in case of catastrophy.

# Part 2 - Implementation

## 1. Design & Implement
The detail implementation step is describe [here]. Belows are the sample schema and data used to ingest to **mysimbdp-coredms**.
##### Sample Schema
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
   processed_date text,
   review_body text,
   review_date text,
PRIMARY KEY (product_id,review_id));
```
#### Sample Data (amazon_reviews_multilingual_US_v1_00.tsv)
| marketplace | customer_id | review_id | product_id | product_parent | product_title | product_category | star_rating | helpful_votes | total_votes | vine | verified_purchase | review_headline | review_body | review_date |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| US | 53096384 | R63J84G1LOX6R | 1563890119 | 763187671 | The Sandman Vol. 1: Preludes and Nocturnes | Books | 4 | 0 | 1 | N | N | ignore the review below | this is the first 8 issues of the series. it is the starting point of all this... it also contains the sound of her wings. | 1995-08-13 |
| US | 52740469 | R3HGFI3WWKPTOH | 630241380X | 206449098 | "Lonesome" Dove [VHS] | Video | 5 | 1 | 1 | N | N | Awesome Good movie | 1999-02-22 |  |
| US | 53096399 | R1BALOA11Z06MT | 1559947608 | 381720534 | The 22 Immutable Laws of Marketing | Books | 4 | 0 | 0 | N | N | awesome | I've always been partial to immutable laws. The tape is entertaining, good car listening.  | 1995-08-17 |
### Step 
- File is loaded from directory
- Clean data before fetch it to CSV reader, remove `\"` character (ex: `"Lonesome"` -> `Lonesome`)
- Process record, add `processed_date` with current formatted timestamp (ex: `2024-02-15 17:51:04`), cast `product_id`, `product_title`, `review_headline` to STRING type.
- Create new file with `.csv` extension and file name format (ex: `review_2024-02-15-17/51/25.csv`).
- Store file in another directory.
- Put record data to **mysimbdp-coredms**.

## 2. Data Partitioning
**Partition Key** <br>
Our partitioning strategy will rely on the partition key of Cassandra, in this case it's a combination of `product_id` and `review_id` (`PRIMARY KEY (product_id,review_id)`).
- `product_id` is the main partition key, as it allows for the distribution of reviews for different products across the cluster. This way, the data related to each product is stored together, promoting more efficient queries for a specific product.
- `review_id` is the cluster key, combine with partition key will always return one unique review record. We assume this type of query is the most frequent one of **mysimbdp**.
With `NetworkTopologyStrategy` for keyspace strategy and replication factor of 3, all replication of data will be distributed evenly across 2 data centers. In my opinion this balance between performance and reliability, more replications seem like a burden to the system.


## 3. Atomic Data Element & Consistency Options
**Atomic Data Unit** <br>
In our system, an atomic data unit to be stored is a single row of record. Each record represents a unique product review with all its associated attributes.

**Consistency Options** <br>
Considering the nature of our platform is for *analytics* purpose, which prioritizes availability and scalability over strict consistency to accommodate large amount of data, missing few records will not cause any noticeable problem. Thus, our system will go with the **Eventual Consistency**, or `ONE` or `TWO` with consistency options in Cassandra term. These levels acknowledge writes with fewer replicas, allowing for lower-latency writes and eventual convergence.

## 4. Measurement
File sample: 
[amazon_reviews_us_Software_v1_00.tsv](https://www.kaggle.com/datasets/cynthiarempel/amazon-us-customer-reviews-dataset?select=amazon_reviews_us_Software_v1_00.tsv)(249.57 MB).
- **Test 1** 
  * Nodes: 3
  * Consistency: ONE
  * Concurrent tasks: 1
  * Batch size: 50
  * **Task time: Failed after ~13 mins (batch too large)**

- **Test 2**
  * Nodes: 4
  * Consistency: ONE
  * Concurrent tasks: 4
  * Batch size: 15
  * **Task time: ~70 mins **

- **Test 3**
  * Nodes: 4
  * Consistency: ONE
  * Concurrent tasks: 4
  * Batch size: 15
  * **Task time: ~70 mins **

- **Test 4**
  * Nodes: 4
  * Consistency: ONE
  * Concurrent tasks: 10
  * Batch size: 5
  * **Task time: ~60 mins **

- **Test 5**
  * Nodes: 4
  * Consistency: ALL
  * Concurrent tasks: 10
  * Batch size: 5
  * **Task time: ~90 mins **

### Summary
This is a long and tedium process of monitoring data getting ingested. Even on the same machine, same configuration, the results came out vastly different each time. Moreover, double the concurrent tasks does not go in hand with double the process time, the same with number of nodes. 


## 5. Failure Problems
With `Batch Size` property of over 20 for the `PutCassandraRecord` Nifi processor, and a TSV file of just around 50.000 records (~250 MB), the system run into failure with `Batch too large`. This surprise me as it does not seem that much of a batch.
To overcome this, simply reduce the number of record per batch to below 15. Another thing is that triple the amount of items per batch does not improve process time.
![Batch too large error](https://i.ibb.co/yRzFWtp/Microsoft-Teams-image.png)


# Part 3 Extension
## 1. Data Lineage
Our system use Apache Nifi which already support data flow visualization to some extend, provide basic lineage of the ingested data. However, if we decide to go without flow programming tools like Nifi, there are several metadatas that can be collected to record data lineage.
- Source information (where the source data come from?)
- Ingestion timestamp (like the `processed_date` in our system, to record the date that data was ingested to the system)
- Transformations (all the transformatin steps were performed on the data)
- Destination information (details about the target data storage like keyspace, cluster nodes, replicas, collection, etc.)
- Execution metrics (task time, total records, successful, failed records).
 **Example**
 ```
{
  "Source": "Amazon Product Reviews",
  "IngestionTimestamp": "2024-02-16T15:45:00",
  "Transformations": ["Data Clean", "Update Schema", "Update File Attribute"],
  "Destination": {"keyspace": "reviews.product_reviews", "nodes": 3, "replicas": 3},
  "ExecutionMetrics": {"TaskTime": "00:05:30", "ReadThroughput": "150 records/s", "WriteThroughput": "120 records/s" "TotalRecords": 1000, "SuccessfulRecords": 950, "FailedRecords": 50}
}
```

## 2. Schema of Service Information
Our platform will choose Consul for service discovery. Thus, the schema for service information will follow Consul configuration format (in JSON).

```
{
  "datacenter": "helsinki",
  "node_name": "consul-helsinki-1",
  "bind_addr": "0.0.0.0",
  "ports": {
    "http": 8500
  },
  "services": [
    {
      "id": "mysimbdp-1",
      "name": "mysimbdp-1",
      "address": "mysimbdp-1-ip",
      "port": 9042,
      "checks": [
        {
          "name": "mysimbdp 1 Status",
          "http": "http://mysimbdp-1-ip:9042",
        }
      ]
    },
    {
      "id": "mysimbdp-2",
      "name": "mysimbdp-2",
      "address": "mysimbdp-2-ip",
      "port": 9042,
      "checks": [
        {
          "name": "mysimbdp 2 Status",
          "http": "http://mysimbdp-2-ip:9043",
        }
      ]
    }
  ]
}
```

## 3. Service Discovery
For service discovery, I would choose Consul for **mysimbdp**. To integrate service discovery into **mysimbdp-dataingest**. The first thing we need is multiple instances of data ingest instances (multiple Nifi instances). For now, the implementation only result in 1 instance.
We can use docker-compose using the official [apache/nifi](https://hub.docker.com/r/apache/nifi) image for setting up multiple Nifi instances. The system can also consider using [apache/nifi-registry](https://hub.docker.com/r/apache/nifi-registry)for storage and management of shared resources so we can share the same template/architecture to multiple tenants.
```
version: '3'
services:
  nifi-registry:
    image: apache/nifi-registry:latest
    ports:
      - "18080:18080"
    environment:
      - NIFI_REGISTRY_WEB_HTTP_PORT=18080

  nifi:
    image: apache/nifi:latest
    ports:
      - "8080:8080"
    environment:
      - NIFI_WEB_HTTP_PORT=8080
      - NIFI_WEB_HTTP_HOST=0.0.0.0
      - NIFI_REGISTRY_CLIENT_SERVICE_URL=http://nifi-registry:18080
    volumes:
      - ./nifi-data:/opt/nifi/nifi-current/data

```

## 4.  mysimbdp-daas
Since now only **mysimbdp-daas** can read and write data into **mysimbdp-coredms**, our **mysimbdp-dataingest** will now have to communicate with **mysimbdp-daas** for data ingestion. How they communicate depends on the messaging mechanism **mysimbdp-daas** exposing to the world. I will assume the service will communicate via Message Queue (ex: RabbitMQ).
For this, Apache Nifi also support `PublishAMQP` processor, so instead of using `PutCassandraRecord` to directly insert data into **mysimbdp-coredms**, the flow file will be send to `PublishAMQP`. **mysimbdp-daas** will take the responsibility of consuming message from the queue for ingestion.

## 5. Data Moving Between Hot and Cold Space
### Constrain
Base on the characteristics of analytical data, in this case amazon product review, we would like to see the insight in a short period of time (maximum 1 year) to make intime decision on where or not to discontinue certain products or improving quality base on customer feedbacks. Thus, the constrain on our **mysimbdp-coredms** could be:
#### **Hot Space**:
- Data less than 12 months old from `processed_date`.
#### **Cold Space:**
- Data over 12 months old from `processed_date`.
- Data about discontinued products.

### Automatically Moving Data
To support moving data from hot space to cold space, we can schedule a Nifi processor (like a cron) to execute a data migration script (ex: Python) that periodically check for data older than 12 months, queries the hot space and inserts data into the cold space.









