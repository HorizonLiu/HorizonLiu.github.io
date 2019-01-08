SparkSession环境

```java
SparkSession sparkSession = SparkSession
                .builder()
                .appName("test Dataset")
                .master("local")
                .getOrCreate();
```

创建dataset

```java
// 创建dataset时可以通过csv、textFile、parquet、hive表等文件进行读取
Dataset<Row> recordsDSRow = sparkSession
                .read()
                .csv("/Users/horizonliu/Desktop/records.csv");

// 指定dataset的类型
Dataset<Record> recordsDS = sparkSession
                .read()
                .csv("/Users/horizonliu/Desktop/records.csv")
                .as(Encoders.bean(Record.class));
```

 `Dataset` 相当于是一张表，该表含有多个列。如果要从原Dataset中获取指定列作为一张新表，可以使用map函数，如：

```java
// 获取指定列作为一个新的dataset
Dataset<String> valuesDS = recordsDS.map(new MapFunction<Record, String>() {
            @Override
            public String call(Record record) throws Exception {
                return record.getValue();
            }
        }, Encoders.STRING());
```



获取dataset的某一列：

```java
Column key = valuesDS.col("key");
// 而如果还想对着一列进行一些其他的操作，可以参见Column的API文档
// 基本的api包含plus、multiply、divide、mod等函数,使用方法如下
Column keyTmp = valuesDS.col("key").multiply(10);//若key列原来类型为int，调用multiply后，新的列的类型变为double
```



按条件过滤dataset某些项从而生成新的dataset， filter函数

```java
// 过滤出列_c0大于1的数据形成新的dataset
Dataset<Row> tmp = recordsDSRow.filter(recordsDSRow.col("_c0").gt(1));
```



选出dataset的某些列作为新的dataset

```java
		Dataset<Row> tmp = recordsDSRow.filter(recordsDSRow.col("_c0").gt(1))
                .groupBy("_c0")
                .agg(recordsDSRow.col("_c0"))
                .withColumn("value", recordsDSRow.col("_c0").multiply(10));
		// 参数分别为Column和String
        tmp.select(tmp.col("value"), tmp.col("_c0")).show();
        tmp.select("value", "_c0").show();
```



关于聚合操作的一些函数

```java
// 聚合操作api
public Dataset<Row> agg(Column expr,
                        Column... exprs);
// 关于agg的参数Column，可以通过如下函数获取
first("columnName");	-- 返回一个group中该列的第一个元素
last("columnName");

max();
mean();
min();
countDistinct();
sum();
sumDistinct();
variance();
等等，这些函数都需要先group，再在group的基础上求最大\最小\平均等等值。
```

