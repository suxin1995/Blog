	根据本机实际情况对操作做适当删减。

本机当前环境：centos7  自带python2.7

安装版本：

| 安装工具 | 版本    | 用途                                        |
| :------- | :------ | :------------------------------------------ |
| Python   | 3.6.5   | 安装airflow及其依赖包、开发airflow的dag使用 |
| MySQL    | 5.7     | 作为airflow的元数据库                       |
| Airflow  | 1.10.10 | 任务调度平台                                |

### 安装Python3

```shell
#python依赖
yum -y install zlib zlib-devel
yum -y install bzip2 bzip2-devel
yum -y install ncurses ncurses-devel
yum -y install readline readline-devel
yum -y install openssl openssl-devel
yum -y install openssl-static
yum -y install xz lzma xz-devel
yum -y install sqlite sqlite-devel
yum -y install gdbm gdbm-devel
yum -y install tk tk-devel
yum install gcc

#安装wget命令
yum -y install wget
#使用wget下载Python源码压缩包到/root目录下
wget -P /root https://www.python.org/ftp/python/3.6.5/Python-3.6.5.tgz
#在当前目录解压Python源码压缩包
tar -zxvf Python-3.6.5.tgz
#进入解压后的文件目录下
cd /root/Python-3.6.5
#检测及校验平台
./configure --with-ssl --prefix=/service/python3
#编译Python源代码
make
#安装Python
make install
#备份原来的Python软连接
mv /usr/bin/python /usr/bin/python2.backup
#制作新的指向Python3的软连接
ln -s /service/python3/bin/python3 /usr/bin/python
#建立pip的软连接
ln -s /service/python3/bin/pip3 /usr/bin/pip 
#若提示已存在仍需同备份python3软链一样mv
#查看Python版本
python -V
#检测pip是否可用
pip
#升级pip
pip install --upgrade pip
#获取yum命令所在位置
whereis yum
#yum: /usr/bin/yum /etc/yum /etc/yum.conf /usr/share/man/man8/yum.8
#编辑yum文件
vi /usr/bin/yum /etc/yum /etc/yum.conf /usr/share/man/man8/yum.8
#进入编辑模式
i
#修改第一行内容（看系统版本，centos7对应2.7，centos6对应2.6）
#修改前：
#!/usr/bin/python
#修改后：
#!/usr/bin/python2.7
#退出编辑模式
esc
#保存文件
：wq
#按上述方式编辑以下文件，修改第一行内容
/usr/libexec/urlgrabber-ext-down
```

### 安装Mysql

```shell
#卸载mariadb
rpm -qa | grep mariadb
rpm -e --nodeps mariadb-libs-5.5.52-1.el7.x86_64
#sudo rpm -e --nodeps mariadb-libs-5.5.52-1.el7.x86_64
rpm -qa | grep mariadb
```

```shell
#下载mysql的repo源
wget -P /root http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
#通过rpm安装
rpm -ivh mysql-community-release-el7-5.noarch.rpm
#安装mysql
yum install mysql-server
#授权
chown -R mysql:mysql /var/lib/mysql
#开启Mysql服务
service mysqld start
#用root用户连接登录mysql:
mysql -uroot
#重置mysql密码
use mysql;
update user set password=password('root') where user='root';
flush privileges;
#为Airflow建库、建用户
#建库:
create database airflow;
#建用户:
create user 'airflow'@'%' identified by 'airflow';
create user 'airflow'@'localhost' identified by 'airflow';
#为用户授权:
grant all on airflow.* to 'airflow'@'%';
grant all on airflow.* to 'root'@'%';
flush privileges;
exit;
```

### 安装Airflow

