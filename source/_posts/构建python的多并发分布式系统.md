---
title: 构建python的多并发分布式系统
date: 2018-06-27 10:29:46
tags: [多并发, python, celery, multiprocessing]
---
### IO密集型的应对
我们一直致力于采用flask（纯python编写的轻量级web框架）来构建快速开发的微服务应用，并将其应用于工作中项目中。    
部署时，采用Gunicorn或者uWSGI作为http服务器，因其支持gevent和monkey_patch，立刻就能使同步的web框架获得数倍的性能提升。（*注 gevent对标准I/O函数做了monkey_patch，把它们变成了异步。并且gevent有greenlet对象能够被用于并发执行。）   

### CPU密集型的应对
然而真实的应用场景下，无论是CPU密集型运算或者是庞大的I/O操作，都会对服务器主机造成相当大的负担。    
比如在我们当前的项目里，有一个很重要的模块需要做算法进行复杂的数学运算。在运算过程中服务器会跑满cpu，并且运算时间很长，大概70秒才可以做完一次处理。如果单纯的将http服务器的worker提升至服务器cpu数量，多并发的request数量也会及其有限，并且压上去的request越多，导致阻塞的时间就会越长，不仅用户体验不好，而且超过了nginx配置的最大连接时间，也会报错。（*注 如果架构中使用了nginx做反向代理的话）  
那么我们将问题梳理一下。在CPU密集型应用中，如何提升服务器性能？（*注 IO密集型的问题，已经采用了Gunicorn或者uWSGI的http服务器帮我们单纯应对了。）    
CPU密集型的问题，我们采用了celery，redis（也可以采用rabbitmq)，及multiprocessing来解决。    
首先我们将CPU密集型的运算从同步处理中拉出来，做成单个的任务模块，扔进消息队列里，然后异步返回给客户端。    
其次将服务器主机集群都注册成统一的消息中间件和任务结果存储地址。这样在运行过程中可以根据任务的多少，动态的可插拔式的添加分布式系统的节点，或者删除节点。
最后对于单节点应用，我们采用了multiporcessing做分核处理，比如单主机单进程一个CPU跑满大概需要70秒，分成4核做并行处理，则会大概花18秒左右跑出结果。
这样，基于微服务的可扩展的多并发分布式系统就构建完成了。其他开发语言的拆分过程思想同理。    
