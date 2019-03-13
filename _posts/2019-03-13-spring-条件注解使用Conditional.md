#### spring常用条件注解介绍

本文主要介绍spring中常用的条件注解`@ConditionalOnBean`, `@ConditionalOnProperty`，个人感觉非常使用，也很容易实现配置化。



#### @ConditionalOnBean

用在类上或者带有@Bean的函数上（可以生成bean，假设生成的bean为example），通常使用语法为`@ConditionalOnBean(xxx.class)` ， 表示若要生成名为example的bean，必须要先有xxx的类，若没有，example无法生成成功。

举个例子：

```java
@Configuration
@ConditionalOnProperty(prefix = "mongo",value = "enable-account", matchIfMissing = false)
//@ConditionalOnProperty(prefix = "mongo", name = "enable-account", havingValue = "true")
public class EnableAccountConfiguration {
}
```



```java
@Configuration
public class MongoConfig {
    @Autowired
    private Config config;

    @Bean(value = "accountMongoTemplate")
    @ConditionalOnBean(EnableAccountConfiguration.class)
    public MongoTemplate accountMongoTemplate() {
        final MongoProperties mongoProperties = config.getAccountMongo();
        return new MongoTemplate(new MongoClient(), mongoProperties.getMongoClientDatabase());
    }
}
```



```java
// 表示这个类能scan到basePackages包中的类，mongoTemplateRef表示accountMongoTemplate和这个包下声明的Repository绑定在一起
// 即若要存入mongodb数据，basePackages下的Repository类将通过accountMongoTemplate进行存储
@EnableMongoRepositories(mongoTemplateRef = "accountMongoTemplate", basePackages = "com.spring.data.instructions.mongo")
@Configuration
@ConditionalOnBean(EnableAccountConfiguration.class)
public class AccountRepositoryConfig {
}
```

上面列举了三个类，`EnableAccountConfiguration`, 使用了@ConditionalOnProperty注解，表示当配置文件中有`enable-count`项才产生这个类，具体使用方法将在下一节专门阐述。

`MongoConfig`中在其函数accountMongoTemplate()中使用了，该函数用来产生一个名为accountMongoTemplate的bean，但是需要建立在bean factory中有EnableAccountConfiguration的前提下。

`AccountRepositoryConfig`也是只有在bean factory中有EnableAccountConfiguration的前提下才会产生。



**注意：ConditionalOnBean 注解条件匹配所检查的bean定义仅限于spring boot 执行过程中application context截至当前所处理的那些bean，因此，强烈建议仅将 ConditionalOnBean 使用于 auto-configuration 类。另外，如果一个候选 bean 需要在另外一个 auto-configuration 完成之后创建(或者不创建)，那么需要确保该 ConditionalOnBean 在那个 auto-configuration 完成之后执行。**



#### @ConditionalOnProperty

用在类上或者带有@Bean的函数上，其作用是当配置中有某个配置项时才生成这个类，一般用在auto-configuration的类上。

```java
@Configuration
// 这里两种使用方法表达的意思一样，都是必须在有enable-account配置时才产生这个bean
// havingValue为true表示name中的值在配置中存在时，生成bean
@ConditionalOnProperty(prefix = "mongo",value = "enable-account", matchIfMissing = false)
//@ConditionalOnProperty(prefix = "mongo", name = "enable-account", havingValue = "true")
public class EnableAccountConfiguration {
}
```

如上面的这个用法，表示当已mongo为前缀的配置项中有enable-account属性时，才产生这个类的bean。

`value`：表示属性名，可以是一个数组；

`matchIfMissing`默认为false，表示没有这个配置时，也能生成上述类；可以将其设置为true，即必须有value配置时，才能生成上述类。