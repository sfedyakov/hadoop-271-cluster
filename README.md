# Apache Hadoop/Yarn 2.7.1 cluster Docker image
This project is actually a clone from https://github.com/lresende/docker-yarn-cluster with some small usability enhancements and some to let the image work in Docker Machine/Compose Swarm clusters. The essence of the latter enhancements is to copy entries from /etc/hosts to nodes across the cluster, because Docker Compose has some bugs with DNS and host aliases.

# Build the image
Before you build, please download the foloowing: Oracle Java and Apache Hadoop.

```
curl -LO http://ftp.jaist.ac.jp/pub/apache/hadoop/core/hadoop-2.7.1/hadoop-2.7.1.tar.gz
curl -LO 'http://download.oracle.com/otn-pub/java/jdk/8u73-b02/jdk-8u73-linux-x64.rpm' -H 'Cookie: oraclelicense=accept-securebackup-cookie'
```

If you'd like to try directly from the Dockerfile you can build the image as:

```
docker build -t sfedyakov/hadoop-271-cluster .
```

# Running Hadoop Cluster
You have several options to run this image:
- Run nodes one by one
- Run cluster on local machine
- Run cluster on Docker Swarm Cluster using Docker machine

## 1. Running nodes one by one
### 1.1. Start Namenode container

In order to use the Docker image you have just build or pulled use:

```
sudo docker run -i -t --name namenode -h namenode sfedyakov/hadoop-271-cluster /etc/bootstrap.sh -bash -namenode
```

You should now be able to access the Hadoop Admin UI at

http://<host>:8088/cluster

**Make sure that SELinux is disabled on the host. If you are using boot2docker you don't need to do anything.**

### 1.2. Start Datanode container

In order to add data nodes to the Hadoop cluster, use:

```
sudo docker run -i -t --link namenode:namenode --dns=namenode sfedyakov/hadoop-271-cluster /etc/bootstrap.sh -bash -datanode
```

You should now be able to access the HDFS Admin UI at

http://<host>:50070

**Make sure that SELinux is disabled on the host. If you are using boot2docker you don't need to do anything.**

## 2. Running Hadoop cluster on local machine with Docker Compose
Running Hadoop cluster on local machine is very straightforward. Just prepare docker-compose.yml similar to what you can find in docker-compose.yml_1machine and run

```
docker-compose scale namenode=1 datanode=3
```

## 3. Running Hadoop cluster on distributed machines with Docker Machine and Docker Compose
This is most advanced way of running Hadoop cluster. And the most cool!

First, you need to prepare the cluster of machines

```
docker-machine create -d virtualbox mh-keystore
docker $(docker-machine config mh-keystore) run -d -p "8500:8500" -h "consul" progrium/consul -server -bootstrap
docker-machine create -d virtualbox --swarm --swarm-master --swarm-discovery="consul://$(docker-machine ip mh-keystore):8500" --engine-opt="cluster-store=consul://$(docker-machine ip mh-keystore):8500" --engine-opt="cluster-advertise=eth1:2376" swarm-master
docker-machine create -d virtualbox --swarm --swarm-discovery="consul://$(docker-machine ip mh-keystore):8500" --engine-opt="cluster-store=consul://$(docker-machine ip mh-keystore):8500" --engine-opt="cluster-advertise=eth1:2376" swarm-01
docker-machine create -d virtualbox --swarm --swarm-discovery="consul://$(docker-machine ip mh-keystore):8500" --engine-opt="cluster-store=consul://$(docker-machine ip mh-keystore):8500" --engine-opt="cluster-advertise=eth1:2376" swarm-02
```

Here we created
- mh-keystore machine with Consul for Docker to preserve Swarm cluster state
- Swarm master machine
- Several Swarm slaves/worker machines

Now we are ready to create an application private overlay network. Overlay network is the one that Docker will handle across distributed machines. It requires creating Consul or Zookeper to preserve state.

```
eval $(docker-machine env --swarm swarm-master)
docker network create --driver overlay --subnet=10.0.9.0/24 appnet
```

At this moment we are actually ready to start our Hadoop cluster with Docker Compose. However, in this case each node will load our container image from Docker HUB. To accelerate the process, lets copy image to the Swarm machines

For that we need to export image

```
docker save -o hdc271.tar sfedyakov/hadoop-271-cluster
```

List all Swarm machines

```
docker-machine.exe ls --filter name=swarm* -q
```

In case you are updating image, please remove it from the mahines first with the following command

```
for z in $(docker-machine ls --filter name=swarm* -q) ; do docker-machine ssh $z docker rmi sfedyakov/hadoop-271-cluster ; done
```


Do the following for each machine from the list above

```
docker-machine ssh swarm-01
docker load -i /<path_to_>/hdc271.tar && docker images && exit
```

Now you are ready to run Hadoop cluster on Docker Swarm cluster.

It is as easy as

```
docker-compose scale namenode=1 datanode=2
```

**Please note that in this manner you can create Hadoop cluster on any infrastructure that Docker Machine supports, such as AWS, DO, OpenStack, Azure, etc.**

Full list is available here https://docs.docker.com/machine/drivers/os-base/

## Testing

You may want to change replication factor to something greater than 1. That's super-easy!

```
for z in $(docker ps -q) ; do docker exec $z sed -i 's/<value>1/<value>2/' /usr/local/hadoop/etc/hadoop/hdfs-site.xml ; done
```


First, prepare test data

```
docker exec -it namenode /bin/bash --login
hdfs dfs -mkdir -p /tmp/wnp/input/ 
curl -LO http://www.gutenberg.org/cache/epub/2600/pg2600.txt 
hdfs dfs -put pg2600.txt /tmp/wnp/input/ 
hdfs dfs -put pg2600.txt /tmp/wnp/input/wnp1.txt 
hdfs dfs -put pg2600.txt /tmp/wnp/input/wnp2.txt 
hdfs dfs -put pg2600.txt /tmp/wnp/input/wnp3.txt 
hdfs dfs -put pg2600.txt /tmp/wnp/input/wnp4.txt 
hdfs dfs -put pg2600.txt /tmp/wnp/input/wnp5.txt 
hdfs dfs -ls /tmp/wnp/input/
rm -f pg2600.txt
exit
```


Now run MapReduce

```
docker exec namenode hadoop jar /usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.1.jar wordcount /tmp/wnp/input/ /tmp/wnp/output/
```

# Limitations
Please be aware of the following
- Exactly one Namenode is allowed
- /etc/hosts are synchronized continuously every 60 seconds. So if you add more nodes during cluster run, new nodes may not be visible to existing ones for about a minute. Hope, Docker will fix their Compose DNS issues!
