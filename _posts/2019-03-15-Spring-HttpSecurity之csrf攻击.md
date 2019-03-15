#### Spring Security之csrf攻击

##### 什么是csrf攻击

下面的内容来自维基百科：

**跨站请求伪造**（英语：Cross-site request forgery），也被称为 **one-click attack** 或者 **session riding**，通常缩写为 **CSRF** 或者 **XSRF**， 是一种挟制用户在当前已登录的Web应用程序上执行非本意的操作的攻击方法。[[1\]](https://zh.wikipedia.org/wiki/%E8%B7%A8%E7%AB%99%E8%AF%B7%E6%B1%82%E4%BC%AA%E9%80%A0#cite_note-Ristic-1) 跟[跨网站脚本](https://zh.wikipedia.org/wiki/%E8%B7%A8%E7%B6%B2%E7%AB%99%E6%8C%87%E4%BB%A4%E7%A2%BC)（XSS）相比，**XSS** 利用的是用户对指定网站的信任，CSRF 利用的是网站对用户网页浏览器的信任。



跨站请求攻击，简单地说，是攻击者通过一些技术手段欺骗用户的浏览器去访问一个自己曾经认证过的网站并运行一些操作（如发邮件，发消息，甚至财产操作如转账和购买商品）。由于浏览器曾经认证过，所以被访问的网站会认为是真正的用户操作而去运行。这利用了web中用户身份验证的一个漏洞：**简单的身份验证只能保证请求发自某个用户的浏览器，却不能保证请求本身是用户自愿发出的**。

##### 结合项目

最近项目在做 账号相关的内容， 难免不了的安全问题的考虑。对于用户密码，采用SRP加密的方式传输；对于用户资料权限和第三方授权登录，采用oauth协议。而在用户登录态的维持方面，一般使用session和cookie的方式来保持，而在这中间是有大大的安全风险的。

就目前的项目来说，用户通过登录接口首先登录，若登录成功，服务器会下发userid和usersig以及sessionId，此后，客户端将userid和usersig也作为cookie的形式传给服务器，已验证用户身份，以后的每一次请求的调用都需要先验证这个身份。

即使如此，这里还是有巨大的安全风险：

1. 浏览器一旦获得 userid和usersig，就可以模拟用户再次发起请求，若这个信息泄露给其他人，其他人就能对拥有这个信息的账号进行访问，窃取资料；
2. 这样的用户身份认证真的安全吗？首先，针对同一用户，userid永远不变；其次，usersig的失效时间是多长，目前的设计是30天内登过这个账号，这个usersig就不会过期（可能前期测试，后续会调小）；再者，调用请求时仅仅验证这两个信息就足够了吗，是否超出一定时长需要重新登录（我想的方式是通过session，即维持一个LoginRemain的session，每次调用请求的时候，检查是否过期，若过期，剔除用户下线并要求用户重新登录）。这是目前考虑到的一些安全问题，我相信在做的过程中会发现越来越多的......



不扯远了，来看下再spring的框架中是如何预防csrf攻击的。

在spring的框架之中，针对http接口开发提供了HttpSecurity类，用户可以自定义安全策略来实现不同的配置。那如何实现自定义安全策略呢，只需要编写类集成自`WebSecurityConfigurerAdapter`抽象类，并实现`configure`接口即可，例如：

```java
@Configuration
@Order(114)
public class GatewaySecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // @formater:off
        http.antMatcher("/")
                .csrf().disable()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS).and()
                .authorizeRequests().anyRequest().anonymous();
    }
}
```

HttpSecurity提供了`.crsf()` 接口来预防csrf攻击，该值默认为`true` ， 可通过`diasable` 的方式取消。在我的项目中肯定取消了啊，不然用cookie来维持登录态怎么搞......

##### 参考链接

1. csrf解释：https://zh.wikipedia.org/wiki/%E8%B7%A8%E7%AB%99%E8%AF%B7%E6%B1%82%E4%BC%AA%E9%80%A0
2. Web安全之CSRF攻击的防御措施 (https://www.cnblogs.com/cxying93/p/6035031.html)
3. CSRF 攻击的应对之道:https://www.ibm.com/developerworks/cn/web/1102_niugang_csrf/index.html