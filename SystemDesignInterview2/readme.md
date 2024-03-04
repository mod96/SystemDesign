# Index
- Geolocational Relation data services
  - Chapter 1. Proximity Service
  - Chapter 2. Nearby Friends
  - Chapter 3. Google Maps
- Loosely coupled message queue System
  - Chapter 4. Distributed Message Queue
  - Chapter 5. Metrics Monitoring and Alert System
  - Chapter 6. Ad click Event Aggregation
- Reservation System
  - Chapter 7. Hotel Reservation System
- Email System
  - Chapter 8. Distributed Email Service
- Data Storage
  - [Chapter 9. S3-like Object Storage](#Chapter-9.-S3-like-Object-Storage)
- Game System
  - Chapter 10. Real-time Gaming Leaderboard
- Currency Exchange Integrity
  - Chapter 11. Payment System
  - Chapter 12. Digital Wallet
  - Chapter 13. Stock Exchange

# Chapter 9. S3-like Object Storage

Object Storage, Virtual cluster map, Erasure coding, Correctness verification, Versioning, Multipart upload, Garbage collection
<p align="center">
    <img src="imgs/9_1.PNG" width="70%" />
    <img src="imgs/9_2.PNG" width="70%" />
</p>

## Step 1. FR & NFR

### FR

- Bucket creation
- Object uploading and downloading
- Object versioning
- Listing objects in a bucket
- object size from few KBs to GBs or more

### NFR

- 100 PB per year
- data durability 6 nines (99.9999%)
- availability 4 nines (99.99%)
- Reduce storage costs while maintaining high degree of reliability and performance

## Step 2. Propose High-level Design

Objects stored inside of object storage are immutable.
Like key-value store, object storage retrieves object data using objects' id.
Most object data is written once and read many times.
Separating metadata and object data simplifies the design like:

<p align="center">
    <img src="imgs/9_3.PNG" width="40%" />
</p>

High level design looks like:

<p align="center">
    <img src="imgs/9_4.PNG" width="70%" />
</p>


## Step 3. Design Deep Dive

### Data Store

#### services
<p align="center">
    <img src="imgs/9_5.PNG" width="70%" />
</p>

`Data Routing Service` provides APIs to access data node cluster. It queries the placement service to get the best data node to store data. It reads data from data nodes and return to API service. It writes data to data nodes. This service is stateless. (Have to choose when to returnto API service: consistency(replication) vs latency(min replica))

`Placement Service` determines which data nodes (primary and replicas) should be chosen to store an object. It maintains virtual cluster map to do so as picture below. Also, it uses heartbeat to continuously monitor all data nodes. Since this service is not stateless, need to use consensus algorithm to build cluster.

<p align="center">
    <img src="imgs/9_6.PNG" width="70%" />
</p>

`Data Node` has a data service daemon running on it. It sends heartbeats to the placement service continuously. This message includes number of disks the node manage, and how much data is stored on each drive.

#### data organization

If we store each object as file, performance suffers for many small files: 
- wastes many data blocks (typical block size 4KB, files smaller than 4KB will waste blocks)
- could exceed inode capacity.

So we use larger files like WAL(write-ahead log). When we save an object, it is appended to an existing read-write file. When the read-write file reaches its capacity threshold(few GBs), the read-write file is marked as read-only and a new read-write file is created.

<p align="center">
    <img src="imgs/9_7.PNG" width="50%" />
</p>

Note that writing a file must be serialized. Multiple cores processing incoming write requests in parallel must take their turns to write to the read-write file. Since this can seriously restrict write throughput, we can provide each core separate read-write file.

#### object lookup & managing

To lookup, object mapping information should be maintained.

<p align="center">
    <img src="imgs/9_8.PNG" width="40%" />
</p>

Since this information doesn't need to be shared across data nodes, we just use SQLite in each Data Nodes. 

Combining informations above, updating data persistence flow shows:

<p align="center">
    <img src="imgs/9_9.PNG" width="70%" />
</p>

#### Erasure coding

Replicating full data makes sense with durability, but it costs xN times more. So we use erasure coding for cost reduction. It chunks data into smaller pieces and creates parities for redundancy. There are lots of algorithms to do this. This slows down access speed since we need to make mathematical calculation for getting a single file, but it's acceptable.

<p align="center">
    <img src="imgs/9_10.PNG" width="60%" />
</p>

#### Correctness verification

There are many checksum algorithms like MD5, SHA1, HMAC, etc.
We save checksum with object, and check if that object is corrupted.

<p align="center">
    <img src="imgs/9_11.PNG" width="60%" />
</p>


### Metadata data model

### Listing objects in a bucket

### Object versioning

### Optimizing uploads of large files

### Garbage collection