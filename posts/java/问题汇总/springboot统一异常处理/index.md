# Springboot统一异常处理


> 参考：  
> https://www.jianshu.com/p/3f3d9e8d1efa  
> https://www.jianshu.com/p/179daa24ef52  
> https://www.cnblogs.com/wang-yaz/p/13225830.html  
> https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-error-handling


本文使用的 `spring boot` 版本为 `2.4.2`

## 使用 ExceptionHandler 注解处理 Controller 异常

Spring 3.0 版本新增了 `@ExceptionHandler` 注解，它的作用是：若在某个 Controller 类中定义一个异常处理方法，并在该异常处理方法上添加 `@ExceptionHandler` 注解（注解中指定需要处理的异常），那么当该 Controller 类中出现指定的异常时，会执行该异常处理方法。该异常处理方法可以使用 spring mvc 提供的数据绑定，比如注入 HttpServletRequest 等，还可以接受一个当前抛出的 Throwable 对象。

示例

```java
package org.msdemt.simple.exception_handler.controller;

import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@Slf4j
@RequestMapping("test")
@RestController
public class UserController{

    @RequestMapping("run")
    public String RuntimeExceptionTest(){
        throw new RuntimeException("runtime exception");
    }

    @RequestMapping("null")
    public String NullPointerExceptionTest(){
        throw new NullPointerException("null pointer exception");
    }

    @ExceptionHandler({NullPointerException.class, RuntimeException.class})
    public String handleException(Exception e) {
        return e.getMessage();
    }
}
```

但是，这样一来，就必须在每个 Controller 类都定义一套这样的异常处理方法，就会造成冗余代码，而且若需要新增一种异常的处理逻辑，就必须修改所有的 Controller 类，很不优雅。

为了解决冗余代码，可以将所有的 Controller 继承一个 BaseController 基类，在 BaseController 统一处理异常。但是这样的代码由侵入性和耦合性，而且由于 Java 是单继承的，如此 Controller 就无法继承其他类了。

示例：

```java
package org.msdemt.simple.exception_handler.controller;

import org.springframework.web.bind.annotation.ExceptionHandler;

public class BaseController {
    @ExceptionHandler({NullPointerException.class, RuntimeException.class})
    public String handleException(Exception e) {
        return e.getMessage();
    }
}
```

```java
package org.msdemt.simple.exception_handler.controller;

import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@Slf4j
@RequestMapping("test")
@RestController
public class UserController extends BaseController{

    @RequestMapping("run")
    public String RuntimeExceptionTest(){
        throw new RuntimeException("runtime exception");
    }

    @RequestMapping("null")
    public String NullPointerExceptionTest(){
        throw new NullPointerException("null pointer exception");
    }
}
```

java 8 版本开始支持接口中定义静态方法或默认方法，将 BaseController 定义为接口，这样 Controller 就可以继承其他类了。


```java
package org.msdemt.simple.exception_handler.controller;

import org.springframework.web.bind.annotation.ExceptionHandler;

public interface BaseController {
    @ExceptionHandler({NullPointerException.class, RuntimeException.class})
    static String handleException(Exception e) {
        return e.getMessage();
    }
}
```

或

```java
package org.msdemt.simple.exception_handler.controller;

import org.springframework.web.bind.annotation.ExceptionHandler;

public interface BaseController {
    @ExceptionHandler({NullPointerException.class, RuntimeException.class})
    default String handleException(Exception e) {
        return e.getMessage();
    }
}
```

```java
package org.msdemt.simple.exception_handler.controller;

import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@Slf4j
@RequestMapping("test")
@RestController
public class UserController implements BaseController{

    @RequestMapping("run")
    public String RuntimeExceptionTest(){
        throw new RuntimeException("runtime exception");
    }

    @RequestMapping("null")
    public String NullPointerExceptionTest(){
        throw new NullPointerException("null pointer exception");
    }
}
```

上述的各种方式，Controller 和 异常处理器（`@ExceptionHandler` 注解的方法）耦合在一起的。

Spring 3.2 版本新增了 `@ControllerAdvice` 注解，可以与 `@ExceptionHandler` 注解配套使用，实现 Controller 和 异常处理器 的解耦：在单独的一个类中，使用 `@ControllerAdvice` 注解该类，并且在该类中添加使用 `@ExceptionHandler` 注解的各种异常的处理方法，实现统一处理所有 Controller 中出现的异常。

Spring 4.3 版本增加了一个注解 `@RestControllerAdvice`，该注解结合了 `@ControllerAdvice` 和 `@ResponseBody` 。


## ControllerAdvice 和 ExceptionHandler 注解组合处理异常

### 异常的优雅判断

一般，我们使用 `if {...}` 等判断语句对异常进行判断，如果需要判断的地方很多，则 if 判断语句特别多，很不优雅。Spring 提供的 `org.springframework.util.Assert` ，能够让我们优雅地判断并抛出异常。示例如下：

```java
@Test
public void test1() {
    Integer i = null;
    if (i == null) {
        throw new IllegalArgumentException("参数 i 不能为 null");
    }
}

@Test
public void test2() {
    Integer i = null;
    Assert.notNull(i, "参数 i 不能为 null");
}
```

`Assert.notNull()` 是怎么实现的呢？如下是 `Assert.notNull()` 的源码

```java
/**
 * Assert that an object is not null.
 * Assert.notNull(clazz, "The class must not be null");
 * Params:
 *     object – the object to check
 *     message – the exception message to use if the assertion fails
 * Throws:
 *     IllegalArgumentException – if the object is null
 */
public static void notNull(@Nullable Object object, String message) {
	if (object == null) {
		throw new IllegalArgumentException(message);
	}
}
```

可以看到， Assert 其实就是帮助我们把 `if {...}` 封装了下，从而使代码更加优雅。

那么我们能不能模仿 `org.springframework.util.Assert` ，也写一个断言类，不过断言失败后抛出的异常不是 `IllegalArgumentException` 这些内置异常，而是我们自定义的异常？

### 自定义 Assert

```java
public interface Assert {
    /**
     * 新建异常
     * @param args
     * @return
     */
    BaseException newException(Object... args);

    /**
     * 新建异常
     * @param t
     * @param args
     * @return
     */
    BaseException newException(Throwable t, Object... args);

    /**
     * 若判断 obj 对象为空，则新建自定义异常并抛出
     *
     * @param obj 待判断对象
     */
    default void assertNotNull(Object obj) {
        if (obj == null) {
            throw newException();
        }
    }
```

其中，`BaseException` 是所有自定义异常的基类。

上面的 `Assert` 断言方法是使用接口的默认方法（jdk8新特性），当断言失败后，抛出的异常不是具体的某个异常，而是由 `newException` 接口方法提供。因为业务逻辑中出现的异常基本都是对应特定的场景，比如根据用户 id 获取用户信息，查询结果为 null ，此时抛出的异常可能为 `UserNotFoundException` ，并且有特定的异常代码（比如 7001）和异常描述（比如“用户不存在”），所以，具体抛出什么异常，由 `Assert` 的实现类决定。

按照上面的说法，岂不是有多少异常，就得定义多少的断言类和异常类？

### 善解人意的 Enum

自定义异常 `BaseException` 有 2 个属性，即 code 、 mess ，可以和枚举 enum 结合

异常枚举接口

```java
/**
 * 返回枚举接口
 */
public interface IResponseEnum {
    /**
     * 获取返回代码
     * @return 返回代码
     */
    String getCode();

    /**
     * 获取返回描述
     * @return 返回描述
     */
    String getMessage();
}
```

基础异常类（包含异常枚举接口）

