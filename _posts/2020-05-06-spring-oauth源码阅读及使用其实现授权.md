本文介绍利用spring oauth实现用户授权功能。文章开始介绍用户授权的步骤，然后给出本文的大纲，从授权服务器配置组件开始介绍（源码阅读），到最后将这些组件融合，构成一个完整的授权服务器。在文章末尾，还会给出spring oauth提供的默认接口及调用方法和文档。

### OAuth实现用户授权

oauth实现用户授权通常分为4步：

- 用户同意授权，获取code
- 通过code获取授权access_token
- 刷新access_token(如果需要，则可通过refresh_token获取新的access_token)
- 拉取用户信息或其他授权的接口信息。

除上述接口外，oauth还提供了用于检测当前token是否有效的工具类接口（check_token）

微信开放平台的授权文档：<https://developers.weixin.qq.com/doc/offiaccount/OA_Web_Apps/Wechat_webpage_authorization.html#0>

### 步骤大纲

- 授权服务器组件--TokenStore 
- 授权服务器组件--authorization_code授权模式下code服务类
- 授权服务器组件--tokenService
- 授权服务器组件--client details
- 授权服务器组件--userDetailService
- 授权服务器组件--authenticationManager
- 授权服务器组件--用户授权处理器UserApprovalHandler
- 授权服务器配置—组件融合

### 组件一、TokenStore

负责token的存储。

在spring oauth的实现中，默认支持mysql、redis、内存、jwt等几种方式存储token，其中前三种是需要进行token的持久化存储的，而jwt则不需要，只需要校验按约定的规则生成+校验token即可。jwt这种方式有优点有缺点，优点是不需要存储，缺点是一旦下发，无法及时的失效、延期。

关于持久化存储的access_token实体类，spring oauth采用`DefaultOAuth2AccessToken` 类保存：

```java
public class DefaultOAuth2AccessToken implements Serializable, OAuth2AccessToken {

	private static final long serialVersionUID = 914967629530462926L;
    
    // token值
	private String value;
    // 过期时间
	private Date expiration;
    // token类型，默认为bear
	private String tokenType = BEARER_TYPE.toLowerCase();
    // refresh_token
	private OAuth2RefreshToken refreshToken;
    // 该token支持的scope
	private Set<String> scope;
    // 其他附加信息
	private Map<String, Object> additionalInformation = Collections.emptyMap();
    // ...... other get/set method
}
```

用户可自定义实现自己的token实体类和token store，如使用mongo存储，并在token中存储授权的用户、申请授权的clientId等，一个例子：

#### AccessToken实体类

```java
@Document(collection = "mongoAccessToken")
@Data
public class MongoAccessToken implements OAuth2AccessToken {
    // token值 -- 主键
    @Id
    private String value;
	// 不持久化到数据库中
    @Transient
    private OAuth2Authentication authentication;
    // 用户认证信息，上个字段authentication的二进制形式 -- 节约空间
    @Field("authentication")
    private Binary authenticationBytes;
    @Field("authenticationId")
    private String authenticationId;

    // 附加信息
    @Field("additionalInformation")
    private Map<String, Object> additionalInformation;
    @Field("scope")
    private Set<String> scope;
    @Field("refreshToken")
    private OAuth2RefreshToken refreshToken;
    @Field("tokenType")
    private String tokenType;
    @Field("expiration")
    private Date expiration;
    @Field("expiresIn")
    private int expiresIn;
    // 授权的用户id、发起授权的clientId
    @Field("userId")
    private String userId;
    @Field("clientId")
    private String clientId;
    // ...construct method and implement method from Oauth2AccessToken
}
```

#### TokenStore实体类

```java
@Service
@Slf4j
public class MongoTokenStore implements TokenStore, ResourceServerTokenServices {
    @Autowired
    @Qualifier("accountMongoTemplate")
    MongoTemplate mongoOperations;

    private AuthenticationKeyGenerator authenticationKeyGenerator = new DefaultAuthenticationKeyGenerator();
    
    //...other method implements from interface TokenStore and ResourceServerTokenServices
}
```

### 组件二、code服务类

该类是用于oauth的authorization_code授权模式时，code的生成和使用。spring oauth默认支持mysql和内存方式存储code，原有的code生成服务类图如下：

