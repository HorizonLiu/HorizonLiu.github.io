#### zuul的主要功能

1. 提供网关服务。实际的网络架构中，一个系统往往包含一个对外服务，和多个内部模块，其他系统访问该系统时，通过对外服务进行转发；
2. 保证网关服务安全性。为了保证对外服务的安全性，往往需要做一些校验机制，比如最简单的用户登录状态等等。

对于上述这两个主要功能，下面将介绍如何使用zuul来进行配置。

#### 前提条件

maven依赖

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zuul</artifactId>
        </dependency>
```

对外服务的主类必须使用`EnablZuulProxy`注解打开zuul路由功能

```java
@SpringBootApplication
@EnableDiscoveryClient 
@EnableZuulProxy
public class AccessApplication {
......
}
```

若要使用微服务名配置zuul转发的路由规则，主类还需要使用`@EnableDiscoveryClient` 打开服务发现机制并且转发的整个框架中需要有一个统一的服务注册中心，Consul或者Eureka等。

在我使用时，使用了consul作为服务注册中心，需在依赖中加入：

```xml
<!--服务发现-->
		<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul-discovery</artifactId>
        </dependency>
        <!--配置-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul-config</artifactId>
            <version>1.10.0.TSF-RELEASE</version>
        </dependency>
```

#### Zuul两大功能的使用

##### 配置路由转发

对于转发，zuul提供了两种转发方式：即 `通过微服务名配置和通过url路径进行配置`。

- 使用微服务名配置

```yaml
zuul:
  routes:
    # 推送系统
    iov-notification:
      path: /iov_push/**
      serviceId: iov-notification
      #stripPrefix: true
```

该配置表明，对于路径为/iov_push/开始的url都会被zuul转发到serviceId为iov-notification的机器上。若原url为/iov_push/hello，那么转发后的路径为：http://iov-notification/hello。 如果想原路径为/iov_push/hello，转发后的路径为http://iov-notification/iov_push/hello，即将匹配的url名也加入到转发后的路径中，则只需要加入`stripPrefix: true `配置即可。

- 使用url配置

```yaml
zuul:
  routes:
    api-a-url:
      path: /api-a-url/**
      url: http://ip:port/
```

该配置表明将/api-a-uri/开始的url会被转发发哦 http://ip:port/ 机器的相应路径下。

**转发过程中的敏感信息**

在zuul请求路由时，会过滤掉http请求头信息中的一些敏感信息，防止他们被传递到下游的外部服务器。默认的敏感头新消息通过sensitiveHeaders参数定义，包括Cookie、Set-Cookie、Authorization三个属性。可以通过下面的方式来进行配置

```yaml
# 配置一：所有转发的敏感头都为空，不会过滤任何东西。不推荐
zuul:
  sensitiveHeaders:
# 配置二：对指定转发进行配置，推荐
zuul:
  route:
    sensitiveHeaders:
```

**超时问题**

在zuul转发时，若下游服务没有在zuul的超时时间内返回，则会抛出异常。这个时候，需要对zuul的超时时间进行配置，如下：

```yaml
# zuul采用ribbon做负载均衡，hystrix做熔断
ribbon:
  # 10s
  ReadTimeout: 10000
  SocketTimeOut: 10000
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 10000
```

##### 校验鉴权

针对这个问题，zuul提供了一套过滤器机制，开发者使用zuul来创建各种校验过滤器，然后指定哪些规则的请求需要执行校验逻辑，只有通过校验的才会被路由到具体的微服务接口，不然就返回错误提示。通过这样的改造，各个业务层的微服务就不再需要非业务性质的校验逻辑了，这使得我们的微服务应用可以更专注于业务逻辑的开发。

zuul过滤器的执行流程可以通过下图来展示。

![image-20190113202811479](https://ws3.sinaimg.cn/large/006tNc79gy1fz58bun8wkj30zm0kan1s.jpg)

由图可知，zuul的过滤器主要分为四大类，即：

- pre filter: 这类过滤器在zuul转发之前进行。只有通过pre filter过滤的请求才能被转发到下一级服务器，通常pre filter就可以用来做账号身份验证、鉴权、记录调试信息等操作；
- routing filter：这种过滤器用于构建发送给微服务或url的请求，并使用apache httpclient或netflix ribbon请求下一级服务。这一级的过滤器主要包含三个，即
  - RibbonRoutingFilter：它的执行顺序为10，是route阶段第一个执行的过滤器，该过滤器只对请求上下文中存在的serviceId参数的请求进行处理，即`只对通过serviceId配置路由规则的请求生效`；
  - SimpleHostRoutingFilter：它的执行顺序为100,route阶段第二个执行的过滤器，该过滤器只对请求上下文中存在的routeHost参数的请求进行处理，即`只对通过url配置路由规则的请求生效`；
  - SendForwardFilter：执行顺序为500，用来`处理路由规则中的forward本地跳转配置`。
- post filter：在zuul转发、下一级服务器处理完成返回后执行。这种过滤器可以用来给响应添加响应头、统一修改响应body、收集统计信息和指标等操作；
- error filter：在其他阶段发生错误时执行。



下面来介绍一下如何使用并自定义自己的filter，在我的项目中，使用了pre filter、error filter和post filter，这里简单做一下介绍。

**自定义pre filter**

主要是在pre filter中验证登录态

```java
@Slf4j
@Component
// 继承自ZuulFilter类
public class ZuulPreFilter extends ZuulFilter {
    @Autowired
    private AccountService accountService;

    @Autowired
    private Gson gson;

    // 设置filter类型，包含pre post error route 四种
    @Override
    public String filterType() {
        return "pre";
    }

    // filter执行优先级，越小优先级越高
    @Override
    public int filterOrder() {
        return 0;
    }

    // 该filter是都执行
    @Override
    public boolean shouldFilter() {
        return true;
    }

    // 在该filter中要执行的操作
    @Override
    public Object run() {
        // 获取请求中的基本信息
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        HttpServletResponse accountResponse = ctx.getResponse();
        
        // 获取请求头中的cookie
        Cookie[] cookies = request.getCookies();
        
        // 向请求头中加入 转发到下一级请求中需要的一些基本信息
        ctx.addZuulRequestHeader(Constant.HEADER_UIN, tinyId);
        ctx.addZuulRequestHeader(Constant.HEADER_APPID, "2");

        // 记录下请求内容
        log.info("send {} request to {} from {} ",
                request.getMethod(), request.getRequestURL().toString(), request.getRemoteAddr());

        /**
         * 验证账号登录态：/ticket/verify。
         * 该请求的cookie中不能带IOV_ACCOUNT_SESSIONID，否则会返回一个新的IOV_ACCOUNT_SESSIONID，
         * 而这个新的sessionId不会跟随zuul进行下一次的账号系统请求，将出现session失效
         * 且修改zuul请求中的cookie也是难上难(设计者的安全考虑)
         */
        HttpHeaders requestHeaders = new HttpHeaders();
        Enumeration<String> headerNames = request.getHeaderNames();
        while (headerNames.hasMoreElements()) {
            String key = headerNames.nextElement();
            String value = request.getHeader(key);
            // 跳过cookie设置
            if (StringUtils.equalsIgnoreCase("cookie", key)) {
                continue;
            }
            requestHeaders.add(key, value);
        }

        // 设置cookie
        if (cookies != null) {
            for (Cookie cookie : cookies) {
                if (StringUtils.equalsIgnoreCase(cookie.getName(), "IOV_ACCOUNT_SESSIONID") ||
                        StringUtils.equalsIgnoreCase(cookie.getName(), "IAM_GATEWAY_SESSIONID")) {
                    continue;
                }
                requestHeaders.add(HttpHeaders.COOKIE, cookie.getName() + "=" + cookie.getValue());
            }
        }

        // 验证登录态
        CommonResponseBody responseBody = null;
        try {
            responseBody = accountService.verify("v2", request, requestHeaders, accountResponse);
        } catch (Exception ex) {
            log.info("catch exception : {}", ex.getStackTrace());
            ex.printStackTrace();
        } finally {
            // 若验证失败，zuul拒绝路由该请求，验账号的结果放入返回体中
            if (responseBody == null || responseBody.getCode() != ErrCode.SUCCESS) {
                log.error("login timeout ,please login again \n");
                ctx.setSendZuulResponse(false);
                ctx.setResponseBody(gson.toJson(responseBody, CommonResponseBody.class));
            }

        return null;
    }
}