```java
@Getter
public class BaseException extends RuntimeException {

    private static final long serialVersionUID = 1L;

    /**
     * 异常信息枚举
     */
    protected IResponseEnum responseEnum;
    /**
     * 异常消息参数
     */
    protected Object[] args;

    public BaseException(IResponseEnum responseEnum) {
        super(responseEnum.getMessage());
        this.responseEnum = responseEnum;
    }

    public BaseException(String code, String message) {
        super(message);
        this.responseEnum = new IResponseEnum() {
            @Override
            public String getCode() {
                return code;
            }

            @Override
            public String getMessage() {
                return message;
            }
        };
    }

    public BaseException(IResponseEnum responseEnum, Object[] args, String message) {
        super(message);
        this.responseEnum = responseEnum;
        this.args = args;
    }

    public BaseException(IResponseEnum responseEnum, Object[] args, String message, Throwable cause) {
        super(message, cause);
        this.responseEnum = responseEnum;
        this.args = args;
    }
}
```

业务异常类（继承基础异常类，也包含了异常枚举接口）

```java
public class BusinessException extends BaseException {

    private static final long serialVersionUID = 1L;

    public BusinessException(IResponseEnum responseEnum, Object[] args, String message) {
        super(responseEnum, args, message);
    }

    public BusinessException(IResponseEnum responseEnum, Object[] args, String message, Throwable cause) {
        super(responseEnum, args, message, cause);
    }
}
```

业务异常断言接口（接口可以多继承，该接口继承了异常枚举接口和自定义断言接口，使用默认方法覆盖了自定义断言接口的新建异常方法）

```java
public interface BusinessExceptionAssert extends IResponseEnum, Assert {

    @Override
    default BaseException newException(Object... args) {
        String msg = this.getMessage();
        if (args != null && args.length > 0) {
            msg = MessageFormat.format(this.getMessage(), args);
        }

        return new BusinessException(this, args, msg);
    }

    @Override
    default BaseException newException(Throwable t, Object... args) {
        String msg = this.getMessage();
        if (args != null && args.length > 0) {
            msg = MessageFormat.format(this.getMessage(), args);
        }

        return new BusinessException(this, args, msg, t);
    }

}
```

业务异常枚举类，该类实现了业务异常断言接口

```java
@Getter
@AllArgsConstructor
public enum ResponseEnum implements BusinessExceptionAssert {

    /**
     * 请求实体为空
     */
    REQUEST_IS_NULL("1001", "Request is null."),
    /**
     * 用户不存在
     */
    USER_NOT_EXIST("1002", "User not exist.")

    ;

    /**
     * 返回码
     */
    private String code;
    /**
     * 返回消息
     */
    private String message;
}
```

使用示例

```java
@RestController
@RequestMapping("/user")
public class UserController {

    private IUserService userService;

    @Autowired
    public UserController(IUserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public R<User> getUserById(@PathVariable("id") Integer id){
        User user = userService.findUserById(id);
        ResponseEnum.USER_NOT_EXIST.assertNotNull(user);
        return R.ok(user);
    }
}
```

上文使用 `assertNotNull()` 判断 `user` 是否为空，

若不使用断言，代码可能如下

```java
if(user == null){
    throw new BusinessException("1002", "User not exist.");
}
```

断言 `assertNotNull()` 方法实现如下

```java
/**
 * 若判断 obj 对象为空，则新建自定义异常并抛出
 *
 * @param obj 待判断对象
 */
default void assertNotNull(Object obj) {
    if (obj == null) {
        throw newException();
    }
}
```

下面来分析下代码 `ResponseEnum.USER_NOT_EXIST.assertNotNull(user);` ，来看是如何实现的

因为枚举类 `ResponseEnum` 实现了接口 `BusinessExceptionAssert`，且接口 `BusinessExceptionAssert` 实现了自定义的 `Assert` 接口，所以枚举类 `ResponseEnum` 中的枚举实例可以调用接口 `Assert` 中的方法（此处为 `assertNotNull()`），方法内判断对象 user 为 null ，调用 `newException()` 方法新建异常，并将异常抛出。

此时 `newException()` 方法由接口 `BusinessExceptionAssert` 的默认方法实现，方法内容如下

```java
@Override
default BaseException newException(Object... args) {
    String msg = this.getMessage();
    if (args != null && args.length > 0) {
        msg = MessageFormat.format(this.getMessage(), args);
    }

    return new BusinessException(this, args, msg);
}
```

此时， this 是 `ResponseEnum.USER_NOT_EXIST` ，获取枚举实例 `ResponseEnum.USER_NOT_EXIST` 中的 message ，并将参数配置到 message 的占位符中（如果有参数的话），然后新建一个业务异常。

```java
public class BusinessException extends BaseException {

    private static final long serialVersionUID = 1L;

    public BusinessException(IResponseEnum responseEnum, Object[] args, String message) {
        super(responseEnum, args, message);
    }

    public BusinessException(IResponseEnum responseEnum, Object[] args, String message, Throwable cause) {
        super(responseEnum, args, message, cause);
    }
}
```

可以看到，业务类异常（BusinessException）是通过构造方法新建时调用父类（`BaseException`）的构造方法创建的

```java
@Getter
public class BaseException extends RuntimeException {

    private static final long serialVersionUID = 1L;

    /**
     * 异常信息枚举
     */
    protected IResponseEnum responseEnum;
    /**
     * 异常消息参数
     */
    protected Object[] args;

    public BaseException(IResponseEnum responseEnum, Object[] args, String message) {
        super(message);
        this.responseEnum = responseEnum;
        this.args = args;
    }
}
```

业务类异常（BusinessException）创建成功后，通过 `throw` 抛出该异常。

这样，就实现了使用枚举类汇总异常实例，然后通过自定义的断言类判断并抛出该异常。

以后每增加一种异常，只需增加一个枚举实例即可。

使用枚举类结合（继承）`Assert`，只需根据特定的异常情况定义不同的枚举实例，就能够针对不同情况抛出特定的异常（这里指特定异常代码和异常消息的异常），这样既不用定义大量的异常类，同时也具备了断言的良好可读性。

### 统一异常处理类

上文使用枚举类结合（继承）`Assert`实现了判断并抛出异常，下面来介绍异常的统一处理。

使用 `@ControllerAdvice` 结合 `@ExceptionHandler` 实现异常的统一处理。

统一异常处理器代码如下