![image-20200506153010573](/Users/horizonliu/blog-Linguo/HorizonLiu.github.io/img/authorization_code之code生成与存储.png)

`AuthorizationCodeServices` 接口提供创建和消费code两个方法，`RandomValueAuthorizationCodeServices`类用于生成随机code（由0-9，A-Z，a-z）组成，默认长度为6位。

如果想自定义code持久化方法，可模仿`JdbcAuthorizationCodeServices` 继承`RandomValueAuthorizationCodeServices` 类并实现即可，一个采用mongo存储的例子：

#### code实体类

```java
@Data
@Document(collection = "mongoAuthorizationCode")
public class MongoAuthorizationCode {
    @Id
    public String code;
    @Transient
    private OAuth2Authentication authentication;
    // 认证信息
    @Field("authentication")
    private Binary authenticationBytes;

    public MongoAuthorizationCode(String code, OAuth2Authentication authentication) {
        this.code = code;
        this.authentication = authentication;
        if (authentication != null) {
            authenticationBytes = new Binary(SerializationUtils.serialize(authentication));
        }
    }
}
```

#### code服务类

```java
@Service
public class MongoAuthorizationCodeService extends RandomValueAuthorizationCodeServices {
    @Autowired
    @Qualifier("accountMongoTemplate")
    MongoTemplate mongoOperations;
    
    @Override
    protected void store(String code, OAuth2Authentication authentication) {
        mongoOperations.save(new MongoAuthorizationCode(code, authentication));
    }

    @Override
    protected OAuth2Authentication remove(String code) {
        MongoAuthorizationCode authorizationCode = mongoOperations.findAndRemove(Query.query(where("code").is(code)), MongoAuthorizationCode.class);
        if (authorizationCode == null) {
            return null;
        }
        return SerializationUtils.deserialize(authorizationCode.getAuthenticationBytes().getData());
    }
}
```

### 组件三、TokenService

负责token的管理

![image-20200506154857993](/Users/horizonliu/blog-Linguo/HorizonLiu.github.io/img/token_service类图.png)

在spring oauth原有的实现中，提供了`DefaultTokenServices` 和`RemoteTokenServices` 两种实现，后者是用在token服务在远程（非本机，另一台服务器）的情况下。这两个类分别实现了三个接口，下面分别介绍：

- `ResourceServerTokenServices`: 

```java
public interface ResourceServerTokenServices {
    // 读取token的认证信息
	OAuth2Authentication loadAuthentication(String accessToken) throws AuthenticationException, InvalidTokenException;
    // 读取token详细信息
	OAuth2AccessToken readAccessToken(String accessToken);
}
```

- `AuthorizationServerTokenServices`: 

```java
public interface AuthorizationServerTokenServices {
    // 根据已认证的用户信息创建一个accessToken
	OAuth2AccessToken createAccessToken(OAuth2Authentication authentication) throws AuthenticationException;
    // 根据refreshToken和请求，刷新token
	OAuth2AccessToken refreshAccessToken(String refreshToken, TokenRequest tokenRequest)
			throws AuthenticationException;
    // 根据认证信息获取token
	OAuth2AccessToken getAccessToken(OAuth2Authentication authentication);
}
```

- `ConsumerTokenServices`

```java
public interface ConsumerTokenServices {
    // 通用方法，DefaultTokenServices实现了该方法，用于移除access_token和refresh_token
	boolean revokeToken(String tokenValue);
}
```

除了实现上面这三个接口外，`DefaultTokenServices` 还包含一些属性，用来打开或关闭某些功能：

```java
	private int refreshTokenValiditySeconds = 60 * 60 * 24 * 30; // default 30 days.
	private int accessTokenValiditySeconds = 60 * 60 * 12; // default 12 hours.
	// 是否支持refresh_token，默认不支持。
	// 若不支持，在/oauth/token接口的返回中不会有refresh_token字段
	private boolean supportRefreshToken = false;
	// 是否支持重用refresh_token。默认支持
	// 若不支持，在使用refresh_token刷新access_token时，refresh_token也会更新；
	// 若支持，刷新access_token时若refresh_token未过期，refresh_token不会更新
	private boolean reuseRefreshToken = true;
	// token store可自定义
	private TokenStore tokenStore;
	// client details可自定义
	private ClientDetailsService clientDetailsService;
	private TokenEnhancer accessTokenEnhancer;
	// 认证管理器。若打开refresh_token功能，这里会有坑，后面详细介绍
	private AuthenticationManager authenticationManager;
```