```shell
#设置临时环境变量
export SLUGIFY_USES_TEXT_UNIDECODE=yes
#添加编辑环境变量
vi /etc/profile

export PS1="[\u@\h \w]\$ "
#Python环境变量
export PYTHON_HOME=/service/python3
export PATH=$PATH:$PYTHON_HOME/bin
#Airflow环境变量
export AIRFLOW_HOME=/root/airflow
export SITE_AIRFLOW_HOME=/service/python3/lib/python3.6/site-packages/airflow
export PATH=$PATH:$SITE_AIRFLOW_HOME/bin

#生效环境变量
source /etc/profile
#安装apache-airflow并且指定1.10.0版本
pip install apache-airflow===1.10.10
#为airflow安装mysql模块
pip install 'apache-airflow[mysql]'
#出现报错： OSError: mysql_config not found
find / -name mysql_config
#如果没有
yum -y install mysql-devel  
#安装完成之后再次pip apache-airflow[mysql]

#采用mysql作为airflow的元数据库
pip install mysqlclient
pip install pymysql
pip install cryptography
#执行命令
python -c "from cryptography.fernet import Fernet; 
print(Fernet.generate_key().decode())"
#这个命令生成一个key，复制这个key然后替换airflow.cfg文件里的fernet_key的值

#为避免初始化元数据库失败  需要修改my.cnf
vi /etc/my.cnf
#在[mysqld]下面（一定不要写错地方）添加如下配置:
explicit_defaults_for_timestamp=true
#重启mysql服务使配置生效
service mysqld restart
#检查配置是否生效
mysql -uroot -proot
mysql> select @@global.explicit_defaults_for_timestamp;
+------------------------------------------+
| @@global.explicit_defaults_for_timestamp |
+------------------------------------------+
|                                        1 |
+------------------------------------------+
1 row in set (0.00 sec)
```

airflow会被安装到python3下的site-packages目录下，完整目录为

```shell
${PYTHON_HOME}/lib/python3.6/site-packages/airflow
#绝对路径/service/python3/lib/python3.6/site-packages/airflow
```

##### 修改airflow配置

```shell
#修改vi airflow.cfg文件

#修改sql_alchemy_conn配置
sql_alchemy_conn = mysql+mysqldb://airflow:airflow@localhost:3306/airflow

#选择性修改webserver地址
base_url = http://192.168.150.128:8085
web_server_port = 8085

#选择性修改executor
#SequentialExecutor是单进程顺序执行任务，默认执行器，通常只用于测试
#LocalExecutor是多进程本地执行任务使用的
#CeleryExecutor是分布式调度使用（当然也可以单机），生产环境常用
#DaskExecutor则用于动态任务调度，常用于数据分析
executor = LocalExecutor

#修改时区
default_timezone = Asia/Shanghai

#选择性修改scheduler线程数控制并发量
parallelism = 32

#选择性修改检测新DAG间隔
min_file_process_interval = 5

```

##### 初始化启动

```shell
#初始化元数据库信息
airflow initdb 
#或者使用airflow resetdb

#启动组件：
airflow webserver -D
#airflow scheduler -D
#遇到bug尚未解决 守护进程运行scheduler会提示过期未检测心跳 直接运行无碍
```



参考文档链接[🔗](https://tech.cuixiangbin.com/?p=1380)

修改中国时区[🔗](https://blog.csdn.net/Crazy__Hope/article/details/83688986)



### Airflow接入钉钉报警

​	打开钉钉客户端-> 机器人管理。创建自定义机器人🤖️获取webhook。安全设置图方便选择了  关键词方式。

<img src="src/2020-12-9-1.png" style="zoom:50%;" />

<img src="src/2020-12-9-2.png" style="zoom:50%;" />

<img src="src/2020-12-9-3.png" style="zoom:50%;" />

​	在Airflow web管理界面创建新连接

<img src="src/2020-12-10-1.png" style="zoom:50%;" />

<img src="src/2020-12-10-2.png" style="zoom:50%;" />

​	保存后编写调用代码

```python
from airflow.contrib.operators.dingding_operator import DingdingOperator

def failure_callback_dingding(self,context):
  message = '自动报警\n  AIRFLOW TASK FAILURE TIPS:\n' \
  'DAG:    {}\n' \
  'TASKS:  {}\n' \
  'Reason: {}\n' \
  .format(context['task_instance'].dag_id,
          context['task_instance'].task_id,
          context['exception'])
  return DingdingOperator(
    task_id='dingding_failure_callback',
    dingding_conn_id='dingding_talk',
    message_type='text',
    message=message,
    at_mobiles=[self.phone],
    # at_all=True,
  ).execute(context)

default_args = {
    'owner': airflow,
    'depends_on_past': False,
    'start_date': datetime(2020, 12, 9),
    'on_failure_callback':config.failure_callback,
}

dag = DAG(
    dag_id=dag_id,  # dag_id
    default_args=default_args,  # 指定默认参数
    schedule_interval=timedelta(minutes=5)
)
```

​	经验证接入成功

<img src="src/2020-12-10-3.png" style="zoom:50%;" />



### Airflow接入邮箱报警