```java
package org.msdemt.simple.unified_exception_handler_demo.exception.handler;

import lombok.extern.slf4j.Slf4j;
import org.msdemt.simple.unified_exception_handler_demo.constant.enums.ArgumentResponseEnum;
import org.msdemt.simple.unified_exception_handler_demo.constant.enums.CommonResponseEnum;
import org.msdemt.simple.unified_exception_handler_demo.constant.enums.ServletResponseEnum;
import org.msdemt.simple.unified_exception_handler_demo.exception.BaseException;
import org.msdemt.simple.unified_exception_handler_demo.exception.BusinessException;
import org.msdemt.simple.unified_exception_handler_demo.i18n.UnifiedMessageSource;
import org.msdemt.simple.unified_exception_handler_demo.pojo.response.ErrorResponse;
import org.springframework.beans.ConversionNotSupportedException;
import org.springframework.beans.TypeMismatchException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.converter.HttpMessageNotReadableException;
import org.springframework.http.converter.HttpMessageNotWritableException;
import org.springframework.validation.BindException;
import org.springframework.validation.BindingResult;
import org.springframework.validation.FieldError;
import org.springframework.validation.ObjectError;
import org.springframework.web.HttpMediaTypeNotAcceptableException;
import org.springframework.web.HttpMediaTypeNotSupportedException;
import org.springframework.web.HttpRequestMethodNotSupportedException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.MissingPathVariableException;
import org.springframework.web.bind.MissingServletRequestParameterException;
import org.springframework.web.bind.ServletRequestBindingException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.context.request.async.AsyncRequestTimeoutException;
import org.springframework.web.multipart.support.MissingServletRequestPartException;

/**
 * 全局异常处理器
 */
@Slf4j
@RestControllerAdvice
public class UnifiedExceptionHandler {
    /**
     * 生产环境
     */
    private final static String ENV_PROD = "prod";

    private UnifiedMessageSource unifiedMessageSource;

    @Autowired
    public UnifiedExceptionHandler(UnifiedMessageSource unifiedMessageSource) {
        this.unifiedMessageSource = unifiedMessageSource;
    }

    /**
     * 当前环境，默认dev环境
     */
    @Value("${spring.profiles.active:dev}")
    private String profile;

    /**
     * 获取国际化消息
     *
     * @param e 异常
     * @return
     */
    public String getMessage(BaseException e) {
        String code = "response." + e.getResponseEnum().toString();
        String message = unifiedMessageSource.getMessage(code, e.getArgs());

        if (message == null || message.isEmpty()) {
            return e.getMessage();
        }

        return message;
    }

    /**
     * 业务异常处理器
     *
     * @param e 异常
     * @return 异常结果
     */
    @ExceptionHandler(value = BusinessException.class)
    public ErrorResponse handleBusinessException(BaseException e) {
        log.error(e.getMessage(), e);

        return new ErrorResponse(e.getResponseEnum().getCode(), getMessage(e));
    }

    /**
     * 自定义异常处理器
     *
     * @param e 异常
     * @return 异常结果
     */
    @ExceptionHandler(value = BaseException.class)
    public ErrorResponse handleBaseException(BaseException e) {
        log.error(e.getMessage(), e);

        return new ErrorResponse(e.getResponseEnum().getCode(), getMessage(e));
    }

    /**
     * servlet容器异常处理
     *
     * @param e 异常
     * @return 异常结果
     */
    @ExceptionHandler({
            //NoHandlerFoundException.class, //该异常由自定义控制器捕获处理
            HttpRequestMethodNotSupportedException.class,
            HttpMediaTypeNotSupportedException.class,
            HttpMediaTypeNotAcceptableException.class,
            MissingPathVariableException.class,
            MissingServletRequestParameterException.class,
            TypeMismatchException.class,
            HttpMessageNotReadableException.class,
            HttpMessageNotWritableException.class,
            HttpMediaTypeNotAcceptableException.class,
            // BindException.class,
            // MethodArgumentNotValidException.class
            ServletRequestBindingException.class,
            ConversionNotSupportedException.class,
            MissingServletRequestPartException.class,
            AsyncRequestTimeoutException.class
    })
    public ErrorResponse handleServletException(Exception e) {
        log.error(e.getMessage(), e);
        String code = CommonResponseEnum.SERVER_ERROR.getCode();
        try {
            ServletResponseEnum servletExceptionEnum = ServletResponseEnum.valueOf(e.getClass().getSimpleName());
            code = servletExceptionEnum.getCode();
        } catch (IllegalArgumentException e1) {
            log.error("class [{}] not defined in enum {}", e.getClass().getName(), ServletResponseEnum.class.getName());
        }

        if (ENV_PROD.equals(profile)) {
            // 当为生产环境, 不适合把具体的异常信息展示给用户, 比如404.
            code = CommonResponseEnum.SERVER_ERROR.getCode();
            BaseException baseException = new BaseException(CommonResponseEnum.SERVER_ERROR);
            String message = getMessage(baseException);
            return new ErrorResponse(code, message);
        }

        return new ErrorResponse(code, e.getMessage());
    }


    /**
     * 参数绑定异常处理器
     *
     * @param e 异常
     * @return 异常结果
     */
    @ExceptionHandler(value = BindException.class)
    public ErrorResponse handleBindException(BindException e) {
        log.error("参数绑定校验异常", e);

        return wrapperBindingResult(e.getBindingResult());
    }

    /**
     * 参数校验(Valid)异常处理器
     * 将校验失败的所有异常组合成一条错误信息
     *
     * @param e 异常
     * @return 异常结果
     */
    @ExceptionHandler(value = MethodArgumentNotValidException.class)
    public ErrorResponse handleValidException(MethodArgumentNotValidException e) {
        log.error("参数绑定校验异常", e);

        return wrapperBindingResult(e.getBindingResult());
    }

    /**
     * 包装绑定异常结果
     *
     * @param bindingResult 绑定结果
     * @return 异常结果
     */
    private ErrorResponse wrapperBindingResult(BindingResult bindingResult) {
        StringBuilder msg = new StringBuilder();

        for (ObjectError error : bindingResult.getAllErrors()) {
            msg.append(", ");
            if (error instanceof FieldError) {
                msg.append(((FieldError) error).getField()).append(": ");
            }
            msg.append(error.getDefaultMessage() == null ? "" : error.getDefaultMessage());

        }

        return new ErrorResponse(ArgumentResponseEnum.PARAMETER_VERIFICATION_FAILED.getCode(), msg.substring(2));
    }

    /**
     * 空指针异常单独处理
     * 若在Exception中处理，则返回的 e.getMessage() 为 null，不太友好
     * @param nullPointerException 空指针异常
     * @return
     */
    @ExceptionHandler(value = NullPointerException.class)
    public ErrorResponse handleNullPointerException(NullPointerException nullPointerException) {
        log.error(nullPointerException.getMessage(), nullPointerException);

        ErrorResponse errorResponse =  new ErrorResponse(CommonResponseEnum.Null_POINTER_EXCEPTION.getCode(),
                CommonResponseEnum.Null_POINTER_EXCEPTION.getMessage());
        return errorResponse;
    }

    /**
     * 未定义异常处理器
     *
     * @param e 异常
     * @return 异常结果
     */
    @ExceptionHandler(value = Exception.class)
    public ErrorResponse handleException(Exception e) {
        log.error(e.getMessage(), e);

        if (ENV_PROD.equals(profile)) {
            // 当为生产环境, 不适合把具体的异常信息展示给用户, 比如数据库异常信息.
            String code = CommonResponseEnum.SERVER_ERROR.getCode();
            BaseException baseException = new BaseException(CommonResponseEnum.SERVER_ERROR);
            String message = getMessage(baseException);
            return new ErrorResponse(code, message);
        }

        ErrorResponse errorResponse =  new ErrorResponse(CommonResponseEnum.SERVER_ERROR.getCode(), e.getMessage());
        return errorResponse;
    }

}
```

信息国际化处理

```java
package org.msdemt.simple.unified_exception_handler_demo.i18n;


import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.MessageSource;
import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.stereotype.Component;

import java.util.Locale;

@Component
public class UnifiedMessageSource {

    private MessageSource messageSource;

    @Autowired
    public UnifiedMessageSource(MessageSource messageSource) {
        this.messageSource = messageSource;
    }

    /**
     * 获取国际化消息
     *
     * @param code 消息code
     * @return
     */
    public String getMessage(String code) {

        return getMessage(code, null);
    }

    /**
     * 获取国际化消息
     *
     * @param code 消息code
     * @param args 参数
     * @return
     */
    public String getMessage(String code, Object[] args) {

        return getMessage(code, args, "");
    }

    /**
     * 获取国际化消息
     *
     * @param code           消息code
     * @param args           参数
     * @param defaultMessage 默认消息
     * @return
     */
    public String getMessage(String code, Object[] args, String defaultMessage) {
        Locale locale = LocaleContextHolder.getLocale();

        return messageSource.getMessage(code, args, defaultMessage, locale);
    }
}
```

