DAG 有向无环图 

​	包含多个Task，其中定义了整个作业流程  每个dag_id 唯一。

​	并行度：

​		 parallelism ：这是用来控制每个airflow worker 可以同时运行多少个task实例。这是airflow集群的全局变量。在airflow.cfg里面配置  默认32

​		concurrency ：这个用来控制 每个dag运行过程中最大可同时运行的task实例数。如果你没有设置这个值的话，scheduler 会从airflow.cfg里面读取默认值 dag_concurrency   默认16

​		max_active_runs : 这个是用来控制在同一时间可以运行的最多的dag runs 数量。这里需要解释一下dag runs ，比如你的dag设置的每天运行，那么在天的时间段内运行某个dag就算是一个dag runs 。按道理每天只会执行一次，但是保不齐，你前天和大前天的dag都没运行，那么就需要补跑，或者你在某一次定时dag触发了之后，又手动触发了，那么就存在，同一个时间点有多个dag runs 。这个参数就是控制这个最大的dag runs  默认16



airflow 踩坑：

​	修改airflow.cfg文件中元数据库地址，需重新初始化连接。如更换另一地址mysql

```
sql_alchemy_conn = mysql+pymysql://user:passwd@localhost:3306/airflow
```



​	在使用airflow initdb 时遇到报错，

>  Global variable explicit_defaults_for_timestamp needs to be on (1) for mysql

​	在需要在新元数据库上mysql配置文件my.cnf中，mysqld下增加属性

```shell
explicit_defaults_for_timestamp = 1
```

​	之后重启mysqld。重新尝试初始化。

https://segmentfault.com/a/1190000019474821 安装参考链接

https://blog.csdn.net/sxf_123456/article/details/88238089  获取数据方式参考链接



关闭dag开关  不会影响当前running 状态的task。跑完后停止dag。未完成的task保留状态。

<img src="src/2020-11-6-1.png" style="zoom:50%;" />

再次打开后，触发执行时间节点会继续执行。

会不断更新数据。直至lastrun 时间+时间间隔 超过当前时间



web界面显示新增dag会有延迟。

https://tech.cuixiangbin.com/?p=1380  安装博客参考

D8zNjA7JuLx2fyW3pknpfKmFBb4knGIy-Ng-ia5VbJ4=



经测试：

task循环依赖会报错 该dag无法启动  提示:

且在

×Broken DAG: [/root/airflow/dags/cdl_demo.py] Cycle detected in DAG. Faulty task: task1 to task2





airflow webserver 
airflow scheduler 





{"dagjob_1_task1":"dagjob_1_task2","dagjob_1_task3":"[dagjob_1_task1,dagjob_1_task2]"}



ps aux | grep airflow |  awk '{print $2}' | xargs kill -9

