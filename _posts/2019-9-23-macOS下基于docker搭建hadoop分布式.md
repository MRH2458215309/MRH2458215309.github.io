

最近在学习大数据技术的原理和应用，需要搭建hadoop分布式集群。通常意义上讲，搭建Hadoop分布式最常见的方式有三种，一种是找多台机器上部署，二是在本地多开几个虚拟机，三是初学者最常用的方法（伪分布式）。前两种方法对硬件的要求有些苛刻，而第三种方式不能够真正体现分布式。所以，我选择利用docker搭建Hadoop分布式集群。
### 什么是docker ###
  Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的镜像中，然后发布到任何流行的 Linux或Windows 机器上，也可以实现虚拟化。容器是完全使用沙箱机制，相互之间不会有任何接口。Docker的基础是linux容器（LXC）等技术。
  在介绍我的搭建过程之前，我先简单的介绍一下docker中的容器和镜像的关系：
  在我的理解下，容器(container) = 镜像(image) + 读写层(read and write layer)。
  我们最开始在docker中下载的是镜像，接下来要对docker镜像进行相关操作，当在镜像中的操作完成后，镜像就变成了加入了读写层后的容器，当我们退出镜像而想保存在镜像中的操作时，我们需要将生成的容器保存成为新的镜像。我在实验的过程中由于一开始没有注意到这个问题，导致自己一直在新的镜像中操作，周而复始。（ps:并且当我们想删除镜像时，需将关于该镜像的所有容器删除才可下载。）
### 安装docker ###
由于本人的电脑为macOS系统，所以以下所有操作均在mac上面完成。

安装docker：在docker官网[下载docker](https://www.docker.com/products/docker-desktop)

### 利用docker搭建Hadoop运行环境 ###
安装docker成功后，开始安装搭建Linux环境，首先下载Linux（默认下载最新版，[更多docker命令](https://www.runoob.com/docker/docker-commit-command.html)）
```
docker pull centos
```

接下来，我们以centos镜像为基准镜像，搭建hadoop运行环境

启动centos镜像
```
docker run -it centos
```
注意：我们在不指定Tag的情况下，默认选择Tag为latest的镜像启动容器。 指定Tag启动命令为：
```
docker run -it centos:latest
```

然后进行java的安装：jdk最好选择java8（下面是三条命令）
```
yum -y install wget 
```
```
wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u141-b15/336fa29ff2bb4ef291e347e091f7f4a7/jdk-8u141-linux-x64.tar.gz"
```
```
tar xzf jdk-8u141-linux-x64.tar.gzyum -y install wget 
```
接着就是hadoop的安装：
```
mkdir /usr/hadoop
```
```
cd /usr/hadoop
```
```
wget 
http://mirrors.sonic.net/apache/hadoop/common/hadoop-2.8.5/hadoop-2.8.5.tar.gz
```
```
tar xvzf hadoop-2.8.5.tar.gz
```

安装好java和hadoop之后，配置环境变量
```
vi ~/.bashrc
```
```
#JAVA VARIABLES  START
export JAVA_HOME=/usr/java/jdk1.8.0_141
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin
#JAVA VARIABLES END
 
#HADOOP VARIABLES START
export HADOOP_HOME=/usr/hadoop/hadoop-2.8.5
export HADOOP_INSTALL=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export PATH=$HADOOP_HOME/sbin:$HADOOP_HOME/bin:$PATH
export CLASSPATH=$($HADOOP_HOME/bin/hadoop classpath):$CLASSPATH
#HADOOP VARIABLES END 
```
运行如下命令使配置生效
```
source ~/.bashrc
```
注意：在配置环境变量的时候java和hadoop的路径要与你本机的真实路径相符

### Hadoop 配置 ###
在配置之前，我们要在Hadoop路径下创建3个目录：
```
mkdir tmp
mkdir namenode
mkdir datanode
```

配置core-site.xml

```
<!-- Put site-specific property overrides in this file. -->
<configuration>
    <property>
            <name>hadoop.tmp.dir</name>
            <value>/usr/hadoop/hadoop-2.8.5/tmp</value>
            <description>A base for other temporary directories.</description>
    </property>
 
    <property>
            <name>fs.default.name</name>
            <value>hdfs://master:9000</value>
            <final>true</final>
            <description>The name of the default file system.  A URI whose
            scheme and authority determine the FileSystem implementation.  The
            uri's scheme determines the config property (fs.SCHEME.impl) naming
            the FileSystem implementation class.  The uri's authority is used to
            determine the host, port, etc. for a filesystem.</description>
    </property>
</configuration> 
```
配置hdfs-site.xml，在搭建集群时，我们需要一个master节点和两个slave节点，所以，我们replication设置为2:


```
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>2</value>
        <final>true</final>
        <description>Default block replication.
        The actual number of replications can be specified when the file is created.
        The default is used if replication is not specified in create time.
        </description>
    </property>
 
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>/usr/local/hadoop/hadoop-2.8.5/namenode</value>
        <final>true</final>
    </property>
 
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>/usr/local/hadoop/hadoop-2.8.5/datanode</value>
        <final>true</final>
    </property>
</configuration>
```
配置mapred-site.xml
首先变更一下现有文件名字

```
cp mapred-site.xml.template mapred-site.xml
```
```
<!-- Put site-specific property overrides in this file. -->
<configuration>
    <property>
        <name>mapred.job.tracker</name>
        <value>master:9001</value>
        <description>The host and port that the MapReduce job tracker runs
        at.  If "local", then jobs are run in-process as a single map
        and reduce task.
        </description>
    </property>
</configuration>
```
配置hadoop-env.sh
```
export JAVA_HOME=/usr/java/jdk1.8.0_141
```
格式化namenode
```
hadoop namenode -format
```
安装ssh并将生成的密钥保存（这里的操作与伪分布式完全相同，参照书本即可完成）
最后，我们将装好hadoop的镜像保存为一个新的镜像,这里注意容器的名称要对应

![avatar](/11.24.59.png)
### hadoop分布式集群搭建 ###
首先，我们要分别在3个终端分别启动一个master节点和两个slave节点，命令如下：
```
docker run  -p 50070:50070 -p 19888:19888 -p 8088:8088  --name  master  -ti -h master linux:hadoop
```
```
docker run -it -h slave1 --name slave1  linux:hadoop  /bin/bash
```
```
docker run -it -h slave2 --name slave2  linux:hadoop  /bin/bash
```
其次，通过ifconfig命令获得三个终端中不同节点的ip，本人三个ip为：
```
172.17.0.2      master
172.17.0.3      slave1
172.17.0.4      slave2
```
修改三个节点的hosts文件，并将上面的3条信息写入：
```
vi /etc/hosts
```
然后在master节点修改相应的文件
```
vi /usr/hadoop/hadoop-2.8.5/etc/hadoop/slaves
```
```
master
slave1
slave2
```
最后，启动hadoop
```
start-all.sh
```
具体成功截图如下：
![avatar](/11.47.47.png)


到此，hadoop完全分布式搭建完成，如有更好的方法或者发现本人的错误，请发送邮件给我（地址在主页）。


  
