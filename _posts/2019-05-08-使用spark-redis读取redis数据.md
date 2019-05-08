## 使用spark-redis读取redis数据 JAVA

### 参考博客

spark-redis官方文档：<https://github.com/RedisLabs/spark-redis/blob/master/doc/getting-started.md>

优秀博客：<https://www.jianshu.com/p/e91076ccc194>

javaRdd和dataframe的相互转换：<https://blog.csdn.net/snail_gesture/article/details/51298752>

### 依赖

```xml
<dependencies>
    <dependency>
      <groupId>com.redislabs</groupId>
      <artifactId>spark-redis</artifactId>
      <version>2.3.1</version>
    </dependency>
  </dependencies>
```

### 建立连接

```java
public static SparkSession getSparkSessionForHiveAndMongo(String appName,String dbName, String colName) {
        String warehouseLocation = new File("spark-warehouse").getAbsolutePath();

        SparkSession sparkSession = SparkSession
                .builder()
                .appName(appName)
                // redis:host/port/password/db
                .config("spark.redis.host", RedisConfig.HOST)
                .config("spark.redis.port", RedisConfig.PORT)
                .config("spark.redis.auth", RedisConfig.AUTH)
                .config("spark.redis.db", RedisConfig.DATA_BASE)
                .master(SparkConfig.MASTER)
                .getOrCreate();

        return sparkSession;
    }

// 或者先创建SparkContext
public static SparkContext getSparkContext() {
        SparkConf sparkConf = new SparkConf()
                .setMaster(SparkConfig.MASTER)
                .setAppName("read data")
                .set("spark.redis.host", RedisConfig.HOST)
                .set("spark.redis.port", String.valueOf(RedisConfig.PORT))
                .set("spark.redis.auth", RedisConfig.AUTH);
        return new SparkContext(sparkConf);
    }
// 再创建sparksession，sqlContext等
protected SparkSession sparkSession = new SparkSession(getSparkContext());
private transient JavaSparkContext javaSparkContext = new JavaSparkContext(sparkSession.sparkContext());
protected transient SQLContext sqlContext = new SQLContext(javaSparkContext);
```

### 读取数据

#### 读取hashset成dataframe

参考链接：<https://github.com/RedisLabs/spark-redis/blob/master/doc/dataframe.md>

存储和读取的方式见链接，这里就不再介绍了。

需要强调的是，使用这种方式读取redis hashset特别不友好。假设说一个redis hashset中的内容如下：

```shell
hgetall reg_total
1) 20190506
2) 20
3) 20190507
4) 6
5) 20190508
6) 30
```

那么读取出来的格式就为：

| col_1    | col_2 | col_3    | col_4 | col_5    | col_6 |
| -------- | ----- | -------- | ----- | -------- | ----- |
| 20190506 | 20    | 20190507 | 6     | 20190508 | 30    |

而在存储时，也是，若存储一个对象Person

```java
public class Person {
    private String name;
    private int age;
}
```

使用链接中的方法存储后，其结构也是一个hashset，即会存储成：

```shell
1) "name"
2) "horizonliu"
3) "age"
4) "25"
```

所以，对于多个人，若要保证读取出来的dataframe的结构如下表所示：

| name       | age  |
| ---------- | ---- |
| john       | 20   |
| horizonliu | 18   |

那就需要将每个人的信息保存成一个单独的hashset。

而这种方式在大多数的存储场景是不适合的，所以需要使用其他方式。

#### 使用RDD读取redis中的数据

参考链接：<https://github.com/RedisLabs/spark-redis/blob/master/doc/rdd.md>

这种方式支持读取redis支持的各种存储格式，看起来很好用，又很简洁。but I wanna say，对于java实现来说，这简直……还没处找文档，接口的使用方式全靠试...

所以这里记录一下，以防下次又忘了。

先来看看scala版本的：

```scala
import com.redislabs.provider.redis._

// 建立连接
val sc = new SparkContext(new SparkConf()
      .setMaster("local")
      .setAppName("myApp")
      // initial redis host - can be any node in cluster mode
      .set("spark.redis.host", "localhost")
      // initial redis port
      .set("spark.redis.port", "6379")
      // optional redis AUTH password
      .set("spark.redis.auth", "passwd")
  )
// 读取数据
val hashRDD = sc.fromRedisHash("keyPattern*")
val hashRDD = sc.fromRedisHash(Array("foo", "bar"))
```

简单的想让人起飞，但是对于Java版本的，使用就没这么简单了。值得注意的点主要有以下两点：

1. 直接使用JAVA版本的`SparkContext` 无法调用到spark-redis的相关函数。这里不是版本问题，而是因为scala版本的可以直接使用隐式转换功能，什么意思呢，看一下源码就知道了：

1. ```scala
   class RedisContext(@transient val sc: SparkContext) extends Serializable {
       def fromRedisHash[T](keysOrKeyPattern: T,
                          partitionNum: Int = 3)
                         (implicit
                          redisConfig: RedisConfig = RedisConfig.fromSparkConf(sc.getConf),
                          readWriteConfig: ReadWriteConfig = ReadWriteConfig.fromSparkConf(sc.getConf)):
     RDD[(String, String)] = {
       keysOrKeyPattern match {
         case keyPattern: String => fromRedisKeyPattern(keyPattern, partitionNum).getHash()
         case keys: Array[String] => fromRedisKeys(keys, partitionNum).getHash()
         case _ => throw new scala.Exception("KeysOrKeyPattern should be String or Array[String]")
       }
     }
   ```

   即会自动从SparkContext中读取RedisConfig和ReadWriteConfig。而java版本的不会。

2. 从Redis中读取出来的数据是RDD类型的，需要转化成JavaRDD再继续进行操作，否则map操作都不好使。

**参考代码**

```java
// 导入包
import com.redislabs.provider.redis.*;

		// Hashset的key
		String hashMapName = "car_location_info_" + currDate;
        SparkContext sparkContext = sparkSession.sparkContext();
		// 创建RedisContext
        RedisContext redisContext = new RedisContext(sparkContext);
        JavaRDD<CarLocationBean> locationRDD = redisContext
                .fromRedisHash(hashMapName,
                        5,
                        RedisConfig.fromSparkConf(sparkContext.getConf()), ReadWriteConfig.fromSparkConf(sparkContext.getConf()))
			// 读取出来的格式是RDD<Tuple2<String,String>>,
            // Tuple2第一个元素是field，第二个元素是value
            // 需转化成JavaRDD
                .toJavaRDD() 
            // 使用map操作将JavaRDD进行转换操作，只保留value
                .map(ele -> {
                    Gson gson = new Gson();
                    CarLocationBean bean = gson.fromJson(ele._2, CarLocationBean.class);
                    if (bean == null) {
                        bean = new CarLocationBean();
                        bean.setCode("no-location");
                    }
                    return bean;
                });
		// 将javaRDD转换成dataframe
        Dataset<Row> locationDS = sqlContext.createDataFrame(locationRDD, CarLocationBean.class);
        locationDS.show();

        // 合并cityCode
        Dataset<Row> addCityCodeDS = resTempDS
                .join(locationDS, 
                      // 该操作是为了指定内连接使用的列，指定的列在结果dataset中只出现一次
                      JavaConversions.asScalaBuffer(Arrays.asList(FieldName.VIN))) 
                .withColumnRenamed("code", "cityCode");

        addCityCodeDS.show();
```

