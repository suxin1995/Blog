

​	hive 主要用来并行分布式处理大量数据。所有查询除了 select * from table 都需要通过mapreduce方式执行。即使一个只有1行1列的表，若通过mapreduce方式查询可能也需要8、9秒。

​	hive在加载数据过程中不会对数据进行任何处理，要访问数据中满足条件的特定值，需要暴力扫描整个数据。因此访问延迟较高。不过由于mapreduce的引入，hive可以并行访问数据，在没有索引的情况下，对大数据量的访问可以体现优势。高延迟的数据访问速度，决定hive不适合在线数据查询。

https://segmentfault.com/a/1190000019299379



​	hbase（nosql数据库）主要解决实时数据查询问题，hive解决数据处理计算问题

​	hdfs管理物理上存放在多个硬盘上的数据文件，而hbase管理的类似于key-value映射表。底层仍然依赖hdfs。

​	hive是基于Hadoop一个数据仓库工具，可以将数据文件映射为数据库表，并提供简单的sql查询，可将sql语句转换成mapreduce任务执行。优点：学习成本低

​	spark数据处理速度秒杀mapreduce。处理数据方式不同。mapreduce分布对数据处理：从集群读取数据、进行一次处理、结果写入集群、从集群读取更新后数据、进行下一次处理依次叠加

Spark会在内存中以接近实时时间完成所有数据分析。





以12月3日的product_storage为测试，数据量为9300w 数据

数据导入：

​		18min 17s

简单查询：

​		select * from product_storage limit 1000；     

​				当前数据库：5s	2s   1.306s       				

​				hive：4.638s 	0.327s	0.379s（不走MR）

​		select count(*) from product_storage；   

​				当前数据库：14.982s     7.175s	6.944s				

​				hive：123.67s   124.520s	124.30s

计算：

SELECT t1.store_id,t2.store_name ,count(1) 
from ods_v2.product_storage as t1  
LEFT JOIN  store as t2  
on  t1.store_id = t2.store_id 
group by t1.store_id, t2.store_name 
ORDER BY t1.store_id DESC;			

​				当前数据库：16:22			

​				hive：