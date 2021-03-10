---
layout:     post
title:      "异常的日常处理"
subtitle:   " \" Exception\""
date:       2021-03-08 12:00:00
author:     "ysbao"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Java

---

> 千里之行，始于足下

### 一、异常的概述

异常就是Java程序在运行过程中出现的错误

![img](/img/java/Exception/picture_01.png)

这是异常的类图

**Throwable是Error和Exception的父类，用来定义所有可以作为异常被抛出来的类。**



**1、Error和Exception区分**：

Error是**编译时错误（CompilerError）和系统错误（VirtualMachineError）**，系统错误在除特殊情况下，都不需要你来关心，基本不会出现，主要是内存溢出OutofMemoryError和StackOverflowError。而编译时错误，如果你使用了编译器，那么编译器会提示。

Exception则是可以被抛出的基本类型，我们需要主要关心的也是这个类。

Exception又分为**RunTimeException和其他Exception**



**2、RunTimeException和其他Exception区分**：

**其他Exception**:**受检查异常**。可以理解为错误，必须要开发者解决以后才能编译通过，解决的方法有两种，1：throw到上层，2，try-catch处理。常见的异常有：IOException、PrintException、SQLException等。

**RunTimeException**：**运行时异常，又称不受检查异常**，因为不受检查，所以在代码中可能会有RunTimeException时Java编译检查时不会告诉你有这个异常，**但是在实际运行代码时则会暴露出来，如果不处理也会被Java自己处理**。比如经典的（1/0 ）ArithmeticException算术异常、 NullPointerException空指针、 ClassCastException类型强制转换异常、ArrayIndexOutOfBoundsException数组下标越界异常、IllegalArgumentException参数异常、FileNotFoundException 文件未找到异常、ClassNotFoundException找不到类异常等。

### 二、异常的处理

> 如果程序出现了问题，我们没有做任何处理，最终jvm会做出默认的处理。jvm默认的处理方法就是在调用 printStackTrace()方法
> 把异常的名称，原因及出现的问题等信息输出在控制台。同时将程序停止运行

#### 1、 try-catch

catch 内部一定要对异常进行处理，一般是输出日志文件中，或者是将错误信息持久化。切勿不做处理把异常吞没。

```java
try { 
     可能出现问题的代码;
 } catch(异常名 变量) {
      针对问题的处理;
 } finally { 
     释放资源;
 }
 
getMessage()：获取异常信息，返回字符串。
toString()：获取异常类名和异常信息，返回字符串。
printStackTrace()：获取异常类名和异常信息，以及异常出现在程序中的位置。返回值void。
```

#### 2、异常声明Throws

有些时候，我们是可以对异常进行处理的，但是又有些时候，我们根本就没有权限去处理某个异常。或者说，我处理不了，我就不处理了。
为了解决出错问题，Java针对这种情况，就提供了另一种处理方案：抛出。

```java
//例如不想处理这个异常，可以由该方法的调用者来处理
//throws表示出现异常的一种可能性，并不一定会发生这些异常
//可以跟多个异常类名，用逗号隔开
public static void method() throws ParseException { 
         String s = "2016-09-03";
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"); 
       Date d = sdf.parse(s);
       System.out.println(d);
   }
```

**Throw**  **则是用来抛出一个具体的异常类型**，抛出的是具体的异常对象名  

 用在方法体内，跟的是异常对象名
 只能抛出一个异常对象名
 表示抛出异常，由方法体内的语句处理
 throw则是抛出了异常，执行throw则一定抛出了某种异常

当我们从方法内部抛出异常之后，就必须对这个异常进行处理（要么在方法上进行声明（抛出）， 或者进行try ()catch{}）。不做处理最终交给jvm处理。

**自定义异常**

> java不可能对所有的情况都考虑到，所以，在实际的开发中，我们可能需要自己定义异常。而我们自己随意的写一个类，是不能作为异常类来看的，要想你的类是一个异常类，就必须继承自Exception或者RuntimeException

```java
/* * 自定义异常测试类 */
public class StudentDemo { 
      public static void main(String[] args) { 
           Scanner sc = new Scanner(System.in);     
           System.out.println("请输入学生成绩："); 
            int score = sc.nextInt(); 
            Teacher t = new Teacher(); 
             try { 
                   t.check(score); 
             }  catch (MyException e) { 
                   e.printStackTrace(); 
             } 
        }
}
/* *自定义 */
class MyException extends Exception { 
           public MyException() { } 
           public MyException(String message) {
                  super(message);
           }
}
//老师类
 class Teacher { 
          public void check(int score) throws MyException { 
               if (score > 100 || score < 0) {
                   throw new MyException("分数必须在0-100之间"); 
               } else { 
                   System.out.println("分数没有问题"); 
          }
 }
```

