楼主前几天写了一篇“[Java子线程中的异常处理（通用）](https://www.cnblogs.com/toplist/p/7594557.html)”文章，介绍了在多线程环境下3种通用的异常处理方法。

但是平时大家的工作一般是基于开发框架进行的（比如Spring MVC，或Spring Boot），所以会有相应特定的异常处理方法，这篇文章要介绍的就是web应用中的异常处理。

想快速解决问题的小伙伴可以只看“解决办法”，想进一步了解细节的小伙伴还可以看“深入剖析”部分。

## 适用场景

使用Spring MVC或Spring Boot框架搭建的web应用

## 解决办法

@ControllerAdvice注解 + @ExceptionHandler注解

实现一个异常处理类，在类上应用@ControllerAdvice注解，并在异常处理方法上应用@ExceptionHandler注解。那么在web应用中，当Controller的@RequestMapping方法抛出指定的异常类型时，@ExceptionHandler修饰的异常处理方法就会执行。

示例：

```java
@ControllerAdvice
public class WebServerExceptionHandler {
    Logger log = LoggerFactory.getLogger(this.getClass());

    public WebServerExceptionHandler() {
    }

    // 指定捕获的异常类型，这里是自定义的SomeException
    @ExceptionHandler({SomeException.class})
    public ResponseEntity<WebServerExceptionResponse> handle(HttpServletResponse response, SomeException ex) {
        WebServerExceptionResponse body = new WebServerExceptionResponse();
        body.setStatus(ex.getStatus());
        body.setMessage(ex.getMessage());
        this.log.info("handle SomeException, status:{}, message:{}",  new Object[]{body.getStatus(), body.getMessage()});
        return new ResponseEntity(body, HttpStatus.valueOf(ex.getStatus()));
    }

    // 指定捕获的异常类型，这里是自定义的OtherException
    @ExceptionHandler({OtherException.class})
    public ResponseEntity<WebServerExceptionResponse> handle(HttpServletResponse response, OtherException ex) {
        WebServerExceptionResponse body = new WebServerExceptionResponse();
        body.setStatus(ex.getStatus());
        body.setMessage(ex.getMessage());
        this.log.info("handle OtherException, status:{}, message:{}",  new Object[]{body.getStatus(), body.getMessage()});
        return new ResponseEntity(body, HttpStatus.valueOf(ex.getStatus()));
    }
}
```

## 深入剖析

### @ControllerAdvice的定义如下：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface ControllerAdvice {

    String[] value() default {};

    String[] basePackages() default {};

    Class<?>[] basePackageClasses() default {};

    Class<?>[] assignableTypes() default {};

    Class<? extends Annotation>[] annotations() default {};

}
```

可以看出它应用在TYPE类型的元素上（也即class或interface），运行时生效。

作用是Controller类的帮助注解，一般搭配@ExceptionHandler注解，用来处理Controller的@RequestMapping修饰的方法抛出的异常。

楼主根据源码的注释整理了5个参数的含义，它们都是用来限定需要处理的Controller的：

* value()：等同于basePackages，表示需要被处理的Controller包名数组，例如 @ControllerAdvice("org.my.pkg")。如果不指定，就代表处理所有的Controller类
* basePackages()：表示需要被处理的Controller包名数组，例如 @ControllerAdvice(basePackages={"org.my.pkg","org.my.other.pkg"})
* basePackageClasses()：通过标记类来指定Controller包名数组
* assignableTypes()：通过类的Class对象来指定Controller包名数组
* annotations()：被注解修饰的Controller需要被处理

性能考虑：不要指定过多的参数和异常处理策略，因为异常检查和处理都是在运行时做的。

### @ExceptionHandler的定义如下：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ExceptionHandler {

    /**
     * Exceptions handled by the annotated method. If empty, will default to any
     * exceptions listed in the method argument list.
     */
    Class<? extends Throwable>[] value() default {};

}
```

可以看出它作用在方法上面，而且参数很好理解，就是需要处理的异常类的Class对象数组。

但是，它对修饰的异常处理方法的参数和返回值有限定，楼主根据源码的注释整理如下：

（1）异常处理方法的参数限定，可以是以下类型，顺序任意：

* 异常类对象
* HttpServletRequest、HttpServletResponse
* HttpSession
* InputStream/Reader、OutputStream/Writer

（2）异常处理方法的返回值限定，最终会写入response流：

* ResponseEntity
* HttpServletResponse
* ModelAndView
* Model
* Map
* View

## 总结

以上就是在Spring web应用中的异常处理方法：使用@ControllerAdvice搭配@ExceptionHandler修饰自定义异常处理方法，处理来自Controller类中的@RequestMapping方法抛出的异常。

使用时需要根据实际情况，合理设置@ControllerAdvice和@ExceptionHandler的参数。
