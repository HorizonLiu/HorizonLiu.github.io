## spring cache+redis自定义注解实现批量删除缓存

### redis + spring cache 接入缓存

#### 添加依赖

```xml
		<!-- cache -->
		<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
        <!--REDIS-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>
```

#### Redis Cache配置类

```java
import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.*;

import java.time.Duration;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;

/**
 * @author horizonliu
 */
@Configuration
@EnableCaching
public class RedisConfig {

    // 缓存过期时间、缓存名称
    @Value("${news.cache.defaultExpireTime}")
    private int defaultExpireTime;
    @Value("${news.cache.expireTime}")
    private int newsExpireTime;
    @Value("${news.cache.name}")
    private String newsCacheName;

    /**
     * redisTemplate 序列化使用的jdkSerializable, 存储二进制字节码, 所以自定义序列化类
     *
     * @param connectionFactory lettuce连接池
     */
    @Bean(name = "redisTemplate")
    public RedisTemplate<String, Object> redisTemplate(LettuceConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(connectionFactory);

        // 使用Jackson2JsonRedisSerialize 替换默认序列化
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        objectMapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer<Object>(Object.class);
        jackson2JsonRedisSerializer.setObjectMapper(objectMapper);

        RedisSerializer stringRedisSerializer = new StringRedisSerializer();

        // 设置key/value/hash_key/hash_value的序列化规则
        // redis数据使用方使用stringRedisSerializer方式进行序列化，修改需和使用方确认
        redisTemplate.setKeySerializer(stringRedisSerializer);
        redisTemplate.setValueSerializer(jackson2JsonRedisSerializer);
        redisTemplate.setHashKeySerializer(stringRedisSerializer);
        redisTemplate.setHashValueSerializer(stringRedisSerializer);

        redisTemplate.afterPropertiesSet();

        return redisTemplate;
    }

    /**
     * 缓存管理器
     *
     * @param lettuceConnectionFactory
     * @return
     */
    @Bean
    public CacheManager cacheManager(LettuceConnectionFactory lettuceConnectionFactory) {
        RedisCacheConfiguration defaultCacheConfig = RedisCacheConfiguration.defaultCacheConfig();
        // 设置缓存管理器管理的缓存的默认过期时间
        defaultCacheConfig = defaultCacheConfig.entryTtl(Duration.ofSeconds(defaultExpireTime))
                // 设置 key为string序列化
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                // 设置value为json序列化
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()))
                // 不缓存空值
                .disableCachingNullValues();

        Set<String> cacheNames = new HashSet<>();
        cacheNames.add(newsCacheName);

        // 对每个缓存空间应用不同的配置
        Map<String, RedisCacheConfiguration> configMap = new HashMap<>();
        configMap.put(newsCacheName, defaultCacheConfig.entryTtl(Duration.ofSeconds(newsExpireTime)));
        // 设置自定义writer，在存进去时gzip压缩，取出来时解压(缓存数据很大时)
        DefaultRedisCacheWriter cacheWriter = new DefaultRedisCacheWriter(lettuceConnectionFactory);

        RedisCacheManager cacheManager = RedisCacheManager.builder(lettuceConnectionFactory)
                .cacheWriter(cacheWriter)
                .cacheDefaults(defaultCacheConfig)
                .initialCacheNames(cacheNames)
                .withInitialCacheConfigurations(configMap)
                .build();
        return cacheManager;
    }
}
```

### 自定义注解批量删除缓存

#### 定义注解

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * @author horizonliu
 * @date 2020/12/3 8:36 下午
 */
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface NewsCacheRemove {

    /**
     * 缓存名称
     *
     * @return
     */
    String value() default "";

    /**
     * 缓存key
     *
     * @return
     */
    String[] key();
}

```

#### 拦截注解实现业务逻辑

注解支持Spel语法，通过`redis scan` 命令获取与注解`NewsCacheRemove key` 模糊匹配的缓存名，最后使用`redis delete` 命令批量删除缓存。

```java
import com.horizonliu.iov.service.RedisService;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.AfterReturning;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.DefaultParameterNameDiscoverer;
import org.springframework.expression.EvaluationContext;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.expression.spel.support.StandardEvaluationContext;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;
import java.util.HashSet;
import java.util.Set;

/**
 * 移除缓存
 *
 * @author horizonliu
 * @date 2020/12/3 8:39 下午
 */
@Aspect
@Component
@Slf4j
public class NewsCacheRemoveAspect {

    @Autowired
    private RedisService redisService;

    /**
     * SpelExpression解析器
     */
    private SpelExpressionParser spelExpressionParser = new SpelExpressionParser();

    /**
     * 参数名发现器
     */
    private DefaultParameterNameDiscoverer parameterNameDiscoverer = new DefaultParameterNameDiscoverer();

    @Pointcut(value = "(execution(* *.*(..)) && @annotation(com.tencent.iov.dealer.aspect.NewsCacheRemove))")
    public void pointcut() {
    }

