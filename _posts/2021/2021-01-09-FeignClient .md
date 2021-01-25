---
layout:     post
title:      "FeignClient 的原理和使用"
subtitle:   " \" FeignClient\""
date:       2021-01-09 12:00:00
author:     "ysbao"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Java
    - Restful工具
---

> 千里之行，始于足下

### 一、原理

Feign 是一个 Java 到 HTTP 的客户端绑定器，Feign 允许您在 Apache HC 等http 库之上编写自己的代码。Feign 以最小的开销将代码连接到 http APIs，并通过可定制的解码器和错误处理（可以写入任何基于文本的 http APIs）将代码连接到 http APIs。

Feign 通过将注解处理为模板化请求来工作。参数在输出之前直接应用于这些模板。尽管 Feign 仅限于支持基于文本的 APIs，但它极大地简化了系统方面，例如重放请求。此外，Feign 使得对转换进行单元测试变得简单。

大概原理如下：

* 启动时，程序会进行包扫描，扫描所有包下所有@FeignClient注解的类，并将这些类注入到spring的IOC容器中。当定义的Feign中的接口被调用时，通过JDK的动态代理来生成RequestTemplate。

* RequestTemplate中包含请求的所有信息，如请求参数，请求URL等。

* RequestTemplate声场Request，然后将Request交给client处理，这个client默认是JDK的HTTPUrlConnection，也可以是OKhttp、Apache的HTTPClient等。

* 最后client封装成LoadBaLanceClient，结合ribbon负载均衡地发起调用。

### 二、处理过程图

![img](/img/java/Feign/picture_01.jpg)

### 三、简单的使用

##### 	1、添加Maven依赖包

```java
 // 注意和Spring boot 版本兼容性问题
	<dependency>
    	<groupId>org.springframework.cloud</groupId>
    	<artifactId>spring-cloud-starter-openfeign</artifactId>
    	<version>2.1.3.RELEASE</version>
    </dependency>
```

##### 2、在main主入口类添加FeignClients启用注解

```java
//包路径可以不加，路径为@FeignClient注解所在的包路径，默认全部生效
@EnableFeignClients("com.example.openfeign.feign.client")
....
```

##### 3、编写FeignClient代码

```java
@FeignClient(
        name = "service",
        url = "${feign.client.config.service.url}",//配置表信息
)
public interface FeignService {
    @RequestMapping(value = "/manager/searchList", method = RequestMethod.GET)
    	public String searchZsiList(@RequestParam("name") String name
    );
}
```

##### 4、直接使用FeignClient

```java
@Autowired
FeignService feignService;
```

### 四、**Http Client 配置**

feign 在默认情况下使用**JDK 原生的 URLConnection** 发送HTTP请求。(没有连接池，保持长连接) 。

可以通过修改 client 依赖换用底层的 client，不同的 http client 对请求的支持可能有差异。

##### 1、添加Maven依赖包

```java
        <dependency>
            <groupId>io.github.openfeign</groupId>
            <artifactId>feign-httpclient</artifactId>
            <version>9.7.0</version>
        </dependency>
```

##### 2、添加属性配置

```java
feign:
  httpclient:
    enabled: true
    maxConnections: 20480
    maxConnectionsPerRoute: 512
    timeToLive: 60
    connectionTimeout: 10000
    userAgent: 'Mozilla/5.0 (Windows NT 6.1; WOW64; rv:37.0) Gecko/20100101 Firefox/37.0'
```

##### 3、修改配置FeignCongfig

```
@Configuration(proxyBeanMethods = false)
@AutoConfigureBefore({FeignClientsConfiguration.class})
public class FeignConfig {
	@Bean
    @Primary
    public Client feignClient(HttpClient httpClient) {
        return new ApacheHttpClient(httpClient);
    }
}
//加载配置调用
@FeignClient(
        name = "service",
        url = "${feign.client.config.service.url}",
        configuration= FeignConfig.class
)
//也可以全局配置，在启动类那边
@EnableFeignClients(defaultConfiguration = FeignConfig.class)
```

### 五、断路器 hystrix 配置

熔断降级策略

**熔断**一般是某个服务故障或者是异常引起的，类似现实世界中的‘保险丝’，当某个异常条件被触发，直接熔断整个服务，**而不是一直等到此服务超时**，为了防止防止整个系统的故障，而采用了一些保护措施。过载保护。比如A服务的X功能依赖B服务的某个接口，当B服务接口响应很慢时，A服务X功能的响应也会被拖慢，进一步导致了A服务的线程都卡在了X功能上，A服务的其它功能也会卡主或拖慢。此时就需要熔断机制，即A服务不在请求B这个接口，**而可以直接进行降级处理。**

**降级**：服务器当压力剧增的时候，根据当前业务情况及流量，对一些服务和页面进行有策略的降级。以此缓解服务器资源的的压力，以保证核心业务的正常运行，同时也保持了客户和大部分客户的得到正确的相应。

通白来说：就是当某个服务熔断之后，服务器将不再被调用，此时客户端可以自己准备一个本地的fallback回调，返回一个缺省值。