介绍了这么多之后，在使用tokenServices时，可以使用默认的`DefaultTokenServices` ，若有自定义功能，只需要在配置时对该service进行自定义配置即可。

### 组件四、ClientDetails

ClientDetailsService用来管理clientId及其对应的权限、token有效期等相关信息。spring oauth实现中，默认支持jdbc和内存方式的clientDetails管理。

![image-20200506164228639](/Users/horizonliu/blog-Linguo/HorizonLiu.github.io/img/client_detail_service.png)

在介绍ClientDetailsService之前，我们先来看看都支持哪些client的配置，其具体的在`ClientDetails`接口中有详细说明。这里我们定义了一个类`MongoClientDetails` 类，该类实现`ClientDetails`接口，将client的相关信息持久化到mongo中。下面根据这个类，来介绍相关配置参数：

```java
@Data
@NoArgsConstructor
@Document("mongo_client_details")
public class MongoClientDetails implements ClientDetails {
    
    private static final long serialVersionUID = -4327246799308139043L;
    
    @Id
    @Field("clientId")
    private String clientId;
    // 应用名称
    @Field("appName")
    private String appName;
    /**
     * 该clientId是否需要clientSecret -- 默认true
     * 如implicit模式不需要clientSecret，则可设为false
     */
    @Field("secretRequired")
    private boolean secretRequired;
    
    @Field("clientSecret")
    private String clientSecret;

    /**
     * 该client可以访问的resources
     */
    @Field("resourceIds")
    private Set<String> resourceIds;

    /**
     * 支持的权限集
     */
    @Field("scope")
    private Set<String> scope;

    /**
     * 该client是否被限制scope
     * 若为false，则该client设置的scope参数将被忽略
     */
    @Field("scoped")
    private boolean scoped;

    /**
     * 支持的授权类型 -- 如authorization_code refresh_token implicit client_credentials
     */
    @Field("authorizedGrantTypes")
    private Set<String> authorizedGrantTypes;

    /**
     * 可支持的回调地址
     */
    @Field("registeredRedirectUri")
    private Set<String> registeredRedirectUri;

    /**
     * 拥有的权限--user/admin等
     */
    @Field("authorities")
    private Collection<GrantedAuthority> authorities;

    /**
     * token有效时长
     */
    @Field("accessTokenValiditySeconds")
    private Integer accessTokenValiditySeconds;

    /**
     * refresh token有效时长
     */
    @Field("refreshTokenValiditySeconds")
    private Integer refreshTokenValiditySeconds;

    /**
     * 对于特定scope，是否需要用户认证
     * 该字段保存不需要用户认证的scope
     */
    @Field("autoApprove")
    private Set<String> autoApprove;

    /**
     * 是否支持所有scope
     */
    @Field("approveAll")
    private boolean approveAll;

    /**
     * 额外信息
     */
    @Field("additionalInformation")
    private Map<String, Object> additionalInformation;

    public MongoClientDetails(String clientId) {
        this.clientId = clientId;
    }
    // 构造函数
    public MongoClientDetails(ClientDetails clientDetails) {
        clientId = clientDetails.getClientId();
        clientSecret = clientDetails.getClientSecret();
        resourceIds = clientDetails.getResourceIds();
        scope = clientDetails.getScope();
        authorizedGrantTypes = clientDetails.getAuthorizedGrantTypes();
        registeredRedirectUri = clientDetails.getRegisteredRedirectUri();
        authorities = clientDetails.getAuthorities();
        accessTokenValiditySeconds = clientDetails.getAccessTokenValiditySeconds();
        refreshTokenValiditySeconds = clientDetails.getRefreshTokenValiditySeconds();
        additionalInformation = clientDetails.getAdditionalInformation();
    }

    @Override
    public boolean isAutoApprove(String scope) {
        return approveAll || autoApprove.contains(scope);
    }
}

```

