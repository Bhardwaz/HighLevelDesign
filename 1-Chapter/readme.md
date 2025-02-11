## Single Server Setup
A journey of a thousand miles begins with a single step.

To start with something simple, everything is running on a single server.
![Single Server Setup](assests\single-server.png)

traffic to your web server comes from two sources: web application and mobile application.

## Database
With the growth of the user base, one server is not enough, and we need multiple servers: one for web/mobile traffic, the other for the database.

Separating web/mobile traffic (web tier) and database (data tier) servers allows them to be scaled independently.
![Single Server Setup](assests\OIP.jpg)

## Which databases to use?
depend upon data weather your data is relational data or not. weather you want ACID
properties or not. 

## Vertical scaling vs horizontal scaling
Vertical scaling, referred to as “scale up”, means the process of adding more power (CPU, RAM, etc.) to your servers. Horizontal scaling, referred to as “scale-out”, allows you to scale by adding more servers into your pool of resources.

## 🔀 Load balancer
A load balancer evenly distributes incoming traffic among web servers that are defined in a load-balanced
![Load Balncer](assests\setup-with-db-replication.png)

users connect to the public IP of the load balancer directly. With this setup, web servers are unreachable directly by clients anymore. For better security, private IPs are used for communication between servers. A private IP is an IP address reachable only between servers in the same network; however, it is unreachable over the internet. The load balancer communicates with web servers through private IPs.

## Database replication
Quoted from Wikipedia: “Database replication can be used in many database management
systems, usually with a master/slave relationship between the original (master) and the copies (slaves)”.
![Database replication](assests\database-replication-diagram.webp)

Advantages of database replication:
* Better Performance * Reliability * High availability

In this technique, we make copy of databases. In these all copies one becomes master db and other become slaves. Master db handle write(delete, update, add) opertation and slaves handle read operation. 

Because read operations are way more then write. So we handle ratio like this.

We can make more then one master db as well and you can read that on internet. That process is complicated so we will focus on one maser db

So if there is one Master db then Single point of failure comes in picture. it handles that preety good. if this happens, one slave get promoted to master db and it make another slave copy of database. Until everything not becomes fine.


In production systems, promoting a new master is more complicated as the data in a slave database might not be up to date. The missing data needs to be updated by running data recovery scripts. Although some other replication methods like multi-masters and circular replication could help, those setups are more complicated; and their discussions are beyond the scope

If only one slave database is available and it goes offline, read operations will be directed to the master database temporarily. As soon as the issue is found, a new slave database will replace the old one. In case multiple slave databases are available, read operations are redirected to other healthy slave databases. A new database server will replace the old one.

## Workflow
Let us take a look:
• A user gets the IP address of the load balancer from DNS.
• A user connects the load balancer with this IP address.
• The HTTP request is routed to either Server 1 or Server 2.
• A web server reads user data from a slave database.
• A web server routes any data-modifying operations to the master database. This includes write, update, and delete operations.

## Cache 
A cache is a temporary storage area that stores the result of expensive responses or frequently accessed data in memory so that subsequent requests are served more quickly. The application performance is greatly affected by calling the database repeatedly. The cache can mitigate this problem.

For Example - Reddis

## Cache tier
The cache tier is a temporary data store layer, much faster than the database. The benefits of having a separate cache tier include 
better system performance, 
ability to reduce database workloads, 
and the ability to scale the cache tier independently.

Considerations for using cache
 Consider using cache when data is read frequently but modified infrequently. Since cached data is stored in volatile memory, a cache server is not ideal for persisting data. For instance, if a cache server restarts, all the data in memory is lost. Thus, important data should be saved in persistent data stores.

• Expiration policy. 
It is a good practice to implement an expiration policy. Once cached data is expired, it is removed from the cache. When there is no expiration policy, cached data will be stored in the memory permanently. It is advisable not to make the expiration date too short as this will cause the system to reload data from the database too frequently.Meanwhile, it is advisable not to make the expiration date too long as the data can become stale.