**自动降级**：超时、失败次数、故障、限流

 （1）配置好**超时时间**(异步机制探测回复情况)；

 （2）不稳的的api调用次数达到一定数量进行降级(异步机制探测回复情况)；

 （3）调用的远程服务出现故障(**dns、http服务错误状态码、网络故障、Rpc服务异常**)，直接进行降级。

**人工降级**：秒杀、双十一大促降级非重要的服务。 

##### 1、添加Maven依赖包

```
       <dependency>
            <groupId>io.github.openfeign</groupId>
            <artifactId>feign-hystrix</artifactId>
            <version>9.7.0</version>
        </dependency>
```

##### 2、添加属性配置

```
feign:
  hystrix:
    enabled: true

hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 15000
  threadpool:
    default:
      coreSize: 40
      maximumSize: 100
      maxQueueSize: 100
```

##### 3、添加降级策略和使用

```java
//FeignConfig 注入bean
@Bean
    public FeignFeignFallback fb() {
        return new FeignFeignFallback();
    }
//定义FeignFeignFallback,这里要继承feign接口,对远程调用对应的方法进行降级处理。
public class FeignFeignFallback implements FeignService {
    @Override
    public String searchList(String name) {
        return "调用searchList方法出错，参数为" +name;
    }
}
//对所有熔断前后进行统一处理。
@Component
public class FeignFactory implements FallbackFactory<FeignService> {
    private static final Logger LOGGER = LoggerFactory.getLogger(FeignFactory.class);
    private final FeignFeignFallback feignFallback;

    public FeignFactory(FeignFeignFallback feignFallback) {
        this.feignFallback = feignFallback;
    }
    @Override
    public FeignService create(Throwable throwable) {
        //打印下异常
        LOGGER.error("调用接口出错：", throwable);
        return feignFallback;
    }
}
//使用
@FeignClient(
        name = "service",
        url = "${feign.client.config.service.url}",
        configuration= FeignConfig.class,
        fallbackFactory = FeignFactory.class 
)
```

### 六、日志配置

 针对每一个Feign客户端，可以配置一个Logger.Level对象，通过该对象控制日志输出内容。
Logger.Level有如下几种选择：
NONE, 不记录日志 (**默认**)。
BASIC, 只记录请求方法和URL以及响应状态代码和执行时间。
HEADERS, 记录请求和应答的头的基本信息。
FULL, 记录请求和响应的头信息，正文和元数据。

```
//添加属性设置日志级别
logging:
  config: classpath:logback-spring.xml
  level:
    org.springframework.boot: info
    org.springframework.security: info
    com.example: debug
    
//FeignConfig 修改并注入
    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.HEADERS;
    }
    
//使用
@FeignClient(
        name = "service",
        url = "${feign.client.config.service.url}",
        configuration= FeignConfig.class,
        fallbackFactory = FeignFactory.class 
)
```

### 七、解码器（Decoder）

解码器作用于Response，用于解析Http请求的响应，提取有用信息数据。

将HTTP响应`feign.Response`解码为指定类型的**单一对象**。当然触发它也有前提：

1. 响应码是2xx
2. 方法返回值既不是void/null，也不是`feign.Response`类型

通常情况下，我们常使用json字符串进行传输。所以使用StringDecoder，比较多。

```
//定义自己的decoder方法
public class DemoDecoder implements Decoder {
    @Override
    public Object decode(Response response, Type type) throws IOException, DecodeException, FeignException {
        if (response.body() == null) {
            throw new DecodeException(response.status(), "没有返回有效的数据");
        }
        String bodyStr = Util.toString(response.body().asReader(Util.UTF_8));
        if (StringUtils.isEmpty(bodyStr)) {
            bodyStr = "";
        }
        //对结果进行转换
        FeignResult result = new FeignResult();
        FeignResult result = (FeignResult) JsonUtil.string2Obj(bodyStr, FeignResult.class);
        //如果返回错误，且为内部错误，则直接抛出异常
       if (result == null || !StatusConstant.API_CALL_SUCCESS.equals(result.getCode())) {
            throw new DecodeException(response.status(), "接口返回错误：" + result.getErrInfo() == null ? bodyStr : result.getErrInfo());
        }
        return result;
    }

```

此外也可以使用feign-jackson 或者feign-gson编码解码，返回JSONObject  对象、

```
<dependency>
	<groupId>io.github.openfeign</groupId>
	<artifactId>feign-gson</artifactId>
	<version>9.7.0</version>
</dependency>
<dependency>
	<groupId>io.github.openfeign</groupId>
	<artifactId>feign-jackson</artifactId>
	<version>9.5.0</version>
</dependency>

//使用，输出javabeen对象
PersonClient client2 = Feign.builder().decoder(new GsonDecoder())
		.target(PersonClient.class, "http://localhost:8080");

```



### 八、引用

[Java 开源项目 OpenFeign —— feign 的基本使用](https://www.cnblogs.com/siweipancc/p/12635294.html)

[原生Feign的解码器Decoder、ErrorDecoder](https://cloud.tencent.com/developer/article/1588501)

[Feign 官网](https://github.com/OpenFeign/feign/wiki/Custom-error-handling)

[Feign使用例子，比较完整](https://www.cnblogs.com/siweipancc/p/12635294.html)

[Feign 使用大全](https://www.jianshu.com/p/0834508b7a6d)