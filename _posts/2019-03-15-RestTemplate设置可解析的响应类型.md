#### RestTemplate 设置可解析的响应类型

今天在做项目的环境迁移工作，恶心的就是环境配置和程序中的可配置参数的更新，这是一件及需要细心与耐心的工作，很无聊。但是，在迁移部署好简单测试时，还是遇到一个未曾出现过的问题，即使用的已经很熟悉的restTemplate。

##### 问题原因

先来看一下问题产生的原因，如下：

```java
org.springframework.web.client.RestClientException: Could not extract response: no suitable HttpMessageConverter found for response type [class com.ctspcl.sms.gateway.manager.sms.dahansantong.base.DaHanSanTongResult] and content type [application/octet-stream]
    at org.springframework.web.client.HttpMessageConverterExtractor.extractData(HttpMessageConverterExtractor.java:110)
    at org.springframework.web.client.RestTemplate.doExecute(RestTemplate.java:655)
    at org.springframework.web.client.RestTemplate.execute(RestTemplate.java:613)
    at org.springframework.web.client.RestTemplate.postForObject(RestTemplate.java:380)

```

#####  问题追踪

上述异常什么意思：即服务器返回的响应是 `application/octet-stream` 格式的，而我的restTemplate没有解析这种响应的能力。一般在后台开发时，一般会以`application/json` 作为响应和请求的格式，而restTemplate若没有指定响应解析类型，则默认也为`application/json` 格式。

已RestTemplate为开始，追踪一下源码：

```java
	// postEntity调用execute函数向服务器发起请求
	@Override
	public <T> ResponseEntity<T> postForEntity(String url, Object request, Class<T> responseType, Object... uriVariables)
			throws RestClientException {

		RequestCallback requestCallback = httpEntityCallback(request, responseType);
		ResponseExtractor<ResponseEntity<T>> responseExtractor = responseEntityExtractor(responseType);
		return execute(url, HttpMethod.POST, requestCallback, responseExtractor, uriVariables);
	}

	// 执行execute
	@Override
	public <T> T execute(String url, HttpMethod method, RequestCallback requestCallback,
			ResponseExtractor<T> responseExtractor, Object... uriVariables) throws RestClientException {

		URI expanded = getUriTemplateHandler().expand(url, uriVariables);
		return doExecute(expanded, method, requestCallback, responseExtractor);
	}

	protected <T> T doExecute(URI url, HttpMethod method, RequestCallback requestCallback,
			ResponseExtractor<T> responseExtractor) throws RestClientException {

		Assert.notNull(url, "'url' must not be null");
		Assert.notNull(method, "'method' must not be null");
		ClientHttpResponse response = null;
		try {
			ClientHttpRequest request = createRequest(url, method);
			if (requestCallback != null) {
				requestCallback.doWithRequest(request);
			}
			response = request.execute();
			handleResponse(url, method, response);
			if (responseExtractor != null) {
                // 对结果进行解析处理
				return responseExtractor.extractData(response);
			}
			else {
				return null;
			}
		}
		catch (IOException ex) {
			String resource = url.toString();
			String query = url.getRawQuery();
			resource = (query != null ? resource.substring(0, resource.indexOf('?')) : resource);
			throw new ResourceAccessException("I/O error on " + method.name() +
					" request for \"" + resource + "\": " + ex.getMessage(), ex);
		}
		finally {
			if (response != null) {
				response.close();
			}
		}
	}
```

接着跟踪到`HttpMessageConverterExtractor` 类：

