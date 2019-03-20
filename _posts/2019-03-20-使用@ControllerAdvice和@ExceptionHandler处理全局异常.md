#### 使用@ControllerAdvice和@ExceptionHandler处理全局异常

在业务开发过程中，往往会在controller层定义很多的异常类型抛出，但如果在每个controller中针对异常进行try-catch将是十分繁琐的，代码不好看，也不容易维护。这里结合spring boot中@ControllerAdvice和@ExceptionHandler注解介绍一种处理全局异常的方法。

其使用方法十分简单。



##### 定义异常类

首先假设我们定义了一个异常类`UserNotExistException` , 用来处理用户不存在的异常，如下：

```java
public class UserNotExistException extends RuntimeException {

	private static final long serialVersionUID = -6112780192479692859L;
	
	private String id;
	
    // 用户id不存在
	public UserNotExistException(String id) {
		super("user not exist");
		this.id = id;
	}

	public String getId() {
		return id;
	}

	public void setId(String id) {
		this.id = id;
	}

}
```



##### 定义全局异常处理类

`ControllerExceptionHandler` 定义如下：

```java
@ControllerAdvice
public class ControllerExceptionHandler {

    // 该函数处理UserNotExistException。并将结果返回为JSON格式，状态码置成500-服务器内部错误
	@ExceptionHandler(UserNotExistException.class)
	@ResponseBody
	@ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
	public Map<String, Object> handleUserNotExistException(UserNotExistException ex) {
		Map<String, Object> result = new HashMap<>();
		result.put("id", ex.getId());
		result.put("message", ex.getMessage());
		return result;
	}
}
```

通过上述方式，在所有controller中抛出`UserNotExistException` 异常都能够在`ControllerExceptionHandler`类中进行处理。



##### 处理使用@Validated校验的异常

加入，我们定义了一个这样的类`User` : 通过`@NotBlank` , `@Past` , `@MyContraint` 注解对参数进行校验。

```java
public class User {
	
	public interface UserSimpleView {};
	public interface UserDetailView extends UserSimpleView {};
	
	private String id;
	
	@MyConstraint(message = "这是一个测试")
	@ApiModelProperty(value = "用户名")
	private String username;
	
	@NotBlank(message = "密码不能为空")
	private String password;
	
	@Past(message = "生日必须是过去的时间")
	private Date birthday;

	@JsonView(UserSimpleView.class)
	public String getUsername() {
		return username;
	}

	public void setUsername(String username) {
		this.username = username;
	}

	@JsonView(UserDetailView.class)
	public String getPassword() {
		return password;
	}

	public void setPassword(String password) {
		this.password = password;
	}

	@JsonView(UserSimpleView.class)
	public String getId() {
		return id;
	}

	public void setId(String id) {
		this.id = id;
	}
	
	@JsonView(UserSimpleView.class)
	public Date getBirthday() {
		return birthday;
	}

	public void setBirthday(Date birthday) {
		this.birthday = birthday;
	}
	
}

```

接下来我们再controller中通过`@Validated` 对参数进行校验：

```java
@PostMapping(value = "/controller_advice", consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
	public String controllerAdvice(@Validated @RequestBody User user) {

		return "success";
	}
```

最后定义全局异常处理类就可以对参数不正确的异常进行返回特定的错误原因到前台。

```java
@ControllerAdvice
public class ControllerExceptionHandler {
	/**
	 * 处理@Validated校验不正确的异常
	 * @param ex
	 * @return
	 */
	@ExceptionHandler(MethodArgumentNotValidException.class)
	@ResponseBody
	@ResponseStatus(HttpStatus.BAD_REQUEST)
	public Map<MethodParameter, Object> handlerInvalidParamsException(MethodArgumentNotValidException ex) {
		Map<MethodParameter, Object> result = new HashMap<>();
		result.put(ex.getParameter(), ex.getMessage());
		result.put(ex.getParameter(), ex.getBindingResult());
		return result;
	}

}

```



##### 延伸

在`ControllerExceptionHandler` 中可以定义多个异常处理函数，分别用来处理不同类型的异常；也可以使用`@ExceptionHandler(Exception.class)` 在一个函数中处理多种异常，使用示例如下：

```java
	@ExceptionHandler(Exception.class)
    @ResponseBody
    public final CommonResponseBody handleAllException(Exception ex) throws Exception {

        int code = ErrCode.EXCEPTION_ERROR;

        if (ex instanceof UndeclaredThrowableException) {
            ex = (Exception) ((UndeclaredThrowableException) ex).getUndeclaredThrowable();
        }

        LOG.error("exception: ", ex);
        // 账号验证失败异常
        if (ex instanceof AccountService.VerifyFailException) {
            code = ErrCode.ACCOUNT_VERIFY_FAILED;
            // session无效异常
        } else if (ex instanceof SessionService.InvalidSessionException) {
            code = ErrCode.INVALID_SESSION_ID;
            // 参数校验错误异常
        } else if (ex instanceof MethodArgumentNotValidException) {
            code = ErrCode.REQUESTBODY_NOT_VALID;
            // 系统错误
        } else if (ex instanceof SessionService.SystemErrorException) {
            code = ErrCode.SYSTEM_ERROR;
            // 服务调用异常
        } else if (ex instanceof BaseService.ServiceCallException){
            code = ((BaseService.ServiceCallException) ex).getErrCode();
        }

        if (code != ErrCode.EXCEPTION_ERROR) {
            // 返回异常对应响应体
            CommonResponseBody response = new CommonResponseBody(code);
            LOG.error("response {} ", response);
            return response;
        }
        // 若上述的异常都不能被处理，再抛出
        throw ex;
    }
```

在上述函数中，判断了抛出的异常属于哪种，并分别进行了处理，最后封装响应包，返回！



##### 参考博客：

https://blog.csdn.net/kinginblue/article/details/70186586