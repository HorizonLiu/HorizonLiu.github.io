#### RestTemplate利用拦截器配置请求

Spring RestTemplate经常被用作客户端向Restful API发送各种请求，也许你也碰到过这种需求，很多请求都需要用到相似或者相同的Http Header。如果在每次请求之前都把Header填入HttpEntity/RequestEntity，这样的代码会显得十分冗余。

Spring提供了ClientHttpRequestInterceptor接口，可以对请求进行拦截，并在其被发送至服务端之前修改请求或是增强相应的信息。拦截器的实现主要分为如下几个步骤：

- 定义一个类RestTemplateInterceptor，实现ClientHttpRequestInterceptor
- 生成一个特殊的RestTemplate，其拦截器为RestTemplateInterceptor
- 使用RestTemplate调用请求

下面以一个实例来介绍拦截器的使用

##### 定义RestTemplateInterceptor

```java
@Component
public class RestTemplateInterceptor implements ClientHttpRequestInterceptor {
    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {
        // 向头部中加入一些信息
        addRequestHeader(request);
        // restTemplate调用其他请求
        ClientHttpResponse response = execution.execute(request, body);
        // 在返回之前也可以做一些其他的操作，如cookie管理。关于手动管理cookie，后面也会介绍
        return response;
    }

    // 向头部中加入一些信息
    private void addRequestHeader(HttpRequest request) {
        HttpHeaders headers = request.getHeaders();

        // 设置请求类型
        headers.setContentType(MediaType.APPLICATION_JSON_UTF8);
        headers.add("testId","add by interceptor");
    }
}
```

这个类实现了ClientHttpRequestInterceptor的intercept接口，并定义了一个将testId加入header中的方法addRequestHeader中。

##### 生成特殊RestTemplate

```java
@Configuration
public class RestTemplateConfig {
    @Autowired
    private RestTemplateInterceptor restTemplateInterceptor;

    @Bean
    public RestTemplate restTemplate() {
        RestTemplate restTemplate = new RestTemplate();
        // 设置restTemplate的interceptor属性
        restTemplate.setInterceptors(Arrays.asList(restTemplateInterceptor));
        return restTemplate;
    }
}
```

##### 编写controller类调用

```java
@RestController
@RequestMapping(value = "/{version}/interceptor")
public class InterceperTestController {
    @Autowired
    private BasicService basicService;

    /**
     * 被调方
     */
    @PostMapping(value = "/server", consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
    public CommonResponseBody beCalled(@PathVariable("version") String version,
                                       @RequestHeader("testId") String testId) {
        return new CommonResponseBody(0, (Object)testId);
    }

    /**
     * 调用方
     */
    @PostMapping(value = "/client")
    public CommonResponseBody call(@PathVariable("version") String version) {
        return basicService.postProxy(version, "interceptor", "server", null, null, CommonResponseBody.class).getBody();
    }

}
```

在上面，主要包含两个接口，/client接口调用了/server接口。/server接口需要一个名为testId的请求头，但是/client并没有传入任何信息，也可以调用/server成功，这是因为在调用/server之前，拦截器在请求头中加入了testId属性。

##### 测试

![测试](https://ws4.sinaimg.cn/large/006tNbRwgy1fy37keu1j2j30zg0lsmz9.jpg)

这时，如果注释掉`headers.add("testId","add by interceptor");` ， 再次测试将会出现如下错误：



![测试](https://ws2.sinaimg.cn/large/006tNbRwgy1fy37nz4u6tj314i0nydiw.jpg)



##### 辅助类

在上述实现当中，实现了一个辅助类BasicService，用来实现通用的post、get透传。

```java
@Service
public class BasicService {

    @Autowired
    private RestTemplate restTemplate;

    private static final String URI_FORMAT = "http://localhost:18080/%s/%s/%s";

    /**
     * 通用post透传接口
     *
     * @param version      版本号
     * @param cmd          一级命令
     * @param subcmd       二级命令
     * @param headers      请求头
     * @param body         请求体
     * @param responseType 返回类型
     * @param <T>
     * @return
     */
    public <T> ResponseEntity<T> postProxy(String version, String cmd, String subcmd, HttpHeaders headers, Object body, Class<T> responseType) {
        String uri = String.format(URI_FORMAT, version, cmd, subcmd);
        HttpEntity request = new HttpEntity<>(body, headers);
        ResponseEntity<T> responseEntity = restTemplate.postForEntity(uri, request, responseType);
        return responseEntity;
    }

    /**
     * 通用get透传接口
     *
     * @param version 版本号
     * @param cmd 一级命令
     * @param subcmd 二级命令
     * @param headers 请求头
     * @param responseType 返回类型
     * @param <T>
     * @return
     */
    public <T> ResponseEntity<T> getProxy(String version, String cmd, String subcmd, HttpHeaders headers, Class<T> responseType) {
        String uri = String.format(URI_FORMAT, version, cmd, subcmd);
        HttpEntity request = new HttpEntity<>(headers);
        ResponseEntity<T> responseEntity = restTemplate.exchange(uri, HttpMethod.GET, request, responseType);
        return responseEntity;
    }

}
```



##### 参考博客

https://www.jianshu.com/p/deb5e5efb724



**完整代码**：https://github.com/HorizonLiu/http_servlet.git