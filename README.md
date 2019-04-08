# Server configuration
I created 3 VMs(running ubuntu) on Google cloud(I choose Google because I have free credits there but this should work on any VM) with the following command:

```
gcloud compute --project=caner-playground instances create instance-1 --zone=europe-north1-a --machine-type=n1-standard-1 --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --no-service-account --no-scopes --image=ubuntu-1810-cosmic-v20190402 --image-project=ubuntu-os-cloud --boot-disk-size=10GB --boot-disk-type=pd-standard --boot-disk-device-name=instance-1
```
![00](https://github.com/DerinMavi/kafka-ha-setup/blob/master/images/00.png)

Then for each VM I installed java and vi editor:
```
sudo apt-get update
sudo apt-get -y install vim
sudo apt -y install default-jre
```

Cretaed a kafka user:
```
sudo useradd kafka -m
sudo passwd kafka
sudo adduser kafka sudo
su -l kafka
chsh --shell /bin/bash
```
# Kafka configuration
Downloaded & extract & configure Kafka:
```
wget http://mirror.netinch.com/pub/apache/kafka/2.2.0/kafka_2.11-2.2.0.tgz
mkdir kafka && tar xzf kafka_2.11-2.2.0.tgz -C kafka --strip-components=1
vi /home/kafka/kafka/config/server.properties
```
To configure I set a unique broker id to each instance:
`broker.id=`

Set the log path:
`log.dirs=/home/kafka/kafka/kafka-logs`

Set the ip addresses of my zookeeper servers:
`zookeeper.connect=10.166.0.2:2181,10.166.0.3:2181,10.166.0.4:2181`

Enabled topic deletion to be able to create/delete some test topics:
`delete.topic.enable=true`

# Zookeeper configuration
Download, extract and configured zookeeper:
```
wget http://www.nic.funet.fi/pub/mirrors/apache.org/zookeeper/zookeeper-3.4.14/zookeeper-3.4.14.tar.gz
mkdir zookeeper && tar xzf zookeeper-3.4.14.tar.gz -C zookeeper --strip-components=1
```
Created a unique id for each zookeeper instance:
`vi /home/kafka/zookeeper/myid`

Renamed the bundled config file:
`mv zookeeper/conf/zoo_sample.cfg zookeeper/conf/zoo.cfg`

Update the config file to set a dataDir and add each zookeeper address:
```
dataDir=/home/kafka/zookeeper
server.1=10.166.0.2:2888:3888
server.2=10.166.0.3:2888:3888
server.3=10.166.0.4:2888:3888
```

# Creating services
I created systemd services to start kafka and zookeper automatiacally on reboot
### Zookeper service
Create the configuration:
```
sudo vi /etc/systemd/system/zookeeper.service
```

I used the following configuration for the zookeper service:
```
[Unit]
Description=Zookeeper Daemon
Requires=network-online.target
After=network-online.target
[Service]
Type=forking
User=kafka
# the directory that the commands will run there   
WorkingDirectory=/home/kafka/zookeeper
ExecStart=/home/kafka/zookeeper/bin/zkServer.sh start /home/kafka/zookeeper/conf/zoo.cfg
ExecStop=/home/kafka/zookeeper/bin/zkServer.sh stop /home/kafka/zookeeper/conf/zoo.cfg
ExecReload=/home/kafka/zookeeper/bin/zkServer.sh restart /home/kafka/zookeeper/conf/zoo.cfg
Restart=on-abnormal
[Install]
WantedBy=multi-user.target
```

Enable and run the service andn check its status: 
```
sudo systemctl enable zookeeper
sudo systemctl start zookeeper
/home/kafka/zookeeper/bin/zkServer.sh status
```

### Kafka service
Create the configuration:
`sudo vi /etc/systemd/system/kafka.service`

I used the following configuration for the kafka service:
```
[Unit]
Description=Kafka Daemon
Wants=syslog.target

# suppose you have a service named zookeeper that it start zookeeper and we want Kafka service run after the zookeeper service
After=zookeeper.service

[Service]    
Type=forking

# the user whom you want run the Kafka start and stop command under
User=kafka

# the directory that the commands will run there   
WorkingDirectory=/home/kafka/kafka 

# Kafka server start command
ExecStart=/home/kafka/kafka/bin/kafka-server-start.sh -daemon /home/kafka/kafka/config/server.properties

# Kafka server stop command
ExecStop=/home/kafka/kafka/bin/kafka-server-stop.sh -daemon

TimeoutSec=30
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable and run the service:
```
sudo systemctl enable kafka
sudo systemctl start kafka
```

# Testing
I created a test topic with replication-factor 3 and 10 partitions:
```
~/kafka/bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 10 --topic test-topic
```

Checking everything works correctly:
```
~/kafka/bin/kafka-topics.sh --describe --topic test-topic --zookeeper localhost:2181
```

![01](https://github.com/DerinMavi/kafka-ha-setup/blob/master/images/01.png)

Started listeners on every instance:
```
~/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test-topic --from-beginning
```

Published some test messages:
```
echo "Hello, World" | ~/kafka/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test-topic > /dev/null
```

I was able to get the published messages on all servers:
![02](https://github.com/DerinMavi/kafka-ha-setup/blob/master/images/02.png)


Next I shutdown instance-2:
![03](https://github.com/DerinMavi/kafka-ha-setup/blob/master/images/03.png)

We can see some partitions lost their leader and failed over to 1 & 3 here:
![04](https://github.com/DerinMavi/kafka-ha-setup/blob/master/images/04.png)

Next I send "HA Works!" message and it is received by instance 1 & 3:
![05](https://github.com/DerinMavi/kafka-ha-setup/blob/master/images/05.png)

