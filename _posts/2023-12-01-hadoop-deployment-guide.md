---
title: "Hadoop Deployment: From Virtual Machines to a Distributed Cluster"
date: 2023-12-01
permalink: /blog/hadoop-deployment-guide/
excerpt: "Hands-on Hadoop deployment with CentOS 7, HDFS, YARN, a three-node cluster, Spark, Python, and troubleshooting."
categories:
  - Big Data
tags:
  - Hadoop
  - HDFS
  - YARN
  - Spark
---

> This English edition preserves the technical steps and screenshots of my [original CSDN tutorial](https://blog.csdn.net/weixin_75027717/article/details/137211303). Archived under **December 2023** as requested; the first day is a sorting placeholder, not a claim of the original day of publication. This is a historical lab, not a modern production security guide.

> **Important:** The original lab used CentOS 7, disabled the firewall and SELinux, recommended a weak password, and included destructive commands. Use strong passwords, appropriate firewall rules, and never format an existing HDFS filesystem or delete existing SSH keys.

# 1. Hadoop deployment

## 1.1 Virtual machine installation

Install **VMware Workstation 17** and **CentOS 7**, remove unnecessary virtual hardware, and choose **NAT** networking.

![VM installation and network configuration 1](https://i-blog.csdnimg.cn/blog_migrate/8d91795aa0bf90f35dc3e496dbdfbbe7.png)

![VM installation and network configuration 2](https://i-blog.csdnimg.cn/blog_migrate/fe76c93f4c773a7133cbf286e686bba6.png)

![VM installation and network configuration 3](https://i-blog.csdnimg.cn/blog_migrate/f96038b11221b94ec847c5d43a27e8bc.png)

At installation: (1) select **Compute Node** or **Server with GUI**; (2) assign the 20 GB virtual disk; (3) enable networking, record the IP range and gateway, and choose **Configure → IPv4 → Manual → Add** to set the static IP and DNS; then configure the hostname.

![IPv4 and hostname settings 1](https://i-blog.csdnimg.cn/blog_migrate/a038f9eb6a0edc62d7a6afb062e460db.png)

![IPv4 and hostname settings 2](https://i-blog.csdnimg.cn/blog_migrate/893a5bf8e2a43920b0102a244b00fc5a.png)

Set the root password and create a **hadoop** user. The original suggested password `123` is insecure: do not use it. Use `ifconfig` or `ip a` to determine the IP address, connect using **Xshell**, power off, and create a **VMware snapshot**.

![User, remote access, and snapshot 1](https://i-blog.csdnimg.cn/blog_migrate/fce0ab2c300ac21f329a2f6992674a58.png)

![User, remote access, and snapshot 2](https://i-blog.csdnimg.cn/blog_migrate/dfd7996e5a62d06d1a0ca1676e5ca240.png)

![User, remote access, and snapshot 3](https://i-blog.csdnimg.cn/blog_migrate/fb0d706f20d636a3474355c105d306da.png)

![User, remote access, and snapshot 4](https://i-blog.csdnimg.cn/blog_migrate/3af6e790c426872cbcd512fc12d2a94d.png)

## 1.2 Basic Linux configuration

Sign in with root privileges (`su`). Edit the hostname, static IP, and host mappings:

```bash
vi /etc/hostname
vi /etc/sysconfig/network-scripts/ifcfg-ens33
vi /etc/hosts
```

![Hostname, network adapter, and hosts file 1](https://i-blog.csdnimg.cn/blog_migrate/66af6eb1b881c7954deb0f5671f44669.png)

![Hostname, network adapter, and hosts file 2](https://i-blog.csdnimg.cn/blog_migrate/9dce45a0f62b484e008c07b4ba5aa9a0.png)

![Hostname, network adapter, and hosts file 3](https://i-blog.csdnimg.cn/blog_migrate/b85d6dec0c3317eeedb6ecaeca6694b6.png)

In the original isolated lab, the firewall and SELinux were disabled, followed by a reboot:

```bash
systemctl disable firewalld.service
vi /etc/selinux/config
reboot
```

![SELinux configuration](https://i-blog.csdnimg.cn/blog_migrate/39012b9b12604f17ad1484feb1b0f023.png)

For real deployments, configure narrow firewall exceptions and suitable access policies instead.

## 1.3 Install Java and Hadoop

Transfer archives using Xftp. Extract into `/home/hadoop/program`:

```bash
tar -zxf hadoop-3.3.1.tar.gz
tar -zxf jdk-8u333-linux-x64.tar.gz
```

![File transfer and extracted packages 1](https://i-blog.csdnimg.cn/blog_migrate/9d966586d19ed812e1b2234da8dea24.png)

![File transfer and extracted packages 2](https://i-blog.csdnimg.cn/blog_migrate/f352cf1725be12fc5f8ff8e1dfa71ceb.png)

The original notes use `/home/hadoop/program/hadoop-3.3` and `/home/hadoop/program/jdk-1.8` as installation paths; check that these directories actually exist after extraction or renaming.

```bash
export JAVA_HOME=/home/hadoop/program/jdk-1.8
export PATH=$JAVA_HOME/bin:$PATH
export HADOOP_HOME=/home/hadoop/program/hadoop-3.3
export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
java -version
hadoop version
```

Persist these `export` statements in `~/.bashrc` and run `source ~/.bashrc`.

![Version checks and shell profile 1](https://i-blog.csdnimg.cn/blog_migrate/69bad61eba38cc74c6af7c4812e62543.png)

![Version checks and shell profile 2](https://i-blog.csdnimg.cn/blog_migrate/323603e32af7d9d739ad1cc45d75e793.png)

## 1.4 Hadoop configuration and local mode

Edit the two Hadoop configuration files (normally under `$HADOOP_HOME/etc/hadoop`):

```bash
vi core-site.xml
vi hdfs-site.xml
```

The original configuration values appear in these screenshots, rather than in copyable text. Verify all XML values against your own IPs and paths.

![core-site.xml and hdfs-site.xml 1](https://i-blog.csdnimg.cn/blog_migrate/50921c7ab4242cf07fee008abeb42e48.png)

![core-site.xml and hdfs-site.xml 2](https://i-blog.csdnimg.cn/blog_migrate/fb5eedff1a43d0c813da3f59af04791a.png)

Enable SSH to the local machine:

```bash
cd
ssh-keygen
ssh-copy-id localhost
```

Run the included grep example in **local mode**:

```bash
cd "$HADOOP_HOME"
mkdir input
cp etc/hadoop/*.xml input
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.1.jar grep input output 'dfs[a-z.]+'
cat output/*
```

![Local-mode test](https://i-blog.csdnimg.cn/blog_migrate/b545770ed0466e2e6f7eac931c5373a3.png)

## 1.5 Pseudo-distributed HDFS

Initialize a **new, empty** test filesystem, start the services, upload files to HDFS, run a MapReduce example, and read the results:

```bash
cd "$HADOOP_HOME"
rm -rf output
bin/hdfs namenode -format
sbin/start-all.sh
bin/hdfs dfs -mkdir -p /user/hadoop
bin/hdfs dfs -mkdir input
bin/hdfs dfs -put etc/hadoop/*.xml input
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.1.jar grep input output 'dfs[a-z.]+'
bin/hdfs dfs -get output
cat output/*
```

The original NameNode web UI was `http://192.168.109.128:9870/` (an example local-network address). **Never reformat a NameNode containing data.** Repeating `mkdir` or job commands without cleaning outputs can cause “already exists” errors.

![HDFS and MapReduce test results 1](https://i-blog.csdnimg.cn/blog_migrate/04251bb050c47cf0790780e5888957b2.png)

![HDFS and MapReduce test results 2](https://i-blog.csdnimg.cn/blog_migrate/ecc1341d099c99eeeb6063e0677a905e.png)

![HDFS and MapReduce test results 3](https://i-blog.csdnimg.cn/blog_migrate/39085f13f9fa64652b0d66e354feab4b.png)

## 1.6 Pseudo-distributed Hadoop with YARN

Configure `mapred-site.xml` and `yarn-site.xml`, and set `HADOOP_MAPRED_HOME` in `~/.bashrc`:

```bash
vi mapred-site.xml
vi yarn-site.xml
# Add to ~/.bashrc:
export HADOOP_MAPRED_HOME=$HADOOP_HOME
source ~/.bashrc
```

![MapReduce and YARN configuration 1](https://i-blog.csdnimg.cn/blog_migrate/dbd7575f44ff1e5fd6d752b2d2265173.png)

![MapReduce and YARN configuration 2](https://i-blog.csdnimg.cn/blog_migrate/03f16fbd93068a25b964f03e9356af83.png)

```bash
cd "$HADOOP_HOME"
start-dfs.sh
start-yarn.sh
bin/hdfs dfs -mkdir -p /user/hadoop
bin/hdfs dfs -mkdir input
bin/hdfs dfs -put etc/hadoop/*.xml input
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.1.jar grep input output 'dfs[a-z.]+'
bin/hdfs dfs -cat output/*
```

## 1.7 Fully distributed cluster

The tutorial's nodes are **bd0, bd1, bd2**. Ensure every machine can resolve the others via `/etc/hosts`. The original lab also disabled its firewall and SELinux:

```bash
systemctl disable firewalld
vi /etc/selinux/config
# Original configuration: SELINUX=disabled
```

Configure SSH from every node to bd0, bd1 and bd2. The source included `rm -rf .ssh`, which is destructive and is deliberately shown as a comment here:

```bash
cd
# Historical instruction ONLY: rm -rf .ssh
ssh-keygen
ssh-copy-id bd0
ssh-copy-id bd1
ssh-copy-id bd2
```

On **bd0**:

```bash
cd ~/.ssh
scp authorized_keys hadoop@bd1:/home/hadoop/.ssh
scp authorized_keys hadoop@bd2:/home/hadoop/.ssh
stop-yarn.sh
stop-dfs.sh
```

![SSH and distributed parameters 1](https://i-blog.csdnimg.cn/blog_migrate/afe75b08646a04e85e63a6299782be48.png)

![SSH and distributed parameters 2](https://i-blog.csdnimg.cn/blog_migrate/6f9f96320e7db81e7579d67dfa7d97c3.png)

![SSH and distributed parameters 3](https://i-blog.csdnimg.cn/blog_migrate/c9dab793fccc45e7d91cc09e8b76bedb.png)

Edit the `workers` file and distributed `yarn-site.xml`:

```bash
vi workers
vi yarn-site.xml
```

![Worker nodes and YARN configuration 1](https://i-blog.csdnimg.cn/blog_migrate/5e59e2b53fa9b5cbc5ca2d76e3710800.png)

![Worker nodes and YARN configuration 2](https://i-blog.csdnimg.cn/blog_migrate/94be218703e4ad67033d4c0b21d048dc.png)

Distribute software and profile files from bd0:

```bash
cd
scp -r program hadoop@bd1:/home/hadoop/
scp -r program hadoop@bd2:/home/hadoop/
scp .bashrc hadoop@bd1:/home/hadoop/
scp .bashrc hadoop@bd2:/home/hadoop/
```

On bd1 and bd2: `source ~/.bashrc`. On bd0, format the NameNode **only for initial setup**, then start and test the cluster:

```bash
hdfs namenode -format
cd "$HADOOP_HOME"
start-dfs.sh
start-yarn.sh
bin/hdfs dfs -mkdir -p /user/hadoop
bin/hdfs dfs -mkdir input
bin/hdfs dfs -put etc/hadoop/*.xml input
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.1.jar grep input output 'dfs[a-z.]+'
bin/hdfs dfs -cat output/*
```

![Cluster output and web interface (port 8080) 1](https://i-blog.csdnimg.cn/blog_migrate/bfe9d22662a1693d86f9c42dda684151.png)

![Cluster output and web interface (port 8080) 2](https://i-blog.csdnimg.cn/blog_migrate/a549c496a68bc7ab4688be2eb9ad8811.png)

## 1.8 Install Spark and Scala

Download [Scala 2.12.15](https://www.scala-lang.org/download/2.12.15.html) and [Spark](https://spark.apache.org/downloads), checking Hadoop and Scala compatibility.

```bash
tar -zxf scala-2.12.15.tgz
tar -zxf spark-3.5.0-bin-hadoop3.tgz
```

The original installation paths were:

```text
/home/hadoop/program/spark-3.5.0-bin-hadoop3
/home/hadoop/program/scala-2.12.15
```

![Spark and Scala directories](https://i-blog.csdnimg.cn/blog_migrate/f9ed5e997229b650d780123bae8dc402.png)

```bash
export SCALA_HOME=/home/hadoop/program/scala-2.12.15
export SPARK_HOME=/home/hadoop/program/spark-3.5.0-bin-hadoop3
export PATH=$SPARK_HOME/bin:$SPARK_HOME/sbin:$SCALA_HOME/bin:$PATH
```

![Spark verification](https://i-blog.csdnimg.cn/blog_migrate/b39c8532db6e16f9ec4af0e3b5b60209.png)

## 1.9 Spark on YARN

Set the Hadoop configuration directory, edit `yarn-site.xml`, distribute the changes, and run the test shown in the screenshots:

```bash
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
source ~/.bashrc
vi "$HADOOP_CONF_DIR/yarn-site.xml"
```

![Spark-on-YARN setup, distribution, and test 1](https://i-blog.csdnimg.cn/blog_migrate/988e23ec9c1bc5c2cb776de944518fbb.png)

![Spark-on-YARN setup, distribution, and test 2](https://i-blog.csdnimg.cn/blog_migrate/3ef5236a1afce239f9ffc82f5f36fcf5.jpeg)

![Spark-on-YARN setup, distribution, and test 3](https://i-blog.csdnimg.cn/blog_migrate/9136817203a0f1cf304a04e61fbfc485.jpeg)

## 1.10 Python installation

The CentOS lab's root commands were:

```bash
yum search python3
yum install python3-devel.x86_64
```

# 2. Deployment lessons

Hadoop installation requires careful version compatibility (VMware, CentOS, Java, Hadoop, Scala, Spark), reliable IP/hostname mappings, valid Java/Hadoop environment variables, accurate XML, SSH authentication and service verification. Test local, pseudo-distributed and fully distributed modes separately; check networking, logs and backups when troubleshooting.

# 3. Common problems and solutions

## Command not found

Check the `PATH` definitions in `~/.bashrc`, then reload them:

```bash
source ~/.bashrc
```

![Command-not-found error](https://i-blog.csdnimg.cn/blog_migrate/1f833243603f0843747c2b7ca6c81b99.png)

## Missing DataNode

Check first-time initialization, environment variables, the output of `ip a`, the IP-to-hostname mapping in `/etc/hosts`, unusual punctuation in hostnames and whether the machine needs a reboot. Do **not** routinely reformat an existing NameNode.

## HDFS starts but YARN encounters resource problems

The original lab adjusted the NodeManager memory and HDFS replication factor:

```xml
<property>
  <name>yarn.nodemanager.resource.memory-mb</name>
  <value>1024</value>
</property>
<property>
  <name>dfs.replication</name>
  <value>2</value>
</property>
```

Choose values according to actual machine memory and DataNode count.

## Nodes cannot communicate

Verify that the nodes can `ping` each other; check IP assignments, routing, firewall configuration and SSH authentication.

## HDFS initialization failed

For `error parsing conf core-site.xml`, examine the XML and logs. In the original example, the `name` element had been mistyped as `neme`; correcting it fixed the error.

![Configuration parsing issue](https://i-blog.csdnimg.cn/blog_migrate/9dbc1a4a1d7b0c45dfb57e1d95df29e6.png)

For `No route to host`, check the firewall, IP assignments and `/etc/hosts`.

## Permission errors

The original repair commands were:

```bash
chown -R hadoop:hadoop /home/hadoop
chmod 700 /home/hadoop
chmod 700 /home/hadoop/.ssh
chmod 644 /home/hadoop/.ssh/authorized_keys
chmod 600 /home/hadoop/.ssh/id_rsa
```

Verify that the Hadoop files belong to the intended user and group. Exercise care before changing ownership recursively.

## Missing NameNode

The original article suggested `hadoop namenode -format` as a possible fix. **This destroys namespace data on an existing installation**. Inspect logs and configuration first; use formatting only when initializing a new, empty test cluster.

---

*Source: [Hadoop deployment tutorial on my CSDN blog](https://blog.csdn.net/weixin_75027717/article/details/137211303). Screenshots are linked to the originals; text and commands have been translated and formatted for this archive.*