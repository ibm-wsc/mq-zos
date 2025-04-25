# Connecting IBM MQ for z/OS to Kafka as a client

#### **Audience level**
Knowledge of IBM MQ for z/OS or Linux
#### **Skillset**
Kafka, IBM MQ for z/OS, SSL/TLS

#### **Background**

If you are looking to gain familiarity with connecting IBM MQ for z/OS to Kafka for event-streaming, this lab is a great place to start. In this lab, we will walk through configuring the open-source Kafka Connector to demonstrate how to capture z/OS events with a standalone Kafka instance. Businesses are looking to capture the valuable insights on z/OS with events, using Kafka. As an MQ administrator, this lab will help you become comfortable with the Kafka architecture. This lab is for development and test purposes only.

#### High-level architecture

![MQ Kafka via Client mode diagram](assets/kafka-mq-clients.png)

#### Pre-requisities
1. Java running on Windows Subsystem for Linux
    
    ```
    openjdk version "21.0.6" 2025-01-21
    OpenJDK Runtime Environment (build 21.0.6+7-Ubuntu-122.04.1)
    OpenJDK 64-Bit Server VM (build 21.0.6+7-Ubuntu-122.04.1, mixed mode, sharing)
    ```

2. Apache Maven running on Windows Subsystem for Linux

    ```
    Apache Maven 3.6.3
    Maven home: /usr/share/maven
    Java version: 21.0.6, vendor: Ubuntu, runtime: /usr/lib/jvm/java-21-openjdk-amd64
    Default locale: en, platform encoding: UTF-8
    OS name: "linux", version: "5.15.167.4-microsoft-standard-wsl2", arch: "amd64", family: "unix"
    ```

3. Apache Kafka downloaded from [here](https://kafka.apache.org/downloads) in binary format from (I am using kafka_2.13-3.9.0)

4. IBM MQ for z/OS. I am using IBM MQ version 9.4. You must use a version of MQ beyond v.8.

5. A SVRCONN channel on a z/OS queue manager running using SSL/TLS. Mine is called USER1.SSL.SVRCONN.


### LAB UNDER CONSTRUCTION, CHECK BACK SOON!
<!-- 
I. Configure Apache Kafka
II. Run the MQ-Kafka source connector
III. Send messages from MQ to Kafka
IV. Stopping Apache Kafka

Stop any Kafka Connect workers and console producers and consumers that you may have left running.

Then, in the Kafka root directory, stop Kafka and ZooKeeper:
``` shell
bin/kafka-server-stop.sh
bin/zookeeper-server-stop.sh
```

**Tech Tip:** Make sure Kafka is fully stopped before stopping ZooKeeper.
-->