#### 目的

本文介绍一种使用redis作分布式http session管理的方法及基本实现。

#### 依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.session</groupId>
            <artifactId>spring-session</artifactId>
            <version>1.3.3.RELEASE</version>
        </dependency>
```

#### 实现

##### 添加配置session

```java
@EnableRedisHttpSession
@Configuration
public class HttpSessionConfig {
    // 使用默认序列化反序列化工具DefaultCookieSerializer
    @Bean
    public CookieSerializer cookieSerializer() {
        DefaultCookieSerializer serializer = new DefaultCookieSerializer();
        serializer.setCookieName("MY_SESSIONID");
        return serializer;
    }
}
```

一般情况下，在做了上面的配置后，若服务器与客户端间有session需要保存时，就会默认保存到redis中了。具体的保存形式可以在你使用的redis中相应的database中去看。

但是我在使用的时候还是遇到一些问题，记录一下，防止大家也踩相同的坑。

##### 序列化失败问题

项目初期是采用的mongodb做的分布式session管理，也使用的DefaultCookieSerializer序列化与反序列化，存储时没什么问题，因此工程中大部分自定义的session都没有`implements Serializable` ，导致序列化失败。

解决方法：当然就是使自定义session实现`Serializable` 接口。

##### config命令不存在问题

项目使用的redis 机器config命令被禁掉了，而使用redis做session管理时，spring程序在启动时会去调用这个config命令，就失败了。

具体报错为：

```java
Context initialization failed
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'enableRedisKeyspaceNotificationsInitializer' defined in class path resource [org/springframework/session/data/redis/config/annotation/web/http/RedisHttpSessionConfiguration.class]: Invocation of init method failed; nested exception is java.lang.IllegalStateException: Unable to configure Redis to keyspace notifications. See http://docs.spring.io/spring-session/docs/current/reference/html5/#api-redisoperationssessionrepository-sessiondestroyedevent
Caused by: java.lang.IllegalStateException: Unable to configure Redis to keyspace notifications. See http://docs.spring.io/spring-session/docs/current/reference/html5/#api-redisoperationssessionrepository-sessiondestroyedevent
Caused by: org.springframework.dao.InvalidDataAccessApiUsageException: ERR unknown command 'CONFIG'; nested exception is redis.clients.jedis.exceptions.JedisDataException: ERR unknown command 'CONFIG'
```

从报错信息中可以看出，**由于enableRedisKeyspaceNotificationInitializer的报错而无法初始化，导致程序启动不了。而造成enableRedisKeyspaceNotificationInitializer报错的原因是无法使用CONFIG命令。解决方案：1）通过申请设置notify-keyspace-events为Egx；2）在程序启动类中添加如下代码：**

```java
    // 关闭spring-session对config的操作
    @Bean
    public static ConfigureRedisAction configureRedisAction() {
        return ConfigureRedisAction.NO_OP;
    }
```

#### 参考博客

使用redis做session共享：https://www.cnblogs.com/chenpi/p/6347299.html

config命令问题：https://cloud.tencent.com/developer/article/1117029