查看异常处理器代码可以看到，代码中将异常分成了几类，实际上只有两大类，一类是 ServletException （在servlet容器中发生的异常），一类是 ServiceException （在业务代码中发生的异常）。

ServletException
- 由 handleServletException 方法处理
  - HttpRequestMethodNotSupportedException
  - HttpMediaTypeNotSupportedException
  - HttpMediaTypeNotAcceptableException
  - MissingPathVariableException
  - MissingServletRequestParameterException
  - TypeMismatchException
  - HttpMessageNotReadableException
  - HttpMessageNotWritableException
  - ServletRequestBindingException
  - ConversionNotSupportedException
  - MissingServletRequestPartException
  - AsyncRequestTimeoutException
- 由 handleBindException 方法处理
  - handleBindException
- 由 handleValidException 方法处理
  - MethodArgumentNotValidException
- 由自定义控制器捕获处理
  - NoHandlerFoundException

ServiceException
- 由 handleBusinessException 方法处理
  - BusinessException
- 由 handleBaseException 方法处理
  - BaseException
- 由 handleNullPointerException 方法处理
  - NullPointerException
- 由 handleException 方法处理
  - Exception

#### 异常处理器说明

##### servlet 容器产生的异常

一个 `http` 请求，在到达 `Controller` `前，Servlet` 容器会对该请求的请求信息和目标控制器信息做一系列校验。

- `NoHandlerFoundException`: 首先根据请求 `url` 查找有没有对应的控制器，若没有则会抛出该异常，即 404 异常；在 springboot 中，该异常不会直接抛出，而是由 `org.springframework.boot.autoconfigure.web.servlet.error.BasicErrorController` 进行处理。
- `HttpRequestMethodNotSupportedException`: 若请求 `url` 匹配到了，获得了对应的匹配结果（匹配结果是一个列表，该 `url` 支持的不同 http 方法，如：Get、Post等），则尝试将请求的 `http` 方法与列表的控制器做匹配，若没有对应 `http` 方法的控制器，则抛出该异常。
- `HttpMediaTypeNotSupportedException`： 然后再对请求头与控制器支持的请求头做比较，比如 `content-type` 请求头，若控制器的参数签名包含注解 `@RequestBody`，但 http 请求的请求头中 `content-type` 的值不包含 `application/json`，那么会抛出该异常。
- `MissingPathVariableException`: 未检测到路径参数，比如控制器的 `url` 为 `/user/{id}`，参数签名包含 `@PathVariable(id)` ，当请求的 `url` 为 `/user` ，没有控制器的 `url` 为 `/user` 的情况下，该请求会被判定为缺少路径参数，从而会抛出 `MissingPathVariableException`
- `MissingServletRequestParameterException`: 缺少请求参数，比如控制器定义了参数 `@RequestParam("id) String id` ，但 http 请求中，未携带该 `id` 参数，则会抛出该异常。
- `TypeMismatchException`: 参数类型匹配失败，比如控制器的接收参数为 `Long` 类型，但 http 请求中传入的值的类型是字符串，那么将会出现类型转换失败的错误，从而抛出该异常。
- `HttpMessageNotReadableException`: http 请求数据反序列化异常，比如控制器接收到 http 请求，请求值是 json 格式，当 spring boot 使用 `jackson` 对该 json 进行反序列化为控制器定义的对象参数时，若反序列化失败，则会抛出该异常。
- `HttpMessageNotWritableException`: spring boot 使用 `jackson` 对控制器返回的数据进行序列化为 `json` 时，发生序列化失败时抛出该异常。比如，控制器对应的方法的返回值是 `User` ，且该控制器含有前面 `@ResponseBody`，则控制器方法返回 `return user;`时，spring boot 会使用 `jackson` 将 `user` 对象序列化为 `json` ，返回给调用方，当对象序列化为 `json` 失败时，抛出该异常。
- `HttpMediaTypeNotAcceptableException`: 控制器返回的报头（`Response Headers`）中的 `content-type` 与 http 请求报头（`Request Headers`）不匹配，会抛出该异常。
- `BindException`: 参数绑定异常，`org.springframework.validation.BindException`，控制器对接受参数根据注解进行规则校验时，若校验失败，则抛出异常。
- `MethodArgumentNotValidException`: 参数校验异常，控制器对接受参数进行校验，校验失败抛出该异常。
- `ServletRequestBindingException`: 参数绑定异常
- `ConversionNotSupportedException`: 类型转换异常
- `MissingServletRequestPartException`: 缺少请求参数异常，如当某个参数配置为 `@RequestParam(required=true)` 时，请求中没有该参数时会抛出异常，注意，当请求参数位于 `controller` 的 `url` 路径中时，默认 `required = true`
- `AsyncRequestTimeoutException`: 异步请求超时异常

##### 业务异常

- `BaseException`: 自定义异常基类
- `BusinessException`: 业务异常
- `NullPointerException`: 业务中产生的空指针异常（若不配置该异常捕获，则由 `Exception` 异常处理器捕获，此时 `e.getMessage()` 为 null，前端处理时可能出现 null 空指针，未避免这种情况，对 `NullPointerException` 单独进行处理）
- `Exception`: 其他异常，若数据库连接失败等异常

#### 异于常人的404

上文提到，当请求没有匹配到控制器时，会抛出 `NoHandlerFoundException` 异常，但实际上默认情况不是这样，默认情况下

如果请求类型为 `text/html`，则 spring boot 匹配不到对应的请求路径，会返回类似如下页面

![图 1](/images/20210202-whitelabel%20error%20page.png)  

如果是其他请求类型（如`application/json`），则返回json数据，类似如下

```json
{
  "timestamp": "2021-02-02T03:09:54.098+00:00",
  "status": 404,
  "error": "Not Found",
  "message": "",
  "path": "/user"
}
```

为什么会这样呢？

因为 `spring boot` 启动后，会自动加载 `/META-INF/spring.factories` 中配置的类，其中就有 `org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration` 和 `org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration`

`WebMvcAutoConfiguration` 部分代码如下

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
@AutoConfigureAfter({ DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class,
		ValidationAutoConfiguration.class })
public class WebMvcAutoConfiguration {
}
```
`@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })` 表示当类路径下存在 `Servlet.class` `DispatcherServlet.class` `WebMvcConfigurer.class` 时，自动加载 `WebMvcAutoConfiguration` 类

`@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)` 表示当类路径下不存在 `WebMvcConfigurationSupport.class` 时，自动加载 `WebMvcAutoConfiguration` 类

加载 `WebMvcConfigurationSupport.class` 类时，该类中使用 `@Bean` 定义的方法都会生成对应的对象并存放在 `spring` 容器中。

重点看下 `EnableWebMvcConfiguration` 类

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(WebProperties.class)
public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration implements ResourceLoaderAware {
	@Bean
	@Override
	public RequestMappingHandlerAdapter requestMappingHandlerAdapter(
			@Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
			@Qualifier("mvcConversionService") FormattingConversionService conversionService,
			@Qualifier("mvcValidator") Validator validator) {
		RequestMappingHandlerAdapter adapter = super.requestMappingHandlerAdapter(contentNegotiationManager,
				conversionService, validator);
		adapter.setIgnoreDefaultModelOnRedirect(
				this.mvcProperties == null || this.mvcProperties.isIgnoreDefaultModelOnRedirect());
		return adapter;
	}
	@Bean
	@Primary
	@Override
	public RequestMappingHandlerMapping requestMappingHandlerMapping(
			@Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
			@Qualifier("mvcConversionService") FormattingConversionService conversionService,
			@Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {
		// Must be @Primary for MvcUriComponentsBuilder to work
		return super.requestMappingHandlerMapping(contentNegotiationManager, conversionService,
				resourceUrlProvider);
	}
	@Bean
	public WelcomePageHandlerMapping welcomePageHandlerMapping(ApplicationContext applicationContext,
			FormattingConversionService mvcConversionService, ResourceUrlProvider mvcResourceUrlProvider) {
		WelcomePageHandlerMapping welcomePageHandlerMapping = new WelcomePageHandlerMapping(
				new TemplateAvailabilityProviders(applicationContext), applicationContext, getWelcomePage(),
				this.mvcProperties.getStaticPathPattern());
		welcomePageHandlerMapping.setInterceptors(getInterceptors(mvcConversionService, mvcResourceUrlProvider));
		welcomePageHandlerMapping.setCorsConfigurations(getCorsConfigurations());
		return welcomePageHandlerMapping;
	}
}
```

