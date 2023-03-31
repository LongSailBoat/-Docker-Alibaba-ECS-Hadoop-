# -Docker-Alibaba-ECS-Hadoop-
# 用Docker实现一台Alibaba轻量ECS搭建Hadoop集群

**-------------------------------------------------------------------------------------------------------------------------------------------------------------**
## 一、开始前的准备

### 1.用SSH软件登录ECS(putty,Xshell等)

### 2.centos安装docker：

    curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun

### 3.启动docker：

    sudo systemctl start docker
  
### 4.拉取一个centos8的镜像：

    docker pull centos:8
  
  
**-------------------------------------------------------------------------------------------------------------------------------------------------------------**
## 二、创建一个容器 java_ssh_proto，用于配置一个包含 Java 和 SSH 的环境

### 1.创建容器：

    docker run -d --name=java_ssh_proto --privileged centos:8 /usr/sbin/init
  
### 2.进入容器：

    docker exec -it java_ssh_proto bash
  
### 3.配置镜像：

    sed -e 's|^mirrorlist=|#mirrorlist=|g' -e 's|^#baseurl=http://mirror.centos.org/$contentdir|baseurl=https://mirrors.ustc.edu.cn/centos|g' -i.bak /etc/yum.repos.d/CentOS-Linux-AppStream.repo /etc/yum.repos.d/CentOS-Linux-BaseOS.repo /etc/yum.repos.d/CentOS-Linux-Extras.repo /etc/yum.repos.d/CentOS-Linux-PowerTools.repo /etc/yum.repos.d/CentOS-Linux-Plus.repo
  
### 4.yum makecache

  这里会出现问题，报错Error: Failed to download metadata for repo 'appstream'
  
  问题原因：在2022年1月31日，CentOS团队终于从官方镜像中移除CentOS 8的所有包。CentOS 8已于2021年12月31日寿终正寝，但软件包仍在官方镜像上保留了一段时间。现在他们被转移到https://vault.centos.org
  
  解决办法：如果你仍然需要运行你的旧CentOS 8，你可以在/etc/yum.repos中更新repos.d使用vault.centos.org代替mirror.centos.org。
  
  解决步骤：
  
    cd /etc/yum.repos.d
    vi CentOS-Linux-BaseOS.repo
  注释已有的baseurl
  添加新的bseurl:
  
    baseurl=https://vault.centos.org/centos/$releasever/BaseOS/$basearch/os/
  
    vi CentOS-Linux-AppStream.repo
  注释已有的baseurl
  添加新的bseurl:
  
    baseurl=https://vault.centos.org/centos/$releasever/AppStream/$basearch/os/ 
  
### 5.再次yum makecache即可成功

### 6.安装 OpenJDK 8 和 SSH 服务：

    yum install -y java-1.8.0-openjdk-devel openssh-clients openssh-server
  
### 7.启用 SSH 服务:

    systemctl enable sshd && systemctl start sshd

### 8.退出容器，然后运行以下两条命令停止容器，并保存为一个名为 java_ssh 的镜像：

    docker stop java_ssh_proto
    docker commit java_ssh_proto java_ssh


**-------------------------------------------------------------------------------------------------------------------------------------------------------------**
## 三、Hadoop 安装

### 1.在hadoop官网(Hadoop 官网地址：http://hadoop.apache.org/)下载压缩包Hadoop 3.1.4 镜像，并将它上传到ecs

### 2.现在以之前保存的 java_ssh 镜像创建容器 hadoop_single：

    docker run -d --name=hadoop_single --privileged java_ssh /usr/sbin/init
  
### 3.将下载好的 hadoop 压缩包拷贝到容器中的 /root 目录下:

    docker cp <你存放hadoop压缩包的路径> hadoop_single:/root/
  
### 4.进入容器root文件夹，解压文件，并将它拷贝到一个常用的地方：

    tar -zxf hadoop-3.1.4.tar.gz
    mv hadoop-3.1.4 /usr/local/hadoop
  
