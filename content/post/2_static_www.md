+++
date = "2018-09-10T23:55:19-07:00"
title = "Week 2: Static Web Site"

+++

With the DNS in place, let’s design the most ubiquitous service on the Internet: **a website**. In this task, we are specifically designing a static website that fits the following criteria:

1. **Read-only content**
When a user accesses the static website, the content is read-only. Users cannot modify or add data to the server.

2. **URL is the direct reflection of a file on the server**
The service serves a file, such as an HTML file or JPEG image, as it is stored on the server. There is no extra processing involved.

Most of the early Internet websites fit into this category; the advent of client-side JavaScript frameworks have made rich content possible without any server-side rendering. In fact, static site generators such as Hugo or Jekyll are making static sites popular again.

**[Requirements]**

Starting a website is simple: run a web server and throw in some HTML files, you get a website running in a few minutes. However, our focus isn't just about getting a website to work, we want the website to scale. Among all popular websites, the one that is most closely associated with a static website is Wikipedia. My goal is to build a static website that can handle the amount of traffic similar to Wikipedia, but without the user edit functionality.

**[Design]**

We plan to handle about as much traffic as Wikipedia, which is around 15 billion page views per month. If each page view turns into an average of 10 requests to the web server, this is equal to 60,000 queries per second (QPS). Besides system load, we also need to consider the amount of storage required to keep all the static files. We turn to Wikipedia statistics to estimate how much storage is needed. As of 2018, there are approximately 46 million articles. With the 300 word average article length, this gives us about 13.8GB text data, a surprisingly small number. However, we need to consider keeping images and media files as well. If we assume 1MB file to be attached with a document, this is about 46TB in total.

Let's sum up the target statistics:

* 60,000 QPS
* 46 million documents
* 46 TB storage (plus backup)

Just how many servers do we need for 60,000 QPS?

QPS is inversely proportional to the time to process a request (TP). The total TP for a request consists of 3 parts: disk seek to find the object, sequential disk read, and processing delay (including network delay). The disk seek is slow, typically on the order of 10ms. The sequential read on 1MB is around 1ms. Processing delay depends on CPU power and algorithm design. In this particular case, the delay should be minimal, since we do not render the data. But to be pessimistic about our system performance, we say that the processing delay is also about 1ms. This brings the total TP to:

    TP = 10ms + 1ms + 1ms = 12ms

The performance with this TP is really bad, about 80 QPS per server. We need about 800 servers to scale this load. In this calculation, we assume a single thread of operation per server. This is not realistic. With asynchronous event-based architecture, a query does not need to wait for the initial seek time to complete before processing the next request, essentially eliminating the seek time out of the TP equation. With that consideration, TP should consist of CPU processing time (such as memory copy and running code), but not I/O idle time (such as disk seek or network delay).

The revised TP calculation:

    TP = 1ms + 1ms = 2ms

This gives us 500 QPS, or 120 servers to handle the load. This is much better. Wikipedia uses about 300 servers to process 60,000 QPS, but mostly because there are a lot more data to pull from for each request.

With static content, we can mostly get away with a network load balancer without session affinity. However, adding session affinity does help with caching, because the same session has a better chance of requesting the same set of resources. Another consideration is that with static content, caching can help a great deal in lowering the overall system performance. 

With these in mind, we will use an application load balancer with session affinity to front the web servers.

As far as the web server implementation is concerned, the most popular choices are Apache and Nginx. Both have large install bases and community support, and both are very reliable. The fundamental difference between Apache and Nginx is how they handle concurrent requests. For Apache, each session is handled by a dedicated thread allocated from a thread pool. Nginx uses an event-driven asynchronous architecture that uses significantly lower resources with a large number of concurrent connections. With our static website designed to serve contents all around the world, I believe Nginx will be a better choice.

With that said, I am actually going to build my own HTTP server in order to gain insights into how caching works, even though Nginx is the more logical choice in production. My simple HTTP server is written in Golang with in-memory caching using memcached. The number of HTTP servers will grow according to CPU load. This is called Horizontal Pod Autoscaling (HPA) in Kubernetes, where the number of containers can be increased or decreased according to the load on each container.

Because caching is a good indicator of the system performance, we want a way to monitor the cache efficiency. One way to do it is to have a separate monitor service polling data from each web server. To do so, the monitor service needs to know how many web servers are currently running, and what are their IP addresses. This gets tricky with auto-scaling where containers come and go dynamically.

A better way is to use a message broker. A message broker is an intermediate service between the senders and receivers. In this case, a web server sends data to a message queue, and the monitor service receives data from the message queue. Both sender and receiver do not need to directly know about each other; they communicate via a named channel. There are many open-source message broker implementations. In this task, RabbitMQ is chosen for its excellent online documentation and Golang binding.

The next consideration is the file system. As web server automatically scales based on load, every web server should have a consistent view of the file system.

For example, the URL **http://www.rebuildtheinternet.com/images/puppy.jpg** should reference the file **images/puppy.jpg** in the file system, regardless of which web server the load balancer forwards request to. In addition, I want to avoid having one slow disk to become the performance hot spot. With these considerations, a distributed file system seems like a viable choice. But which one? An orchestration platform like Kubernetes provides file system abstraction via Volumes. On my Minikube simulation environment, the only choice is using the local file path. In production, both Ceph and Gluster FS can provide good scalability solutions.

**[System Architecture]**

We are now ready to design the static web server:

![Static Web Site System Architecture](/images/task2_webserver.png)
Figure 1: Static Web Server Architecture

The system is fronted by a load balancer with session affinity. A number of web servers are auto-scaled based on CPU utilization. The content comes from the distributed filesystem and cached in memory using memcached. Caching statistics (hit vs. miss) is monitored with each web server sending its cache decision to a message queue. A separate monitor service subscribes to this message queue and compiles this data.

The results are available on [GitHub](https://github.com/hc6internet/rebuildtheinternet/tree/master/task2).

**[Lessons Learned]**

1. Reduce dependency in each application

    A lot of time was spent on connecting the web server to local memcached, and running the container with RabbitMQ client. As the number of dependencies increase, so does the complexity of each application. Reducing dependency will certainly make any task more efficient. On a separate note, making each application single purpose will also make things easier in the long run. 

2. Separate data from code

    My initial thought on cache statistics is to expose REST API on the web server and have cache monitor to poll data from them. Aside from the fact that this is difficult with auto-scaling, cache data is now dispersed among many web servers. The first problem with this approach is how often do I have to poll each web server. I quickly realize that there is almost no way to make sure that data is not lost. This makes me realize that data should be better off separated from the code that processes it. In this case, the web server immediately punts the data out to a message broker. This separation not only makes auto-scaling works so much easier; it also allows dependent pods (web servers and cache monitor) to be started up independently.

**[References]**

1. *Alexa ranking* - <https://www.alexa.com/topsites>

2. *Wikipedia Statistics* - <https://stats.wikimedia.org/v2/#/all-projects>

3. *Apache vs. Nginx: Practical Considerations* - <https://www.digitalocean.com/community/tutorials/apache-vs-nginx-practical-considerations>

4. *Run a scalable, available and cheap static site on S3 or Github* - <http://highscalability.com/blog/2011/8/22/strategy-run-a-scalable-available-and-cheap-static-site-on-s.html>

5. *Big data storage wars: Ceph vs Gluster* - <https://technologyadvice.com/blog/information-technology/ceph-vs-gluster/>

6. Exploring message brokers: RabiitMQ, Kafka, ActiveMQ, and Kestrel - <https://dzone.com/articles/exploring-message-brokers>



