# Realtime-Streaming-PCF-BDS

This demo aims to solve problem that appeared in DEBS challenge 2015

http://www.debs2015.org/call-grand-challenge.html

Stream of taxi data comes in real time for new york region. Region is to be divided into areas, each measuring 300mx300m. Following questions needs to be solved in real time


image::problemstmt.jpg[]


Question 1 - For every 10 seconds, find out the top 10 routes where taxies are plying the most from one area to another area.

Question 2 - For every 10 seconds, find out the top 3 routes (areas) 

Question 3 - For every 10 seconds, find out free taxies (show only 50 in the map) available in the region

Question 4 - For every 10 seconds, find out how much data is incorrectly reported by taxi drivers

Question 5 - For every 10 seconds, find out how much time does your product takes to compute the above 4 things

Below questions need not be answered in real time -

Question 6 - For overall data, find out who are the taxi drivers not reporting the data correctly

Question 7 - For overall data, find out top 10 taxi drivers earning the most

=== Solution Architecture

We use following Pivotal products to implement the solution

a. SpringXD (real time data gets ingested via SpringXD
b. Spark (real time streams are sent to Spark for computing Q1-5 above)
c. Gemfire (output from Spark goes into Gemfire)
d. PHD/HAWQ (streams are also sent to PHD so that we could answe Q6 and A7 above)
e. Application using PHP and Google Charts

See the below diagram for the stack used. NP stands for network packets so the stream would come as traffic on machine network port. SpringXD would be listening to this port.


image::solarchitect.jpg[]


=== Start Demo

* Start Gemfire

ssh as gpadmin/changme into your running instance. Run gfsh command
[source,bash]
----
$ gfsh
    _________________________     __
   / _____/ ______/ ______/ /____/ /
  / /  __/ /___  /_____  / _____  /
 / /__/ / ____/  _____/ / /    / /
/______/_/      /______/_/    /_/    v8.1.0

Monitor and Manage GemFire

gfsh>
----

Start locator now using following command 

gfsh>start locator --name=locator --port=41111 --properties-file=/var/www/html/streamtaxi/gemfire-files/gemfire-server.properties --initial-heap=50m --max-heap=50m

[source,bash]
----
gfsh>start locator --name=locator --port=41111 --properties-file=/var/www/html/streamtaxi/gemfire-files/gemfire-server.properties --initial-heap=50m --max-heap=50m
Starting a GemFire Locator in /home/gpadmin/locator...
....
Locator in /home/gpadmin/locator on ip-172-31-26-122.ap-southeast-1.compute.internal[41111] as locator is currently online.
Process ID: 146328
Uptime: 16 seconds
GemFire Version: 8.1.0
Java Version: 1.7.0_67
Log File: /home/gpadmin/locator/locator.log
JVM Arguments: -DgemfirePropertyFile=/var/www/html/streamtaxi/gemfire-files/gemfire-server.properties -Dgemfire.enable-cluster-configuration=true -Dgemfire.load-cluster-configuration-from-dir=false -Xms50m -Xmx50m -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=60 -Dgemfire.launcher.registerSignalHandlers=true -Djava.awt.headless=true -Dsun.rmi.dgc.server.gcInterval=9223372036854775806
Class-Path: /opt/pivotal/gemfire/Pivotal_GemFire_810/lib/gemfire.jar:/opt/pivotal/gemfire/Pivotal_GemFire_810/lib/locator-dependencies.jar

Successfully connected to: [host=ip-172-31-26-122.ap-southeast-1.compute.internal, port=1099]

Cluster configuration service is up and running.
----

Start server now using following command

gfsh> start server --name=server1 --cache-xml-file=/var/www/html/streamtaxi/gemfire-files/xml/server-cache.xml --initial-heap=50m --max-heap=100m --J=-Dgemfire.start-dev-rest-api=true --J=-Dgemfire.http-service-port=8081 --J=-Dgemfire.http-service-bind-address=localhost

[source,bash]
----
gfsh>start server --name=server1 --cache-xml-file=/var/www/html/streamtaxi/gemfire-files/xml/server-cache.xml --initial-heap=50m --max-heap=100m --J=-Dgemfire.start-dev-rest-api=true --J=-Dgemfire.http-service-port=8081 --J=-Dgemfire.http-service-bind-address=localhost
Starting a GemFire Server in /home/gpadmin/server1...
----

Make sure you see all the four regions listed below by running "list regions" command

[source,bash]
----
gfsh>list regions
List of regions
---------------
FreeTaxiList
ProcessData
RouteData
TaxiData
----

Come out of gfsh shell using exit command. And check that curl is working

$curl -i http://localhost:8081/gemfire-api/v1

you should see list of all regions in json format


* Start SpringXD

Use following command to run SpringXD - Note: the command will not terminate.

$ export JAVA_OPTS="-XX:PermSize=512m"

$ $XD_HOME/bin/xd-singlenode

You should wait and see following output and then proceed further

[source,bash]
----
2015-08-18T04:09:23-0700 1.2.1.RELEASE INFO DeploymentsPathChildrenCache-0 container.DeploymentListener - Path cache event: type=INITIALIZED
2015-08-18T04:09:23-0700 1.2.1.RELEASE INFO DeploymentSupervisor-0 zk.ContainerListener - Container arrived: Container{name='f6641b76-a6d0-4b46-956a-29c891140105', attributes={groups=, host=admin.local.com, id=f6641b76-a6d0-4b46-956a-29c891140105, ip=172.31.26.122, pid=148562}}
2015-08-18T04:09:23-0700 1.2.1.RELEASE INFO DeploymentSupervisor-0 zk.ContainerListener - Scheduling deployments to new container(s) in 15000 ms 
----

Start another terminal and run springXD shell command where you will be creating streams

$ $XD_SHELL/bin/xd-shell


[source,bash]
----
[gpadmin@admin ~]$ $XD_SHELL/bin/xd-shell
 _____                           __   _______
/  ___|          (-)             \ \ / /  _  \
\ `--. _ __  _ __ _ _ __   __ _   \ V /| | | |
 `--. \ '_ \| '__| | '_ \ / _` |  / ^ \| | | |
/\__/ / |_) | |  | | | | | (_| | / / \ \ |/ /
\____/| .__/|_|  |_|_| |_|\__, | \/   \/___/
      | |                  __/ |
      |_|                 |___/
eXtreme Data
1.2.1.RELEASE | Admin Server Target: http://localhost:9393
Welcome to the Spring XD shell. For assistance hit TAB or type "help".
xd:>

----

Note - SpringXD Flo is also running on http://IPAddress:9393/admin-ui  where you could create streams using drag and drop. In this demo, we would be creating using command line interface.

* Start HTTPD

Just in case httpd is not running, become root (passwd is changeme) and run "$service httpd start" command.


* Start Pivotal Hadoop 

Login to http://IPAdress:8080 (user/passwd - admin/admin) and follow this link:../springxd/start_phd.pdf[guide] to start PHD and HAWQ.

Run following command on shell to delete all files (if present);

[source,bash]
----
$hadoop fs -rm /xd/streamtaxi/*
----
* Create Streams

Go back to the XD Shell command line and run the following command -

xd:>module list

[source,bash]
----
xd:>module list
      Source              Processor           Sink                     Job
  ------------------  ------------------  -----------------------  -----------------
      file                aggregator          aggregate-counter        filejdbc
      ftp                 bridge              counter                  filepollhdfs
      gemfire             filter              field-value-counter      ftphdfs
      gemfire-cq          http-client         file                     gpload
      http                json-to-tuple       ftp                      hdfsjdbc
      jdbc                object-to-json      gauge                    hdfsmongodb
      jms                 script              gemfire-json-server      jdbchdfs
      kafka               scripts             gemfire-server           sparkapp
      mail                shell               gpfdist                  sqoop
      mongodb             splitter            hdfs                     timestampfile
      mqtt                transform           hdfs-dataset
      rabbit                                  jdbc
      reactor-ip                              kafka
      reactor-syslog                          log
      sftp                                    mail
      syslog-tcp                              mongodb
      syslog-udp                              mqtt
      tail                                    null
      tcp                                     rabbit
      tcp-client                              redis
      time                                    rich-gauge
      trigger                                 router
      twittersearch                           shell
      twitterstream                           spark-taxi
                                              splunk
                                              tcp
                                              throughput-sampler
----

You will see that there is a module spark-taxi in Sink. This is nothing but a spark module which has been uploaded already in SpringXD. This spark module is written in java and contains the business logic of getting stream data. Stream data is collected over a window of 10 seconds and then business logic is applied to find out answers of Q1-Q5 and upload the data in Gemfire's region. The jar file is located at /var/www/html/streamtaxi/jar/spark-taxi-0.1.0.jar. 

We will make the source code public soon.

Create your first stream

xd:>stream create --name stream-topx --definition "tcp --outputType=text/plain --decoder=LF | spark-taxi " --deploy

This stream basically listens to all data coming to tcp default port and sending it to the spark module. When you run SpringXD in singlenode configuration, you could also have spark running inside SpringXD. In a real world scenario, Spark will be running separately.

[source,bash]
----
xd:>stream create --name stream-topx --definition "tcp --outputType=text/plain --decoder=LF | spark-taxi " --deploy
Created and deployed new stream 'stream-topx'
xd:>
----

Make sure it is deployed correctly by checking that there are no errors in SpringXD single node terminal 

* Start taxi data on network port

Run the following command to start streaming data on network port

$cat /var/www/html/streamtaxi/sampledata/sorted_data.csv | nc localhost 1234

Access your application at http://IPaddress/streamtaxi/index.html and see that the data is being shown on the website

Note that there are three buttons, "Top 10 Areas(RT)", "Top 3 Routes(RT)" and "Free Taxies(RT)". Click on these button to see the streamed and processed data. 

If you click on "Analytics on HD" button, you would not see any data because we are running sql queries on Hadoop via HAWQ. However, we have not created any stream that puts the data on hadoop. So in next section let's create a tap on existing stream and put the data on Hadoop.

* Create hdfs tap stream

Run following command on XD shell

xd:>stream create --name hdfsstream --definition "tap:stream:stream-topx > hdfs --directory=/xd/streamtaxi --fileExtension=csv --fileName=sorted_data --rollover=300M --idleTimeout=10" --deploy

[source,bash]
----
xd:>stream create --name hdfsstream --definition "tap:stream:stream-topx > hdfs --directory=/xd/streamtaxi --fileExtension=csv --fileName=sorted_data --rollover=300M --idleTimeout=10" --deploy
Created and deployed new stream 'hdfsstream'
---- 

This stream gets a duplicate from our earlier stream and puts it on HDFS.

If you now click on the "Analytics on HD" button, you could see sql queries being run correctly and Google charts are properly shown.