### Consistency: 
This involves keeping the data store and the cache in sync. Inconsistency can happen because data-modifying operations on the data store and cache are not in a single transaction. When scaling across multiple regions, maintaining consistency between the data store and cache is challenging. For further details, refer to the paper titled “Scaling Memcache at Facebook” published by Facebook

### Mitigating failures: 
A single cache server represents a potential single point of failure (SPOF), defined in Wikipedia as follows: “A single point of failure (SPOF) is a part of a system that, if it fails, will stop the entire system from working” [8]. As a result, multiple cache servers across different data centers are recommended to avoid SPOF. Another recommended approach is to overprovision the required memory by certain percentages.
This provides a buffer as the memory usage increases.

### Eviction Policy: 
Once the cache is full, any requests to add items to the cache might cause existing items to be removed. This is called cache eviction. Least-recently-used
(LRU) is the most popular cache eviction policy. Other eviction policies, such as the Least Frequently Used (LFU) or First in First Out (FIFO), can be adopted to satisfy different use cases.

## CDN 
A CDN is a network of geographically dispersed servers used to deliver static content. CDN servers cache static content like images, videos, CSS, JavaScript files, etc.

## Considerations of using a CDN
### Cost: 
CDNs are run by third-party providers, and you are charged for data transfers in and out of the CDN. Caching infrequently used assets provides no significant benefits so you should consider moving them out of the CDN.

### Setting an appropriate cache expiry: 
For time-sensitive content, setting a cache expiry
time is important. The cache expiry time should neither be too long nor too short. If it is too long, the content might no longer be fresh. If it is too short, it can cause repeat reloading of content from origin servers to the CDN.
 
### 🌍 CDN fallback: 
You should consider how your website/application copes with CDN
failure. If there is a temporary CDN outage, clients should be able to detect the problem and request resources from the origin.

### Invalidating files: 
You can remove a file from the CDN before it expires by performing one of the following operations:
• Invalidate the CDN object using APIs provided by CDN vendors.
• Use object versioning to serve a different version of the object. To version an object, you can add a parameter to the URL, such as a version number. For example, version number 2 is added to the query string: image.png?v=2

1. Static assets (JS, CSS, images, etc.,) are no longer served by web servers. They are fetched from the CDN for better performance.
2. The database load is lightened by caching data

## Stateless web tier
A good practice is to store session data in the persistent storage such as relational database or NoSQL. Each web server in the cluster can access state data from databases. This is called stateless web tier.

![Sticky sessions](assests\Diagram--1-.jpg)
user A’s session data and profile image are stored in Server 1. To authenticate
User A, HTTP requests must be routed to Server 1. If a request is sent to other servers like Server 2, authentication would fail because Server 2 does not contain User A’s session data. Similarly, all HTTP requests from User B must be routed to Server 2

The issue is that every request from the same client must be routed to the same server. This can be done with *sticky sessions* in most load balancers. however, this adds the *overhead*. 
❌ *Adding or removing servers is much more difficult with this approach. It is also challenging to handle server failures.*

## Stateless architecture
HTTP requests from users can be sent to any web servers, which fetch state data from a shared data store. State data is stored in a shared data store and kept out of web servers. A stateless system is simpler, more robust, and scalable.

![Stateless architecture](assests\f8c96ab3-03c4-4935-9165-8e814e380b8a_1233x721.jpg)

we move the session data out of the web tier and store them in the persistent
data store. The shared data store could be a relational database, Memcached/Redis, NoSQL, etc. The NoSQL data store is chosen as it is easy to scale. Autoscaling means adding or removing web servers automatically based on the traffic load. After the state data is removed out of web servers, ⚖️ *auto-scaling of the web tier is easily achieved* by adding or removing servers based on traffic load.

🤔 As we move out state out outside the web tier. it becomes relatively easy to scale 
our system. As the state is no longer exist on web tier. we can skip about user authorization bcz storing a state in one server can hinder the stability then we start to think about user's state as well. in this case we need to shuffle state acroos the servers and its hard and slows down the scalability of system 

