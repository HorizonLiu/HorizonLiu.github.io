## 使用NamedParameterJdbcTemplate查询数据



### 需求

想使用如下语句查询mysql中的数据

```sql
select * from tableName where column in ("xxx","xxxx")
```

### 解决方式一

通过拼接sql语句的方式将所有要查询的条件进行拼接，然后通过`JdbcTemplate.query()` 的方式进行查询。 

```java
StringBuilder jobTypeInClauseBuilder = new StringBuilder();
for(int i = 0; i < jobTypes.length; i++) {
    Type jobType = jobTypes[i];

    if(i != 0) {
        jobTypeInClauseBuilder.append(',');
    }

    jobTypeInClauseBuilder.append(jobType.convert());
}
```

这种方式显然不优雅，JDBC有一中更优雅的方式，即`NamedParameterJdbcTemplate`.

### 解决方式二

使用`NamedParameterJdbcTemplate`

```java
// 注入
@Autowired
private NamedParameterJdbcTemplate namedParameterJdbcTemplate;

// 查询条件
Set<Integer> ids = new HashSet<>();

MapSqlParameterSource parameters = new MapSqlParameterSource();
parameters.addValue("ids", ids);

// getRowMapper为查询结果到bean的映射
List<Foo> foo = getJdbcTemplate().query("SELECT * FROM foo WHERE a IN (:ids)",
     parameters, getRowMapper());
```

### stackoverflow链接

这里面介绍了几种使用方式，可以参考下。

<https://stackoverflow.com/questions/1327074/how-to-execute-in-sql-queries-with-springs-jdbctemplate-effectivly>