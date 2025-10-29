# DragonflyDB, Redis and Redis-Sentinel

<img width="876" height="430" alt="image" src="https://github.com/user-attachments/assets/c0053f4f-556a-4cd2-acfb-3004307ba6c7" />


### Redis
Redis (REmote DIctionary Server) is an open source, in-memory, NoSQL key/value store that is used primarily as an application cache or quick-response database. Redis stores data in memory, rather than on a disk or solid-state drive (SSD), which helps deliver unparalleled speed, reliability, and performance.

### DragonflyDB
Dragonfly emerges as a compelling open-source alternative to traditional in-memory stores like Redis. Built for efficiency and compatibility, Dragonfly delivers exceptional performance while consuming fewer resources, making it an ideal choice for high-throughput applications. However, as your system grows, safeguarding against data loss and downtime requires more than just raw speed—it demands robust redundancy.

### Redis-Sentinel
Redis Sentinel is a distributed system designed to provide high availability for Redis deployments. It works by monitoring Redis master and replica instances, detecting failures, and automatically performing failover operations to ensure continuous service.

##  Table of Contents

1. [DragonflyDB + Sentinel Bare-Metal Setup](01-dragonflydb-bare-metal.md)
2. [DragonflyDB + Sentinel Kubernetes Setup](02-dragonflydb-kubernetes.md)
3. [Redis + Sentinel Bare-Metal Setup](03.redis-sentinel.md)
4. [HAProxy in front of Redis/Dragonfly](redis-sentinel-ha.md)

##  Feedback is very much appreciated

Based on the feedback this repo will be updated. Feel free to contribute or use this for your own study!