```java
@Configuration(proxyBeanMethods = false)
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport
```

```java
public class WebMvcConfigurationSupport implements ApplicationContextAware, ServletContextAware {
	/**
	 * Return a {@link RequestMappingHandlerMapping} ordered at 0 for mapping
	 * requests to annotated controllers.
	 */
	@Bean
	@SuppressWarnings("deprecation")
	public RequestMappingHandlerMapping requestMappingHandlerMapping(
			@Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
			@Qualifier("mvcConversionService") FormattingConversionService conversionService,
			@Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {

		RequestMappingHandlerMapping mapping = createRequestMappingHandlerMapping();
		mapping.setOrder(0);
        ... ...
    }

	/**
	 * Return a handler mapping ordered at 1 to map URL paths directly to
	 * view names. To configure view controllers, override
	 * {@link #addViewControllers}.
	 */
	@Bean
	@Nullable
	public HandlerMapping viewControllerHandlerMapping(
			@Qualifier("mvcConversionService") FormattingConversionService conversionService,
			@Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {
    }

	/**
	 * Return a {@link BeanNameUrlHandlerMapping} ordered at 2 to map URL
	 * paths to controller bean names.
	 */
	@Bean
	public BeanNameUrlHandlerMapping beanNameHandlerMapping(
			@Qualifier("mvcConversionService") FormattingConversionService conversionService,
			@Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {

		BeanNameUrlHandlerMapping mapping = new BeanNameUrlHandlerMapping();
		mapping.setOrder(2);
        ... ...
    }

	/**
	 * Return a {@link RouterFunctionMapping} ordered at 3 to map
	 * {@linkplain org.springframework.web.servlet.function.RouterFunction router functions}.
	 * Consider overriding one of these other more fine-grained methods:
	 * <ul>
	 * <li>{@link #addInterceptors} for adding handler interceptors.
	 * <li>{@link #addCorsMappings} to configure cross origin requests processing.
	 * <li>{@link #configureMessageConverters} for adding custom message converters.
	 * <li>{@link #configurePathMatch(PathMatchConfigurer)} for customizing the {@link PathPatternParser}.
	 * </ul>
	 * @since 5.2
	 */
	@Bean
	public RouterFunctionMapping routerFunctionMapping(
			@Qualifier("mvcConversionService") FormattingConversionService conversionService,
			@Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {

		RouterFunctionMapping mapping = new RouterFunctionMapping();
		mapping.setOrder(3);
        ... ...
    }

    
	/**
	 * Return a handler mapping ordered at Integer.MAX_VALUE-1 with mapped
	 * resource handlers. To configure resource handling, override
	 * {@link #addResourceHandlers}.
	 */
	@Bean
	@Nullable
	public HandlerMapping resourceHandlerMapping(
			@Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
			@Qualifier("mvcConversionService") FormattingConversionService conversionService,
			@Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {

        ... ...
        addResourceHandlers(registry);  //该方法会被子类覆盖
        ... ...
        return simpleUrlHandlerMapping
    }
}
```

执行到上文的 addResourceHandlers(registry)，该方法会被子类，即 `EnableWebMvcConfiguration` 中的 `addResourceHandlers` 方法覆盖

`org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration.EnableWebMvcConfiguration#addResourceHandlers`

```java
@Override
protected void addResourceHandlers(ResourceHandlerRegistry registry) {
	super.addResourceHandlers(registry);
	if (!this.resourceProperties.isAddMappings()) {
		logger.debug("Default resource handling disabled");
		return;
	}
	ServletContext servletContext = getServletContext();
	addResourceHandler(registry, "/webjars/**", "classpath:/META-INF/resources/webjars/");
	addResourceHandler(registry, this.mvcProperties.getStaticPathPattern(), (registration) -> {
		registration.addResourceLocations(this.resourceProperties.getStaticLocations());
		if (servletContext != null) {
			registration.addResourceLocations(new ServletContextResource(servletContext, SERVLET_LOCATION));
		}
	});
}
```

该方法，添加了 `/webjars/**` `/**` 的路径匹配。

所以请求 `http://localhost:8080/user` 地址，即使我们没有配置 `/user` 对应的 `Controller` ，最终也会被 `SimpleUrlHandlerMapping` 处理。


比如当我们请求 `http://localhost:8080/user`，并且 Controller 没有匹配 `/user` 的控制器（注意：若有匹配 `/user/hello` 的控制器，但是没有匹配 `/user` 的控制器，也不会匹配到`http://localhost:8080/user`）时，springboot 也会匹配到处理器。

#### 404的复现

当请求不存在的url路径时，会出现 `NoHandlerFoundException` 异常

发送请求，当前项目没有符合 `/user` 的 `Controller`，所以请求该地址时会出现 `NoHandlerFoundException`

```http
GET http://localhost:8080/user
Accept: application/json
```

`spring mvc` 中，所有的请求由 `org.springframework.web.servlet.DispatcherServlet` 分发

```java
/**
 * Process the actual dispatching to the handler.
 * <p>The handler will be obtained by applying the servlet's HandlerMappings in order.
 * The HandlerAdapter will be obtained by querying the servlet's installed HandlerAdapters
 * to find the first that supports the handler class.
 * <p>All HTTP methods are handled by this method. It's up to HandlerAdapters or handlers
 * themselves to decide which methods are acceptable.
 * @param request current HTTP request
 * @param response current HTTP response
 * @throws Exception in case of any kind of processing failure
 */
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HttpServletRequest processedRequest = request;
	HandlerExecutionChain mappedHandler = null;
	boolean multipartRequestParsed = false;

	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

	try {
		ModelAndView mv = null;
		Exception dispatchException = null;

		try {
			processedRequest = checkMultipart(request);
			multipartRequestParsed = (processedRequest != request);

			// Determine handler for the current request.
			mappedHandler = getHandler(processedRequest);
			if (mappedHandler == null) {
				noHandlerFound(processedRequest, response);
				return;
			}

			// Determine handler adapter for the current request.
			HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

			// Process last-modified header, if supported by the handler.
			String method = request.getMethod();
			boolean isGet = "GET".equals(method);
			if (isGet || "HEAD".equals(method)) {
				long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
				if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
					return;
				}
			}

			if (!mappedHandler.applyPreHandle(processedRequest, response)) {
				return;
			}

			// Actually invoke the handler.
			mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

			if (asyncManager.isConcurrentHandlingStarted()) {
				return;
			}

			applyDefaultViewName(processedRequest, mv);
			mappedHandler.applyPostHandle(processedRequest, response, mv);
		}
		catch (Exception ex) {
			dispatchException = ex;
		}
		catch (Throwable err) {
			// As of 4.3, we're processing Errors thrown from handler methods as well,
			// making them available for @ExceptionHandler methods and other scenarios.
			dispatchException = new NestedServletException("Handler dispatch failed", err);
		}
		processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
	}
	catch (Exception ex) {
		triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
	}
	catch (Throwable err) {
		triggerAfterCompletion(processedRequest, response, mappedHandler,
				new NestedServletException("Handler processing failed", err));
	}
	finally {
		if (asyncManager.isConcurrentHandlingStarted()) {
			// Instead of postHandle and afterCompletion
			if (mappedHandler != null) {
				mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
			}
		}
		else {
			// Clean up any resources used by a multipart request.
			if (multipartRequestParsed) {
				cleanupMultipart(processedRequest);
			}
		}
	}
}
```

