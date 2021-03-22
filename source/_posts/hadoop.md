## 概念

yarn 主要包括ResourceManager、NodeManager

## 安装

安装好jdk1.8的linux系统下

1. 创建hadoop用户

   ```sh
   $ sudo useradd -m hadoop -s /bin/bash  #创建hadoop用户，并使用/bin/bash作为shell
   $ sudo passwd hadoop                   #为hadoop用户设置密码，之后需要连续输入两次密码
   $ sudo adduser hadoop sudo             #为hadoop用户增加管理员权限
   $ su - hadoop                          #切换当前用户为用户hadoop
   $ sudo apt-get update                  #更新hadoop用户的apt,方便后面的安装
   ```

2. 设置ssh和无密码登陆

   ```sh
   $ sudo apt-get install openssh-server   #安装SSH server
   #需要版本对应，安装报依赖不合格的话就apt install 对应版本的client即可
   $ ssh localhost                         #登陆SSH，第一次登陆输入yes
   $ exit                                  #退出登录的ssh localhost
   $ cd ~/.ssh/                            #如果没法进入该目录，执行一次ssh localhost
   $ ssh-keygen -t rsa　#然后的三次输入都确定即可 直接回车
   $ cat ./id_rsa.pub >> ./authorized_keys #加入授权
   $ ssh localhost                         #此时已不需密码即可登录localhost，如果失败则可以搜索SSH免密码登录来寻求答案
   ```

3. 下载hadoop-xxx.tar.gz解压到/usr/local/hadoop...

   ```sh
   sudo tar -zxvf hadoop-xxx.tar.gz -C /usr/local
   cd /usr/local
mv hadoop-xxx/ hadoop/ #将/usr/local/hadoop-xxx 改名为 /usr/local/hadoop
   sudo chown -R hadoop ./hadoop #授权  
sudo chmod 777 /usr/local/hadoop
   ```

4. 配置环境变量 (已配置JAVA的环境变量)

   ```shell
   sudo vim /etc/profile
   ```

   ```sh
   export JAVA_HOME=/usr/local/jdk1.8
   export HADOOP_HOME=/usr/local/hadoop
   export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$JAVA_HOME/bin
   
   #export CLASSPATH=$($HADOOP_HOME/bin/hadoop classpath):$CLASSPATH
   export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
   #export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib:$HADOOP_COMMON_LIB_NATIVE_DIR"
   ```

   ```sh
   source /etc/profile #使配置文件生效
   ```

5. ```hadoop version```即可查看版本

## 配置

可以在官网下查看各配置作用,左下角config

```http
https://hadoop.apache.org/docs/r2.10.0/
```

hadoop有默认配置文件和自定义配置文件，jar包中可以找到default.xml文件

- 默认配置文件

  | 默认文件           | jar包                                   |
  | ------------------ | :-------------------------------------- |
  | core-default.xml   | hadoop-common-x.x.x.jar                 |
  | hdfs-default.xml   | hadoop-hdfs-x.x.x.jar                   |
  | yarn-default.xml   | hadoop-yarn-common-x.x.x.jar            |
  | mapred-default.xml | hadoop-mapreduce-client-core-.x.x.x.jar |

- 自定义配置文件

  | 默认文件        | jar包                                   |
  | --------------- | :-------------------------------------- |
  | core-site.xml   | hadoop-common-x.x.x.jar                 |
  | hdfs-site.xml   | hadoop-hdfs-x.x.x.jar                   |
  | yarn-site.xml   | hadoop-yarn-common-x.x.x.jar            |
  | mapred-site.xml | hadoop-mapreduce-client-core-.x.x.x.jar |

### 伪分布式配置

> cd /usr/local/hadoop

1. 配置hadoop-env.sh,添加环境变量

   ```sh
   export JAVA_HOME=/usr/local/jdk1.8
   ```

2. 修改core-site.xml文件

   > file:/  本地模式使用的协议
   >
   > hdfs://  分布式模式使用的协议

   ```xml
   <configuration>
   	<!--其他临时目录-->
   	<property>
   		<name>hadoop.tmp.dir</name>
   		<value>file:/usr/local/hadoop/tmp</value>
   		<description>Abase for other temporary directories.</description>
   	</property>
   	<!--默认文件系统的名称。一个URI，其模式和权限决定文件系统的实现。uri的模式决定了命名文件系统实现类的配置属性(fs. schema .impl)。uri的权限用于确定文件系统的主机、端口等。默认file:///-->
   	<property>
   		<name>fs.defaultFS</name>
   		<value>hdfs://localhost:9000</value>
   	</property>
   </configuration>
   ```

3. 修改hdfs-site.xml

   >  dfs.replication副本数：默认3

   ```xml
   <configuration>
   	<!--默认的块复制。可以在创建文件时指定复制的实际数目。如果在创建时未指定复制，则使用默认值 3-->
   	<property>
   		<name>dfs.replication</name>
   		<value>1</value>
   	</property>
   	<property>
   		<name>dfs.namenode.name.dir</name>
   		<value>file:/usr/local/hadoop/tmp/dfs/name</value>
   	</property>
   	<!-- 确定DFS数据节点应该将其块存储在本地文件系统的哪个位置。如果这是一个以逗号分隔的目录列表，那么数据将存储在所有命名的目录中，通常在不同的设备上。对于HDFS存储策略，目录应该使用相应的存储类型([SSD]/[DISK]/[ARCHIVE]/[RAM_DISK])进行标记。如果目录没有显式标记的存储类型，则默认存储类型为DISK。如果本地文件系统权限允许，将创建不存在的目录。默认值 file://${hadoop.tmp.dir}/dfs/data -->
   	<property>
   		<name>dfs.datanode.data.dir</name>
   		<value>file:/usr/local/hadoop/tmp/dfs/data</value>
   	</property>
   </configuration>
   ```

