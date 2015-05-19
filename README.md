## Hive streaming workshop
This demo is part of a 'Interactive Query with Apache Hive' webinar.

The webinar recording and slides are available at http://hortonworks.com/partners/learn

#### Demo overview

1. [Start HDP 2.2 sandbox and enable Hive features like transactions, queues, preemption, Tez and sessions](https://github.com/abajwa-hw/hdp22-hive-streaming#part-1---start-sandbox-vm-and-enable-hive-features)
2. [Sqoop - import PII data of users from MySql into Hive ORC table](https://github.com/abajwa-hw/hdp22-hive-streaming#part-2---import-data-from-mysql-to-hive-orc-table-via-sqoop)
3. [Flume - import browsing history of users e.g. userid,webpage,timestamp from simulated weblogs into Hive ORC table](https://github.com/abajwa-hw/hdp22-hive-streaming#part-3---import-web-history-data-from-log-file-to-hive-orc-table-via-flume) 
4. [Storm - import tweets for those users into Hive ORC table](https://github.com/abajwa-hw/hdp22-hive-streaming#part-4-import-tweets-for-users-into-hive-orc-table-via-storm) 
5. [Analyze tables to populate statistics](https://github.com/abajwa-hw/hdp22-hive-streaming#part-5-analyze-table-to-populate-statistics)
6. [Run Hive queries to correlate the data from thee different sources](https://github.com/abajwa-hw/hdp22-hive-streaming#part-6-run-hive-query-to-correlate-the-data-from-thee-different-sources)
7. [What to try next?](https://github.com/abajwa-hw/hdp22-hive-streaming#what-to-try-next)

Note: the recommended way to run the SQL queries mentioned below is using Beeline client
```
beeline -u 'jdbc:hive2://localhost:10000'
```
##### Part 1 - Start sandbox VM and enable Hive features 

- Download HDP 2.2 sandbox VM image (Sandbox_HDP_2.2_VMware.ova) from [Hortonworks website](http://hortonworks.com/products/hortonworks-sandbox/)
- Import Sandbox_HDP_2.2_VMware.ova into VMWare and set the VM memory size to 8GB
- Now start the VM
- After it boots up, find the IP address of the VM and add an entry into your machines hosts file e.g.
```
192.168.191.241 sandbox.hortonworks.com sandbox    
```
- Connect to the VM via SSH (password hadoop) and start Ambari server
```
ssh root@sandbox.hortonworks.com
/root/start_ambari.sh
```
After bringing up Ambari, make the below config changes to Hive and YARN and restart these components

  - Under Hive config, increase memory settings: 
```
hive.heapsize=512
hive.tez.container.size=512
hive.tez.java.opts=-Xmx410m
```
  - Under Hive config, turn on Hive txns (only worker threads property needs to be changed on 2.2 sandbox):
```
hive.support.concurrency=true
hive.txn.manager=org.apache.hadoop.hive.ql.lockmgr.DbTxnManager
hive.compactor.initiator.on=true
hive.compactor.worker.threads=2
hive.enforce.bucketing=true
hive.exec.dynamic.partition.mode=nonstrict
```
  - Under YARN config, increase YARN memory settings:
```
yarn.nodemanager.resource.memory-mb=4096
yarn.scheduler.minimum-allocation-mb=512
yarn.scheduler.maximum-allocation-mb=4096
```
  - Under YARN config, under Capacity Scheduler, define queues - change these two existing properties:
```
yarn.scheduler.capacity.root.default.capacity=50
yarn.scheduler.capacity.root.queues=default,hiveserver	
```
  - Under YARN config, under Capacity Scheduler, define sub-queues - add below new properties:
```
yarn.scheduler.capacity.root.hiveserver.capacity=50
yarn.scheduler.capacity.root.hiveserver.hive1.capacity=50
yarn.scheduler.capacity.root.hiveserver.hive1.user-limit-factor=4
yarn.scheduler.capacity.root.hiveserver.hive2.capacity=50
yarn.scheduler.capacity.root.hiveserver.hive2.user-limit-factor=4
yarn.scheduler.capacity.root.hiveserver.queues=hive1,hive2
```
  - Under YARN config, under Custom yarn-site add these new properties to enable preemption:
```
yarn.resourcemanager.scheduler.monitor.enable=true
yarn.resourcemanager.scheduler.monitor.policies=org.apache.hadoop.yarn.server.resourcemanager.monitor.capacity.ProportionalCapacityPreemptionPolicy
yarn.resourcemanager.monitor.capacity.preemption.monitoring_interval=1000
yarn.resourcemanager.monitor.capacity.preemption.max_wait_before_kill=5000
yarn.resourcemanager.monitor.capacity.preemption.total_preemption_per_round=0.4
```
  - Under Hive config, enable Tez and sessions (few of these already set in 2.2 sandbox)
```
hive.execution.engine=tez
hive.server2.tez.initialize.default.sessions=true
hive.server2.tez.default.queues=hive1,hive2
hive.server2.tez.sessions.per.default.queue=1
hive.server2.enable.doAs=false
hive.vectorized.groupby.maxentries=10240
hive.vectorized.groupby.flush.percent=0.1
```

More details on Hive streaming ingest can be found here: https://cwiki.apache.org/confluence/display/Hive/Streaming+Data+Ingest

More details on the above parameters can be found in the webinar slides, available at http://hortonworks.com/partners/learn

##### Part 2 - Import data from MySQL to Hive ORC table via Sqoop 
- Pull the latest Hive streaming code/scripts
```
cd
git clone https://github.com/abajwa-hw/hdp22-hive-streaming.git 
```
- Inspect CSV of user personal data
```
cat ~/hdp22-hive-streaming/data/PII_data_small.csv
```
- Import users personal data into MySQL
```
mysql -u root -p
#empty password

create database people;
use people;
create table persons (people_id INT PRIMARY KEY, sex text, bdate DATE, firstname text, lastname text, addresslineone text, addresslinetwo text, city text, postalcode text, ssn text, id2 text, email text, id3 text);
LOAD DATA LOCAL INFILE '~/hdp22-hive-streaming/data/PII_data_small.csv' REPLACE INTO TABLE persons FIELDS TERMINATED BY ',' LINES TERMINATED BY '\n';
```
- Now verify that the data was imported
```
select people_id, firstname, lastname, city from persons where lastname='SMITH';
exit;
```

- Notice there is no Hive table called persons yet

Hue: http://sandbox.hortonworks.com:8000/beeswax/tables/

Ambari View for Hive: http://sandbox.hortonworks.com:8080/#/main/views/HIVE/0.2.0/MyHive

- Optional: point Sqoop to a newer version of mysql connector. This is a workaround needed when importing large files using Sqoop, to avoid "GC overhead limit exceeded" error.  See [SQOOP-1617](https://issues.apache.org/jira/browse/SQOOP-1617) and [SQOOP-1400](https://issues.apache.org/jira/browse/SQOOP-1400) for more info
```
cp -a /usr/hdp/current/ranger-admin/ews/webapp/WEB-INF/lib/mysql-connector-java-5.1.31.jar /usr/share/java/
ln -sf /usr/share/java/mysql-connector-java-5.1.31.jar /usr/share/java/mysql-connector-java.jar
ls -la /usr/share/java/my*
```

- Import data from MySQL to Hive ORC table using Sqoop 
```
sqoop import --verbose --connect 'jdbc:mysql://localhost/people' --table persons --username root --hcatalog-table persons --hcatalog-storage-stanza "stored as orc" -m 1 --create-hcatalog-table 
```
Note: if importing large files you should also add the following argument: --fetch-size -2147483648

- Now notice persons table created and has data

http://sandbox.hortonworks.com:8000/beeswax/table/default/persons
![Image](../master/screenshots/screenshot-persons-data.png?raw=true)

- Notice the table is stored in ORC format by clicking view file location and then on part-m-00000
http://sandbox.hortonworks.com:8000/filebrowser/view//apps/hive/warehouse/persons/part-m-00000
![Image](../master/screenshots/screenshot-persons-HDFS.png?raw=true)

- Compare the format of table persons against the format of sample_07 which is stored in text format

http://sandbox.hortonworks.com:8000/filebrowser/view//apps/hive/warehouse/sample_07/sample_07
![Image](../master/screenshots/screenshot-sample-HDFS.png?raw=true)

##### Part 3 - Import web history data from log file to Hive ORC table via Flume 

- Create table webtraffic to store the userid and web url enabling transactions and partition into day month year 
````
create table if not exists webtraffic (id int, val string) 
partitioned by (year string,month string,day string) 
clustered by (id) into 7 buckets 
stored as orc 
TBLPROPERTIES ("transactional"="true");
````

- Now lets configure the Flume agent. High level:
  - The *source* will be of type exec that tails our weblog file using a timestamp intersept (i.e. flume interseptor adds timestamp header to the payload)
  - The *channel* will be a memory channel which is ideal for flows that need higher throughput but could lose the data in the event of agent failures
  - The *sink* will be of type Hive that writes userid and url to default.webtraffic table partitioned by year, month, day
  - More details about each type of source, channel, sink are available [here](http://flume.apache.org/FlumeUserGuide.html) 
- In Ambari > Flume > Config > flume.conf enter the below and restart Flume
```

## Flume NG Apache Log Collection
## Refer to https://cwiki.apache.org/confluence/display/FLUME/Getting+Started
##

agent.sources = webserver
agent.sources.webserver.type = exec
agent.sources.webserver.command = tail -F /tmp/webtraffic.log
agent.sources.webserver.batchSize = 20
agent.sources.webserver.channels = memoryChannel
agent.sources.webserver.interceptors = intercepttime
agent.sources.webserver.interceptors.intercepttime.type = timestamp

## Channels ########################################################
agent.channels = memoryChannel
agent.channels.memoryChannel.type = memory
agent.channels.memoryChannel.capacity = 2000000


## Sinks ###########################################################

agent.sinks = hiveout
agent.sinks.hiveout.type = hive
agent.sinks.hiveout.hive.metastore=thrift://localhost:9083
agent.sinks.hiveout.hive.database=default
agent.sinks.hiveout.hive.table=webtraffic
agent.sinks.hiveout.hive.batchSize=1
agent.sinks.hiveout.hive.partition=%Y,%m,%d
agent.sinks.hiveout.serializer = DELIMITED
agent.sinks.hiveout.serializer.fieldnames =id,val
agent.sinks.hiveout.channel = memoryChannel
```

- Start tailing the flume agent log file in one terminal...
```
tail -F /var/log/flume/flume-agent.log
```

- After a few seconds the agent log should contain the below
```
02 Jan 2015 20:35:31,782 INFO  [lifecycleSupervisor-1-0] (org.apache.flume.source.ExecSource.start:163)  - Exec source starting with command:tail -F /tmp/webtraffic.log
02 Jan 2015 20:35:31,782 INFO  [lifecycleSupervisor-1-1] (org.apache.flume.instrumentation.MonitoredCounterGroup.register:119)  - Monitored counter group for type: SINK, name: hiveout: Successfully registered new MBean.
02 Jan 2015 20:35:31,782 INFO  [lifecycleSupervisor-1-1] (org.apache.flume.instrumentation.MonitoredCounterGroup.start:95)  - Component type: SINK, name: hiveout started
02 Jan 2015 20:35:31,783 INFO  [lifecycleSupervisor-1-1] (org.apache.flume.sink.hive.HiveSink.start:611)  - hiveout: Hive Sink hiveout started
02 Jan 2015 20:35:31,785 INFO  [lifecycleSupervisor-1-0] (org.apache.flume.instrumentation.MonitoredCounterGroup.register:119)  - Monitored counter group for type: SOURCE, name: webserver: Successfully registered new MBean.
02 Jan 2015 20:35:31,785 INFO  [lifecycleSupervisor-1-0] (org.apache.flume.instrumentation.MonitoredCounterGroup.start:95)  - Component type: SOURCE, name: webserver started
```

- Using another terminal window, run the createlog.sh script which will generate 400 dummy web traffic log events at a rate of one event per second
```
cd ~/hdp22-hive-streaming
./createlog.sh ./data/PII_data_small.csv 400 >> /tmp/webtraffic.log
```
- Start tailing the webtraffic file in another terminal
```
tail -F /tmp/webtraffic.log
```
- The webtraffic.log should start displaying the webtraffic
```
581842607,http://www.google.com
493259972,http://www.yahoo.com
607729813,http://cnn.com
53802950,http://www.hortonworks.com
```

- The Flume agent log should start outputting below
```
02 Jan 2015 20:42:37,380 INFO  [SinkRunner-PollingRunner-DefaultSinkProcessor] (org.apache.flume.sink.hive.HiveWriter.commitTxn:251)  - Committing Txn id 14045 to {metaStoreUri='thrift://localhost:9083', database='default', table='webtraffic', partitionVals=[2015, 01, 02] }
```
- After 6-7min, notice that the script has completed and the webtraffic table now has records created

http://sandbox.hortonworks.com:8000/beeswax/table/default/webtraffic
![Image](../master/screenshots/screenshot-webtraffic-data.png?raw=true)

- Notice the table is stored in ORC format

http://sandbox.hortonworks.com:8000/filebrowser/view//apps/hive/warehouse/webtraffic
![Image](../master/screenshots/screenshot-webtraffic-HDFS.png?raw=true)



##### Part 4: Import tweets for users into Hive ORC table via Storm

- Install maven from epel (*or [as an Ambari Service](https://github.com/randerzander/maven-service) or [manually](https://gist.github.com/seanorama/532caf72797a31f1a856)*)
```
curl -o /etc/yum.repos.d/epel-apache-maven.repo https://repos.fedorapeople.org/repos/dchen/apache-maven/epel-apache-maven.repo
yum -y install apache-maven
```
- Make hive config changes to enable transactions, if not already done above
```
hive.txn.manager = org.apache.hadoop.hive.ql.lockmgr.DbTxnManager
hive.compactor.initiator.on = true
hive.compactor.worker.threads > 0 
```
- Create a Twitter consumer key/secret
  1. Open https://apps.twitter.com/
  2. Click 'Create New App'
  3. Use whatever Name & URL you like.
  4. Once created, click "Keys and Access Tokens"
  5. Click "Create my access token".
  6. Use the consumer and access details in the next step

- Add your Twitter consumer key/secret, token/secret in [~/hdp22-hive-streaming/src/test/HiveTopology.java](https://github.com/abajwa-hw/hdp22-hive-streaming/blob/master/src/test/HiveTopology.java#L40)
  - *Tip: (edit from SSH with `vi` or `nano -w`)*

- Create hive table for tweets that has transactions turned on and ORC enabled
```
create table if not exists user_tweets (twitterid string, userid int, displayname string, created string, language string, tweet string) clustered by (userid) into 7 buckets stored as orc tblproperties("orc.compress"="NONE",'transactional'='true');
```
- Run below
```
sudo -u hdfs hadoop fs -chmod +w /apps/hive/warehouse/user_tweets
```

- Optional: build the storm uber jar (may take 10-15min first time). You can skip this to use the pre-built jar in the target dir. 
```
cd ~/hdp22-hive-streaming
mvn package
```

- In case your system time is not accurate, fix it to avoid errors from Twitter4J
```
yum install -y ntp
service ntpd stop
ntpdate pool.ntp.org
service ntpd start
```
- Using Ambari, make sure Storm is started first *(it is stopped by default on the sandbox)* and twitter_topology does not already exist
- Open up the Storm webui

http://sandbox.hortonworks.com:8744/
![Image](../master/screenshots/screenshot-storm-home.png?raw=true)

- Run the topology on the cluster and notice twitter_topology appears on Storm webui
```
cd ~/hdp22-hive-streaming
storm jar ./target/storm-integration-test-1.0-SNAPSHOT.jar test.HiveTopology thrift://sandbox.hortonworks.com:9083 default user_tweets twitter_topology
```

Note: to run in local mode *(i.e. without submitting it to cluster)*, run the above without the twitter_topology argument

- In Storm UI, drill down into the topology to see the details and refresh periodically. The numbers under emitted, transferred and acked should start increasing.
![Image](../master/screenshots/screenshot-storm-topology.png?raw=true)

In Storm UI, you can also click on "Show Visualization" under "Topology Visualization" to see the topology visually
![Image](../master/screenshots/screenshot-storm-visualization.png?raw=true)

- After 20-30 seconds, kill the topology from the Storm UI or using the command below to avoid overloading the VM
```
storm kill twitter_topology
```

- After a few seconds, query the Hive table and notice it now contains tweets
```
select * from user_tweets;
```

![Image](../master/screenshots/screenshot-usertweets-data.png?raw=true)

  - You may encounter the below error through Hue when browsing this table. This is because in this version, Hue beeswax does not support UTF-8 and there were such characters present in the tweets
```
'ascii' codec can't decode byte 0xf0 in position 62: ordinal not in range(128)
```
  - To workaround replace ```default_filters=['unicode', 'escape'],``` with ```default_filters=['decode.utf8', 'unicode', 'escape'],``` in /usr/lib/hue/desktop/core/src/desktop/lib/django_mako.py
```
cp  /usr/lib/hue/desktop/core/src/desktop/lib/django_mako.py  /usr/lib/hue/desktop/core/src/desktop/lib/django_mako.py.orig
sed -i "s/default_filters=\['unicode', 'escape'\],/default_filters=\['decode.utf8', 'unicode', 'escape'\],/g" /usr/lib/hue/desktop/core/src/desktop/lib/django_mako.py
```

- Notice the table is stored in ORC format

http://sandbox.hortonworks.com:8000/filebrowser/view/apps/hive/warehouse/user_tweets
![Image](../master/screenshots/screenshot-usertweets-HDFS.png?raw=true)


- In case you want to empty the table for future runs, you can run below 
```
delete from user_tweets;
```
Note: the 'delete from' command are only supported in 2.2 when Hive transactions are turned on)

##### Part 5: Analyze table to populate statistics

- Run table statistics
```
analyze table persons compute statistics;
analyze table user_Tweets compute statistics;
analyze table webtraffic partition(year,month,day) compute statistics;
```

- Run column statistics
```
analyze table persons compute statistics for columns;
analyze table user_Tweets compute statistics for columns;
analyze table webtraffic partition(year,month,day) compute statistics for columns;
```


##### Part 6: Run Hive query to correlate the data from thee different sources

- Check size of PII table
```
select count(*) from persons;
```
returns 400 rows

- Correlate browsing history with PII data
```
select  p.firstname, p.lastname, p.sex, p.addresslineone, p.city, p.ssn, w.val
from persons p, webtraffic w 
where w.id = p.people_id;
```
Notice the last field contains the browsing history:
![Image](../master/screenshots/screenshot-query1.png?raw=true)

- Correlate tweets with PII data
```
select t.userid, t.twitterid, p.firstname, p.lastname, p.sex, p.addresslineone, p.city, p.ssn, t.tweet 
from persons p, user_tweets t 
where t.userid = p.people_id;
```
Notice the last field contains the Tweet history:
![Image](../master/screenshots/screenshot-query2.png?raw=true)

- Correlate all 3
```
select  p.firstname, p.lastname, p.sex, p.addresslineone, p.city, p.ssn, w.val, t.tweet
from persons p, user_tweets t, webtraffic w 
where w.id = t.userid and t.userid = p.people_id
order by p.ssn;
```
Notice the last 2 field contains the browsing and Tweet history:
![Image](../master/screenshots/screenshot-query3.png?raw=true)


##### What to try next?

- Enhance the sample Twitter Storm topology
  - Import the above Storm sample into Eclipse on the sandbox VM using an *Ambari stack for VNC* and use the Maven plugin to compile the code. Steps available at https://github.com/abajwa-hw/vnc-stack
  - Update [HiveTopology.java](https://github.com/abajwa-hw/hdp22-hive-streaming/blob/master/src/test/HiveTopology.java#L250) to pass hashtags or languages or locations or Twitter user ids to filter Tweets
  - Add other Bolts to this basic topology to process the Tweets (e.g. rolling count) and write them to different components (like HBase, Solr etc). Here is a HDP 2.2 sample project showing a more complicated topology with Tweets being generated from a Kafka producer and being emitted into local filesystem, HDFS, Hive, Solr and HBase: https://github.com/abajwa-hw/hdp22-twitter-demo 
  
- Use Sqoop to import data into ORC tables from other databases (e.g. Oracle, MSSQL etc). See [this blog entry](http://hortonworks.com/hadoop-tutorial/import-microsoft-sql-server-hortonworks-sandbox-using-sqoop/) for more details

- Experiment with Flume
  - Change the Flume configuration to use different channels (e.g. FileChannel or Spillable Memory Channel) or write to different sinks (e.g HBase). See the [Flume user guide](http://flume.apache.org/FlumeUserGuide.html) for more details.
  