重点看如下代码

```java
// Determine handler for the current request.
mappedHandler = getHandler(processedRequest);
if (mappedHandler == null) {
	noHandlerFound(processedRequest, response);
	return;
}
```

它的作用是使用 `getHandler()` 方法获取 `request` 的处理器（`handler`），如果没有这个 `request` 对应的处理器，则执行 `noHandlerFound()` 方法

接着看下 `getHandler()` 方法里做了什么

```java
/**
 * Return the HandlerExecutionChain for this request.
 * <p>Tries all handler mappings in order.
 * @param request current HTTP request
 * @return the HandlerExecutionChain, or {@code null} if no handler could be found
 */
@Nullable
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
	if (this.handlerMappings != null) {
		for (HandlerMapping mapping : this.handlerMappings) {
			HandlerExecutionChain handler = mapping.getHandler(request);
			if (handler != null) {
				return handler;
			}
		}
	}
	return null;
}
```

处理器映射器共 5 个，如下图

![图 2](/images/20210202-处理器映射器.png)  


对这 5 个处理器映射器遍历，分别执行 `getHandler()` 方法，看看是否有符合该 `request` 的处理器执行链（`HandlerExecutionChain`）

遍历的第一个处理器映射器是 `RequestsMappingHandlerMapping` 对应的类是 `org.springframework.web.servlet.mvc.method.RequestMappingInfoHandlerMapping`。

执行`getHandler()` 方法时执行的是 `org.springframework.web.servlet.handler.AbstractHandlerMapping` 的 getHandler() 方法，因为 RequestMappingInfoHandlerMapping 类没有实现 getHandler() 方法，但是它的父类 AbstractHandlerMethodMapping 实现了 getHandler() 方法，所以它继承下来了。

```java
public abstract class RequestMappingInfoHandlerMapping extends AbstractHandlerMethodMapping<RequestMappingInfo>
```

![图 4](/images/20210202-getHandler-1.png)  

再执行 `getHandlerInternal()` 方法，该方法在 `RequestMappingInfoHandlerMapping` 类中有定义，所以会执行该类中定义的 `getHandlerInternal()` 方法

```java
@Override
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
	request.removeAttribute(PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);
	try {
		return super.getHandlerInternal(request);
	}
	finally {
		ProducesRequestCondition.clearMediaTypesAttribute(request);
	}
}
```

该方法移除了 `request` 中的一个 `attribute` ，然后调用了父类 `AbstractHandlerMethodMapping` 的 `getHandlerInternal()` 方法，如下图

![图 5](/images/20210202-getHandlerInternal-1.png)  

该方法中，获取了请求的地址，此处为 `/user`，然后查找是否有处理该地址的处理器方法（`HandlerMethod`）

此时没有找到处理 `/user` 地址的处理器方法，返回 null

然后进行下一个处理器映射器的遍历，当遍历到 `SimpleUrlHandlerMapping` 时，会发现竟然找到处理 `/user` 的处理器了，明明我们没有定义 `/user` 的 `Controller`，怎么还找到对应处理器了呢？

![图 6](/images/20210202-simpleUrlHandlerMapping.png)  

如上图，在 `SimpleUrlHandlerMapping` 中的 `ResourceHttpRequestHandler` 会使用 `/webjars/**` 和 `/**` 作为 pattern 对 url 路径进行匹配，此处 `/user` 是肯定可以被 `/**` 匹配到的，所以 `/user` 可以查询到 `HandlerExecutionChain`。

`handlerMappings` 是一个 `ArrayList` ，遍历时是有序的，所以会按照 `RequestMappingHandlerMapping` -> `BeanNameUrlHandlerMapping` -> `RouterFunctionMapping` -> `SimpleUrlHandlerMapping` -> `WelcomePageHandlerMapping` 的顺序进行遍历，不用担心 `SimpleUrlHandlerMapping` 中的 `/**` `pattern` 会影响其他的 `Controller`

找到 /user 对应的处理器了，再次回到如下代码

```java
// Determine handler for the current request.
mappedHandler = getHandler(processedRequest);
if (mappedHandler == null) {
	noHandlerFound(processedRequest, response);
	return;
}
```

```java
/**
 * No handler found -> set appropriate HTTP response status.
 * @param request current HTTP request
 * @param response current HTTP response
 * @throws Exception if preparing the response failed
 */
protected void noHandlerFound(HttpServletRequest request, HttpServletResponse response) throws Exception {
	if (pageNotFoundLogger.isWarnEnabled()) {
		pageNotFoundLogger.warn("No mapping for " + request.getMethod() + " " + getRequestUri(request));
	}
	if (this.throwExceptionIfNoHandlerFound) {
		throw new NoHandlerFoundException(request.getMethod(), getRequestUri(request),
				new ServletServerHttpRequest(request).getHeaders());
	}
	else {
		response.sendError(HttpServletResponse.SC_NOT_FOUND);
	}
}
```

所以，即使我们没有定义 `/user` 对应的 `Controller` ，也不会抛出 `NoHandlerFoundException` 异常

然后执行到如下代码

```java
// Determine handler adapter for the current request.
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
```

根据上面查到的 `Handler` （`ResourceHttpRequestHandler`） 查找对应的 `Adapter` （`ha` 为 `HttpRequestHandlerAdapter`）

```java
// Actually invoke the handler.
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

调用 `HttpRequestHandlerAdapter` 的 `handle()` 方法，其中传入的 `handler` 参数是 `ResourceHttpRequestHandler`

```java
public class HttpRequestHandlerAdapter implements HandlerAdapter {