    /**
    * 若方法被@NewsCacheRemove注解修饰，在方法执行完毕后，执行如下业务逻辑
    * 1. 解析Spel语法，获取要移除缓存的key列表
    * 2. 通过redis scan命令获取与key模糊匹配的键集合
    * 3. 通过redis delete命令批量删除缓存
    */
    @AfterReturning(value = "pointcut()")
    public void process(JoinPoint joinPoint) {

        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        NewsCacheRemove cacheRemove = method.getAnnotation(NewsCacheRemove.class);
        if (cacheRemove == null) {
            return;
        }
        // 需要移除的正则key
        String[] keys = cacheRemove.key();
        log.info("NewsCacheRemove keys:{}", keys);

        String[] parameterNames = parameterNameDiscoverer.getParameterNames(method);
        if (parameterNames != null && parameterNames.length > 0) {
            // 获取方法参数值
            EvaluationContext context = new StandardEvaluationContext();
            Object[] args = joinPoint.getArgs();
            for (int i = 0; i < args.length; ++i) {
                // 替换spel里的变量值为实际值，比如 #news-> news
                context.setVariable(parameterNames[i], args[i]);
            }
            // 解析出实际的参数信息
            Set<String> spelActualKeys = new HashSet<>();
            for (String key : keys) {
                String actualKey = spelExpressionParser.parseExpression(key).getValue(context).toString();
                // 若实际参数为空，跳过
                if (StringUtils.isBlank(actualKey)) {
                    continue;
                }
                spelActualKeys.add(actualKey);
            }
            // 删除缓存
            Set<String> toDeleteKeys = new HashSet<>();
            for (String key : spelActualKeys) {
                Set<String> regexKey = redisService.scanKeys("*" + key + "*");
                log.info("find pattern:{} keys:{}", spelActualKeys, regexKey);
                toDeleteKeys.addAll(regexKey);
            }
            long count = redisService.deleteKeys(toDeleteKeys);
            log.info("total size:{}, delete success count:{}", toDeleteKeys.size(), count);

        }
    }
}

```

#### 使用注解

在根据品牌/车系 分页查询资讯时，缓存查询结果：缓存键值为`news::[品牌_车系名_页码_页大小]`；

在保存或更新车型资讯内容时，删除与之相关的缓存：缓存键值为`*<品牌名>* ` 和 `*<车系名>*`。

```java
@Component
@CacheConfig(cacheNames = "news")
@Slf4j
public class NewsCacheRepository {

    @Autowired
    private RedisService redisService;

    @Autowired
    private NewsRepository newsRepository;
    
    @Cacheable(key = "#brandName.concat('_').concat(#serialName).concat(#page).concat('_').concat(#pageSize)", sync = true)
    public NewsPageQueryResp findByBrandAndSerialName(String brandName, String serialName, int page, int pageSize) {
        Page<News> news = newsRepository.findByMasterBrandNameRegexAndSerialNameRegex(brandName, serialName, getPageQuery(page, pageSize));
        return getPageRes(page, pageSize, news);
    }

    @Cacheable(key = "'all_'.concat(#page).concat('_').concat(#pageSize)")
    public NewsPageQueryResp findAll(int page, int pageSize) {
        Page<News> news = newsRepository.findAll(getPageQuery(page, pageSize));
        return getPageRes(page, pageSize, news);
    }

    /**
     * 保存/更新 资讯信息：同时删除与该资讯相关品牌/车系相关的缓存
     *
     * @param news 资讯
     */
    @NewsCacheRemove(key = {"#news.masterBrandName", "#news.serialName"})
    public void saveNews(News news) {
        newsRepository.save(news);
    }
}
```

### redis操作辅助类

```java
@Service
@Slf4j
public class RedisService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    /**
     * 设置值，并赋予超时时间
     *
     * @param key      键
     * @param value    值
     * @param expire   过期时长
     * @param timeUnit 过期时长单位
     */
    public void setExpire(String key, Object value, long expire, TimeUnit timeUnit) {
        redisTemplate.opsForValue().set(key, value, expire, timeUnit);
    }

    /**
     * 判断某个键是否存在
     *
     * @param key 键
     * @return true/false
     */
    public boolean isExist(String key) {
        Boolean hasKey = redisTemplate.hasKey(key);
        return hasKey != null && hasKey;
    }

    /**
     * 以count为步长查找符合pattern条件的keys
     *
     * @param pattern 匹配条件
     * @return Set<String>  返回匹配条件的key
     */
    public Set<String> scanKeys(String pattern) {

        log.info("pattern:{}", pattern);
        return redisTemplate.execute(new RedisCallback<Set<String>>() {
            @Override
            public Set<String> doInRedis(@Nonnull RedisConnection connection) throws DataAccessException {
                Set<String> tmpKeys = new HashSet<>();
                ScanOptions options = ScanOptions.scanOptions().match(pattern).count(1000).build();
                // 迭代一直查找，直到找到redis中所有满足条件的key为止(cursor变为0为止)
                Cursor<byte[]> cursor = connection.scan(options);
                while (cursor.hasNext()) {
                    tmpKeys.add(new String(cursor.next()));
                }
                return tmpKeys;
            }
        });
    }

    /**
     * 删除指定键
     *
     * @param key 键
     */
    public void deleteKey(String key) {
        redisTemplate.delete(key);
    }

    /**
     * 删除指定键集
     *
     * @param keys 键集
     */
    public long deleteKeys(Set<String> keys) {
        Long deleteCount = redisTemplate.delete(keys);
        return deleteCount == null ? 0 : deleteCount;
    }

}
```

