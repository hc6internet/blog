+++
date = "2018-08-31T23:18:17-07:00"
title = "Week 1: Domain Name Service"

+++

What is the first thing that needs to be brought up? We need to be able to find where each service is. Given the assumption that the computers and infrastructure are still there, then the first thing we need is a name mapping service that translates a meaningful service name to its public IP address.

This brings me to the first task: build a DNS service.

**[Requirements]**

I am going to limit the scope to designing an authoritative DNS on “rebuildtheinternet.com”, rather than the entire DNS hierarchy. The goal is to build a DNS service that scales and to accommodate frequent changes as we add more services and hosts to that domain. 

**[Design]**

Suppose I am building a DNS service for a popular domain that serves 10 million queries per day. This translates to about 115 queries per second, a relatively small load that can be handled by a single machine. Therefore, the design consideration should focus more on reliability rather than scalability. To make DNS service reliable, we will have multiple name server behind a load balancer for failover. In our design, we will use 2 DNS servers behind the load balancer.

DNS runs primarily on UDP port 53, although it can also run on TCP. There is no requirement for session affinity on the load balancer, so a simple source IP based load balancer sitting in front of multiple name servers will do.

To accommodate frequent changes, we need to consider the ease of management and propagating configurations across multiple name servers. With a handful of servers, this may not seem like a big deal; but consider making changes multiple times a day, having a database backend rather than a flat file should make operation easier. Also, the service bring-up time from a large zone file will be longer than database backend due to more work in parsing the file. With these considerations, we will use a DNS implementation that uses a database backend.

When it comes to choosing a name server implementation, BIND is the most obvious choice. BIND takes a flat file that is read in during initialization, and as we have discussed earlier, is not the ideal choice. Although another argument can be made about BIND being a long-tested stable implementation. This is typically a strong point when it comes to choosing a particular implementation.

Another choice is to use PowerDNS. PowerDNS uses a database backend to store zone information, a Web interface for statistics tracking, and REST API for configuration. Since each DNS server only needs read-only access to the database, we can add read replica to the master database where PowerDNS reads from. The configuration will be centralized via REST API, and data synchronization is achieved by database replication. PowerDNS has been around since the late 1990s, and even though it is not as widely used as BIND, it is still a stable implementation with strong community support.

**[System Architecture]**

With these considerations, I choose to deploy PowerDNS with a MySQL database backend, with one read-replica per PowerDNS instance. The overall architecture looks like this:

![DNS System Architecture](/images/task1_dns.png)
Figure 1: DNS service architecture

The load balancer fronts 2 PowerDNS servers for redundancy. Each server reads from its individual MySQL read replica. One of these PowerDNS instances is used for configuration, which writes to the master MySQL database. The configuration is written via REST API provided by PowerDNS itself. A convenient Python wrapper PowerDNS-CLI is used to configure DNS zone data in the backend database. Since PowerDNS-CLI runs internally, there is no need to expose REST API to reduce the risk of being compromised.

**[Lessons Learned]**

1. Operation is critical

    This exercise makes me truly appreciate how much difference a good operations engineer can make. A lot of time was spent on operations such as creating docker images, making MySQL replicas, creating load balancers on Kubernetes, etc. Having an experienced operation engineer with strong networking background will make all the differences.

2. Don’t trust everything online

    The first PowerDNS docker file I found online didn’t work with an unpopulated database. And since I intend to configure the backend through PowerDNS’s REST front end, this is not a viable solution. Finally, I was able to find one that works, but not without wasting a few good hours.

**[References]**

1. *Scaling DNS* - <https://www.sanog.org/resources/sanog14/sanog14-devastation-dns-scalability.pdf>
* *Google Public DNS Blog* - <https://googleblog.blogspot.com/2012/02/google-public-dns-70-billion-requests.html>
* *Alpine Linux based PowerDNS docker image* - <https://github.com/pschiffe/docker-pdns>
* *PowerDNS CLI* - <https://github.com/pbertera/PowerDNS-CLI>

---
Contact me @ <hc6internet@gmail.com>

