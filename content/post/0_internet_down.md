+++
date = "2018-08-23T23:13:50-07:00"
title = "Week 0: When the Internet goes dark"
tags = ["technology", "scalability", "internet", "education", "microservice"]

+++

The Internet seems like a party that never ends. Services appear to be always on and is available whenever we need it, and wherever we need it. Most of us take the Internet for granted; we don’t know what to do when we are not connected.

But what if it does go down? What if all your favorite services disappear overnight, and you, only you have the power to restore the Internet?

Do you know where to start? Do you know how to start?

Those are the questions I have been asking myself. And in my search to answer these questions, I have decided to take on the challenge of rebuilding Internet services one-by-one, in the span of a year. Of course, it’s impossible to build the entire Internet from scratch. It took very many incredibly smart people, countless hours, immense amount of money and iterations of trial-and-error for the Internet to exist as it is today. So for this exercise to be meaningful, I have to limit the scope.

**[Scope]**

The assumptions are:

1. The Internet infrastructure is still intact. The services may not be available, but switches, routers, and other network infrastructure components are still there for me to build services as I see fit. 
* Services have gone down, but data has been preserved. I assume that all the data are still somewhere in some format freely available to me. All the opensource codes are not lost, and I can recreate services assuming that all the modern tools are freely available.
* It’s the flow of data that counts, not the presentation. For example, if I am to rebuild Facebook, I can claim victory when the essential data flow and information exchange are in place, rather than having an exact replica of the web page.

To rebuild the Internet, I will try to bring back services one by one assuming every time I turn on a service, everyone will start using it. Therefore, scalability and reliability are primary concerns.

**[Tools]**

Open source tools will be used to create and simulate the services. As much as possible, I will create each service using containers and open source orchestration tools. I choose the combination of Docker and Kubernetes for their popularity and the fact that I can simulate a simple Kubernetes cluster (minikube) right on my laptop. Most of the core services will be directly coming from Github. The primary programming languages will be either in Golang, Python or NodeJS.

Each task will hopefully last about a week, while more complicated services will last longer. At the end of the 52-week period, I hope to have a mini-Internet that mimics most of the services I rely on.

Now, let’s switch off the light on the Internet and start rebuilding it.
