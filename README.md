# Server configuration
I created 3 VMs(running ubuntu) on Google cloud(I choose Google because I have free credits there but this should work on any VM) with the following command:

```
gcloud compute --project=caner-playground instances create instance-1 --zone=europe-north1-a --machine-type=n1-standard-1 --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --no-service-account --no-scopes --image=ubuntu-1810-cosmic-v20190402 --image-project=ubuntu-os-cloud --boot-disk-size=10GB --boot-disk-type=pd-standard --boot-disk-device-name=instance-1
```

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