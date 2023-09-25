# Spark client running into StandAlone cluster in Docker

Apache Spark is an open-source, distributed processing system used for big data workloads.

In this demo, a Spark container uses a Spark StandAlone cluster as a resource management and job scheduling technology to perform distributed data processing.

This Docker image contains Spark binaries prebuilt and uploaded in Docker Hub.

Spark client access a remote Hive metastore created in a MySQL Server for Spark catalog.


## Start Swarm cluster

1. start swarm mode in node1
```shell
$ docker swarm init --advertise-addr <IP node1>
$ docker swarm join-token worker  # issue a token to add a node as worker to swarm
```

2. add 3 more workers in swarm cluster (node2, node3, node4)
```shell
$ docker swarm join --token <token> <IP nodeN>:2377
```

3. label each node to anchor each container in swarm cluster
```shell
docker node update --label-add hostlabel=hdpmst node1
docker node update --label-add hostlabel=hdp1 node2
docker node update --label-add hostlabel=hdp2 node3
docker node update --label-add hostlabel=hdp3 node4
```

4. create an external "overlay" network in swarm to link the 2 stacks (hdp and spk)
```shell
docker network create --driver overlay mynet
```

5. start the Spark cluster, mysql server and Hadoop standalone
```shell
$ docker stack deploy -c docker-compose.yml spk
$ docker stack ps spk
nx0huvb6ent1   spk_hdpmst.1    mkenjis/ubhdp_img:latest          node1     Running         Running 36 minutes ago             
7sgbf3tgwcug   spk_spk1.1      mkenjis/ubspkcluster_img:latest   node2     Running         Running 36 minutes ago             
jgr9as5irt6r   spk_spk2.1      mkenjis/ubspkcluster_img:latest   node3     Running         Running 36 minutes ago             
rdgxc68jrdub   spk_mysql.1     mkenjis/mysql:5.7                 node4     Running         Running 36 minutes ago             
ux3l1ywtvjf7   spk_spk_mst.1   mkenjis/ubspkcluster_img:latest   node1     Running         Running 36 minutes ago
```

## Set up MySQL server

1. download hive binaries and unpack it
```shell
wget https://archive.apache.org/dist/hive/hive-1.2.1/apache-hive-1.2.1-bin.tar.gz
tar -xzf apache-hive-1.2.1-bin.tar.gz
```

2. in mysql server node, run hive script to create metastore tables
```shell
/root/staging/apache-hive-1.2.1-bin/scripts/metastore/upgrade/mysql
mysql -uroot -p metastore < hive-schema-1.2.0.mysql.sql
Enter password:
```

## Set up Spark client

1. access spark client node
```shell
$ docker container exec -it <spk_cli ID> bash
```

2. copy hive-site.xml into $SPARK_HOME/conf

3. copy core-site.xml and hdfs-site.xml from hadoop master into $SPARK_HOME/conf

4. replicate *.xml to spk1 and spk2
```shell
scp *.xml root@spk1:${SPARK_HOME}/conf
```

5. start spark-shell installing mysql jar files
```shell
$ spark-shell --packages mysql:mysql-connector-java:5.1.49 --master spark://<container_hostname>:7077
2021-12-05 11:09:14 WARN  NativeCodeLoader:62 - Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
2021-12-05 11:09:40 WARN  Client:66 - Neither spark.yarn.jars nor spark.yarn.archive is set, falling back to uploading libraries under SPARK_HOME.
Spark context Web UI available at http://802636b4d2b4:4040
Spark context available as 'sc' (master = yarn, app id = application_1638723680963_0001).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.3.2
      /_/
         
Using Scala version 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_181)
Type in expressions to have them evaluated.
Type :help for more information.

scala> 
```


