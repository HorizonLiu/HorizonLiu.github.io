### 使用spring jpa Specifications进行多条件查询

JPA2引入了一个标准API，使得我们能以编程的方式构建查询。通过编写条件，可以为域定义查询的`where` 子句。下面来看一下如何使用`Specifications` 来建立多条件查询。

#### 定义表结构

在mysql数据库中创建一张表`banner`， 其中含有的字段及类型如下所示，每条记录的主键是`id`, 自动生成

```java
@Entity
@Table(name = "banner")
@Data
@AllArgsConstructor
@NoArgsConstructor
@ToString
public class Banner {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Integer id;
    private Integer moduleId;
    private Integer bannerPos;
    private String bannerName;
    private DeviceTypes deviceTypes;
    private String imageNormal;
    private String imageHd;
    private String action;
    private Integer priority = -1;
    private Long beginTime;
    private Long endTime;
    private String extra;
    private Integer status = -1;
    private Integer userScope = -1;
    private String whiteList;
    private Long createTime;
    private Long modifyTime;
    private String operator;
}
```

#### 自定义JpaRepository

```java
// 该接口继承自JpaRepository和JpaSpecificationExecutor类
public interface BannerRepository extends JpaRepository<Banner, Integer>, JpaSpecificationExecutor {

    /**
     * 根据banner失效时间和status查询记录，并按banner id 升序排列
     * @param ts
     * @param status
     * @return
     */
    @Query("SELECT b FROM Banner b WHERE b.endTime >= ?1 AND b.status = ?2 ORDER BY b.id ASC")
    List<Banner> getByTimeAndStatus(Long ts, Integer status);

    /**
     * 根据banner id 查询记录
     * @param id
     * @return
     */
    List<Banner> findById(Integer id);

    /**
     * 根据banner id 更新指定记录的status
     * @param id
     * @param status
     * @return
     */
    @Modifying
    @Transactional
    @Query("UPDATE Banner b set b.status = ?2 WHERE b.id = ?1")
    int updateStatus(Integer id, Integer status);

    /**
     * 查找某些特定id的banner记录
     * @param ids
     * @return
     */
    List<Banner> findByIdIn(Collection<Integer> ids);
}
```

#### 自定义关于表banner的条件查询类

```java
// GetBannerInfoListRequest为查询请求，包含可用于查询的字段
public class BannerSpecs {
    public static Specification<Banner> where(GetBannerInfoListRequest getBannerInfoListRequest) {
        return new Specification<Banner>() {
            @Override
            public Predicate toPredicate(Root<Banner> root, CriteriaQuery<?> criteriaQuery, CriteriaBuilder criteriaBuilder) {
                List<Predicate> predicates = new ArrayList<Predicate>();
                // 若查询条件字段为空，直接返回
                if (getBannerInfoListRequest == null) {
                    return null;
                }
                // 根据status进行equal等值查询
                if (getBannerInfoListRequest.getStatus() != null) {
                    predicates.add(criteriaBuilder.equal(root.get("status"), getBannerInfoListRequest.getStatus()));
                }
                // 根据moduleId进行equal等值查询
                if (getBannerInfoListRequest.getModuleId() != null) {
                    predicates.add(criteriaBuilder.equal(root.get("moduleId"), getBannerInfoListRequest.getModuleId()));
                }
                // 根据bannerPos进行equal等值查询
                if (getBannerInfoListRequest.getBannerPos() != null) {
                    predicates.add(criteriaBuilder.equal(root.get("bannerPos"), getBannerInfoListRequest.getBannerPos()));
                }
                // 根据bannerName进行模糊匹配查询
                if (getBannerInfoListRequest.getBannerName() != null) {
                    predicates.add(criteriaBuilder.like(root.get("bannerName"), "%" + getBannerInfoListRequest.getBannerName() + "%"));
                }
                // 查询beginTime在某段时间范围内的记录
                if (getBannerInfoListRequest.getBeginTime() != null) {
                    predicates.add(criteriaBuilder.gt(root.get("beginTime"), getBannerInfoListRequest.getBeginTime()));
                }
                if (getBannerInfoListRequest.getEndTime() != null) {
                    predicates.add(criteriaBuilder.lt(root.get("beginTime"), getBannerInfoListRequest.getEndTime()));
                }
                return criteriaQuery.where(predicates.toArray(new Predicate[0])).getRestriction();
            }
        };
    }
}
```

这样就建立了多条件查询的工具类，那么如何使用它呢？

#### 进行多条件查询

接下来就可以进行多条件查询了，首先我们来看看在自定义`JpaRepository` 时继承的接口类 `JpaSpecificationExecutor`，它当中包含了如下接口：

```java
public interface JpaSpecificationExecutor<T> {
    // 获取满足条件的一个元素
	T findOne(Specification<T> spec);
    // 获取满足条件的所有元素
	List<T> findAll(Specification<T> spec);
    // 通过分页的形式返回满足条件的所有元素
	Page<T> findAll(Specification<T> spec, Pageable pageable);
    // 通过排序的形式返回满足条件的所有元素
	List<T> findAll(Specification<T> spec, Sort sort);
    // 返回满足条件的元素的个数
	long count(Specification<T> spec);
}
```

也就是说，我们可以根据这些接口和自定义的多条件查询类来过滤出我们想要的信息了。

针对上述定义的表结构，我们可以这样使用：

```java
@Autowired
private BannerRepository bannerRepository;

// 查询条件
GetBannerInfoListRequest request = new GetBannerInfoListRequest(1,2,3,"test",1234,5678);
Specification<Banner> bannerFilter = BannerSpecs.where(request); 
// 查询中status=1,moduleId=2,bannerPos=3,bannerName中含有test、起始时间在1234~5678之间 的所有元素
List<Banner> banners = bannerRepository.findAll(bannerFilter);
```



#### 参考博客：

<https://docs.spring.io/spring-data/jpa/docs/2.1.6.RELEASE/reference/html/#jpa.query-methods.at-query>