有了ClientDetails实体类，就可以在该类的基础上构建Service类`MongoClientDetailsService`， 和`JdbcClientDetailsService`一样，该类需要实现`ClientDetailsService`,`ClientRegistrationService` 接口。

- ClientDetailsService -- 获取clientId

```java
public interface ClientDetailsService {
	// 获取clientId详细信息
  ClientDetails loadClientByClientId(String clientId) throws ClientRegistrationException;
}
```

- ClientRegistrationService — clientId的增删改查

```java
public interface ClientRegistrationService {

	void addClientDetails(ClientDetails clientDetails) throws ClientAlreadyExistsException;

	void updateClientDetails(ClientDetails clientDetails) throws NoSuchClientException;

	void updateClientSecret(String clientId, String secret) throws NoSuchClientException;

	void removeClientDetails(String clientId) throws NoSuchClientException;
	
	List<ClientDetails> listClientDetails();

}
```

### 组件五、UserDetailService

要使用refresh_token模式，必须配置UserDetailService。若有自定义用户实体（该用户实体必须继承`UserDetails` 类），则需要重写该类，一个例子：

```java
@Service
public class MongoUserDetailsService implements UserDetailsService {
    @Autowired
    private UserRepository userRepository;

    /**
     * 根据tinyId加载用户信息
     * @param username here is tinyid
     * @return
     * @throws UsernameNotFoundException
     */
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException, NumberFormatException {
        UserDetails user = userRepository.findById(Long.valueOf(username)).orElse(null);
        if (user == null) {
            throw new UsernameNotFoundException("Invalid username or password.");
        }
        return user;
    }
}
```

### 组件六、AuthenticationManager

认证管理器，`AuthenticationManager` 是spring security认证框架中的一个接口，oauth中的password和refresh_token授权模式必须依赖于该类。默认情况下，会使用spring security自动注入的bean，但在refresh_token模式时，需对AuthenticationManager进行额外配置，否则会报错，参见：

**避坑指南**：https://www.cnblogs.com/luas/p/12118694.html

**解决方案**：https://stackoverflow.com/questions/34716636/no-authenticationprovider-found-on-refresh-token-spring-oauth2-java-config

### 组件七、UserApprovalHandler

![image-20200506172315859](/Users/horizonliu/blog-Linguo/HorizonLiu.github.io/img/approval_setting.png)

`UserApprovalHandler`用来检查授权请求是否被当前用户允许过，它包含四个接口：

```java
public interface UserApprovalHandler {
    // 检查该授权请求是否被用户允许过 -- authorizationRequest中的approved字段来标志
	boolean isApproved(AuthorizationRequest authorizationRequest,
			Authentication userAuthentication);
    // 在用户授权前拦截（用户授权页面之前）--可用于跳过用户授权页(静默授权)
	AuthorizationRequest checkForPreApproval(AuthorizationRequest authorizationRequest,
			Authentication userAuthentication);
    // 在设置授权请求(approval params)参数后，检查参数前，再次更新authorizationRequest。在授权请求中包含富文本参数时可使用  -- 没搞明白到底咋用
	AuthorizationRequest updateAfterApproval(AuthorizationRequest authorizationRequest,
			Authentication userAuthentication);
    // 获取授权参数
	Map<String, Object> getUserApprovalRequest(AuthorizationRequest authorizationRequest,
			Authentication userAuthentication);
}
```

而`UserApprovalHandler`的实现包含三个类： `DefaultUserApprovalHandler`、 `TokenStoreUserApprovalHandler`  和 `ApprovalStoreUserApprovalHandler`。其中`DefaultUserApprovalHandler` 的授权处理器没有任何记忆性，而其他两个通过已申请的token或者approval来保存之前的授权请求处理，如果之前申请过token（approval），该token（approval）没过期且授权的scope包含此次请求的scope，那此时则不需要用户再次授权。

`ApprovalStoreUserApprovalHandler` 的approval的读写通过ApprovalStore接口进行，spring boot的源码中使用jdbc、inMemory、tokenStore分别实现了ApprovalStore。

`TokenStoreUserApprovalHandler` 使用token来保持这种记忆性，但需要做一个token<->Approval的转换。