## Data Centers
users are geoDNS-routed, also known as geo-routed, to the closest data center, with a split traffic of x% in US-East and (100 – x)% in US-West. geoDNS is a DNS service that allows domain names to be resolved to IP addresses based on the location of a user.

In the event of any significant data center outage, we direct all traffic to a healthy data center.
data center 2 (US-West) is offline, and 100% of the traffic is routed to data
center 1 (US-East)

Several technical challenges must be resolved to achieve multi-data center setup:

### 🧠 Traffic redirection: 
Effective tools are needed to direct traffic to the correct data center.
GeoDNS can be used to direct traffic to the nearest data center depending on where a user is located.

### 🧠 Data synchronization: 
Users from different regions could use different local databases or caches. In failover cases, traffic might be routed to a data center where data is unavailable.
A common strategy is to replicate data across multiple data centers. A study
shows how Netflix implements asynchronous multi-data center replication

### 🧠 Test and deployment: 
With multi-data center setup, it is important to test your website/application at different locations. Automated deployment tools are vital to keep services consistent through all the data centers.

To further scale our system, we need to decouple different components of the system so they can be scaled independently. Messaging queue is a key strategy employed by many real world distributed systems to solve this problem.

## Message Queue
The basic architecture of a message queue is simple. Input services, called producers/publishers, create messages, and publish them to a message queue. Other services or servers, called consumers/subscribers, connect to the queue, and perform actions defined by the messages.

## Database scaling
### 🚀 Vertical Scaling 

According to
Amazon Relational Database Service (RDS) [12], you can get a database server with 24 TB of RAM. This kind of powerful database server could store and handle lots of data. For example, stackoverflow.com in 2013 had over 10 million monthly unique visitors, but it only had 1 master database [13]. However, vertical scaling comes with some serious drawbacks:

🔴 You can add more CPU, RAM, etc. to your database server, but there are hardware
limits. If you have a large user base, a single server is not enough.
🔴 Greater risk of single point of failures.
🔴 The overall cost of vertical scaling is high. Powerful servers are much more expensive.

### 📈 Horizontal Scaling
Horizontal scaling, also known as sharding, is the practice of adding more servers.

Sharding separates large databases into smaller, more easily managed parts called shards. Each shard shares the same schema, though the actual data on each shard is unique to the shard.
shows an example of sharded databases. User data is allocated to a database server based on user IDs. Anytime you access data, a hash function is used to find the
corresponding shard. In our example, user_id % 4 is used as the hash function. If the result equals to 0, shard 0 is used to store and fetch data. If the result equals to 1, shard 1 is used.
The same logic applies to other shards.

The most important factor to consider when implementing a sharding strategy is the choice of the sharding key. Sharding key (known as a partition key) consists of one or more columns that determine how data is distributed. As shown in Figure 1-22, “user_id” is the sharding key. A sharding key allows you to retrieve and modify data efficiently by routing database queries to the correct database. When choosing a sharding key, one of the most important criteria is to choose a key that can evenly distributed data.Sharding is a great technique to scale the database but it is far from a perfect solution. 
⚠️ It introduces complexities and new challenges to the system:

Resharding data: Resharding data is needed when 1) a single shard could no longer hold
more data due to rapid growth. 2) Certain shards might experience shard exhaustion faster than others due to uneven data distribution. When shard exhaustion happens, it requires updating the sharding function and moving data around. Consistent hashing, which will be discussed in Chapter 5, is a commonly used technique to solve this problem.

Celebrity problem: This is also called a hotspot key problem. Excessive access to a specific shard could cause server overload. Imagine data for Katy Perry, Justin Bieber, and Lady Gaga all end up on the same shard. For social applications, that shard will be overwhelmed with read operations. To solve this problem, we may need to allocate a shard for each celebrity. Each shard might even require further partition.

Join and de-normalization: Once a database has been sharded across multiple servers, it is hard to perform join operations across database shards. A common workaround is to denormalize the database so that queries can be performed in a single table.





















