# Spark 持久化

## 概念
cache,persist,checkpoint，以上算子都可以将RDD持久化，持久化的单位是partition。cache和persist都是懒执行的。必须有一个action类算子触发执行。checkpoint算子不仅能将RDD持久化到磁盘，还能切断RDD之间的依赖关系。

## cache
默认将RDD的数据持久化到内存中。cache是懒执行。
注意：chche () = persist()=persist(StorageLevel.Memory_Only)

测试代码：

```
SparkConf conf = new SparkConf();
 conf.setMaster("local").setAppName("CacheTest");
 JavaSparkContext jsc = new JavaSparkContext(conf);
 JavaRDD<String> lines = jsc.textFile("./NASA_access_log_Aug95");

 lines = lines.cache();
 long startTime = System.currentTimeMillis();
 long count = lines.count();
  long endTime = System.currentTimeMillis();
 System.out.println("共"+count+ "条数据，"+"初始化时间+cache时间+计算时间="+ 
          (endTime-startTime));
 
  long countStartTime = System.currentTimeMillis();
 long countrResult = lines.count();
 long countEndTime = System.currentTimeMillis();
 System.out.println("共"+countrResult+ "条数据，"+"计算时间="+ (countEndTime-
           countStartTime));
 
  jsc.stop();
```

## persist：
可以指定持久化的级别。最常用的是MEMORY_ONLY和MEMORY_AND_DISK。”_2”表示有副本数。
持久化级别如下：
<div align=center><img src="https://raw.githubusercontent.com/shuainuo/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/persist.png" width="400"></div>


### cache和persist的注意事项：
1. cache和persist都是懒执行，必须有一个action类算子触发执行。
2. cache和persist算子的返回值可以赋值给一个变量，在其他job中直接使用这个变量就是使用持久化的数据了。持久化的单位是partition。
3. cache和persist算子后不能立即紧跟action算子。
4. cache和persist算子持久化的数据当applilcation执行完成之后会被清除。
错误：rdd.cache().count() 返回的不是持久化的RDD，而是一个数值了。

## checkpoint
checkpoint将RDD持久化到磁盘，还可以切断RDD之间的依赖关系。checkpoint目录数据当application执行完之后不会被清除。

### checkpoint 的执行原理：

当RDD的job执行完毕后，会从finalRDD从后往前回溯。
当回溯到某一个RDD调用了checkpoint方法，会对当前的RDD做一个标记。
Spark框架会自动启动一个新的job，重新计算这个RDD的数据，将数据持久化到HDFS上。
优化：对RDD执行checkpoint之前，最好对这个RDD先执行cache，这样新启动的job只需要将内存中的数据拷贝到HDFS上就可以，省去了重新计算这一步。
使用：

```
SparkConf conf = new SparkConf();
 conf.setMaster("local").setAppName("checkpoint");
 JavaSparkContext sc = new JavaSparkContext(conf);
 sc.setCheckpointDir("./checkpoint");
 JavaRDD<Integer> parallelize = sc.parallelize(Arrays.asList(1,2,3));
 parallelize.checkpoint();
 parallelize.count();
 sc.stop();
```