使用方可以根据自己的需要，分别来自定义实现上述接口。

下面来分别看一下这三个类的实现，为了简洁，只看`checkForPreApproval` 函数：

#### DefaultUserApprovalHandler

```java
	public AuthorizationRequest checkForPreApproval(AuthorizationRequest authorizationRequest, Authentication userAuthentication) {
		return authorizationRequest;
	}
```

#### TokenStoreUserApprovalHandler

```java
	public AuthorizationRequest checkForPreApproval(AuthorizationRequest authorizationRequest, Authentication userAuthentication) {

        boolean approved = false;
        // 看申请授权的clientId是都支持autoApprove，若支持，则直接设置不再需要用户授权并直接返回
        String clientId = authorizationRequest.getClientId();
        Set<String> scopes = authorizationRequest.getScope();
        if (clientDetailsService != null) {
            try {

                ClientDetails client = clientDetailsService.loadClientByClientId(clientId);
                approved = true;
                for (String scope : scopes) {
                    if (!client.isAutoApprove(scope)) {
                        approved = false;
                    }
                }
                if (approved) {
                    authorizationRequest.setApproved(true);
                    return authorizationRequest;
                }
            } catch (ClientRegistrationException e) {
                logger.warn("Client registration problem prevent autoapproval check for client=" + clientId);
            }
        }
        
        // 若clientService为空
        OAuth2Request storedOAuth2Request = requestFactory.createOAuth2Request(authorizationRequest);

        OAuth2Authentication authentication = new OAuth2Authentication(storedOAuth2Request, userAuthentication);
        // ...省略部分日志打印代码
        //  根据用户认证信息获取token，若token存在且未过期 -- 设置approved为true
        OAuth2AccessToken accessToken = tokenStore.getAccessToken(authentication);
        if (accessToken != null && !accessToken.isExpired()) {
            if (logger.isDebugEnabled()) {
                logger.debug("User already approved with token=" + accessToken);
            }
            // A token was already granted and is still valid, so this is already approved
            approved = true;
        } else {
            // 否则设置approved为：
            logger.debug("Checking explicit approval");
            approved = userAuthentication.isAuthenticated() && approved;
        }

        authorizationRequest.setApproved(approved);
        return authorizationRequest;
    }
```

#### ApprovalStoreUserApprovalHandler

```java
	public AuthorizationRequest checkForPreApproval(AuthorizationRequest authorizationRequest,
			Authentication userAuthentication) {

		String clientId = authorizationRequest.getClientId();
		Collection<String> requestedScopes = authorizationRequest.getScope();
		Set<String> approvedScopes = new HashSet<String>();
		Set<String> validUserApprovedScopes = new HashSet<String>();

		if (clientDetailsService != null) {
			try {
				ClientDetails client = clientDetailsService.loadClientByClientId(clientId);
                // clientId支持自动授权的scope
				for (String scope : requestedScopes) {
					if (client.isAutoApprove(scope)) {
						approvedScopes.add(scope);
					}
				}
                // 若自动授权的scope完全包含请求的scope
				if (approvedScopes.containsAll(requestedScopes)) {
					// gh-877 - if all scopes are auto approved, approvals still need to be added to the approval store.
					Set<Approval> approvals = new HashSet<Approval>();
					Date expiry = computeExpiry();
					for (String approvedScope : approvedScopes) {
                        // 创建approval并存储
						approvals.add(new Approval(userAuthentication.getName(), authorizationRequest.getClientId(),
								approvedScope, expiry, ApprovalStatus.APPROVED));
					}
					approvalStore.addApprovals(approvals);
                    // 设置请求中的approved为true并返回
					authorizationRequest.setApproved(true);
					return authorizationRequest;
				}
			}
			catch (ClientRegistrationException e) {
				logger.warn("Client registration problem prevent autoapproval check for client=" + clientId);
			}
		}
        
        // 通过认证的用户名和clientId查询approval，若查询得到scope已授权过且包含此次请求中的所有scope，则设置approved为true并返回
		// Find the stored approvals for that user and client
		Collection<Approval> userApprovals = approvalStore.getApprovals(userAuthentication.getName(), clientId);

		// Look at the scopes and see if they have expired
		Date today = new Date();
		for (Approval approval : userApprovals) {
			if (approval.getExpiresAt().after(today)) {
				if (approval.getStatus() == ApprovalStatus.APPROVED) {
					validUserApprovedScopes.add(approval.getScope());
					approvedScopes.add(approval.getScope());
				}
			}
		}
		
		// If the requested scopes have already been acted upon by the user,
		// this request is approved
		if (validUserApprovedScopes.containsAll(requestedScopes)) {
			approvedScopes.retainAll(requestedScopes);
			// Set only the scopes that have been approved by the user
			authorizationRequest.setScope(approvedScopes);
			authorizationRequest.setApproved(true);
		}

		return authorizationRequest;
	}
```

