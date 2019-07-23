## spring boot处理filter抛出的异常

对于controller抛出的异常，可以使用@ControllerAdvice来进行全局处理（具体使用方法参见：

<https://horizonliu.github.io/2019/03/20/%E4%BD%BF%E7%94%A8@ControllerAdvice%E5%92%8C@ExceptionHandler%E5%A4%84%E7%90%86%E5%85%A8%E5%B1%80%E5%BC%82%E5%B8%B8/>）。那对于filter抛出的异常，又该怎么来处理呢？

### 问题来源

项目中使用了spring oauth2框架来做鉴权，我本地使用了Mongo数据库来对token进行存储，并对其中的一些方法进行了override。

```java
// 定义存储token的mongo
public class MongoTokenStore implements TokenStore, ResourceServerTokenServices {
    Override
    public OAuth2Authentication readAuthentication(OAuth2AccessToken token) {
        return readAuthentication(token.getValue());
    }

    @Override
    public OAuth2Authentication readAuthentication(String token) {
        MongoAccessToken accessToken = mongoOperations.findById(token, MongoAccessToken.class);
        if (accessToken == null) {
            return null;
        }
        return accessToken.getAuthentication();
    }

    @Override
    public void storeAccessToken(OAuth2AccessToken token, OAuth2Authentication authentication) {
        mongoOperations.save(new MongoAccessToken(token, authentication));
    }
    
    @Override
    public OAuth2Authentication loadAuthentication(String accessToken) throws AuthenticationException, InvalidTokenException {
        MongoAccessToken oAuth2AccessToken = (MongoAccessToken) readAccessToken(accessToken);
        if (oAuth2AccessToken == null) {
            // 抛出InvalidTokenException异常
            // 该类来自于org.springframework.security.oauth2.common.exceptions包
            throw new InvalidTokenException();
        }
        if (oAuth2AccessToken.isExpired()) {
            // 抛出AccountExpiredException异常
            // 该类来自于org.springframework.security.authentication
            throw new TokenExpireException();
        }
        return oAuth2AccessToken.getAuthentication();
    }
    
    // ...... 其他重写函数
}
```

其中自定义的`loadAuthentication` 函数抛出了oauth框架下的token过期和token无效异常，这两个异常都属于`RunTimeException`而这个函数被oauth框架下的client和provider分别调用。

