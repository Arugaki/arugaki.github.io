---
layout: post
title: Strimzi Kafka Operator 使用记录
tags:
  - Openshift
  - Operator
  - Kafka

---

Strimzi kafka operator 介绍及其部署

### 在 Openshift 中部署

<a href="https://strimzi.io/quickstarts/okd/"> 官方部署地址 </a>

首先在 Openshift 集群中

```
oc login -u system:admin

// 创建用于部署的namespace 
oc new-project kafka

// 部署 Strimzi kafka operator 
oc apply -f https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.11.3/strimzi-cluster-operator-0.11.3.yaml -n kafka
```

此时查看 pod strimzi-cluster-operator-* 的日志发现如下报错

```
2019-05-16 02:33:33 ERROR Main:150 - Cluster Operator verticle in namespace kafka failed to start
io.fabric8.kubernetes.client.KubernetesClientException: kafkas.kafka.strimzi.io is forbidden: User "system:serviceaccount:kafka:strimzi-cluster-operator" cannot watch kafkas.kafka.strimzi.io in the namespace "kafka": no RBAC policy matched
	at io.fabric8.kubernetes.client.dsl.internal.WatchConnectionManager$2.onFailure(WatchConnectionManager.java:198) ~[cluster-operator-0.11.3.jar:0.11.3]
	at okhttp3.internal.ws.RealWebSocket.failWebSocket(RealWebSocket.java:546) ~[cluster-operator-0.11.3.jar:0.11.3]
	at okhttp3.internal.ws.RealWebSocket$2.onResponse(RealWebSocket.java:188) ~[cluster-operator-0.11.3.jar:0.11.3]
	at okhttp3.RealCall$AsyncCall.execute(RealCall.java:153) ~[cluster-operator-0.11.3.jar:0.11.3]
	at okhttp3.internal.NamedRunnable.run(NamedRunnable.java:32) ~[cluster-operator-0.11.3.jar:0.11.3]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) ~[?:1.8.0_212]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) ~[?:1.8.0_212]
	at java.lang.Thread.run(Thread.java:748) [?:1.8.0_212]
```

解决方案如下, 原因可见 <a href="https://github.com/strimzi/strimzi-kafka-operator/issues/1292"> issue </a> 

```
oc adm policy add-cluster-role-to-user strimzi-cluster-operator-namespaced --serviceaccount strimzi-cluster-operator -n kafka
oc adm policy add-cluster-role-to-user strimzi-entity-operator --serviceaccount strimzi-cluster-operator -n kafka
oc adm policy add-cluster-role-to-user strimzi-topic-operator --serviceaccount strimzi-cluster-operator -n kafka
```

执行完上述命令后删除pod后解决

```
// 部署一个单点kafka 的 demo
oc apply -f https://raw.githubusercontent.com/strimzi/strimzi-kafka-operator/0.11.3/examples/kafka/kafka-persistent-single.yaml -n kafka
```

等待集群部署成功, 最好还是查看一下zk或kafka的日志, 确认没有异常

![](https://raw.githubusercontent.com/arugaki/arugaki.github.io/master/images/strimzi_1.jpg)

测试kafka是否正常提供服务

```
// producer 
oc run kafka-producer -ti --image=strimzi/kafka:0.11.3-kafka-2.1.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --broker-list my-cluster-kafka-bootstrap:9092 --topic my-topic

// consumer
oc run kafka-consumer -ti --image=strimzi/kafka:0.11.3-kafka-2.1.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning
```

在生产者端发送消息

![](https://raw.githubusercontent.com/arugaki/arugaki.github.io/master/images/strimzi_2.jpg)

在消费者端可以接收到消息

![](https://raw.githubusercontent.com/arugaki/arugaki.github.io/master/images/strimzi_3.jpg)

