+++
title = "Week 3: Analytics Service"
date = "2018-09-21T22:35:37-07:00"
tags = ["rebuild", "technology", "scalability", "internet", "education", "microservice", "network", "software", "programming", "system"]
+++

In this task, we are trying to build a simple **site analytics service**. An analytics service provides insights into how users access your content. An example of such a service is [Google Analytics](https://analytics.google.com/analytics/web/).

This task is different in that in the previous tasks, we have been mostly focused on read-only data. This week, we are going to look at how "write" affects design decisions.

**[Requirements]**

In addition to the usual scalability requirements, the analytics service should do the followings:

1. **Collect basic information about user traffic**

	Information collected includes the number of visitors, which page do they access, type of browsers, and how does a user find this page. In this exercise, we are focusing on only the basic information, rather than building a comprehensive analytics service.

2. **Be lightweight and invisible**

	We don't want the analytics service to add burden to the website. This means that it should not affect the viewing experience, and should not affect loading time. It should also be easy for the site owner to add the capability. Lastly, it should work on all browsers without special features enabled on the browser.

To limit the scope of the problem, we will focus on the data collection aspect, not monitoring and analysis. The system will collect data from users of the websites that signed up for the service and store them in a database.

**[Design]**

Let's first look at how to collect user information. To be clear, a **user** means the person who browses the website, and **site** refers to the website the user accesses. It is the **site owner** that signs up for the analytics service, but the service collects data about **how users access the site**. 

Let’s consider how a site owner uses our service. First, a site owner signs up and obtain a unique Site ID. This site ID is used to embed a piece of code into their website that is triggered by a page access. When users access the website, the embedded code collects information and send it to the analytics service along with the site ID.

With JavaScript ubiquitously used on almost all browsers, we can use a JavaScript code snippet to collect the following information directly on the browser.

* URL
* User-agent
* Time of access
* URL Referrer

URL tells us which page user accesses, assuming the site is serving static content. For sites that do massive server-side rendering, the URL may not correctly reflect the page user accesses. For this task, it's good enough to just know the URL. User-agent tells us what type of browser user uses to access the page. It's an interesting data point to understand what platform and browser user uses. Time of access is recorded by the JavaScript to more accurately reflect the time of access. Lastly, the URL referrer tells us how this user arrives at the current page, either a link from another website or as a result of search engine.

One missing piece of information is the **user**. Since the code snippet runs on the browser, it is the user that makes a connection to the analytics service, not the site. Therefore, we can identify the user by the source IP address of the connection.

One way of writing data to a web-based service is to use HTTP POST request. However, due to Cross-Origin Resource Sharing (CORS) restrictions enforced on most modern browsers, and the fact that the website and analytics service are on different domains, this approach won't work.

Instead, we add a single-pixel image to the website, use JavaScript to first collect relevant data, and rewrite the image URL to the analytics server. All the user information are appended to the URL as query strings. The web page loads the image, and because it is single-pixel, it is not visible on the website. Loading a third-party image URL is allowed by almost every browser, so this approach works very well. The image URL approach is possible because sending analytics data is a one-way street; the user sends the data to the server without expecting any information in return.

An image URL, after rendered by the JavaScript code looks something like the following:

	<img src="http://analytics-service/analytics.gif?
	ID=12345678&UserAgent=ChromeBrowser&
	URL=https%3A%2F%2Fwww.rebuildtheinternet.com%2F>

The above URL indicates that this embedded link is coming from site ID "12345678", the user accesses the page on a Chrome browser, and the URL is "https://www.rebuildtheinternet.com/". 

We now look at how to build the analytics service. The service responds to one simple gif file request, without actually serving any content. The information we need are coming from the query strings. After these information are collected, we need to write it to a database for monitoring. The database is replicated to a read-only copy, where data will be retrieved by the monitoring system when requested by the site owner.

Since it is the user that provides (write) the data, and the site owner that monitors (read) the data, we can replicate the database without worrying about “read-your-own-write” problem caused by replication lag. The reader and writer are not the same person in our case.

Another potential problem with replication lag is “monotonic read”. This problem arises when data from 2 distinct reads come from 2 different replicas, possibly results in data discrepancy. To address this problem, we partition the data based on the site ID. This way, the site owner who queries the database by site ID always queries from the same replica, thereby avoid the monotonic read problem. Partitioning the data this way also makes it easier to scale when more and more sites sign up for the service.

Partition helps with scaling the service. But what is our partition strategy? In this particular task, as long as we can make sure that a particular site ID is written and read from the same partition, we are okay. This leads us to using a MD5 hash on the site ID as the hash key:

	PartitionID = MD5(SiteID) % N

Where N is the total number of partitions.

This algorithm is simple enough that it can be built into the analytics server’s logic. Hashing has the downside where database query becomes difficult for range-based queries. Fortunately for us, a site owner reading data will not be performing range-based queries based on site ID.

This strategy works fine until 2 things happen:

1. The analytics service grows so popular that N needs to increase. This requires all the application logic on the analytics servers to change. 
* One of the sites grows so popular that a partition needs to accommodate that, perhaps giving that site its dedicated database.

In both cases, we can update the application code and restart the analytics servers. It's not hard to do, and with proper image versioning and careful planning, there may not even be much downtime. However, it's still an operational hassle.

In scenario #1, we also need to know that site X has been stored in partition Y, so either the application logic "remembers" where each site should be or rearrange data to accommodate that change.

Let’s discuss scenario #2 more carefully. Typically a service needs to scale when it gets popular. A site analytics service has an interesting property that its service needs to scale when its customer’s service gets popular. Fortunately, because we are analysing their service, we can build a simple model to predict that a site will grow exponentially in the near future. In response to the future growth, we can modify our partition strategy to better fit the future growth of some superstar site. This prediction requirement implies that we are not simply scaling based on CPU utilization, but based on the data we collected. 

With these considerations, we will create a separate service called **partition service** to decide which database partition to write to. The front-end analytics server always queries the partition service, and write to the database it is assigned to.

This requirement also tells us that the communication between the analytics server and the partition service is going to be very busy. To accommodate that, the communication mechanism needs to be both efficient and reliable.

Let’s first look at efficiency. Since the analytics server and the partition service typically lies in the same data center, the round trip network delay is very short. However, when the incoming message rate is high, any extra processing delay will add up and eventually becomes the system bottleneck. To reduce delay, we use **binary encoding format** to send and receive data. We use **Protocol Buffer** to define the message format and compile it into a language-specific file. Using Protobol Buffer reduces efficiency in two ways. First, it is binary encoded, so data received can be directly mapped into a data structure, reducing the need for parsing. Second, the payload footprint is small. Unlike REST API where HTTP is the transport mechanism, Protocol Buffer directly runs on top of TCP.

Another requirement is reliability. A TCP connection is reliable in the sense that each byte needs to be acknowledged by the receiver. However, since TCP is a stream-based protocol, using TCP means that the application code needs to consider message boundaries, otherwise the service can receive partial, or multiple message contents in a single packet. To reducthe e complexity of application code, we use **ZeroMQ** to ensure a message is always received in full.

In production, we may want to avoid using the simple modulus function as a partition strategy. This strategy will require partition rebalancing operation to move around a large amount of data. When the system is already under load (the reason for rebalancing in the first place), moving around a large amount of data will make the problem even worse. One strategy is to use a larger partition value N than the number of nodes. This way, when the number of nodes increases, we can pick just a portion of all partitions that are currently assigned to each node to the new node, thereby reducing the amount of data movement.

**[System Architecture]**

Let’s now put everything together to build the analytics service:

![Analytics Service Architecture](/images/task3_analytics.png)
Figure 1: Analytics Service Architecture

The analytics service will be auto-scaled based on CPU utilization. The number of databases will scale manually based on the size of the data. When scaling happens, the partition service will be updated to reflect the database number. Each database is replicated to at least one replica for monitoring purposes. A monitor request selects the replica based on the site ID. This monitor service will also periodically check and predict if a site will grow exponentially in the near future; in this case, it notifies the partition service to update its algorithm.

**[Code Structure]**

The resuls are available on [GitHub](https://github.com/hc6internet/rebuildtheinternet/tree/master/task3).

* browser-code:
	The JavaScript code snippet that connects to the analytics server. In production, this code should be compacted into just a couple lines for ease of deployment.

* proto:
	The Protocol Buffer message format for the communication between analytics server and partition service. The resultant Go file is copied into both "partition-select" and "analytics-server" folders.

* analytics-server:
	The analytics server, written in Go, is used to collect information when a request to "analytics.gif" is received.

* partition-select:
	The partition service reads messages from the analytics server (via ZeroMQ) and responds with the database partition ID. 

**[Lessons Learned]**

1. Partition is a hard problem

	One of the big challenges is about rebalancing partitions. When a partition is skewed due to an exceptionally popular site, we may need to assign more resource to that site. This causes skewed partition and requires partition rebalance. Rebalance moves data around databases and will possibly incur downtime. In this task, partition rebalance is only briefly mentioned, but not implemented simply because it is a big topic by itself. We will get back to it at a later time.

2. Functionality over scalability 

	Scalability is definitely important. But a service without the proper functionality does not have scalability issue to speak of; there just won't be enough user. When user base starts to grow, we can always think about how to scale the system, provided the original design is reasonably flexible to scale. Even though this is an exercise about building scalable services, I should still refrain from building an infinitely scalable service. Nothing will get done with that mindset.

3. Users should be informed of collecting information

	Any site that collects user information needs to inform the user. This is due to the recent GDPR compliance requirement. Strictly speaking, this is the responsibility of the site owner, not the analytics service provider. We mention that because this service is about collecting user data, but if you use browser cookie in any way even if you are not monetizing based on user data, it is still your responsibility to inform the user. 

**[References]**

1. *Tracking Code Overview* - <https://developers.google.com/analytics/resources/concepts/gaConceptsTrackingOverview>

2. *Cross-Origin Resource Sharing* - <https://en.wikipedia.org/wiki/Cross-origin_resource_sharing>

3. *Google Protocol Buffers* - <https://developers.google.com/protocol-buffers/>

4. *ZeroMQ* - <http://zeromq.org/>

5. *Go interface for zmq version 4* - <https://github.com/pebbe/zmq4>

6. *Designing Data-Intensive Applications* by *Martin Kleppmann*