	@Override
	@Nullable
	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		((HttpRequestHandler) handler).handleRequest(request, response);
		return null;
	}
}
```

执行 `ResourceHttpRequestHandler` 的 `handleRequest()` 方法

```java
@Override
public void handleRequest(HttpServletRequest request, HttpServletResponse response)
		throws ServletException, IOException {

	// For very general mappings (e.g. "/") we need to check 404 first
	Resource resource = getResource(request);
	if (resource == null) {
		logger.debug("Resource not found");
		response.sendError(HttpServletResponse.SC_NOT_FOUND);
		return;
	}
}
```

`SimpleUrlHandlerMapping` 中的 `/**` 被匹配到后，spring 依次去 `classpath:/META-INF/resources/`、 `classpath:/resources/`、 `classpath:/static/`、 `classpath:/public/` 查找该资源

根据请求 `url` 中的路径 `user` 在 资源目录中 找不到资源，返回的 `resource` 为 `null` ，执行 `response.sendError(404)` ，并返回

而 `response.sendError(404)`，会重新请求 `/error` 地址，不信可以测试下 

测试代码

```java
@Slf4j
@RequestMapping("/user")
@RestController
public class UserController {

    @RequestMapping("/hello")
    public String hello(HttpServletResponse response) throws IOException {
        response.sendError(404);
        return "123";
    }
}
```

`spring boot` 还有一个 `mvc` 相关的自动配置类，即 `org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration`

该类主要内容如下

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class })
// Load before the main WebMvcAutoConfiguration so that the error View is available
@AutoConfigureBefore(WebMvcAutoConfiguration.class)
@EnableConfigurationProperties({ ServerProperties.class, WebMvcProperties.class })
public class ErrorMvcAutoConfiguration {

    private final ServerProperties serverProperties;

	@Bean
	@ConditionalOnMissingBean(value = ErrorController.class, search = SearchStrategy.CURRENT)
	public BasicErrorController basicErrorController(ErrorAttributes errorAttributes,
			ObjectProvider<ErrorViewResolver> errorViewResolvers) {
		return new BasicErrorController(errorAttributes, this.serverProperties.getError(),
				errorViewResolvers.orderedStream().collect(Collectors.toList()));
	}
}

```

ServerProperties 类是参数配置类，可以配置 `server.error` 相关参数

```java
@ConfigurationProperties(prefix = "server", ignoreUnknownFields = true)
public class ServerProperties
```

重点是

```java
// Load before the main WebMvcAutoConfiguration so that the error View is available
@AutoConfigureBefore(WebMvcAutoConfiguration.class)
```

该类 `ErrorMvcAutoConfiguration` 需要在 `WebMvcAutoConfiguration` 自动配置类之前进行配置

同时，配置该类（`ErrorMvcAutoConfiguration` ）时，会新建 控制器 `BasicErrorController` ，该控制器主要内容如下

```java
@Controller
@RequestMapping("${server.error.path:${error.path:/error}}")
public class BasicErrorController extends AbstractErrorController {
	@RequestMapping(produces = MediaType.TEXT_HTML_VALUE)
	public ModelAndView errorHtml(HttpServletRequest request, HttpServletResponse response) {
		HttpStatus status = getStatus(request);
		Map<String, Object> model = Collections
				.unmodifiableMap(getErrorAttributes(request, getErrorAttributeOptions(request, MediaType.TEXT_HTML)));
		response.setStatus(status.value());
		ModelAndView modelAndView = resolveErrorView(request, response, status, model);
		return (modelAndView != null) ? modelAndView : new ModelAndView("error", model);
	}

	@RequestMapping
	public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {
		HttpStatus status = getStatus(request);
		if (status == HttpStatus.NO_CONTENT) {
			return new ResponseEntity<>(status);
		}
		Map<String, Object> body = getErrorAttributes(request, getErrorAttributeOptions(request, MediaType.ALL));
		return new ResponseEntity<>(body, status);
	}
}
```

该 `Controller` 匹配的 `url` 默认为 `/error` ，这样，当 `response.sendError(404)`，会重新请求 `/error` 地址时，就会请求到该 `Controller` 中。

如果请求头中的 `content-type` 是 `text/html` ，则会对应到 `errorHtml()` 方法，返回  `Whitelabel Error Page` 页面；`content-type` 是其他类型，调用 `error()` 方法，返回 json 数据。

至此，404 页面整个流程就走通了。






#### 404页面返回自定义格式的json

首先，需要让请求路径不存在的地址，找不到对应的处理器

网上搜索到了两种方式

第一种方式是配置 `spring.web.resources.add-mappings=false`，关闭静态资源映射。这种方式的缺点是项目配置了该参数后，本项目或依赖本项目的其他项目就不能使用静态资源映射了，示例：如果类路径下 `/resources` 目录存在  `resources.html`，无法访问该 html 了，并且访问 `http://localhost:8080/error` ，也会出现 404页面。

第二种方式是配置 `spring.mvc.static-path-pattern=/statics/**`，即使用 `/statics/**` 替换 默认的静态资源地址 `/**`，示例：如果类路径下 `/resources` 目录存在  `resources.html`，可以通过 `http://localhost:8080/static/resources.html`
访问。但是，访问 `http://localhost:8080/static/test.html` 还是会出现404页面的（静态资源目录里没有 `test.html` 文件），并且访问 `http://localhost:8080/error` ，也会出现 404页面。

`org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration.EnableWebMvcConfiguration#addResourceHandlers`  

```java
@Override
protected void addResourceHandlers(ResourceHandlerRegistry registry) {
	super.addResourceHandlers(registry);
	if (!this.resourceProperties.isAddMappings()) {
		logger.debug("Default resource handling disabled");
		return;
	}
	ServletContext servletContext = getServletContext();
	addResourceHandler(registry, "/webjars/**", "classpath:/META-INF/resources/webjars/");
	addResourceHandler(registry, this.mvcProperties.getStaticPathPattern(), (registration) -> {
		registration.addResourceLocations(this.resourceProperties.getStaticLocations());
		if (servletContext != null) {
			registration.addResourceLocations(new ServletContextResource(servletContext, SERVLET_LOCATION));
		}
	});
}
```

```java
@ConfigurationProperties("spring.web")
public class WebProperties {
	
    public static class Resources {

		private static final String[] CLASSPATH_RESOURCE_LOCATIONS = { "classpath:/META-INF/resources/",
				"classpath:/resources/", "classpath:/static/", "classpath:/public/" };

		/**
		 * Whether to enable default resource handling.
		 */
		private boolean addMappings = true;

    }
}
```

```java
@ConfigurationProperties(prefix = "spring.mvc")
public class WebMvcProperties {

	/**
	 * Path pattern used for static resources.
	 */
	private String staticPathPattern = "/**";
}
```


上述两种方式的原因可以看以上代码

上面两种方式或多或少都有些缺点，比如第一个配置了之后就不能访问静态资源了，第二个配置了之后静态资源请求url会改变，并且不能彻底避免404的问题

下面介绍第三种方式，主要思路是通过子类的方式，覆盖重写404页面的逻辑，改成符合自己要求的输出

首先，自定义一个模仿 `org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration` 自定义一个配置类 `CustomErrorMvcConfig` ，覆盖 `BasicErrorController` 实例的生成方法，生成我们自定义的默认地址为 `/error` 的控制器。

注意，该配置类需要在 `ErrorMvcAutoConfiguration` 之前进行加载，同时将 `basicErrorController` 方法上的 `search` 属性改为 `search = SearchStrategy.ALL` ，才能在本项目作为其他项目的依赖导入后，其他项目也能生效（当然，本项目`/META-INF/spring.factories` 里需要添加自动配置类）。

```java
package org.msdemt.simple.unified_exception_handler_demo.config;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.AutoConfigureBefore;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
import org.springframework.boot.autoconfigure.condition.SearchStrategy;
import org.springframework.boot.autoconfigure.web.ServerProperties;
import org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration;
import org.springframework.boot.autoconfigure.web.servlet.WebMvcProperties;
import org.springframework.boot.autoconfigure.web.servlet.error.BasicErrorController;
import org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration;
import org.springframework.boot.autoconfigure.web.servlet.error.ErrorViewResolver;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.boot.web.servlet.error.ErrorAttributes;
import org.springframework.boot.web.servlet.error.ErrorController;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.DispatcherServlet;

import javax.servlet.Servlet;
import java.util.stream.Collectors;

