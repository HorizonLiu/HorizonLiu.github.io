#### Spring boot传参方式

从事javaweb后台接口开发时，有不同的传参方式，针对不同的传参方式，后台必须要对应不同的接收方法，否则会出现415、406等各种各样的问题。这篇文章将针对spring boot的不同传参方式进行总结。



##### 传参方式

这里从web开发常用调试工具postman出发，看一下都有哪些传参方式。由下图可知：对于post 方法，可以通过：

- params
- authorization
- headers
- body
- pre-request script
-  test

几种方式来传参。前四种是经常用到的，后面两种没用过，等以后用过了再做分享吧。



![postman参数方式截图](https://ws1.sinaimg.cn/large/006tNbRwgy1fy3zo1s69ej31lu0eaq4o.jpg)

##### 使用params传参

![企业微信截图_2d0ddb14-4112-4c18-a4d1-a4eb28a61388](https://ws1.sinaimg.cn/large/006tNbRwgy1fy41zzt3n1j316c0pkdj8.jpg)



使用params参数传递参数时，参数会拼接到url中。在后台，需要使用@RequestParam注解来接收参数。

```java
	// 可以通过String的方式接收
	@PostMapping(value = "/params/key")
    public CommonResponseBody uploadParams(@PathVariable("version") String version,
                                           @RequestParam("testId") String testId,
                                           @RequestParam("date") String date) {

        Params data = new Params(version, testId, date);
        return new CommonResponseBody(0, data);
    }

	// 也可以通过map的方式接收
    @PostMapping(value = "/params/map")
    public CommonResponseBody uploadParams(@PathVariable("version") String version,
                                           @RequestParam Map<String,String> paramMap ) {

        Params data = new Params();
        data.setVersion(version);
        if (paramMap.containsKey("testId")) {
            data.setTestId(paramMap.get("testId"));
        }
        if (paramMap.containsKey("date")) {
            data.setDate(paramMap.get("date"));
        }
        return new CommonResponseBody(0, data);
    }
```

##### 使用Authorization传参

这一项主要是用来鉴权使用的，主要支持oauth、basic auth等鉴权方式。

在这里生成token后，该token会被带在url中去请求。

![image-20181213104727336](https://ws4.sinaimg.cn/large/006tNbRwgy1fy4xc2wr33j31l60k6aff.jpg)

关于oauth鉴权相关的知识可以参考阮一峰的博客：`http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html`



##### headers传参

headers是调用一个连接时的请求头部，包含http请求中的一些常用参数，用户也可以自定义参数放入头部。若服务端需要在请求头中加入某个字段时，可以使用@RequestHeader注解来添。

```java
	// 使用String方式接收
	@PostMapping(value = "/headers/key", consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
    public CommonResponseBody uploadHeader(@PathVariable("version") String version,
                                           @RequestHeader("testId") String testId,
                                           @RequestHeader("date") String date) {
        Params data = new Params(version, testId, date);
        return new CommonResponseBody(0, data);
    }

	// 使用map方式接收。需要注意的是，在这种情况下，由于没有给map中的每个字段指定名字，若字段名中有大写字母，会统一转换为小写。如这里，若我传入的是testId字段，服务器接收到的是testid字段。
    @PostMapping(value = "/headers/map", consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
    public CommonResponseBody uploadsHeader(@PathVariable("version") String version,
                                            @RequestHeader Map<String,String> headerMap) {


        Params data = new Params();
        data.setVersion(version);
        if (headerMap.containsKey("testid")) {
            data.setTestId(headerMap.get("testid"));
        }
        if (headerMap.containsKey("date")) {
            data.setDate(headerMap.get("date"));
        }
        return new CommonResponseBody(0, data);
    }
```



##### 通过请求体body传参

![image-20181213174558891](https://ws4.sinaimg.cn/large/006tNbRwgy1fy59fidzj7j30za05qmxw.jpg)

通过请求体传参，可接收的参数主要包括如下编码方式：form-data表单、x-www-form-urlencode、raw和binary几种。其中form-data和x-www-form-urlencode可以通过@RequestParam注解、MultiValueMap来接收；raw中支持的比较多，如下：

![image-20181213174916712](https://ws2.sinaimg.cn/large/006tNbRwgy1fy59ix1upej30oe0fcgmm.jpg)

最常用的就是JSON格式，通常用@RequestBody注解接收；而binary可以用来上传文件。

在这里就不再详细用代码说明了。



##### 其他

有的时候，我们需要在一个函数里面调用另一个接口，这个时候就需要手动封装body、header了。那么对于不同的被调用方来说，其消费的body类型也不一样，究竟用什么来装body呢，下面以x-www-form-urlencoded编码方式的来说明。

```java
	@Resource
    private RestTemplate commonRestTemplate;

	// 调用方
	@PostMapping(value = "/test/send", consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
    public String testSend() {
        MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
        params.add("hello", "xxxx");
        params.add("xiha", "test");

        // 请求头
        HttpHeaders headers = new HttpHeaders();
        //  请勿轻易改变此提交方式，大部分的情况下，提交方式都是表单提交
        MediaType type = MediaType.parseMediaType("application/x-www-form-urlencoded; charset=UTF-8");
        headers.setContentType(type);
        headers.add("Accept", MediaType.APPLICATION_JSON.toString());
        //我们发起 HTTP 请求还是最好加上"Connection","close" ，有利于程序的健壮性
        headers.set("Connection", "close");

        HttpEntity<MultiValueMap<String, String>> requestEntity = new HttpEntity<>(params, headers);
        //  执行HTTP请求
        ResponseEntity<String> response = commonRestTemplate.postForEntity("http://localhost:8080/test/recv", requestEntity, String.class);
        return response.getBody();
    }

	// 被调用方
    @PostMapping(value = "/test/recv", consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE)
    public String testRecv(@RequestBody MultiValueMap params) {

        return params.toString();
    }
```



##### 辅助类

```java
    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    class Params {
        private String version;
        private String testId;
        private String date;
    }
```

