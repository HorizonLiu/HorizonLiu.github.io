本文接收一种使用cas-client + spring boot 接入子系统（假设为运营系统）至单点登录系统的方法。



#### 1. 添加依赖

```xml
<dependency>
    <groupId>net.unicon.cas</groupId>
    <artifactId>cas-client-autoconfig-support</artifactId>
    <version>2.3.0-GA</version>      
</dependency>
<!-- 参考开源项目链接：https://github.com/Unicon/cas-client-autoconfig-support -->
```

#### 2. 在启动类上加上@EnableCasClient

```java
	@SpringBootApplication
    @Controller
    @EnableCasClient // 注解
    public class MyApplication { .. }
```

#### 3. 添加sso相关filter

```java
package com.tencent.cloud.iov.account.security;

import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.jasig.cas.client.authentication.AuthenticationFilter;
import org.jasig.cas.client.util.AssertionThreadLocalFilter;
import org.jasig.cas.client.util.HttpServletRequestWrapperFilter;
import org.jasig.cas.client.validation.Cas20ProxyReceivingTicketValidationFilter;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

import java.util.Arrays;
import java.util.HashMap;
import java.util.Map;

/**
 * @author horizonliu
 * @date 2019/8/19 2:43 PM
 */
@Data
@Slf4j
@Configuration
@RefreshScope
@ConfigurationProperties(prefix = "cas")
public class CASAutoConfig {

    // cas server url前缀
    private String serverUrlPrefix;
    // cas server 登录页url
    private String serverLoginUrl;
    // cas client 地址
    private String clientHostUrl;

    /**
     * 负责跳转登录页进行用户授权，用户授权后会得到一个ticket
     */
    @Bean
    @Primary
    public FilterRegistrationBean casAuthenticationFilter() {
        FilterRegistrationBean registration = new FilterRegistrationBean();
        registration.setFilter(new AuthenticationFilter());
        // 设定匹配的路径
        registration.setUrlPatterns(Arrays.asList("/v2/sso/client/access_subsystem"));
        Map<String,String> initParameters = new HashMap<>();
        // 设置cas server登录地址和cas client地址
        initParameters.put("casServerLoginUrl", serverLoginUrl);
        initParameters.put("serverName", clientHostUrl);
        registration.setInitParameters(initParameters);
        // 设定加载的顺序
        registration.setOrder(2);
        return registration;
    }
    
    /**
     * 负责校验ticket 
     */
    @Bean
    @Primary
    public FilterRegistrationBean casValidationFilter(){
        FilterRegistrationBean registration = new FilterRegistrationBean();
        registration.setFilter(new Cas20ProxyReceivingTicketValidationFilter());
        // 设定匹配的路径，只有下面的路径才会去进行ticket校验
        registration.setUrlPatterns(Arrays.asList("/v2/sso/client/access_subsystem"));
        Map<String, String> initParameters = new HashMap<>();
        initParameters.put("casServerUrlPrefix", serverUrlPrefix);
        initParameters.put("serverName", clientHostUrl);
        registration.setInitParameters(initParameters);
        registration.setOrder(1);
        return registration;
    }

    /**
     * 该过滤器负责实现HttpServletRequest请求的包裹， 
     * 比如允许开发者通过HttpServletRequest的getRemoteUser()方法获得SSO登录用户的登录名，可选配置。
     */
    @Bean
    @Primary
    public FilterRegistrationBean casHttpServletRequestWrapperFilter(){
        FilterRegistrationBean registration = new FilterRegistrationBean();
        registration.setFilter(new HttpServletRequestWrapperFilter());
        // 设定匹配的路径
        registration.setUrlPatterns(Arrays.asList("/v2/sso/client/access_subsystem"));
        registration.setOrder(3);
        return registration;
    }

    /**
     * 该过滤器使得开发者可以通过org.jasig.cas.client.util.AssertionHolder来获取用户的登录名。
     * 比如AssertionHolder.getAssertion().getPrincipal().getName()
     * 或者request.getUserPrincipal().getName()
     */
    @Bean
    @Primary
    public FilterRegistrationBean casAssertionThreadLocalFilter(){
        FilterRegistrationBean registration = new FilterRegistrationBean();
        registration.setFilter(new AssertionThreadLocalFilter());
        // 设定匹配的路径
        registration.setUrlPatterns(Arrays.asList("/v2/sso/client/access_subsystem"));
        registration.setOrder(4);
        return registration;
    }
}
```

#### 4. 添加配置项