@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
@ConditionalOnClass({Servlet.class, DispatcherServlet.class})
// Load before the main WebMvcAutoConfiguration so that the error View is available
@AutoConfigureBefore({WebMvcAutoConfiguration.class, ErrorMvcAutoConfiguration.class})
@EnableConfigurationProperties({ServerProperties.class, WebMvcProperties.class})
public class CustomErrorMvcConfig {

    private final ServerProperties serverProperties;

    private CustomErrorAttributes customErrorAttributes;

    @Autowired
    public CustomErrorMvcConfig(ServerProperties serverProperties, CustomErrorAttributes customErrorAttributes) {
        this.serverProperties = serverProperties;
        this.customErrorAttributes = customErrorAttributes;
    }

    @Bean
    @ConditionalOnMissingBean(value = ErrorController.class, search = SearchStrategy.ALL)
    public BasicErrorController basicErrorController(ErrorAttributes errorAttributes,
                                                     ObjectProvider<ErrorViewResolver> errorViewResolvers) {
        return new CustomErrorController(customErrorAttributes, this.serverProperties.getError(),
                errorViewResolvers.orderedStream().collect(Collectors.toList()));
    }
}
```

自定义的错误控制器 `CustomErrorController` ，需要继承 `org.springframework.boot.autoconfigure.web.servlet.error.BasicErrorController`，并且覆盖 `errorHtml()`方法，这样如果使用 `text/html` 请求一个404页面，也会显示我们期望的 json


`BasicErrorController` 类中的 `error()` 方法不能直接覆盖，因为它的父类 `org.springframework.boot.autoconfigure.web.servlet.error.AbstractErrorController`

```java
public abstract class AbstractErrorController implements ErrorController {
    private final ErrorAttributes errorAttributes;

	public AbstractErrorController(ErrorAttributes errorAttributes, List<ErrorViewResolver> errorViewResolvers) {
		Assert.notNull(errorAttributes, "ErrorAttributes must not be null");
		this.errorAttributes = errorAttributes;
		this.errorViewResolvers = sortErrorViewResolvers(errorViewResolvers);
	}
```

`ErrorAttributes` 不能为空

`BasicErrorController` 没有空参的构造方法，如下

```java
@RequestMapping("${server.error.path:${error.path:/error}}")
public class BasicErrorController extends AbstractErrorController {

	private final ErrorProperties errorProperties;

	/**
	 * Create a new {@link BasicErrorController} instance.
	 * @param errorAttributes the error attributes
	 * @param errorProperties configuration properties
	 */
	public BasicErrorController(ErrorAttributes errorAttributes, ErrorProperties errorProperties) {
		this(errorAttributes, errorProperties, Collections.emptyList());
	}

	/**
	 * Create a new {@link BasicErrorController} instance.
	 * @param errorAttributes the error attributes
	 * @param errorProperties configuration properties
	 * @param errorViewResolvers error view resolvers
	 */
	public BasicErrorController(ErrorAttributes errorAttributes, ErrorProperties errorProperties,
			List<ErrorViewResolver> errorViewResolvers) {
		super(errorAttributes, errorViewResolvers);
		Assert.notNull(errorProperties, "ErrorProperties must not be null");
		this.errorProperties = errorProperties;
	}
}
```

它的子类 `CustomErrorController` 必须显式调用父类构造方法，且 `errorAttributes` 不能为 `null` ，所以 CustomErrorMvcConfig 中新建 `CustomErrorController` 实例 时必须传入 `customErrorAttributes`


`CustomErrorAttributes` 类的内容如下

```java
package org.msdemt.simple.unified_exception_handler_demo.config;

import org.msdemt.simple.unified_exception_handler_demo.constant.enums.CommonResponseEnum;
import org.springframework.boot.web.error.ErrorAttributeOptions;
import org.springframework.boot.web.servlet.error.DefaultErrorAttributes;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.WebRequest;

import java.util.HashMap;
import java.util.Map;

@Component
public class CustomErrorAttributes extends DefaultErrorAttributes {

    public CustomErrorAttributes() {

    }

    @Override
    public Map<String, Object> getErrorAttributes(WebRequest webRequest, ErrorAttributeOptions options) {
        Map<String, Object> errorAttributes = new HashMap<>();

        //404错误 统一返回如下信息
        errorAttributes.put("code", CommonResponseEnum.NO_MATCHIING_BUSINESS_FOUND.getCode());
        errorAttributes.put("mess", CommonResponseEnum.NO_MATCHIING_BUSINESS_FOUND.getMessage());

        return errorAttributes;
    }
}
```

配置 spring 当找不到请求路径的处理器时，将异常（`NoHandlerFoundException`）抛出

`org.springframework.web.servlet.DispatcherServlet`

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {

                ... ...

				// Determine handler for the current request.
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

                ... ...
}

/**
 * No handler found -> set appropriate HTTP response status.
 * @param request current HTTP request
 * @param response current HTTP response
 * @throws Exception if preparing the response failed
 */
protected void noHandlerFound(HttpServletRequest request, HttpServletResponse response) throws Exception {
	if (pageNotFoundLogger.isWarnEnabled()) {
		pageNotFoundLogger.warn("No mapping for " + request.getMethod() + " " + getRequestUri(request));
	}
	if (this.throwExceptionIfNoHandlerFound) {
		throw new NoHandlerFoundException(request.getMethod(), getRequestUri(request),
				new ServletServerHttpRequest(request).getHeaders());
	}
	else {
		response.sendError(HttpServletResponse.SC_NOT_FOUND);
	}
}
```

通过上面的配置，我们实现了若请求一个未定义的地址，找不到该地址对应的handler，能够进入 `mappedHandler == null` 的判断，但是此时还无法抛出 NoHandlerFoundException 异常。

`org.springframework.boot.autoconfigure.web.servlet.WebMvcProperties#throwExceptionIfNoHandlerFound`

```java
@ConfigurationProperties(prefix = "spring.mvc")
public class WebMvcProperties {
	/**
	 * Whether a "NoHandlerFoundException" should be thrown if no Handler was found to
	 * process a request.
	 */
	private boolean throwExceptionIfNoHandlerFound = false;
}
```

`org.springframework.web.servlet.DispatcherServlet#noHandlerFound`

```java
protected void noHandlerFound(HttpServletRequest request, HttpServletResponse response) throws Exception {
	if (pageNotFoundLogger.isWarnEnabled()) {
		pageNotFoundLogger.warn("No mapping for " + request.getMethod() + " " + getRequestUri(request));
	}
	if (this.throwExceptionIfNoHandlerFound) {
		throw new NoHandlerFoundException(request.getMethod(), getRequestUri(request),
				new ServletServerHttpRequest(request).getHeaders());
	}
	else {
		response.sendError(HttpServletResponse.SC_NOT_FOUND);
	}
}
```

如上，`throwExceptionIfNoHandlerFound` 的值默认时 `false` ，此时不会新建并抛出 `NoHandlerFoundException` 异常，需要将该值置为 `true` ，所以需要修改本项目的 `application.properties` ， 添加配置 `spring.mvc.throw-exception-if-no-handler-found=true`


最后，为了上文定义的配置类，能够在本项目作为依赖时引入其他项目后生效，即自动加载放入spring容器中，需要在 /META-INF/spring.factories 文件中添加类引用。

如下

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.msdemt.simple.unified_exception_handler_demo.i18n.UnifiedMessageSource,\
  org.msdemt.simple.unified_exception_handler_demo.exception.handler.UnifiedExceptionHandler,\
  org.msdemt.simple.unified_exception_handler_demo.config.CustomErrorAttributes,\
  org.msdemt.simple.unified_exception_handler_demo.config.CustomErrorMvcConfig
```

至此，springboot 统一异常处理就完成了。