### 5.配置环境变量：

    echo "export HADOOP_HOME=/usr/local/hadoop" >> /etc/bashrc
    echo "export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin" >> /etc/bashrc 
  
### 6.然后退出 docker 容器并重新进入，这时，echo $HADOOP_HOME 的结果应该是 /usr/local/hadoop

    echo "export JAVA_HOME=/usr" >> $HADOOP_HOME/etc/hadoop/hadoop-env.sh
    echo "export HADOOP_HOME=/usr/local/hadoop" >> $HADOOP_HOME/etc/hadoop/hadoop-env.sh
  
### 7.输入hadoop version检查hadoop安装情况出现问题bash: hadoop: command not found

  解决办法：
  
  vi /etc/profile，进入编辑器后，添加Hadoop路径：
  
    export HADOOP_HOME=/usr/local/hadoop
    export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
  
  退出编辑器，source /etc/profile，再修改启动项：
  
    vi ~/.bashrc
  
  在最后添加：
    source /etc/profile
  
  这里hadoop version就可以显示hadoop版本了
  
  
**-------------------------------------------------------------------------------------------------------------------------------------------------------------**
## 四、HDFS配置与使用

### 1.进入hadoop_single容器，新建用户，名为 hadoop：

    adduser hadoop
  
  安装一个小工具用于修改用户密码和权限管理：
  
    yum install -y passwd sudo
  
  设置 hadoop 用户密码:
  
    passwd hadoop
  
  修改 hadoop 安装目录所有人为 hadoop 用户：
  
    chown -R hadoop /usr/local/hadoop
  
  用文本编辑器修改 /etc/sudoers 文件，在
    root    ALL=(ALL)       ALL
  之后添加一行
    hadoop  ALL=(ALL)       ALL
  然后退出容器。
  
### 2.在hadoop用户下登录容器

  执行vi ~/.bashrc在最后添加 
  
    source /etc/profile
  
  这里的操作是为了之后用hadoop用户登录不用再一直执行这样的启动项操作
  
### 3.关闭并提交容器 hadoop_single 到镜像 hadoop_proto：

    docker stop hadoop_single
    docker commit hadoop_single hadoop_proto
  
**-------------------------------------------------------------------------------------------------------------------------------------------------------------**
_接下来的4-9实现的是单容器下的HDFS，用多个容器实现hadoop集群的可以跳过这段，直接从步骤10开始_

### 4.创建新容器 hdfs_single：

    docker run -d --name=hdfs_single --privileged hadoop_proto /usr/sbin/init
  
### 5.进入刚建立的容器：

    docker exec -it hdfs_single su hadoop
  
  whoami显示是hadoop用户
  
  生成 SSH 密钥：
  
    ssh-keygen -t rsa
    
  这里可以一直按回车直到生成结束
  
  然后将生成的密钥添加到信任列表：
  
    ssh-copy-id hadoop@172.17.0.2
  
  查看容器 IP 地址：
  
    ip addr | grep 172
  
  从而得知容器的 IP 地址是 172.17.0.2
  
### 6.在启动 HDFS 以前我们对其进行一些简单配置，Hadoop 配置文件全部储存在安装目录下的 etc/hadoop 子目录下，所以我们可以进入此目录：

    cd $HADOOP_HOME/etc/hadoop
  
  修改两个文件：core-site.xml 和 hdfs-site.xml
  
  在 core-site.xml 中，我们在 标签下添加属性：
  
	<property>
		<name>fs.defaultFS</name>
    	<value>hdfs://172.17.0.2:9000</value>
	</property>
  
  在 hdfs-site.xml 中的 标签下添加属性：
  
	<property>
    <name>dfs.replication</name>
    <value>1</value>
	</property>
  
### 7.格式化文件结构：

    hdfs namenode -format
  
