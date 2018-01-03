# Myth源码解析系列之（一）-项目简介

## Myth 介绍

Myth 是一个基于消息队列的分布式事务开源框架, 基于java语言来开发（JDK1.8），支持dubbo，springcloud,motan等rpc框架进行分布式事务。

## 特征
* RPC框架支持 : dubbo,motan,springcloud。

* 消息中间件支持 : jms(activimq),amqp(rabbitmq),kafka,roceketmq。

* 本地事务存储支持 : redis,mogondb,zookeeper,file,mysql。

* 事务日志序列化支持 ：java，hessian，kryo，protostuff。

* 采用Aspect AOP 切面思想与Spring无缝集成，天然支持集群,高可用,高并发。

* 配置简单，集成简单，源码简洁，稳定性高，已在生产环境使用。

* 内置经典的分布式事务场景demo工程，并有swagger-ui可视化界面可以快速体验。

## 环境要求

* JDK 1.8+

* Maven 3.2.x

* Git

* RPC framework dubbo or motan or springcloud

* Message Oriented Middleware


<b>大家有任何问题或者建议欢迎沟通 ，欢迎加入QQ群：162614487 进行交流</b>