### 授权服务器配置

在介绍完授权服务的各组件后，下面介绍如何将这些组件使用起来组成一个完整的授权服务器。

#### 配置授权服务器

```java
@Configuration
// 使用注解启用授权服务器默认配置
@EnableAuthorizationServer
public class OAuth2AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

    // clientSecret加密--bcrypt
    @Autowired
    private PasswordEncoder passwordEncoder;
    // 自定义mongo存储clientId
    @Autowired
    private MongoClientDetailsService mongoClientDetailsService;
    // 自定义mongo token store
    @Autowired
    MongoTokenStore mongoTokenStore;
    // 自定义authorization_code模式的code服务
    @Autowired
    MongoAuthorizationCodeService mongoAuthorizationCodeService;
    // 自定义用户授权处理类
    @Autowired
    RoleUserApprovalHandler userApprovalHandler;
    // 用户服务--refresh_token模式使用
    @Autowired
    private MongoUserDetailsService mongoUserDetailsService;

    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        security.allowFormAuthenticationForClients()
                .passwordEncoder(passwordEncoder)
            	// check_token所有用户都可访问
                .checkTokenAccess("permitAll()");
//                .checkTokenAccess("isAuthenticated()");
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        // 设置clientDetailService
        clients.withClientDetails(mongoClientDetailsService);
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        // 设置token存储等其他
        endpoints.tokenStore(mongoTokenStore)
                .authorizationCodeServices(mongoAuthorizationCodeService)
                .userApprovalHandler(userApprovalHandler)
                .authenticationManager(createPreAuthProvider())
                .userDetailsService(mongoUserDetailsService);
        endpoints.tokenServices(defaultTokenServices());
        super.configure(endpoints);
    }

    // 定义默认tokenService
    @Bean
    public DefaultTokenServices defaultTokenServices() {
        DefaultTokenServices defaultTokenServices = new DefaultTokenServices();
        defaultTokenServices.setTokenStore(mongoTokenStore);
        defaultTokenServices.setClientDetailsService(mongoClientDetailsService);
        // 避坑指南：https://www.cnblogs.com/luas/p/12118694.html
        // solution：https://stackoverflow.com/questions/34716636/no-authenticationprovider-found-on-refresh-token-spring-oauth2-java-config
        defaultTokenServices.setAuthenticationManager(createPreAuthProvider());
        // 支持refresh_token模式，且refresh_token可复用
        defaultTokenServices.setSupportRefreshToken(true);
        defaultTokenServices.setReuseRefreshToken(true);
        return defaultTokenServices;
    }

    private ProviderManager createPreAuthProvider() {
        PreAuthenticatedAuthenticationProvider provider = new PreAuthenticatedAuthenticationProvider();
        provider.setPreAuthenticatedUserDetailsService(new UserDetailsByNameServiceWrapper<>(mongoUserDetailsService));
        return new ProviderManager(Arrays.asList(provider));
    }
}
```

#### 设置oauth端点访问权限

##### /oauth/token

```java
// 访问/oauth/token需认证
@Configuration
@Order(109)
public class OAuth2TokenWebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // @formatter:off
        http
            .requestMatchers()
                .antMatchers("/oauth/token")
                .and()
//            .authenticationProvider(passwordAuthenticationProvider)
            .authorizeRequests()
                .antMatchers("/oauth/token").authenticated()
                .and()
            .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
            .csrf()
                .disable()
        ;
        // @formatter:on
    }
}
```