### 三、异常和Spring事务

　Spring的事务管理默认是针对unchecked exception回滚，也就是**默认对Error异常和RuntimeException异常以及其子类进行事务回滚****，**且必须对抛出异常**，若使用try-catch对其异常捕获则不会进行回滚！（Error异常和RuntimeException异常抛出时不需要方法调用throws或try-catch语句）；

   **checked异常,checked异常必须由try-catch语句包含或者由方法throws抛出，且事务默认对checked异常不进行回滚。**

**@Transactional注解**

**在默认的代理模式下，只有目标方法由外部调用，才能被 Spring 的事务拦截器拦截。在同一个类中的两个方法直接调用，是不会被 Spring 的事务拦截器拦截**

用法

* 在需要事务管理的地方加@Transactional 注解。@Transactional 注解可以被应用于接口定义和接口方法、类定义和类的 public 方法上。

* Isolation属性：**事务的隔离级别**，默认值为 Isolation.DEFAULT。使用底层数据库默认的隔离级别。
* propagation 属性:**事务的传播行为**，默认值为 Propagation.REQUIRED。
* value 和 transactionManager 属性,当配置了多个事务管理器时，可以使用该属性指定选择哪个事务管理器

*   在项目中，@Transactional(rollbackFor=Exception.class)，如果类加了这个注解，那么这个类里面的方法抛出异常，就会回滚，数据库里面的数据也会回滚。
* 在@Transactional注解中如果不配置rollbackFor属性,**那么事物只会在遇到RuntimeException的时候才会回滚**,加上rollbackFor=Exception.class,可以让事物在遇到非运行时异常时也回滚。

**编写注意事项：**

Service中实现对事务的控制时，必须抛出异常（事务作用域内），不能捕捉处理，否则不会回滚事务。

controller 层，对异常统一处理，一、保存每天的操作日志，可以记录每个人请求地址，操作类型和，操作时间等等。打印出入参日志和返回结果（入库）。前后端分离，前端请求token校验，以及敏感参数过滤。（使用自定义注解环绕aop和拦截器都行）、二、全局异常处理，将异常信息持久化数据库，便于排除。

### 四、Spring 全局异常捕捉处理

@ControllerAdvice 和 @RestControllerAdvice都是对Controller进行增强的，可以全局捕获spring mvc抛的异常。
 RestControllerAdvice = ControllerAdvice + ResponseBody，方法自动返回json数据。

作用：异常处理情况，可以封装成统一的数据格式。对特定的异常返回特定的描述给用户，用户体验比较好，异常管理比较方便。

```java
@RestControllerAdvice
public class DefaultExceptionHandler {

	private static final Logger log = LoggerFactory.getLogger(DefaultExceptionHandler.class);

	/**
	 * 权限校验失败
	 */
	@ExceptionHandler(AuthorizationException.class)
	public AjaxResult handleAuthorizationException(AuthorizationException e) {
		log.error(e.getMessage(), e);
		return AjaxResult.error("您没有数据的权限，请联系管理员添加");
	}

	/**
	 * 请求方式不支持
	 */
	@ExceptionHandler({ HttpRequestMethodNotSupportedException.class })
	public AjaxResult handleException(HttpRequestMethodNotSupportedException e) {
		log.error(e.getMessage(), e);
		return AjaxResult.error("不支持' " + e.getMethod() + "'请求");
	}

	/**
	 * 拦截未知的运行时异常
	 */
	@ExceptionHandler(RuntimeException.class)
	public AjaxResult notFount(RuntimeException e) {
		log.error("运行时异常:", e);
		return AjaxResult.error("运行时异常:" + e.getMessage());
	}

	/**
	 * 系统异常
	 */
	@ExceptionHandler(Exception.class)
	public AjaxResult handleException(Exception e) {
		log.error(e.getMessage(), e);
		return AjaxResult.error("服务器错误，请联系管理员");
	}

	/**
	 * 自定义演示模式异常
	 */
	@ExceptionHandler(DemoModeException.class)
	public AjaxResult demoModeException(DemoModeException e) {
		return AjaxResult.error("演示模式，不允许操作");
	}

}
```

### 五、引用

[Spring Boot 中使用 @Transactional 注解配置事务管理](https://blog.csdn.net/nextyu/article/details/78669997)

[Spring Boot @RestControllerAdvice 统一异常处理](https://www.jianshu.com/p/47aeeba6414c)

