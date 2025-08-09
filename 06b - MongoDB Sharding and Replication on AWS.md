# MongoDB Sharding and Replication on AWS

**NOTE: PLEASE READ THE ENTIRE CASE. A lot of the commands currently written here will need to be changed based on your environment. In particular, anything enclosed in `<>` will need to be changed.**

## Overview

**Sharding** is a method for distributing data across multiple machines. MongoDB uses sharding to support deployments with very large data sets and high throughput operations.

Database systems with large data sets or high throughput applications can challenge the capacity of a single server. For example, high query rates can exhaust the CPU capacity of the server. Working set sizes larger than the system’s RAM stress the I/O capacity of disk drives.

There are two methods for addressing system growth: vertical and horizontal scaling.

*Vertical Scaling* involves increasing the capacity of a single server, such as using a more powerful CPU, adding more RAM, or increasing the amount of storage space. Limitations in available technology may restrict a single machine from being sufficiently powerful for a given workload. Additionally, Cloud-based providers have hard ceilings based on available hardware configurations. As a result, there is a practical maximum for vertical scaling.

*Horizontal Scaling* involves dividing the system dataset and load over multiple servers, adding additional servers to increase capacity as required. While the overall speed or capacity of a single machine may not be high, each machine handles a subset of the overall workload, potentially providing better efficiency than a single high-speed high-capacity server. Expanding the capacity of the deployment only requires adding additional servers as needed, which can be a lower overall cost than high-end hardware for a single machine. The trade off is increased complexity in infrastructure and maintenance for the deployment.

MongoDB supports horizontal scaling through sharding.

## Sharded Cluster
A MongoDB sharded cluster consists of the following components:

