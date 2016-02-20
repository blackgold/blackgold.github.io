---
layout: post
title: FailOverResolver
---

Often the software we develop depends on some external service like a database, fileserver etc. To reach to healthy and best service (fastest response time, least loaded etc ) we typically use a loadbalancer or use dns as loadbalancer. loadbalancer service is good but you make decisions on how the laodbalancer service sees the external secive you depend on. The loadbalancer service may be in a difference network and what it sees as a best server maynot be the best server from your network. Failover Resolver helps your software determine the best server from the box on which your software is running. 


https://github.com/blackgold/FailOverResolver

---
