# RADOS

RADOS stands for Reliable Autonomic Distributed Object Storage.

### WHY?
- Improve the scalability of storage clusters
- High-throughput and low-latency storage required at a **cheaper cost**
- Consistent management of data placement, failure detection, and failure
  recovery places an increasingly **large burden on client** ,controller, or
  metadata directory nodes, limiting scalability. (_because storage devices are
  treated as dumb_)

### WHAT?

RADOS aims to provide **balanced distribution of data** and workload across a
**heterogenous** storage cluster while providing applications with illusion of
**single logic store** (removing burden from clients) with well defined **safety
semantics** and **strong consistency** guarantees.

ASK: Safety semantics how??

Question: What comes under safety semantics and strong consistency ??

**Intelligent Devices**, Storage nodes semi-autonomously - self manage:
* Replication
* Failure Detection
* Failure Recovery


## 2. Scalable Cluster Management

RADOS = OSD + Monitor cluster

OSD:
- RAM
- Network Interface
- Attached disk drive or RAID

Monitor:
- stand alone processes

### 2.1 Cluster Map

As system scales, new storage and **decommissioning** of old devices, devices
fail and recover on a **continuous basis**, and large amounts of data are
**created and destroyed**. We need a way to have consistent view of data.

Question: How was this consistent view maintained before Cluster Map? Lookup
tables?, If yes then lookup tables are problematic because that will have
problems of "lots of stale data", "single point of failure", "work only being
done by clients"

**Versioned Cluster map**, ensures:
- Specifies data distribution of entire system
- consistent read and write access to data objects
- Monitor uses cluster map to manage storage cluster
- replicated by both clients and OSD
- incremental updates, lazy propagation

**Versioned Cluster map**, OSD with _complete knowledge_ of distribute data,
semi autonomously _self manage_ - using P2P protocols:
- data replication
- consistently and safely process updates (Question: How consistently and safely?)
- participate in failure detection
- respond to device failure (re-replicate or migrate object)

**Eases the burden on Monitor Cluster**, which manages the master copy of map -
**enabling the system to scale**.


Each cluster map has the following information: 
|Information|Description|
|-----------|-----------|
|Map epoch|map version |
|OSDs that are up and down|network address of all healthy & unhealthy nodes|
|OSDs that are in and out|nodes included in PGs and nodes that are not. |
|Number of PGs||
|CRUSH|placement rules to map each PG to OSDs|

* epoch incremented, whenever data is affected (eg: OSD device failure, PG
  groups changes etc)
    * Allows everyone to agree on the current data distribution
    * Helps figure out out-of-data information for all systems
* Updates delivered as incremental maps, small messages describing the
  difference between two successive epochs

Possible state of OSDs: 
|OSD State|Description|
|---------|-----------|
|(up, in)| present in an PG and actively serving I/O|
|(down, out)|Failed|
|(up, out)|Online but idle. eg - newly added devices which aren't immediately used.|
|(down, in)|Unreachable but data hasn't yet been migrated off. eg - intermittently unreachable, data migration in progress before decommissioning.|

The above possible states facilitates a variety os scenarios.

### Map Propagation

- OSDs are responsible for sharing map updates along with inter OSD
  communication + hearbeat messages
    - Only the diff is shared, ensure minimal amount of data present in messages
- Updates are lazily propagated, OSDs receive updates as and when they interact
  with the cluster
- Each OSD does:
    - maintains history of past incremental updates 
    - tag all messages (_heartbeat messages included_) with its latest epoch
      (_helps in finding stale data_)
    - keeps track of most recent epoch present at each peer (_helps in sending
      updates to relevant peers_)
    - periodically sends heartbeat messages for failure detection
- When contacting peers with old epoch, **incremental updates** are shared to
  bring that peer in sync.
- Map updates are shared only b/w communicating OSDs (i.e the OSD who share PGs)
    - **preemptive map sharing strategy**, OSD will always share an update when
      contancting a peer
    - duplicate messages - bounded by number of peers, peers are determined by
      number of PGs it manage

#### OSD booting

- OSD booting up, it informs the monitor with it's latest epoch it is booting up
  with
- Monitor cluster, updates the OSD state in the global cluster map as up
    - Monitor shares the incremental updates to bring it fully up to date
- Newly booted OSD, sends heartbeart messages and incremental updates to it's
  "unaware" peers (set of devices who are affected by its status change)
    - The incremental update shared by new OSD includes the last 30 seconds of
      updates


### 2.2 Data Placement

RADOS **pseudo-randomly distributes objects** to devices, maintaining a **balanced distribution** of data on all devices.

<PHOTO of entire data pass through from conf talk>

#### Step 1: Map objects to Placement groups

- Placement groups (PGs) are a logical collection of objects
- Client's Data is split into various objects
- Objects PG, determined by:
    * o, object name,
    * r, desired level of replication
    * m, bit mask - total number of placement groups
- number of placement groups periodically adjusted when cluster scales

#### Step 2: Map Placement groups to OSD

- CRUSH: Controlled, Scalable, Decentralized Placement of Replicated Data
    * algorithm to calculate pseudo random mapping
    * a robust replica distribution algorithm, similar to a hash function (but
      safe)
    * maintains balanced distribution, when devices join/leave - most PG remain
      where they are
- maps each PG to an ordered list of r OSDs
    * The order of the OSD decided which are primary and which are replicas

