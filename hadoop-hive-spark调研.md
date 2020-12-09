

​	hbase（nosql数据库）主要解决实时数据查询问题，hive解决数据处理计算问题

​	hdfs管理物理上存放在多个硬盘上的数据文件，而hbase管理的类似于key-value映射表。底层仍然依赖hdfs。

​	hive是基于Hadoop一个数据仓库工具，可以将数据文件映射为数据库表，并提供简单的sql查询，可将sql语句转换成mapreduce任务执行。优点：学习成本低



### hive效率

​	hive 主要用来并行分布式处理大量数据。所有查询除了 select * from table 都需要通过mapreduce方式执行。即使一个只有1行1列的表，若通过mapreduce方式查询可能也需要8、9秒。

​	hive在加载数据过程中不会对数据进行任何处理，要访问数据中满足条件的特定值，需要暴力扫描整个数据【没有索引时】，因此访问延迟较高。不过由于mapreduce的引入，hive可以并行访问数据，在没有索引的情况下，对大数据量【TB、PB级】的访问可以体现优势。高延迟的数据访问速度，决定hive不适合在线数据查询。

##### hive索引优化

注：索引可以避免全表扫描和资源浪费加快group by语句查询的计算速度。但是Hive只有有限的索引功能。没有关系型数据库中主键、外键的概念。且会消耗大量额外资源创建索引。在执行查询任务时，会先额外生成一个mr job，根据对索引列的过滤条件，从索引表中过滤出索引列的值对应的hdfs文件路径及偏移量，输出到hdfs上的一个文件中，然后根据这些文件中的hdfs路径和偏移量，筛选原始input文件，生成新的split,作为整个job的split,这样就达到不用全表扫描的目的。

但实际使用中一般不会使用索引：

使用过程比较繁琐

每次查询时候都要先用一个job扫描索引表，如果索引列的值非常稀疏，那么索引表本身也会非常大；  

索引表不会自动rebuild表有数据新增或删除，那么必须手动rebuild索引表数据



### spark 与 mapreduce 速度对比

​		spark数据处理速度秒杀mapreduce。处理数据方式不同。mapreduce分布对数据处理：从集群读取数据、进行一次处理、结果写入集群、从集群读取更新后数据、进行下一次处理依次叠加。mapreduce将中间结果保存在文件中，提高可靠性、减少内存占用但是牺牲了性能。整个计算过程会不断重复地往磁盘里读写中间结果。这样的读写数据会引起大量的网络传输以及磁盘读写，极其耗时

​	Spark会在内存中以接近实时时间完成所有数据分析。Hadoop每次shuffle操作后，必须写到磁盘，而Spark在shuffle后不一定落盘，可以cache到内存中，以便迭代时使用。Hadoop每次MapReduce操作，启动一个Task便会启动一次JVM，基于进程的操作。而Spark每次MapReduce操作是基于线程的，只在启动Executor时启动一次JVM，内存的Task操作是在线程复用的。
每次启动JVM的时间可能就需要几秒甚至十几秒，那么当Task多了，这个时间Hadoop不知道比Spark慢了多少。

总结：Spark比Mapreduce运行更快，主要得益于其对mapreduce操作的优化以及对JVM使用的优化。



修改spark并行度 

SET spark.sql.shuffle.partitions=



集群：2核8G  * 4 虚拟机 

> hadoop的关键在io 
>
> spark的关键在内存



以12月2日的product_storage为测试，数据量为9300w 数据

数据导入：

​		18min 17s

简单查询：

​		select * from product_storage limit 1000; 

​				当前数据库：0.213s	0.265s   0.073s       				

​				hive：4.711s 	0.489s	0.436s（不走MR）

​				spark: 5.394s	0.834s    0.81s

​		select count(*) from product_storage;

​				当前数据库：8.42s     7.021s	7.089s				

​				hive：	123.67s   124.520s	124.30s

​				spark: 	55.09s	47.254s	47.795s		

计算：

select store_id, date, count(1) from ods_v2.product_storage group by store_id, date ORDER BY store_id,date;

​		当前数据库：35.598s  35.543s  36.742s

​		hive：851.180s   784.20s  792.349s

​		spark: 324.47s    284.69s   274.623s

 

数据倾斜问题：

​	原因：count、distinct、group by、join等操作，都会触发Shuffle动作，一旦触发，所有相同key的值就会拉到一个或几个节点上，就容易发生单点问题。因为数据分布不均匀，导致大量的数据分配到了一个节点。