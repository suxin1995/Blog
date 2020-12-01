# mac 搭建Hadoop集群（一）

### 配置必要java环境

##### 删除已有jdk

````shell
rpm -qa | grep jdk
````

​	查看当前环境是否安装openjdk  卸载不干净jdk 防止版本冲突

<img src='src/2020-12-1-3.png' style='zoom:70%'>

​	卸载已安装jdk与jdk-configs

````shell
yum remove *openjdk*
yum remove copy-jdk-configs-3.3-10.el7_5.noarch
````

##### 上传解压jdk

​	当前使用版本：jdk1.8.0-202	

​	上传当前合适JDK安装包到/opt目录下 并解压。

````
tar xzvf jdk-8u202-linux-x64.tar.gz
````

##### 设置环境路径

​	设置JAVA_HOME：

````shell
vi /etc/profile
````

​	并在文本中编辑模式下添加路径：

````shell
export JAVA_HOME=/opt/jdk1.8.0_202
export PATH=$PATH:$JAVA_HOME/bin:$JAVA_HOME/sbin
````

​	:wq 退出保存后重启该文件使其生效:  source  /etc/profile

##### 节点同步

​	最后，将上述操作同步到其他节点。同样的删除openjdk，然后将master节点上已经解压的jdk复制发送到子节点上：

````shell
scp -r /opt/jdk1.8.0_202 root@slave1:/opt/
scp -r /opt/jdk1.8.0_202 root@slave2:/opt/
scp -r /opt/jdk1.8.0_202 root@slave3:/opt/
````

​	同样将profile文件复制发送到其他节点覆盖，避免重复修改

```shell
scp /etc/profile root@slave1:/etc/
scp /etc/profile root@slave2:/etc/
scp /etc/profile root@slave3:/etc/
```

​	最后，每个节点分别执行source /etc/profile 使其生效。通过java -version 测试是否配置成功：

<img src='src/2020-12-1-4.png' style='zoom:70%'>



### 安装Hadoop



##### 上传并解压

​	当前使用版本：hadoop-2.7.3

​	cd 到资源路径解压：

```shell
tar zxvf hadoop-2.7.3.tar.gz
```

详见二