### Why Placement groups ?

Three ways of replication:

<PHOTO OF why placement groups>

| replication type   | description<br>(3 way replication) | disk fails                                                                                                                                                                              | Concurrent Failure<br> (Triple failures)                                                                                                                                                                                            |   |
|--------------------|------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---|
| replicated disks   | each disk is replicated            | * empty spare (of same size) required to recover<br>* bottlenecked, single disk throughput <br>    (only as fast as the disk can write)<br>* two devices completely involved in process | * In this setup, data loss usually happen<br>    across different devices.<br>* Very rare for the entire rack to go down<br>* Very few data failures cause complete data<br>    loss                                                |   |
| replicated PGs     | each PG is replicated              | * PG from lost device present in other surviving<br>  devices.<br>* Parallel recovery process<br>* did NOT need a spare for recovery, redistributed<br>  the PG across existing devices | * Balance b/w disk and objects<br>* Some triple failures cause data loss<br>* CRUSH reduces chances, it uses replication<br>   level to replicate PG on different hosts.<br>* Erasure coding, allows us to reconstruct<br>    data  |   |
| replicated objects | each object is replicated          | * each device participates in recovery                                                                                                                                                  | * High possibility that data is present only<br>     on the three nodes that went down<br>* Every triple failure causes data loss<br><br>                                                                                           |   |

What happens if enough disk fails, [Stack overflow answer](https://stackoverflow.com/questions/51934067/ceph-what-happens-when-enough-disks-fail-to-cause-data-loss).


## 3. Intelligent Storage Devices

Data distribution encapsulated in cluster map allows OSDs to self manage the below using peer-like protocols (eg: gossip):
- Data Redundancy (Replication)
- Failure Detection
- Failure Recovery

## 3.1 Replication

<FIGURE 2: FROM PAPER>

- Client talks to only ONE OSD, the replication are then managed by OSD peers
  themselves.
    - The cluster ensures: replicas are safely updated and consistent read/write
      preserved
    - Question: What happens when the replica goes down during a write?
- Once all replicas are updates, a single acknowledgment is returned to the
  client.

Three types of replication:

||Primary-Copy|Chain|Splay|
|-|-----------|-----|-----|
|Reads|Primary|Tail|Tail|
|Writes|Primary|Primary|Primary|
|Replication|Parallel|Serial|Parallel|

## 3.2 Strong Consistency

RADOS is a CP system. RADOS features consistency and Partition tolerance over availability.

There are two types in consistency:
- **Read Consistency**: Any read operation from any node will return the most
  recent value
- **Write Consistency**: When a write operation is performed, data is propagated
  to all relevant nodes , before it is considered successfull


#### Write Consistency

All messages from both (clients + OSD) are tagged with sender's map epoch
    - this ensures, update operation are applied in a fully consistent manner

We have the following cases:

1. Good path
    - All the OSD involved in the update are live and functioning
    - The replication strategies makes sure that, the write is propagated to all
      nodes before considered as successfull
2. Client contacts wrong OSD
    - This can happen, when the client has old map and the PG that client wants
      to update changed it's membership. This can happen when new devices are
      added
    - The wrong OSD will respond with appropriate incrementals so that the
      client can redirect the request
    - This avoids the need to pro-actively share map updates
3. Client contacts OSD, but participating primary OSD are unaware of the PG
   membership change
    - Updates still processed by old members
    - Assuming PG replica knows of the change
        - When primary OSD forwards the update to replica, replica responds back
          with new incremental map updates
        - Next time, when client contacts the primary - the primary will
          redirect the client to the correct OSD
    - Updates processed by old member is safe bcoz
        - Any set of OSD newly responsible for a PG are required to contact all
          previously involved nodes in order to determing the PG contents
        - This also ensures that, new OSD get the data before the old OSD stop
          communicating
4. Client contacts primary OSD, but during the replication proces - replica
   fails
    - Since peer OSD share heartbeat messages, if any of the peer does not
      receive a heartbeat message within a time frame, the update is blocked.
    - Client gets an error, this ensures that half bad replication does not
      happen

#### Read Consistency

This is a bit harder than updates. 

Scenario: In event of network failure, OSD becomes partially unreachable:
  - OSD servicing reads for a PG could be declared failed
  - yet, OSD is still reachable

How do we prevent reads operation being processed by old OSDs?, Heartbeat
messages:
  - Peer OSDs (the OSD responsible for PG) needs to share heartbeat messages
    with each other for the PG to remain redable
  - If OSD servicing reads, does not get heartbeats from other replicas in H
    seconds, it will block reads
      - This makes sure that, we do not serve if any of the participating OSD is
        down or was moved away from PG membership
  - Before another OSD takes over primary, it should receive a positive ACK from
    old primary

## 3.3 Failure Detection
- RADOS uses point-to-point message passing libary for communication
    - a pattern in which producers send messages typically intended for a single
      consumer.
    - sender <--> queue <--> receiver
    - basically means any message b/w two OSD is unique
- Failure, unable to connect to a TCP socket after limited number of reconnect
  attempts
- Failure is reported to Monitor
- Nodes exchange Heartbeat messages with their peers to detect device failures
    - i.e if heartbeat messages fails, the OSD knows something is wrong and
      reports to monitor
- OSD who discover they are marked down, kill themselves

## 3.4 Data Migration and Failure Recovery