```java
	public T extractData(ClientHttpResponse response) throws IOException {
		MessageBodyClientHttpResponseWrapper responseWrapper = new MessageBodyClientHttpResponseWrapper(response);
		if (!responseWrapper.hasMessageBody() || responseWrapper.hasEmptyMessageBody()) {
			return null;
		}
    	// 获取response的媒体格式
		MediaType contentType = getContentType(responseWrapper);

		for (HttpMessageConverter<?> messageConverter : this.messageConverters) {
			if (messageConverter instanceof GenericHttpMessageConverter) {
				GenericHttpMessageConverter<?> genericMessageConverter =
						(GenericHttpMessageConverter<?>) messageConverter;
				if (genericMessageConverter.canRead(this.responseType, null, contentType)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Reading [" + this.responseType + "] as \"" +
								contentType + "\" using [" + messageConverter + "]");
					}
					return (T) genericMessageConverter.read(this.responseType, null, responseWrapper);
				}
			}
			if (this.responseClass != null) {
				if (messageConverter.canRead(this.responseClass, contentType)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Reading [" + this.responseClass.getName() + "] as \"" +
								contentType + "\" using [" + messageConverter + "]");
					}
					return (T) messageConverter.read((Class) this.responseClass, responseWrapper);
				}
			}
		}

		throw new RestClientException("Could not extract response: no suitable HttpMessageConverter found " +
				"for response type [" + this.responseType + "] and content type [" + contentType + "]");
	}

	// 获取响应格式，若contentType为空，默认为MediaType.APPLICATION_OCTET_STREAM;
	private MediaType getContentType(ClientHttpResponse response) {
		MediaType contentType = response.getHeaders().getContentType();
		if (contentType == null) {
			if (logger.isTraceEnabled()) {
				logger.trace("No Content-Type header found, defaulting to application/octet-stream");
			}
			contentType = MediaType.APPLICATION_OCTET_STREAM;
		}
		return contentType;
	}
```

从上面可以看出，如果从response中获取的contentType为空，则默认设置为`MediaType.APPLICATION_OCTET_STREAM` 类型。



那我们再来看看，若作为服务端的自己，如果不设置响应类型，返回客户端的将是什么类型呢？

以 `HttpServletResponse` 追踪到 `ResponseIncludeWrapper`， 其`getContentType()` 如下：

```java
	@Override
    public String getContentType() {
        if (contentType == null) {
            String url = request.getRequestURI();
            String mime = context.getMimeType(url);
            if (mime != null) {
                setContentType(mime);
            } else {
                // return a safe value
                setContentType("application/x-octet-stream");
            }
        }
        return contentType;
    }
```



##### 解决问题

既然是不支持的响应类型，这里添加相应的解析器即可，解决方法：

```java
		// 关掉cookie自动管理，设置服务最大连接数 和 每个接口最大连接数
		CloseableHttpClient httpClient
                = HttpClients.custom().disableCookieManagement()
                .setSSLHostnameVerifier(new NoopHostnameVerifier())
                .setMaxConnTotal(8192)
                .setMaxConnPerRoute(256)
                .build();
        HttpComponentsClientHttpRequestFactory requestFactory
                = new HttpComponentsClientHttpRequestFactory();
        requestFactory.setHttpClient(httpClient);
		// 设置请求超时时长
        requestFactory.setConnectionRequestTimeout(30000);
        requestFactory.setConnectTimeout(30000);
        requestFactory.setReadTimeout(30000);
		// 设置restTemplate拦截器
		RestTemplate restTemplate = new RestTemplate(requestFactory);
        List<ClientHttpRequestInterceptor> interceptorList = new ArrayList<>();
        interceptorList.add(new RestTemplateHeaderInterceptor());
        restTemplate.setInterceptors(interceptorList);
		// 设置可解析的响应类型-json和cotet-stream；编码格式utf-8
        MappingJackson2HttpMessageConverter mappingJackson2HttpMessageConverter = new MappingJackson2HttpMessageConverter();   mappingJackson2HttpMessageConverter.setSupportedMediaTypes(Arrays.asList(MediaType.APPLICATION_JSON, MediaType.APPLICATION_OCTET_STREAM));
        restTemplate.getMessageConverters().add(mappingJackson2HttpMessageConverter);
        restTemplate.getMessageConverters().add(new StringHttpMessageConverter(StandardCharsets.UTF_8));
        return restTemplate;
```



##### 参考博客

1. RestTemplate对于响应的处理解析：https://anguslean.github.io/2018/07/06/Java/Framework/spring/RestTemplate%E5%AF%B9%E4%BA%8E%E5%93%8D%E5%BA%94%E7%9A%84%E5%A4%84%E7%90%86%E8%A7%A3%E6%9E%90/
2. Could not extract response: no suitable HttpMessageConverter found for response type：http://www.technicalkeeda.com/spring-tutorials/could-not-extract-response-no-suitable-httpmessageconverter-found-for-response-type