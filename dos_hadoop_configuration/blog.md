# hadoop配置与wordcount



参考的博客大多都是hadoop2.x和低版本的java之上的，配置过程写出来看似很简单，看别人的博客也感觉步骤都差不多，但是自己配置时候出了很多问题：datanode启动不了，网页不能正常显示，datanode莫名死掉，resourcemanager启动不了，nodemanager启动不了，mapreduce过程中无法连接到slave等等。这个过程看博客看日志折腾了许多时间才弄好，记录一下。

我是在虚拟机中安装了四个linux系统作为节点，所需环境相同，因此这里先配置一台，然后用虚拟机自带的功能直接复制得到其他三台。

环境：

* Macos , Parallels Desktop
* Linux 16.04
* Jdk 1.8.0
* Hadoop 3.2.0



## Java 环境配置

在oracle官网下载最新的jdk压缩文件，复制到安装的目标目录下解压：

```bash
sudo tar -zxvf jdk-12_linux-x64_bin.tar.gz
sudo rm jdk-12_linux-x64_bin.tar.gz
```

然后配置环境变量。可以写在~/.bashrc或者/etc/profile中，其中~/.bashrc是在用户的主目录下，只对当前用户生效，/etc/profile是所有用户的环境变量。

```bash
vim /etc/profile
```

在末尾加入jdk的环境变量

```sh
JAVA_HOME=/usr/lib/jdk-12
CLASSPATH=.:$JAVA_HOME/lib.tools.jar
PATH=$JAVA_HOME/bin:$PATH
export JAVA_HOME CLASSPATH PATH
```

之后``source /etc/profile``生效，``java —version``检查是否配置正确。

在后面启动resourcemanager时候出现了问题，**更换成了jdk8**，过程同上。



## ssh 免密钥连接

接着安装hadoop，过程放在下一部分，安装好了之后复制生成三个相同环境的虚拟机。我用的是parallels，相比于其他的比较稳定易用。

