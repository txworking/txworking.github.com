---
layout: post
title: "使用Spark对ElasticSearch进行读取"
date: 2015-01-14 17:22:00 +0800
comments: true
categories: [elasticsearch, spark]
---

ElasticSearch for Apache Hadoop是ES提供的工具库，让Hadoop、Pig、Hive等可以比较原生的方式去和ES交互。
目前提供了mapreduce、hive、pig、cascading、spark、storm的集成。

下面以Spark为例，演示如何利用这个工具库去读取ES记录

[官方指南](http://www.elasticsearch.org/guide/en/elasticsearch/hadoop/current/spark.html)

## 安装
我使用的是spark-shell交互式环境进行的测试，所以需要手动下载elasticsearch-hadoop的jar包。
在maven项目中可以通过添加
```xml
<dependency>
    <groupId>org.elasticsearch</groupId>
    <artifactId>elasticsearch-hadoop</artifactId>
    <version>2.1.0.Beta3</version>
</dependency>
```

这是包含了所有支持的jar包，也可以下载单独的spark支持包。

对于Spark，还需要下载Kryo，来替代Spark自带的序列化包。

最后elasticsearch-hadoop只支持Java 1.7以上版本，所以需要看看Java环境是否匹配。

启动spark-shell时，使用以下命令加载特定jar包

```
./bin/spark-shell -jars ./elasticsearch-hadoop-2.1.0.Beta3.jar;./kryo-3.0.0/jar
```

指定序列化工具
```
import com.esotericsoftware.kryo.KryoSerializable
import org.apache.spark.SparkConf

val conf = new SparkConf()
conf.set("spark.serializer", classOf[KryoSerializer].getName)
```

## 读取
```
import org.apache.hadoop.conf.Configuration
import org.elasticsearch.hadoop.mr.EsInputFormat
import org.apache.hadoop.io.Text
import org.apache.hadoop.io.MapWritable

val conf = new Configuration()                          
conf.set("es.resource", "highrisk/blacklist") //指定读取的索引名称           
conf.set("es.nodes", "127.0.0.1")
conf.set("es.query", "?q=me*")  //使用query字符串对结果进行过滤                        
val esRDD = sc.newHadoopRDD(conf, classOf[EsInputFormat[Text, MapWritable]],     
                                  classOf[Text], classOf[MapWritable]))
val docCount = esRDD.count();
```
读取出来的记录为key-value形式，key为Text类型，value为MapWritable类型。
接下来就可以利用Spark对esRDD进行各种map、flatmap、reduceByKey的操作了。

## 写入
参考[用Spark处理数据导入ElasticSearch](http://chenlinux.com/2014/09/04/spark-to-elasticsearch/)
```
import org.apache.spark.SparkConf
import org.elasticsearch.spark._

val conf = new SparkConf()
conf.set("es.index.auto.create", "true")
conf.set("es.nodes", "127.0.0.1")

val numbers = Map("one" -> 1, "two" -> 2, "three" -> 3)
val airports = Map("OTP" -> "Otopeni", "SFO" -> "San Fran")
sc.makeRDD(Seq(numbers, airports)).saveToEs("spark/docs")

```

目前elasticsearch-hadoop对Spark的支持还比较简单，想要对记录进行过滤就只有通过query字符串或者全部读取后在Spark中过滤，对规模比较大的索引或者复杂的过滤查询不友好。

## 配置

主要的配置
* es.resource 
* es.resource.read    默认与es.resource相同
* es.resource.write   默认与es.resource相同
* es.nodes 默认为localhost
* es.port  默认为9200
* es.query 默认为none

完整的请查看[Configuration](http://www.elasticsearch.org/guide/en/elasticsearch/hadoop/current/configuration.html)

