OpenADS
======
OpenADS is a Big Data analytics framework designed to consume and monitor network traffic and mine hidden anomalies using advanced machine learning techniques. In current date, OpenADS is still at it's conceptual stage where it is designed to work at a massive scale. The system believes to act as an extensible and reliable platform to enrich traditional Intrusion Detection System (IDS). OpenADS is unique at it's nature with the architecture supported by Berkeley Data Stack (BDS).

Contents
--------
* [Features](#features)
* [System requirements](#system-requirements)
	* [Dependencies](#dependencies)
	* [Platforms](#platforms)
* [Important prerequisites](#prerequisites)
	* [Pcap](#pcap)
	* [Syslog](#syslog)
	* [Spark](#spark)
	* [Zeppelin](#zeppelin)
* [How to use](#how-to-use)
	* [How to build](#how-to-build)
	* [How to run](#how-to-run)
	* [How to use Docker](#how-to-use-docker)
* [Contacts](#contacts)

Features
--------
* Streaming computation runs on Spark platform.
* Capture various network data.
* Support real-time analysis via Machine Learning techniques.


System requirements
-------------------

#### Dependencies ####
* Java 1.8
* libpcap 1.1.1
* WinPcap 4.1.2
* jna 4.1.0
* slf4j-api 1.7.12
* logback-core 1.0.0
* logback-classic 1.0.0
* rsyslogd 8.16.0
* zeppelin 0.6.0

#### Platforms ####
The software is tested on Ubuntu 16.04 LTS


Important prerequisites
-----------------------

#### Pcap ####
Pcap4j needs root's right to access network and device. So, before deploying, please ensure to run the following line:
	* `setcap cap_net_raw,cap_net_admin=eip /path/to/java`
for example, mine is `setcap cap_net_raw,cap_net_admin=eip /usr/local/java`

If you run java command now, you might receive the following error: <br/>
	java: error while loading shared libraries: libjli.so: cannot open shared object file: No such file or directory

To ensure the java can run properly, you could run the following:
	* `ln -s /usr/local/java/jre/lib/amd64/jli/libjli.so /usr/lib/` Or `echo /usr/local/java/jre/lib/amd64/jli/ > /etc/ld.so.conf`
Refer to the issue link: https://github.com/kaitoy/pcap4j/issues/63

#### Syslog ####
For Linux User: to receive syslog data from remote data source, you must do two things:
* Configure the rsyslog service in data source (after installing rsyslog in Linux):　
    * `sudo vim /etc/rsyslog.conf`
    * add `*.* @Your_IP:Your_Port;RSYSLOG_SyslogProtocol23Format`(UDP) `*.* @@localhost:514;RSYSLOG_SyslogProtocol23Format`(TCP) in the file;
    * restart the service `sudo service rsyslog restart`;
* Configure the receiver system:
    * Allow the port in your Firewall:
        * `iptables -A INPUT -p tcp -s Your_IP --dport Your_Port -j ACCEPT`;
        * `iptables -A INPUT -p udp -s Your_IP --dport Your_Port -j ACCEPT`;
    * Grant permission to java (iff Your_Port is lower than 1024, such as 514):
        * Grant Permission: `sudo setcap cap_net_bind_service+ep Your_Java_Path/bin/java`
        * Find libjli.so: `find $JAVA_HOME -name 'libjli.so'`
        * ln -s `Path2Java/lib/amd64/jli/libjli.so /usr/lib/`

For local test, just use localhost and port 514.

#### Spark ####
If you do not run the project in docker, you have to download and configure Spark by referring to the [Spark documentation](http://spark.apache.org/docs/1.6.2/).

#### Zeppelin ####
If you do not run the project in docker, you have to download and configure [Apache Zepplin](http://zeppelin.apache.org) by the following steps.

* Download
	* wget -c http://www-us.apache.org/dist/zeppelin/zeppelin-0.6.0/zeppelin-0.6.0-bin-all.tgz
	* tar -xzvf zeppelin-0.6.0-bin-all.tgz
* Avoid port conflict with Spark
	* cp zeppelin-site.xml.template zeppelin-site.xml
	* vim zeppelin-site.xml:
    ```
    <property>
    <name>zeppelin.server.port</name>
    <value>8090</value>
    <description>Server port.</description>
    </property>
	```
* Run Daemon
	* ./zeppelin-version-bin-all/bin/zeppelin-daemon.sh start

* Issues
	* When using external Spark, the notebook should first list the following:
	```
	%dep
	z.reset()
	z.load("org.apache.spark:spark-streaming-twitter_2.10:1.4.1")
	```
	Because the external Spark is lack of such configuration, and running streaming program may cause error.

	* Since we need OpenADS jar as dependency, there two ways to load such a dependency:
		* z.load("Dependency_OpenADS.jar")
		* configure settings in zeppelin-env.sh, using SPARK_SUBMIT_OPTIONS --jars.

	*In this demo, we deploy the second method. Please modify its path to the jar, the demo code works in Docker. Please refer to [dependency management](https://zeppelin.apache.org/docs/0.6.0/manual/dependencymanagement.html).*

How to use
----------

#### How to build ####
To build the project, you just need to run the `maven_package.sh` to package the project.

#### How to run ####
To run the project, you should submit the task to Spark. Below is a demo code:

* To run it locally \
```
	~/spark/bin/spark-submit \
    --class "com.scorelab.openads.receiver.PcapReceiver" \
    --master local[*] \
    ./target/OpenADS-0.1-SNAPSHOT-jar-with-dependencies.jar \
    ./configuration/config.properties
```
* To run it on servers \
```
	~/spark/bin/spark-submit \
    --class "com.scorelab.openads.receiver.PcapReceiver" \
    --master Spark_Master_Address \
    ./target/OpenADS-0.1-SNAPSHOT-jar-with-dependencies.jar \
    ./configuration/config.properties
```
* Without user-defined configuration \
The properties is optional, you could leave it alone and you could use the defaul settings, below is the example: \
```
    ~/spark/bin/spark-submit \
    --class "com.scorelab.openads.receiver.PcapReceiver" \
    --master local[*] \
    ./target/OpenADS-0.1-SNAPSHOT-jar-with-dependencies.jar
```

#### How to use Docker ####
To run it in docker, you could follow the steps below.

* Biuld Docker Container \
	docker build -t your_user/your_container_name:version.

* Run Docker Container \
	docker run -it -p 8088:8088 -p 8042:8042 -p 4040:4040 -h localhost your_user/your_container_name:version bash

* To run it on Yarn \
```
    ~/spark/bin/spark-submit \
    --files $SPARK_HOME/conf/metrics.properties \
    --class "com.scorelab.openads.receiver.PcapReceiver" \
    --master yarn-cluster \
    `Path to the jar`/OpenADS-0.1-SNAPSHOT-jar-with-dependencies.jar \
    file://`Path to the config`/config.properties\
```
&ensp;&ensp; OR:  \
```
    ~/spark/bin/spark-submit \
    --class "com.scorelab.openads.receiver.PcapReceiver" \
    --master yarn-client \
    `Path to the jar`/OpenADS-0.1-SNAPSHOT-jar-with-dependencies.jar \
    file://Path to the config/config.properties
```
* Version <br/>
Hadoop 2.6.0 and Apache Spark v1.6.0 on Centos

* Issues <br/>
	* WebUI of Spark: <br/>
	To see the webUI of Spark, you have to first run a spark job or init a spark shell. For example:\
    ```
    spark-shell \
    --master yarn-client \
    --driver-memory 1g \
    --executor-memory 1g \
    --executor-cores 4
    ```

Contacts
--------
SCoRe Lab: info@scorelab.org
Website: [http://www.scorelab.org](http://www.scorelab.org)