##### /oauth/authorize

```java
// 访问/oauth/authorize用户认证通过
@Configuration
@Order(110)
@ConditionalOnBean(EnableAccountConfiguration.class)
public class OAuth2WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    UserSigAuthenticationProvider userSigAuthenticationProvider;
    @Autowired
    UserSigRememberMeService userSigRememberMeService;

    @Autowired
    OAuth2AuthenticationEntryPoint oAuth2AuthenticationEntryPoint;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // @formatter:off
        http
                .antMatcher("/oauth/authorize")
                .csrf().disable()
                .authorizeRequests().antMatchers("/oauth/authorize").hasRole("USER")
                .and()
            	// 自定义认证服务 -- 校验cookie中的用户登录信息
                .authenticationProvider(userSigAuthenticationProvider)
            	// 记住我配置 -- cookie中的用户登录信息
                .rememberMe().rememberMeServices(userSigRememberMeService)
                .and()
               .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
            // 若认证失败，使用oAuth2AuthenticationEntryPoint进行处理，比如跳转到登录页面
            .exceptionHandling().authenticationEntryPoint(oAuth2AuthenticationEntryPoint);
    }
}
```

### 使用授权服务器

经过上面的配置后，就可以使用spring oauth提供的端点接口进行访问，并获取相关的token。

spring oauth提供的接口有：

- /oauth/authorize : authorization_code模式下用于获取用户授权code的接口；
- /oauth/token：获取token
- /oauth/check_token : 校验token
- /oauth/token_key : 同/oauth/check_token
- /oauth/confirm_success : 展示给用户进行授权的页面，可自定义
- /oauth/error ： 授权过程中失败时展示给用户的页面

#### /oauth/authorize调用

方法GET

| 参数          | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| client_id     | 应用的API Key                                                |
| redirect_uri  | 业务回调url                                                  |
| scope         | 请求权限                                                     |
| response_type | 授权模式：code或token（code: authorization_code模式; token: implicit模式） |
| state         | 随机值，用于校验流程                                         |
| nonce         | 随机值，用于防重放                                           |

业务需要登录/授权时重定向到这个地址，该接口会根据配置(autoApprove等)让用户登录/确认/自动授权。

用户登录认证流程完成后，会HTTP 302跳转到上述提供的重定向URL：redirect_uri

- code模式时：redirect_uri?code=123
- token模式时：redirect_uri#access_token=xxx

scope为要申请的权限，由资源服务器提供，并在对应的clientId中进行配置。

#### /oauth/token 调用

- /oauth/token
  - Authorization Code模式下先调用/oauth/authorize获取code，再使用code和client_secret来换取access_token
  - Client Credentials模式下使用client_secret直接换取access_token

方法POST，参数用form表单方式提交

| 参数          | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| grant_type    | authorization_code或client_credentials或refresh_token，为refresh_token时刷新token |
| code          | code模式时需要，业务获得的code                               |
| client_id     | 应用的API Key                                                |
| client_secret | 应用的Secret Key                                             |
| redirect_uri  | code模式时需要，authorize时填写的redirect_uri                |
| refresh_token | grant_type为refresh_token时填写                              |

成功时返回(id_token为可选)

```json
{
    "access_token":"20e25782-1d04-485e-b8ca-d97c7cec90b3",
    "expires_in" : 3600, // access_token有效时间，单位s
    "token_type":"bearer",
    "scope":"profile", // 用户授权确认了的scope
    "refresh_token":"bccc08e2-1b71-4d8a-93eb-a6cc312c8ea8" // 用于刷新token
}
```

失败时返回

```json
{
    "error":"invalid_grant",
    "error_description":"Invalid authorization code: Ucdy64"
}
```

#### /oauth/check_token 调用

业务获取access_token后调用该接口获取access_token内的内容

方法GET

| 参数  | 说明         |
| ----- | ------------ |
| token | access token |

成功时返回

```json
{
    "user_name":"144115205301725057",
    "authorities":["ROLE_USER"],
    "client_id":"a",
  	"exp": 1584780570,
    "scope":["profile"]
}
```

失败时返回

```json
{
    "error":"invalid_token",
    "error_description":"Token was not recognised"
}
```