- [shard](https://docs.mongodb.com/manual/core/sharded-cluster-shards/): Each shard contains a subset of the sharded data. Each shard can be deployed as a replica set.
- [mongos](https://docs.mongodb.com/manual/core/sharded-cluster-query-router/): The mongos acts as a query router, providing an interface between client applications and the sharded cluster.
- [config](https://docs.mongodb.com/manual/core/sharded-cluster-config-servers/) servers: Config servers store metadata and configuration settings for the cluster. As of MongoDB 3.4, config servers must be deployed as a replica set (CSRS).

### Shard Keys
To distribute the documents in a collection, MongoDB partitions the collection using the [shard key](https://docs.mongodb.com/manual/reference/glossary/#term-shard-key). This means that one shard will have a certain portion of the entire collection. The shard key must be indexed for efficient querying.

The choice of shard key affects the performance, efficiency, and scalability of a sharded cluster. A cluster with the best possible hardware and infrastructure can be bottlenecked by the choice of shard key. The choice of shard key and its backing index can also affect the sharding strategy that your cluster can use.

### Chunks
MongoDB partitions sharded data into **chunks**. A chunk is a contiguous range of shard key values within a particular shard. Each chunk has an inclusive lower and exclusive upper range based on the shard key. Chunk ranges are inclusive of the lower boundary and exclusive of the upper boundary. MongoDB splits chunks when they grow beyond the configured chunk size, which by default is 64 megabytes. MongoDB migrates chunks when a shard contains too many chunks of a collection relative to other shards. 

### High Availability and Replication
A sharded cluster can continue to perform partial read / write operations even if one or more shards are unavailable. While the subset of data on the unavailable shards cannot be accessed during the downtime, reads or writes directed at the available shards can still succeed.

In production environments, individual shards should be deployed as replica sets, providing increased redundancy and availability. **These should be deployed in seperate disaster zones to maximize availability.** 

## Objectives

Your goal is to setup a production MongoDB Cluster. Each shard must be replicated accross two different servers accross separate failure zones. The figure below is a sample diagram of the cluster you will be constructing sans networking configuration. 

![network diagram](https://drive.google.com/uc?id=1AtXx2jaenS4cPRAYJoUBMHC4veYBVOPc)


This cluster will be implemented on AWS. Make sure you remember how to configure the **VPC, subnets, routes, and security groups**. **Ensure you only use t2.micro to optimize for cost.** It should also persist accross shutdowns and interruptions.


## Steps

You will need at least 9 AWS EC2 instances to implement this infrastructure. You will first create the Config Server databases, then create the shard databases, then lastly, configure the mongos server.

### Creating a Config Database
Config servers store the metadata for a sharded cluster. The metadata reflects state and organization for all data and components within the sharded cluster. The metadata includes the list of chunks on every shard and the ranges that define the chunks.

#### Start the Configuration Server

The next step is to start the mongodb server. This will be a configuration server, so it is not as simple as running the service. First, check if there is an existing mongodb server running. You can check this by listing the processes with `sudo systemctl status mongod`. You can kill the process using the command `sudo systemctl status mongod`.

#### Initiate Config Server Replicas
Now, you must indicate that this database is part of replica set. First, ssh to one of the instances then connect to one of the MongoDB servers using this command `mongo --port 27019 --host <ip address>`. Then, run the following MongoDB command to indicate the replica set.


```javascript
rs.initiate(
  {
    _id: "csrs",
    configsvr: true,
    members: [
      {"_id" : 0, "host" : "<ip address>:27019"},
      {"_id" : 1, "host" : "<ip address>:27019"}
    ]
  }
)

rs.status()
```

The `rs.status()` command should show a primary and a secondary server. 


### Creating the Shards

A shard contains a subset of sharded data for a sharded cluster. Together, the cluster’s shards hold the entire data set for the cluster.

As of MongoDB 3.6, shards must be deployed as a replica set to provide redundancy and high availability.

Users, clients, or applications should only directly connect to a shard to perform local administrative and maintenance operations.

Performing queries on a single shard only returns a subset of data. Connect to the mongos to perform cluster level operations, including read or write operations.

#### Start the Shard Server

Since each shard will be replicated and there will be three shards in this cluster, you will need to have three unique `<replicasetname>`. You will also to configure all 6 servers using the commands discussed.

#### Initiate Shard Replica Sets
Initiate the replica set similar to how it was done in the configuration servers.

```bash
mongo --port 27018 --host <ip address>
```

```javascript
rs.initiate(
  {
    _id: "<replicasetname>",
    members: [
      {"_id" : 0, "host" : "<ip address>:27018"},
      {"_id" : 1, "host" : "<ip address>:27018"}
    ]
  }
)

rs.status()
```

### Initiating Mongos

The last step is to initiate the mongos server. MongoDB mongos instances route queries and write operations to shards in a sharded cluster. mongos provide the only interface to a sharded cluster from the perspective of applications. Applications never connect or communicate directly with the shards.

The mongos tracks what data is on which shard by caching the metadata from the config servers. The mongos uses the metadata to route operations from applications and clients to the mongod instances. A mongos has no persistent state and consumes minimal system resources.

#### Set Location of Config Servers
SSH to the mongos server and run the following command:
```bash
sudo mongos --port 27017 --bind_ip 0.0.0.0 \
--configdb csrs/<ip>:27019,<ip>:27019 \
--logpath /var/log/mongodb/mongod.log \
--fork
```

The following block is a more detailed explanation of each option in this command.

```bash
sudo mongos --port 27017 --bind_ip 0.0.0.0 \ #Set the IP address and port. The bind_ip is 0.0.0.0 because we want to listen for connections from anywhere.
--configdb csrs/<ip>:27019,<ip>:27019 \ #Set the location of the config servers
--logpath /var/log/mongodb/mongod.log \ #Set the location of where the logs will be saved
--fork #Runs the application in the background
```

#### Add Shard Locations
The mongos server must be made aware of the location of the shards and their replicas. This can be done by connecting to the mongos server from the mongo instance by running `mongo` on the instance.

After connecting to the mongos server, run the following command:

```javascript
sh.addShard( '<replicasetname1>/<ip>:27018,<ip>:27018' );
sh.addShard( '<replicasetname2>/<ip>:27018,<ip>:27018' );
sh.addShard( '<replicasetname3>/<ip>:27018,<ip>:27018' );
sh.status()
db.adminCommand({listShards : 1})
```

#### Enable Sharding on the Database
The last step is to configure the collection to be sharded. You will be sharding the grades collection based on the class_id. While connected to the mongos server, run the following commands.

```javascript
use sample
db.<collection>.createIndex(   { "<shardkey>" :  "hashed" },   { background : true })
sh.enableSharding("sample")
sh.shardCollection("sample.<collection>", { "<shardkey>" : "hashed"}, false, {"numInitialChunks" : 3})
```
Download the data.zip file again from this [link]( https://s3.amazonaws.com/cs129.1/data/data.zip) (refer to[ Lab - MongoDB CRUD](https://moodle.ateneo.edu/ls/mod/assign/view.php?id=42230)). Import the data using `mongoimport -d sample -c grades grades.json`. MongoDB should distributed the data accross the different shards. You can check the distribution by running `db.grades.getShardDistribution()`. The output should be similar but **maybe not** exactly the same as the following.

```
Shard shard0 at shard0/10.0.0.103:27018,10.0.0.228:27018
 data : 6.54MiB docs : 29476 chunks : 2
 estimated data per chunk : 3.27MiB
 estimated docs per chunk : 14738

Shard shard1 at shard1/10.0.0.120:27018,10.0.0.177:27018
 data : 8.15MiB docs : 36705 chunks : 2
 estimated data per chunk : 4.07MiB
 estimated docs per chunk : 18352

Shard shard2 at shard2/10.0.0.114:27018,10.0.0.135:27018
 data : 7.51MiB docs : 33819 chunks : 2
 estimated data per chunk : 3.75MiB
 estimated docs per chunk : 16909

Totals
 data : 22.21MiB docs : 100000 chunks : 6
 Shard shard0 contains 29.47% data, 29.47% docs in cluster, avg obj size on shard : 233B
 Shard shard1 contains 36.7% data, 36.7% docs in cluster, avg obj size on shard : 233B
 Shard shard2 contains 33.81% data, 33.81% docs in cluster, avg obj size on shard : 233B
```

The thing to look out for is that the data is evenly distributed across the three shards.

## Expected Output

1. Infrastructure diagram including subnets, security groups, IP addresses, and global infrastructure components (AZs, regions, etc.)

2. Actual infrastructure deployed on AWS. You will be required to demonstrate this during the defense

3. Also include the security roles from the security slideset. Be sure they are created in the proper database. Refer to this link https://docs.mongodb.com/manual/core/security-users/#sharded-cluster-users

4. Be prepared to edit the infrastructure as necessary during the defense based on suggested improvements made by the instructor

5. Be prepared to answer questions about the infrastructure during the defense


## Additional Notes

### AWS Cost
**Be aware of your remaining credits.** Stop your instances when you're not using them. Stopping the instances may change the Public IP addresses, but it will not change the private IP addresses. If you configure the instances correctly, stopping and starting should maintain the state of the replicas and shards. I highly suggest organizing your scripts before running them (i.e. Change the items that need to be changed, input IP addresses, etc.).

### Groupings
This lab has several steps and moving parts. You will be configuring a total of 9 servers as well as the AWS infrastructure. Make use of all the members of your group. 


**Instructions below are not applicable for online semester.**

### Case Choice and Defense Signups
**Only one representative per group should select a choice.** This is important because if another member were to sign-up, s/he would be taking someone else's slot. 

Sign-ups for defenses will be posted at 12nn on November 28. When signing up, list the group members in the description using the following format:

```
<IDNumber>-<LastName>
<IDNumber>-<LastName>
```
  
  e.g. 
  
  ```
  23800-Saavedra
  22345-Medalla
  22335-Pulmano
  ```
  
  **Do not move defense times without first consulting the instructor.** This should only be done during special circumstances. If for any reason, a group is not available at any of the defense slots, please contact the instructor ASAP. Any special consideration is at the discretion of the instructor.
  