4. 格式化NameNode

   ```sh
   bin/hdfs namenode -format
   ```

   > 有数据的时候格式化namenode会使/usr/local/hadoop/tmp/dfs/data/name/current/VERSION中的clusterID会和datanode的clusterID不一致,所以必须先删除DataNode中的数据

5. 启动

   ```shell
   sbin/start-all.sh 
   ```

   启动完成可以使用jps查看进程

   > DataNode
   >
   > NameNode
   >
   > NodeManager
   >
   > ResourceManager
   >
   > SecondaryNameNode

6. 配置yarn (非必须)  重启

   > 1. 在/usr/local/hadoop-xxx/etc/hadoop目录下
   >
   >    ```shell
   >    cp mapred-site.xml.template mapred-site.xml
   >    ```
   >
   > 2. 修改mapred-site.xml
   >
   >    ```xml
   >    <configuration>
   >    	<!-- 用于执行MapReduce作业的运行时框架。可以是local, classic 或者 yarn -->
   >    	<property>
   >    		<name>mapreduce.framework.name</name>
   >    		<value>yarn</value>
   >    	</property>
   >    </configuration>
   >    ```
   >
   > 3. 修改yarn-site.xml
   >
   >    ```xml
   >    <configuration>
   >    	<!-- 一个逗号分隔的服务列表，其中服务名应该只包含A- z a - z 0 -9_，并且不能以数字开头。 shuffle混洗重组模式 -->
   >    	<property>
   >    		<name>yarn.nodemanager.aux-services</name>
   >    		<value>mapreduce_shuffle</value>
   >    	</property>
   >    </configuration>
   >    ```
   >
   > 4. 单独启动resourcemanager 和 nodemaneger
   >
   >    ```shell
   >    sbin/yarn-daemon.sh start resourcemanager
   >    sbin/yarn-daemon.sh start nodemanager
   >    ```

7.  启动历史资源管理器

   >  就能在8088的记录中点击<u>history</u>链接，展示对历史mp的history链接中显示详细信息

   - 配置mapred-site.xml

     ```xml
     <property>
     	<name>mapreduce.jobhistory.address</name>
     	<value>localhost:10020</value>
     </property>
     <property>
     	<name>mapreduce.jobhistory.webapp.address</name>
     	<value>localhost:19888</value>
     </property>
     ```

   - 启动/关闭 历史服务器

     ```shell
     sbin/mr-jobhistory-daemon.sh start historyserver
     sbin/mr-jobhistory-daemon.sh stop historyserver
     ```

8. 配置日志聚集 (需要重启NodeNanager ResourceNanager HistoryNanager)

   > 应用运行完成之后，将运行日志信息上传到hdfs。方便开发调试。

   - 配置yarn-site.xml

     ```xml
     <property>
     	<name>yarn.log-aggregation-enable</name>
         <value>true</value>
     </property>
     <!--日志保留7天-->
     <property>
     	<name>yarn.log-aggregation.retain-seconds</name>
         <value>604800</value>
     </property>
     ```



###### 安全模式

```sh
bin/hadoop dfsadmin -safemode leave
#enter - 进入安全模式
#leave - 强制NameNode离开安全模式
#get - 返回安全模式是否开启的信息
#wait - 等待，一直到安全模式结束。
```

## hadoop example 

### 词频统计 wordcount

1. 本地创建words.txt并写入一些单词

   > touch ~/test/words.txt

2. 上传到hdfs

   > hdfs dfs -put ~/test /input

3. map reduce

   ```sh
   hadoop jar /usr/local/hadoop-2.10.0/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.10.0.jar wordcount /input /output
   ```

4. 查看结果

   > hdfs dfs -cat /output/part-r-00000

### 正则抽取 grep

```
hadoop jar /usr/local/hadoop-2.10.0/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.10.0.jar grep /input /output-grep '正则表达式'
```



## 常用命令

### 基础命令

- 创建文件夹

  > hdfs dfs -mkdir [xx] /...
  >
  > ​	-p 多级创建  可以-mkdir -p /first/second/xx

- 查看ls

  > hdfs dfs -ls [xx] /...
  >
  > ​	-R 多级查看  可以将父目录下的子目录也显示出来

- 上传到hdfs

  > hdfs dfs -put 本地路径 hdfs路径、

- 删除

  > hdfs dfs -rm -r /xx  #删除文件夹
  >
  > hdfs dfs -rm /xxx/xx.text  #删除文件

### 安全模式

```sh
bin/hadoop dfsadmin -safemode leave
#enter - 进入安全模式
#leave - 强制NameNode离开安全模式
#get - 返回安全模式是否开启的信息
#wait - 等待，一直到安全模式结束。
```



## shuffle

混洗重组





## 分布式搭建

1. 克隆4台虚拟机，并分别配置ip