### 8.启动 HDFS：

    start-dfs.sh
  
### 9.JPS看到NN,SNN,DN进程

**-------------------------------------------------------------------------------------------------------------------------------------------------------------**
_接下来的10-18是用多个容器实现hadoop集群，总体思路是这样的，我们先用一个包含 Hadoop 的镜像进行配置，配置成集群中所有节点都可以共用的样子，然后再以它为原型生成若干个容器，构成一个集群_

### 10.首先，我们将使用之前准备的 hadoop_proto 镜像启动为容器：

    docker run -d --name=hadoop_temp --privileged hadoop_proto /usr/sbin/init
  
### 11.进入 Hadoop 的配置文件目录：

    cd $HADOOP_HOME/etc/hadoop
  
  首先编辑 workers ，更改文件内容为：
  
    dn1
	dn2
  
  然后编辑 core-site.xml，在 中添加以下配置项：
  
  <!-- 配置 HDFS 主机地址与端口号 -->
	<property>
    	<name>fs.defaultFS</name>
    	<value>hdfs://nn:9000</value>
	</property>
	<!-- 配置 Hadoop 的临时文件目录 -->
	<property>
    	<name>hadoop.tmp.dir</name>
    	<value>file:///home/hadoop/tmp</value>
	</property>
  
  配置 hdfs-site.xml，在 中添加以下配置项：
  
  <!-- 每个数据块复制 2 份存储 -->
	<property>
    	<name>dfs.replication</name>
    	<value>2</value>
	</property>
	<!-- 设置储存命名信息的目录 -->
	<property>
    	<name>dfs.namenode.name.dir</name>
    	<value>file:///home/hadoop/hdfs/name</value>
	</property>
  
### 12.最后需要配置一下 SSH ：

	ssh-keygen -t rsa -P "" -f ~/.ssh/id_rsa
	ssh-copy-id -i ~/.ssh/id_rsa hadoop@localhost
  
### 13.需要补的免密操作：

  先将root/.ssh下的id_rsa.pub发送给anthorzied_keys：
  
    cat id_rsa.pub >> authorized_keys
  
  再将root/.ssh下的文件复制一份给home/hadoop/,ssh下，并在root下把id_rsa操作：
  
    chmod 777 id_rsa
    
  让id_rsa变成其他用户可读，
  
    chmod 666 known_hosts
    
  让known_hosts其他用户可写，这样就可以了
  
### 14.到此为止，集群的原型就配置完毕了，可以退出容器并上传容器到新镜像 cluster_proto ：

    docker stop hadoop_temp
	docker commit hadoop_temp cluster_proto
  
### 15.部署集群，首先，要为 Hadoop 集群建立专用网络 hnet ：

    docker network create --subnet=172.20.0.0/16 hnet
  
### 16.创建集群容器：

    docker run -d --name=nn --hostname=nn --network=hnet --ip=172.20.1.0 --add-host=dn1:172.20.1.1 --add-host=dn2:172.20.1.2 --privileged cluster_proto /usr/sbin/init
    docker run -d --name=dn1 --hostname=dn1 --network=hnet --ip=172.20.1.1 --add-host=nn:172.20.1.0 --add-host=dn2:172.20.1.2 --privileged cluster_proto /usr/sbin/init
    docker run -d --name=dn2 --hostname=dn2 --network=hnet --ip=172.20.1.2 --add-host=nn:172.20.1.0 --add-host=dn1:172.20.1.1 --privileged cluster_proto /usr/sbin/init

### 17.进入命名节点：

    docker exec -it nn su hadoop
  格式化 HDFS：
  
    hdfs namenode -format
  
  启动 HDFS：
  
    start-dfs.sh
  
### 18.在NN中JPS可以看到Namenode进程和SecondaryNamenode进程，在DN1和DN2中可以看到DataNode进程，Hadoop集群搭建完毕

  
  
  

  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