![image-20190723161815935](http://ww2.sinaimg.cn/large/006tNc79gy1g59ugnwbzaj31bs0ea0wv.jpg)

#### CheckTokenEndpoint中的使用

```java
	@RequestMapping(value = "/oauth/check_token")
	@ResponseBody
	public Map<String, ?> checkToken(@RequestParam("token") String value) {

		OAuth2AccessToken token = resourceServerTokenServices.readAccessToken(value);
        // 已经抛出了token无效和过期两种情况的异常
		if (token == null) {
			throw new InvalidTokenException("Token was not recognised");
		}

		if (token.isExpired()) {
			throw new InvalidTokenException("Token has expired");
		}

		OAuth2Authentication authentication = resourceServerTokenServices.loadAuthentication(token.getValue());

		Map<String, ?> response = accessTokenConverter.convertAccessToken(token, authentication);

		return response;
	}

	// 异常处理
	@ExceptionHandler(InvalidTokenException.class)
	public ResponseEntity<OAuth2Exception> handleException(Exception e) throws Exception {
		logger.info("Handling error: " + e.getClass().getSimpleName() + ", " + e.getMessage());
		// This isn't an oauth resource, so we don't want to send an
		// unauthorized code here. The client has already authenticated
		// successfully with basic auth and should just
		// get back the invalid token error.
		@SuppressWarnings("serial")
		InvalidTokenException e400 = new InvalidTokenException(e.getMessage()) {
			@Override
			public int getHttpErrorCode() {
				return 400;
			}
		};
		return exceptionTranslator.translate(e400);
	}
```

`check_token`接口会抛出`InvalidTokenException` 异常，而该异常通过`handleException` 函数进行处理，给客户端返回400错误码。

#### OAuth2AuthenticationManager中鉴定token

```java
	// 抛出InvalidTokenException异常和loadAuthentication中的异常
	// 一旦抛出就会给客户端返回500的错误，所以这里需要自定义错误码，返回正确的提示，让客户端方便处理
	// 该函数在OAuth2AuthenticationProcessingFilter中调用
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {

		if (authentication == null) {
			throw new InvalidTokenException("Invalid token (token not found)");
		}
		String token = (String) authentication.getPrincipal();
		OAuth2Authentication auth = tokenServices.loadAuthentication(token);
		if (auth == null) {
			throw new InvalidTokenException("Invalid token: " + token);
		}

		Collection<String> resourceIds = auth.getOAuth2Request().getResourceIds();
		if (resourceId != null && resourceIds != null && !resourceIds.isEmpty() && !resourceIds.contains(resourceId)) {
			throw new OAuth2AccessDeniedException("Invalid token does not contain resource id (" + resourceId + ")");
		}

		checkClientDetails(auth);

		if (authentication.getDetails() instanceof OAuth2AuthenticationDetails) {
			OAuth2AuthenticationDetails details = (OAuth2AuthenticationDetails) authentication.getDetails();
			// Guard against a cached copy of the same details
			if (!details.equals(auth.getDetails())) {
				// Preserve the authentication details from the one loaded by token services
				details.setDecodedDetails(auth.getDetails());
			}
		}
		auth.setDetails(authentication.getDetails());
		auth.setAuthenticated(true);
		return auth;

	}
```

#### OAuth2ClientAuthenticationProcessingFilter

```java
@Override
	public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
			throws AuthenticationException, IOException, ServletException {

		OAuth2AccessToken accessToken;
		try {
			accessToken = restTemplate.getAccessToken();
		} catch (OAuth2Exception e) {
			BadCredentialsException bad = new BadCredentialsException("Could not obtain access token", e);
			publish(new OAuth2AuthenticationFailureEvent(bad));
			throw bad;			
		}
		try {
			OAuth2Authentication result = tokenServices.loadAuthentication(accessToken.getValue());
			if (authenticationDetailsSource!=null) {
				request.setAttribute(OAuth2AuthenticationDetails.ACCESS_TOKEN_VALUE, accessToken.getValue());
				request.setAttribute(OAuth2AuthenticationDetails.ACCESS_TOKEN_TYPE, accessToken.getTokenType());
				result.setDetails(authenticationDetailsSource.buildDetails(request));
			}
			publish(new AuthenticationSuccessEvent(result));
			return result;
		}
		catch (InvalidTokenException e) {
			BadCredentialsException bad = new BadCredentialsException("Could not obtain user details from token", e);
			publish(new OAuth2AuthenticationFailureEvent(bad));
			throw bad;			
		}

	}
```

由上面可知，`MongoTokenStore` 中的`loadAuthentication` 抛出的异常除了在`CheckTokenEndpoint` 中无法从新处理外（CheckTokenEndpoint已经定义了处理方法），其他的都可以对Filter的异常进行捕捉，重新定义。下面介绍下如何捕捉Filter中的异常。

### 捕获Filter中的异常

#### 定义`GlobalExceptionHandler`

```java
@Component
@Slf4j
public class GlobalExceptionFilter extends OncePerRequestFilter {

    @Autowired
    protected ObjectMapper objectMapper;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws ServletException, IOException {
        try {
            chain.doFilter(request, response); // 执行各filter
        } catch (Exception ex) {
            log.info("已捕获异常 {}", ex.getClass().getName());

            // 捕获并处理相关异常，若不能处理的，直接抛出
            if (ex instanceof TokenExpireException
                    || ex instanceof AccountExpiredException) {
                response.setContentType("application/json;charset=UTF-8");
                objectMapper.writeValue(response.getWriter(), new BaseResponse(ErrorCode.OAUTH_TOKEN_EXPIRED));
            } else if (ex instanceof InvalidTokenException
                    || ex instanceof org.springframework.security.oauth2.common.exceptions.InvalidTokenException) {
                response.setContentType("application/json;charset=UTF-8");
                objectMapper.writeValue(response.getWriter(), new BaseResponse(ErrorCode.INVALID_OAUTH_TOKEN));
            } else {
                throw ex;
            }
        }
    }
}
```

#### 注册Filter

```java
@Configuration
@Slf4j
public class WebConfig {

    @Bean
    public FilterRegistrationBean filterRegistrationBean() {
        log.info("register global exception filter");
        FilterRegistrationBean bean = new FilterRegistrationBean();
        bean.setFilter(globalExceptionFilter());
        bean.addUrlPatterns("/*"); // 任何接口路径都要执行
        bean.setOrder(Integer.MIN_VALUE); // 优先级最高

        return bean;
    }

    @Bean
    public GlobalExceptionFilter globalExceptionFilter() {
        return new GlobalExceptionFilter();
    }
}
```

#### 自定义Exception类替换oauth2框架异常

```java
public class TokenExpireException extends RuntimeException {
    private static final long serialVersionUID = -8660624962428959868L;
}
```

```java
/**
 * 定义oauth token无效异常 -- 在GlobalExceptionFilter中处理
 * 若使用oauth2自定义InvalidTokenException的异常，将会被oauth鉴权框架捕获，不能自定义返回体
 *
 * @author horizonliu
 * @date 2019/7/23 2:46 PM
 */
public class InvalidTokenException extends RuntimeException {
    private static final long serialVersionUID = -929203033741539948L;
}
```

定义上述两个Exception并替换`MongoTokenStore` 类中`loadAuthentication` 抛出的异常：

```java
@Override
    public OAuth2Authentication loadAuthentication(String accessToken) throws AuthenticationException, InvalidTokenException {
        MongoAccessToken oAuth2AccessToken = (MongoAccessToken) readAccessToken(accessToken);
        if (oAuth2AccessToken == null) {
            throw new InvalidTokenException();
        }
        if (oAuth2AccessToken.isExpired()) {
            throw new TokenExpireException();
        }
        return oAuth2AccessToken.getAuthentication();
    }
```

### 参考链接

<https://juejin.im/entry/5cb720f5e51d456e2446fc95>