```yml
cas:
  serverUrlPrefix: https://sso.xxx.com/   # cas server服务url前缀
  serverLoginUrl: https://sso.xxx.com/   # cas server登录页面
  clientHostUrl: https://xxx.xxx.xxx.com  # 或者为http://ip:port格式
```

#### 5. 编写controller实现/v2/sso/client/access_subsystem

```java
 	@GetMapping("/access_subsystem")
    public ModelAndView accessSubsystem(@RequestParam(name = "sysid", required = false) String sysid,
                                        @RequestParam(name = Const.PARAMS_APPID, defaultValue = "1") String appid,
                                        HttpSession session) {

        Config config = this.config.getConfig(appid);
        // 获取HttpSession中的cas_assertion
        Object object = session.getAttribute(AbstractCasFilter.CONST_CAS_ASSERTION);
        // 若是Assertion类，去cas server处获取用户相关信息
        if (object instanceof Assertion) {
            String uid = null;
            // 获取登录用户的用户名
            Assertion ass = (Assertion) session.getAttribute(AbstractCasFilter.CONST_CAS_ASSERTION);
            log.info("ass:{} ", ass);
            if (ass != null && ass.getPrincipal() != null) {
                uid = ass.getPrincipal().getName();
            }
            if (sysid == null || uid == null) {
                return new ModelAndView(new RedirectView(config.getSsoServerLoginUrl()));
            }
            // 根据uid、appid、sysid从cas server处获取其他信息--这里是获取子系统的tinyId
            String sysUserName = getSysUidFromSSO(sysid, uid, appid);
            long tinyId = sysUserName == null ? 0 : Long.valueOf(sysUserName);
            if (tinyId == 0) {
                log.warn("didn't get tinyId from sso system");
                return new ModelAndView(new RedirectView(config.getSsoServerLoginUrl()));
            }
            // 获取子系统的tinyId后生成该账号在子系统的登录cookie
            postProcessLogin(Long.valueOf(appid), tinyId, AuthenticateFactor.THIRD_OPEN_LOGIN, false);
            // 重定向到运营管理系统
            if ("sysoms".equalsIgnoreCase(sysid)) {
                return new ModelAndView(new RedirectView(config.getOpmSysUrl()));
            }

            // 若是UserSigAuthentication类(自定义的关于本系统登录态的类)
        } else if (object instanceof UserSigAuthentication) {
            // 校验登录态有效性，若有效直接重定向到子系统，若无效跳转到登录页
            UserSigAuthentication authentication = (UserSigAuthentication) object;
            String usersig = (String) authentication.getCredentials();
            AuthTicket authTicket;
            try {
                authTicket = this.config.parseAuthTicket(usersig);
            } catch (Exception e) {
                log.error("decode ticket error", e);
                throw new BadCredentialsException("usersig parse exception");
            }
            if (authTicket == null) {
                log.warn("invalid ticket " + usersig);
                throw new BadCredentialsException("usersig parse error");
            }
            long tinyid = (long) authentication.getPrincipal();
            long appidInTicket = (long) authentication.getDetails();
            ErrorCode result = ticketService.verify(tinyid, appidInTicket, authTicket);
            if (result.getCode() == ErrorCode.SUCC.getCode() || result.getCode() == ErrorCode.NEED_REFRESH.getCode()) {
                // 直接重定向运营管理系统
                return new ModelAndView(new RedirectView(config.getOpmSysUrl()));
            } else {
                return new ModelAndView(new RedirectView(config.getSsoServerLoginUrl()));
            }
        }

        return new ModelAndView(new RedirectView(config.getSsoServerLoginUrl()));
    }

```

至此，就完成了运营系统接入单点登录系统的过程。此时，访问/v2/sso/client/access_subsystem，若未在 https://sso.xxx.com/页面登录过，则会跳转到该页面要求用户登录，登录成功后会得到一个ticket通过casValidationFilter去校验，若校验成功，执行/v2/sso/client/access_subsystem中的操作，一旦操作成功就可以成功跳转到运营系统页面；若已在 https://sso.xxx.com/页面登录过，则浏览器会有ticket，将直接通过casValidationFilter校验该ticket的有效性。

#### 6. 参考博客

 [OAuth2实现单点登录SSO](https://www.cnblogs.com/cjsblog/p/10548022.html)

单点登录原理与简单实现](https://www.cnblogs.com/ywlaker/p/6113927.html)

[cas server + cas client 单点登录 原理介绍](https://www.cnblogs.com/xiatian0721/p/8136305.html)

cas 获取登录的用户名<https://blog.csdn.net/wangyy130/article/details/51922804>