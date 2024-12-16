# mongodb-replication

### <a name="content">Table of contents:</a>

1. [Introduction](#intro)

    1.1 [What is Replication ?](#rep)

    1.2 [Why is Replication Important ?](#imp)

2. [Replication in Mongodb Database](#rep_mongo)

    2.1 [Setting Replication](#set)
    
    2.2 [Testing Replication](#test)
   
    2.3 [Testing Failover](#failover)

## <a name="intro">1. Introduction</a>

In modern applications, ensuring data availability, reliability, and fault tolerance is essential. This is where replication comes into play.

This article Explores the concept of replication, its importance, and how it is implemented in MongoDB, one of the most popular NoSQL databases.

### <a name="rep">1.1 What is Replication ?</a>

Replication is the process of copying and maintaining data across multiple servers. It ensures that the same data is available in more than one place, providing data redundancy and availability.

### <a name="imp">1.2 Why is Replication Important ?</a>

Replication is crucial for system reliability and fault tolerance. It helps keep data accessible even if a server fails, enhances system performance by distributing read operations, and supports disaster recovery by having backups ready in real time.

## <a name="rep_mongo">2. Replication in Mongodb Database</a>

MongoDB uses replication to create a `replica set`, which is a group of servers with one primary node and multiple secondary nodes. The primary node handles all write operations, and the secondary nodes replicate data from the primary. If the primary node fails, one of the secondary nodes is automatically promoted to primary to ensure continuous availability. 

MongoDB replication also supports read scaling by allowing read operations on secondary nodes.

### <a name="set">2.1 Setting Replication</a>

**[1] -**  We will have 3 mongodb servers and start them from the same machine on three different ports :

• 127.0.0.1 => 27018
• 127.0.0.1 => 27019
• 127.0.0.1 => 27020

**[2] -**  Create 3 folders in `rep` folder, required for data directory and log file :

`• mkdir rep/data1 rep/data2/ rep/data3`

**[3] -** Start all the three mongod process:

`• mongod --bind_ip 127.0.0.1 --port 27018 --replSet rs0 --dbpath rep/data1 --oplogSize 200 --logpath rep/data1/mongo.log --fork`

`• mongod --bind_ip 127.0.0.1 --port 27019 --replSet rs0 --dbpath rep/data2 --oplogSize 200 --logpath rep/data2/mongo.log --fork`

`• mongod --bind_ip 127.0.0.1 --port 27020 --replSet rs0 --dbpath rep/data3 --oplogSize 200 --logpath rep/data3/mongo.log --fork`


Here’s a breakdown of the options :

1. `--bind_ip 127.0.0.1`
- Restricts MongoDB to listen for connections only from specifiec IP address(es)
- `127.0.0.1` means MongoDB will accept connections only from the local machine
- We can provide multiple Ip address(es) like: `127.0.0.1,10.17.57.10`

2. `--port 27020`
- Specifies the port MongoDB will use to listen for incoming connections 
- Default port is `27017`.

3. `--replSet rs0`
- Enables replication by assigning the instance to a replica set named rs0.
- This option does not start the replica set immediately but prepares the instance for replication.

4. `--dbpath rep/data3`
- Sets the directory where MongoDB will store its data files.
- The directory must exist and be writable by the MongoDB process.
- This is critical because it determines where your data is physically stored on the disk.

5. `--oplogSize 200`
- Allocates 200 MB for the oplog (operation log) used for replication.
- The oplog records all operations that modify the database and helps synchronize replica set members.

6. `--logpath rep/data3/mongo.log`
- Sets the file where MongoDB will write its log messages.

7. `--fork` (optional)
- Runs the MongoDB process in the background (linux specific).
- Don't use `--fork` option if you are running this command from windows, instead you can run each command in 3 sperate terminals

**[4] -** Test Connectivity to each instance (optional):

`• mongosh --port 27018`
`• mongosh --port 27019`
`• mongosh --port 27020`


**[5] -** Connect to one of the mongod process, and create a variable with member information and replica set name. 

```js
rconfig={
 _id:"rs0",
 members:[
  { _id:0, host:"127.0.0.1:27018" },
  { _id:1, host:"127.0.0.1:27019" },
  { _id:2, host:"127.0.0.1:27020" }
 ]
}
```
- make sure to use the replica set name used while starting up the mongod servers (`rs0` in our example) on the `_id` property.
- you can type `rconfig` to check the value of the variable.

**[6] -** Initiate the replication configuration
```js
rs.initiate(rconfig)
```

**[7] -** Checking replication status:

1. The current mongo shell prompt displays if the shell you connected to is a primary or a secondary :

   ```
   rs0 [direct: secondary] test>
   rs0 [direct: primary] test>`
   ```

2. To check the status of a replica set and provides detailed information about the replica set configuration, the state of its members, and their synchronization status :

   ```js
   rs.status()
   ```

3. To  check the role of the current MongoDB instance in a replica set or standalone server, provides information about whether the instance is the primary, secondary, or a standalone server:

   ```js
   db.isMaster()
   ```


### <a name="test">2.2 Testing Replication</a>

**[1] -** connect to the primary service.
**[2] -** create a database and a collectio nand insert at least one document in it : 
```js
use movies
db.films.insertOne({name: "Godzilla"})
```
**[3] -** we need to check if the database, collection and the record have been replicated in other replicas:

- connect to the other 2 secondary members and checked:
- Both the database [movies] and the collection [films] have been replicated.
- but when trying to read the record `db.films.find();`  you will encounter this error:

  `
  MongoServerError: not primary and secondaryOk=false - consider ... readPreference in the connection string
  `
-  we need to enable reading from the secondary:
`
• db.getMongo().setReadPref("primaryPreferred")
`
- other options:
   - `primary`: Default. Queries only to the primary node.
   - `primaryPreferred`: Prefer the primary but fall back to secondaries if necessary. (ensure consistency with the latest writes)
   - `secondary`: Queries only to secondary nodes.
   - `secondaryPreferred` : Prefer secondaries but fall back to the primary if no secondary is available.
  - `nearest` : Queries the node (primary or secondary) with the lowest network latency.

### <a name="failover">2.3 Testing Failover</a> 

What if something happen to the primary, will the secondary be selected or not ?
1.  Shutdown the `primary` instance and see if the `secondary` was selected or not.
    - in windows just `ctrl + c` inside the terminal or just close it
    - in linux terminates the process or inide the mongo shell of the primary => 
    - or use admin then => `db.shutdownServer()` (will close the terminal)

2. connect to another service, one of them will be elected to be the primary. (check shell or `db.isMaster()` or `rs.status()`) => to check who's is the primary and also for the server who has been shut down :

   - `stateStr`: '(not reachable/healthy)',
   - `lastHeartbeatMessage`: 'Error connecting to 127.0.0.1:27019 :: caused by :: No connection could be made becau ...

3. try to bring the primary server back and check again the status.
   - it will return back as a secondary.