```

**自定义post filter**

主要是在post filter中将 下一级服务返回体中 的msg根据用户需要设置为中文文案或者英文文案，并加入需要的信息到响应的cookie中。

```java
@Slf4j
@Component
public class ZuulPostFilter extends ZuulFilter {
    @Autowired
    private Gson gson;

    // 业务相关 -- 不必关心
    @Autowired
    private CloudConfig cloudConfig;

    // 业务相关 -- 不必关心
    @Autowired
    private GetLanguageService getLanguageService;

    @Override
    public String filterType() {
        return "post";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        try {
            // 获取当前请求基本信息
            RequestContext ctx = RequestContext.getCurrentContext();
            HttpServletRequest request = ctx.getRequest();
            // 获取响应body，转换成json格式
            InputStream stream = ctx.getResponseDataStream();
            String bodyStr = StreamUtils.copyToString(stream, Charset.forName("UTF-8"));
            log.info("\n\norigin response body is: {}\n\n", bodyStr );
            CommonResponseBody responseBody = gson.fromJson(bodyStr, CommonResponseBody.class);
            // 若responseBody为空，直接返回
            if (responseBody == null) {
                return null;
            }

            // 获取userId
            String userId = null;
            Cookie[] cookies = request.getCookies();
            if (cookies != null) {
                for (Cookie cookie : cookies) {
                    if ("userid".equals(cookie.getName())) {
                        userId = cookie.getValue();
                        break;
                    }
                }
            }

            // 设置返回前端的文案
            if (null != cloudConfig) {
                // 默认返回中文
                if (userId == null || userId.trim().length() <= 0) {
                    log.info("no user has been login , show chinese error msg\n");
                    responseBody.setMsg(cloudConfig.getCHNErrMsg(responseBody.getCode()));
                } else {
                    // 否则根据用户设置的语言属性返回中/英文错误文案
                    CommonResponseBody body = getLanguageService.getLanguage("v2", userId);
                    if (body.getData() != null && StringUtils.equalsIgnoreCase("EN", body.getData().toString())) {
                        responseBody.setMsg(cloudConfig.getENErrorMessage(responseBody.getCode()));
                    } else {
                        responseBody.setMsg(cloudConfig.getCHNErrMsg(responseBody.getCode()));
                    }
                }
            }

            log.info("final response body: {}\n", responseBody);
            
            // 设置返回体body
            String newResponseBodyStr = gson.toJson(responseBody);
            ctx.setResponseBody(newResponseBodyStr);

            // 设置响应的 cookie
            HttpServletResponse response = ctx.getResponse();
            List<Cookie> cookieLst = new ArrayList<>();
            // 可以在这加一下你想加的东西，并加到响应的cookie中
            if (cookieLst != null) {
                for (Cookie cookie : cookieLst) {
                    cookie.setPath("/");
                    response.addCookie(cookie);
                    log.info("post filter after {} : {}", cookie.getName(), cookie.getValue());
                }
            }
        } catch (IOException ex) {
            throw new RuntimeException(ex);
        }

        return null;
    }
}
```



**error filter**

这里做的比较简单，就是记录下异常信息。

```java
@Component
@Slf4j
public class ZuulErrorFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return "error";
    }

    @Override
    public int filterOrder() {
        return 10;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        Throwable throwable = ctx.getThrowable();

        // 记录异常信息
        if (throwable != null) {
            log.error("this is a error filter : {}", throwable.getCause().getMessage());
            ctx.set("error.status_code", HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
            ctx.set("error_exception",throwable.getCause());
        }

        return null;
    }
}
```

#### 参考博客

https://juejin.im/post/5a2f31e56fb9a044fc44b346

http://cloud.spring.io/spring-cloud-netflix/single/spring-cloud-netflix.html#_zuul_timeouts