![](https://github.com/liuoorui/MyBlog/raw/master/dos_hadoop_configuration/pics/copy_system.png)

接着就是分布式的部分了。纯的分布式是很难实现的，hadoop仍然是用一个master来集中式地管理数据节点，master并不存储数据，而是将数据存储在datanode之中，这里命名为slave1, slave2, slave3三个datanode，网络连接均为桥接。因此master需要能免密钥登陆到slave。添加节点的ip地址（为了在ip变化时候不用再重新配置，可以配置静态ip）：

```bash
vim /etc/hosts
192.168.31.26   master
192.168.31.136  slave1
192.168.31.47   slave2
192.168.31.122  slave3

vim /etc/hostname
master # 分别配置slave1, slave2, slave3

ping slave1 # 测试
```

安装ssh，这个在ubuntu官方的源里面很慢，我试图换到国内的清华和阿里云等的源，但里面是没有的，也可能是有不同的版本之类的原因吧。懒得去管的话直接耐心等待就好了。

```bash
sudo apt-get install ssh
```

然后生成公钥和私钥：

```bash
ssh-keygen -t rsa
```

这里默认路径是用户主目录下.ssh，一路回车就好了。

使每台主机能够免密钥连接自己：

```bash
cp .id_rsa.pub authorized_keys
```

接着为了使master能够免密钥连接到slave，将master的公钥追加到每个slave的authorized_keys中。

![](https://github.com/liuoorui/MyBlog/raw/master/dos_hadoop_configuration/pics/ssh_authorized_keys.png)
然后测试是否能够正常连接：

```bash
ssh slave1
```



## 安装配置hadoop

从官网下载hadoop3.2，解压到/usr/lib/。并且将读权限分配给hadoop用户

```bash
cd /usr/lib
sudo tar –xzvf hadoop-3.2.0.tar.gz
chown –R hadoop:hadoop hadoop #将文件夹"hadoop"读权限分配给hadoop普通用户
sudo rm -rf hadoop-3.2.0.tar.gz
```

添加环境变量：

```bash
HADOOP_HOME=/usr/lib/hadoop-3.2.0
PATH=$HADOOP_HOME/bin:$PATH
export HADOOP_HOME PATH
```

接着是最重要的配置hadoop部分，分别配置HADOOP_HOME/etc/hadoop/下的以下几个文件：

*hadoop-env.sh*

```sh
export JAVA_HOME=/usr/lib/jdk1.8.0_201
```

*core-site.xml*

```xml
<configuration>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/usr/lib/hadoop-3.2.0/tmp</value>
        <description>Abase for other temporary directories.</description>
    </property>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://master:9000</value>
    </property>
</configuration>

```

*hdfs-site.xml*

```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
    <property>
      <name>dfs.name.dir</name>
      <value>/usr/lib/hadoop-3.2.0/hdfs/name</value>
    </property>
    <property>
      <name>dfs.data.dir</name>
      <value>/usr/lib/hadoop-3.2.0/hdfs/data</value>
    </property>
</configuration>

```

*yarn-site.xml*

```xml
<configuration>
    <property>
      <name>yarn.resourcemanager.address</name>
      <value>master:8032</value>
    </property>
    <property>
      <name>yarn.resourcemanager.scheduler.address</name>
      <value>master:8030</value>
    </property>
    <property>
      <name>yarn.resourcemanager.resource-tracker.address</name>
      <value>master:8031</value>
    </property>
    <property>
      <name>yarn.resourcemanager.admin.address</name>
      <value>master:8033</value>
    </property>
    <property>
      <name>yarn.resourcemanager.webapp.address</name>
      <value>master:8088</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
        <value>org.apache.hadoop.mapred.ShuffleHandler</value>
    </property>
</configuration>

```

*mapred-site.xml*

```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
        <property>
        <name>mapred.job.tracker</name>
        <value>master:49001</value>
    </property>
    <property>
        <name>mapred.local.dir</name>
        <value>/usr/lib/hadoop-3.2.0/var</value>
    </property>

        <property>
                <name>yarn.app.mapreduce.am.env</name>
                <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
        </property>
        <property>
                <name>mapreduce.map.env</name>
                <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
        </property>
        <property>
                <name>mapreduce.reduce.env</name>
                <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
        </property>
</configuration>
```

*workers*

```
slave1
slave2
slave3
```

这些做完之后就配置完了，接着将整个文件夹复制到其他三台主机就完成了。



## 启动

格式化namenode

```bash
hdfs namenode -format # 前提是已经将HADOOP_HOME添加到环境变量中
```

![](https://github.com/liuoorui/MyBlog/raw/master/dos_hadoop_configuration/pics/formated.png)

如果看到如上INFO说明这一步成功了。然后运行start脚本：

```bash
./sbin/start-all.sh # 在hadoop 2.x版本放在./bin/下面
```

![](https://github.com/liuoorui/MyBlog/raw/master/dos_hadoop_configuration/pics/start_all.png)

用``jps``查看Java进程，master应该包含NameNode, SecondaryNameNode, ResourceManager，slave应该包含DataNode, NodeManager。这里很常见的问题包括没有datanodes，没有访问权限，resouecemanager不能启动等，一些原因我写在下面了，大部分都是配置出了问题，查看log文件就能找到原因。

通过``master:9870``可以网页查看集群状态。

![](https://github.com/liuoorui/MyBlog/raw/master/dos_hadoop_configuration/pics/web.png)



## WordCount示例程序

wordcount可以说是hadoop学习过程中的"hello world"，网上可以找到源码，也可以自己写，我这里直接用了官方$HADOOP_HOME/share/hadoop/mapreduce/下的示例程序。

先将输入文件传到dfs中，我这里是自己写了两个含有"hadoop", "hello", "world"单词的txt文件。然后运行示例程序：

```bash
hdfs dfs -mkdir /in
hdfs dfs -put ~/Desktop/file*.txt /in
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.0.jar wordcount /in /out
```

![](https://github.com/liuoorui/MyBlog/raw/master/dos_hadoop_configuration/pics/wordcount_res.png)

这里可以看到mapreduce分为map和reduce过程。mapreduce分为map，shuffle，reduce过程，先将大任务分到各个节点分别计算，然后shuffle是按照一定的规则将不同的key值分到不同的节点做整合，然后提交任务再reduce整合。查看结果：

```bash
hdfs dfs -cat /out/part-r-00000
```

![](https://github.com/liuoorui/MyBlog/raw/master/dos_hadoop_configuration/pics/wordcount_out.png)

至此hadoop集群环境才能说是正确安装了。接下来就是修改wordcount代码自己玩了，上手后就可以自己写了。



## 一些遇到的问题

* 复制配置好的文件夹时候不小心复制错了，复制成了之前一次配置失败时候用过的文件夹，导致datanode启动一直失败，但是全程无提示。谷歌好久解决不了。后来看datanode的log文件找到错误的地方，是core-site.xml出了问题，修改之后重新格式化，启动成功。

  这个悲伤的故事告诉我们，出了问题先去看看log文件定位错误，大家的错误千奇百怪，谷歌不是万能的。

* **没有resourcemanager和nodemanager**：查看日志找到原因为classNoFound(javax.XXXXXXX)。发现是由于java9以上的一些限制，默认禁用了javax的API，参考博客得到解决办法有两个：

  1. 在``yarn-env.sh``中添加(但是我试过不可行，由于本人不会java，因此放弃深究)

     ```sh
     export YARN_RESOURCEMANAGER_OPTS="--add-modules=ALL-SYSTEM"
     export YARN_NODEMANAGER_OPTS="--add-modules=ALL-SYSTEM"
     ```

  2. 更换为jdk8

* 第一次运行wordcount程序时候将$HADOOP_HOME/etc/hadoop整个文件夹全传入作为输入，结果出错，根据log发现是内存不足，我的每个虚拟机只开了1G的内存。由此可见这样的配置只是仅仅能够作为熟悉hadoop分布式环境用途，根本达不到能够解决问题的条件。