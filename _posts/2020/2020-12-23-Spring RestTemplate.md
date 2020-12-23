---
layout:     post
title:      "RestTemplate 的使用"
subtitle:   " \" Spring RestTemplate\""
date:       2020-12-23 12:00:00
author:     "ysbao"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Java
    - Restful工具
---

> 千里之行，始于足下

### 前言

我们在项目中经常要使用第三方的 **Rest API** 服务，比如短信、快递查询、天气预报等等。这些第三方只要提供了 **Rest Api** ，你都可以使用 `RestTemplate` 来调用它们。

服务和服务之间的调用，通常分为webservice接口和http接口，目前比较流行的还有 RPC 和消息队列。http接口远程服务调用是基于Http协议。

Http协议是建立在TCP协议基础之上的，当浏览器需要从服务器获取网页数据的时候，会发出一次Http请求。Http会通过TCP建立起一个到服务器的连接通道，当本次请求需要的数据完毕后，Http会立即将TCP连接断开，这个过程是很短的。所以Http连接是一种短连接，是一种无状态的连接。

Java中，通常使用以下连接库。

* JDK 自带的 HttpURLConnection
* Apache 的 HttpClient
* OKHttp3

RestTemplate 是 Spring 提供的用于访问 Rest 服务的客户端库。它提供了一套接口，上述常用的连接库都实现了这个接口。所以支持包括HttpClient, OkHttp。在使用上更加方便和灵活。能够大大提高客户端的编写效率。

虽然 RestTemplate 是一个很不错的 HTTP Client，但 Netflix 已经开源了一个更好地 HTTP Client 工具 - Feign。它是一个声明式的 HTTP Client，在易用性、可读性等方面大幅领先于现有的工具。

### RestTemplae 简单使用

```java
package com.tcl.scm.common.utils;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.nio.charset.StandardCharsets;
import java.util.HashMap;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import com.alibaba.fastjson.JSONArray;
import com.alibaba.fastjson.JSONObject;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.tcl.scm.common.config.RestTemplateConfig;
import com.tcl.scm.common.exception.WorkflowExecption;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.*;
import org.springframework.http.client.ClientHttpResponse;
import org.springframework.http.client.SimpleClientHttpRequestFactory;
import org.springframework.http.converter.StringHttpMessageConverter;
import org.springframework.util.MultiValueMap;
import org.springframework.web.client.DefaultResponseErrorHandler;
import org.springframework.web.client.ResponseErrorHandler;
import org.springframework.web.client.RestClientException;
import org.springframework.web.client.RestTemplate;

public class RestTemplateUtils {
	public static Logger log = LoggerFactory.getLogger(RestTemplateUtils.class);
    // 默认时间五秒，不重试。
    private final static int CONNEC_TIMEOUT = 5000;
    private final static int READ_TIMEOUT   = 5000;
    private final static int RETRY_COUNT    = 1;

    /**
     * https 请求 GET
     * @param url           地址
     * @param connecTimeout 连接时间  毫秒
     * @param readTimeout   读取时间  毫秒
     * @param retryCount    重试机制
     * @return String 类型
     */
    public static String getHttp(String url, int connecTimeout, int readTimeout, int retryCount) {
        RestTemplate restTemplate = RestTemplateConfig.getRestTemplate();
        String result = null; // 返回值类型;
        for (int i = 1; i <= retryCount; i++) {
            try {
                result = restTemplate.getForObject(url, String.class);
                return result;
            } catch (RestClientException e) {
            	log.error("-----------开始-----------重试count: " + i);
                e.printStackTrace();
            }
        }
        return null;
    }

    /**
     * https 请求 GET
     * @param url           地址
     * @return String 类型
     */
    public static String getHttp(String url) {
        RestTemplate restTemplate = RestTemplateConfig.getRestTemplate();
        String result = null; // 返回值类型;
        for (int i = 1; i <= RETRY_COUNT; i++) {
            try {
                result = restTemplate.getForObject(url, String.class);
                return result;
            } catch (RestClientException e) {
                log.error("-----------开始-----------重试count: " + i);
                e.printStackTrace();
            }
        }
        return null;
    }

    /**
     * https 请求 PUT
     * @param url           地址
     * @return String 类型
     */
    public static String putHttp(String url,Map<String,Object> map) {
        try{
            RestTemplate restTemplate = RestTemplateConfig.getRestTemplate();
            String result = null; // 返回值类型;
            HttpHeaders header = new HttpHeaders();

            header.setContentType(MediaType.APPLICATION_JSON_UTF8);
            ObjectMapper mapper = new ObjectMapper();
            String value = mapper.writeValueAsString(map);
            HttpEntity<String> entity = new HttpEntity<String>(value,header);
            for (int i = 1; i <= RETRY_COUNT; i++) {
                try {
                    ResponseEntity<String > response = restTemplate.exchange(url.toString(), HttpMethod.PUT, entity, String.class);
                    result = response.getBody();
                    return result;
                } catch (RestClientException e) {
                    log.error("-----------开始-----------重试count: " + i);
                    e.printStackTrace();
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }
        return null;
    }


    /**
     * http 请求 post
     * @param url           地址
     * @param params        参数
     * @param headersMap    header
     * @param connecTimeout 连接时间
     * @param readTimeout   读取时间`
     * @param retryCount    重试机制
     * @return String 类型
     */
    public static String postHttp(String url, JSONObject params, Map<String,String> headersMap, int connecTimeout, int readTimeout, int retryCount) {
        RestTemplate restTemplate =  RestTemplateConfig.getRestTemplate();
        // 设置·header信息
        HttpHeaders requestHeaders = new HttpHeaders();
        requestHeaders.setAll(headersMap);
        HttpEntity<JSONObject> requestEntity = new HttpEntity<JSONObject>(params, requestHeaders); // josn utf-8 格式
        String result = null; // 返回值类型;
        for (int i = 1; i <= retryCount; i++) {
            try {
                result = restTemplate.postForObject(url, requestEntity, String.class);
                return result;
            } catch (RestClientException e) {
                log.error("-----------开始-----------重试count: " + i);
                e.printStackTrace();
            }
        }
        return null;
    }

    /**
     * https 请求 PUT
     * @param url           地址
     * @return String 类型
     */
    public static String putHttp(String url, JSONObject params) {
        try{
            RestTemplate restTemplate = RestTemplateConfig.getRestTemplate();
            String result = null; // 返回值类型;
            HttpHeaders requestHeaders = new HttpHeaders();
            requestHeaders.setContentType(MediaType.APPLICATION_JSON_UTF8);
            HttpEntity<JSONObject> requestEntity = new HttpEntity<JSONObject>(params, requestHeaders); // josn utf-8 格式
            for (int i = 1; i <= RETRY_COUNT; i++) {
                try {
                    ResponseEntity<String > response = restTemplate.exchange(url.toString(), HttpMethod.PUT, requestEntity, String.class);
                    result = response.getBody();
                    return result;
                } catch (RestClientException e) {
                    log.error("-----------开始-----------重试count: " + i);
                    e.printStackTrace();
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }
        return null;
    }

    /**
     * http 请求 post
     * @param url           地址
     * @param params        参数
     * @param headersMap    header
     * @return String 类型
     */
    public static String postHttp(String url, JSONObject params) {
        RestTemplate restTemplate = RestTemplateConfig.getRestTemplate();
        // 设置·header信息
        HttpHeaders requestHeaders = new HttpHeaders();
        HttpEntity<JSONObject> requestEntity = new HttpEntity<JSONObject>(params, requestHeaders); // josn utf-8 格式
        String result = null; // 返回值类型;
        for (int i = 1; i <= RETRY_COUNT; i++) {
            try {
                result = restTemplate.postForObject(url, requestEntity, String.class);
                return result;
            } catch (RestClientException e) {
                log.error("-----------开始-----------重试count: " + i);
                e.printStackTrace();
            }
        }
        return null;
    }

    /**
     * http 请求 post
     * @param url           地址
     * @param params        参数
     * @param headersMap    header
     * @return JSONArray 类型
     */
    public static String postHttp(String url, JSONArray jsonArray) {
        RestTemplate restTemplate = RestTemplateConfig.getRestTemplate();
        // 设置·header信息
        HttpHeaders requestHeaders = new HttpHeaders();
        HttpEntity<JSONArray> requestEntity = new HttpEntity<JSONArray>(jsonArray, requestHeaders); // josn utf-8 格式
        String result = null; // 返回值类型;
        for (int i = 1; i <= RETRY_COUNT; i++) {
            try {
                result = restTemplate.postForObject(url, requestEntity, String.class);
                return result;
            } catch (RestClientException e) {
                log.error("-----------开始-----------重试count: " + i);
                e.printStackTrace();
            }
        }
        return null;
    }

    /**
     * http 请求 post
     * @param url           地址
     * @param params        参数
     * @param headersMap    header
     * @return String 类型
     */
    public static String postHttp(String url, JSONObject params, Map<String,String> headersMap) {
        RestTemplate restTemplate = RestTemplateConfig.getRestTemplate();
        // 设置·header信息
        HttpHeaders requestHeaders = new HttpHeaders();
        requestHeaders.setAll(headersMap);
        HttpEntity<JSONObject> requestEntity = new HttpEntity<JSONObject>(params, requestHeaders); // josn utf-8 格式
        String result = null; // 返回值类型;
        for (int i = 1; i <= RETRY_COUNT; i++) {
            try {
                result = restTemplate.postForObject(url, requestEntity, String.class);
                return result;
            } catch (RestClientException e) {
                log.error("-----------开始-----------重试count: " + i);
                e.printStackTrace();
            }
        }
        return null;
    }


    /**
     * 加密参数类型请求  application/x-www-form-urlencoded
     * MultiValueMap<String, Object>
     * 采用 HttpEntity<MultiValueMap<String, Object>> 构造
     * http 请求 post
     *
     * @param url           地址
     * @param postParameters 参数
     * @param headersMap    header
     * @param connecTimeout 连接时间
     * @param readTimeout   读取时间
     * @param retryCount    重试机制
     * @return String 类型
     */
    public static String postHttpEncryption(String url, MultiValueMap<String, Object> postParameters, Map<String,String> headersMap, int connecTimeout, int readTimeout, int retryCount) {
        RestTemplate restTemplate = RestTemplateConfig.getRestTemplate();
        // 设置·header信息
        HttpHeaders requestHeaders = new HttpHeaders();
        requestHeaders.setAll(headersMap);
        HttpEntity<MultiValueMap<String, Object>> requestEntity = new HttpEntity<>(postParameters, requestHeaders);
        String result = null; // 返回值类型;
        for (int i = 1; i <= retryCount; i++) {
            try {
                result = restTemplate.postForObject(url, requestEntity, String.class);
                return result;
            } catch (RestClientException e) {
                log.error("-----------开始-----------重试count: " + i);
                e.printStackTrace();
            }
        }
        return null;
    }

    /**
     * 加密参数类型请求  application/x-www-form-urlencoded
     * MultiValueMap<String, Object>
     * 采用 HttpEntity<MultiValueMap<String, Object>> 构造
     * http 请求 post
     * @param url           地址
     * @param postParameters 参数
     * @param headersMap    header
     * @return String 类型
     */
    public static String postHttpEncryption(String url, MultiValueMap<String, Object> postParameters) {
        RestTemplate restTemplate = RestTemplateConfig.getRestTemplate();
        HttpEntity<MultiValueMap<String, Object>> requestEntity = new HttpEntity<>(postParameters);
        String result = null; // 返回值类型;
        for (int i = 1; i <= RETRY_COUNT; i++) {
            try {
                result = restTemplate.postForObject(url, requestEntity, String.class);
                return result;
            } catch (RestClientException e) {
                log.error("-----------开始-----------重试count: " + i);
                e.printStackTrace();
            }
        }
        return null;
    }
/**
* 获取restTemplate 这里使用SimpleClientHttpRequestFactory获取
*
**/
    private static RestTemplate simpeClient(String url, int connecTimeout, int readTimeout) {
       SimpleClientHttpRequestFactory requestFactory = new SimpleClientHttpRequestFactory();
        requestFactory.setConnectTimeout(connecTimeout);
        requestFactory.setReadTimeout(readTimeout);
        RestTemplate restTemplate = new RestTemplate(requestFactory);
        restTemplate.getMessageConverters().set(1, new StringHttpMessageConverter(StandardCharsets.UTF_8)); // 设置编码集
        restTemplate.setErrorHandler(new DefaultResponseErrorHandler()); //error处理
        return restTemplate;
    }

    /**
     * @ClassName: DefaultResponseErrorHandler
     * @Description: TODO
     * @author:
     * @date: 2
     */
    private static class DefaultResponseErrorHandler implements ResponseErrorHandler {

        /**
         * 对response进行判断，如果是异常情况，返回true
         */
        @Override
        public boolean hasError(ClientHttpResponse response) throws IOException {
            return response.getStatusCode().value() != HttpServletResponse.SC_OK;
        }

        /**
         * 异常情况时的处理方法
         */
        @Override
        public void handleError(ClientHttpResponse response) throws IOException {
            BufferedReader br = new BufferedReader(new InputStreamReader(response.getBody()));
            StringBuilder sb = new StringBuilder();
            String str = null;
            while ((str = br.readLine()) != null) {
                sb.append(str);
            }
            throw new WorkflowExecption(sb.toString(),"");
        }
    }

    /**
     * 
     * @Title: createHTTPURLParams()
     * @Description: TODO 构造参数，返回?<>&<> ... 封装URL将map转化为URL参数。
     * @param uriVariables
     * @Return String
     */
    public static String createHTTPURLParams(Map<String,Object> uriVariables) {
        StringBuffer params=new StringBuffer("?");
        for(String keyStr:uriVariables.keySet()) {
            params.append(keyStr+"="+uriVariables.get(keyStr)+"&");
        }
        return params.substring(0, params.length()-1);
    }
}

```

### RestTemplae 解决Https访问和Socket InteAddress 用尽的问题

HttpComponentsClientHttpRequestFactory 底层使用HttpClient访问远程HTTP服务请求，使用HTTPClient可以配置链接池。 配置成功后对于高频且短的大量Request请求，控制 Request的请求量，不会跑出IntAddress use out的异常。

 **在进行请求时，最好使用资源池管理Socket**；如果每次请求都创建Socket的话，  那么一定要记得请求完毕后释放Socket，可通过将Socket定义为一个Field，使用完毕后，进行释放。

```java
package com.tcl.scm.common.config;

import org.apache.http.client.HttpClient;
import org.apache.http.impl.client.DefaultConnectionKeepAliveStrategy;
import org.apache.http.impl.client.DefaultHttpRequestRetryHandler;
import org.apache.http.impl.client.HttpClientBuilder;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.impl.conn.PoolingHttpClientConnectionManager;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.web.client.DefaultResponseErrorHandler;
import org.springframework.web.client.RestTemplate;

import java.util.concurrent.TimeUnit;

public class RestTemplateConfig {

    private static RestTemplate restTemplate;

    static {
        // 长链接保持时间长度20秒
        PoolingHttpClientConnectionManager poolingHttpClientConnectionManager =
                new PoolingHttpClientConnectionManager(20, TimeUnit.SECONDS);
        // 设置最大链接数
        poolingHttpClientConnectionManager.setMaxTotal(2*getMaxCpuCore() + 3 );
        // 单路由的并发数
        poolingHttpClientConnectionManager.setDefaultMaxPerRoute(2*getMaxCpuCore());

        HttpClientBuilder httpClientBuilder = HttpClients.custom();
        httpClientBuilder.setConnectionManager(poolingHttpClientConnectionManager);

        // 重试次数3次，并开启
        httpClientBuilder.setRetryHandler(new DefaultHttpRequestRetryHandler(3,true));
        HttpClient httpClient = httpClientBuilder.build();
        // 保持长链接配置，keep-alive
        httpClientBuilder.setKeepAliveStrategy(new DefaultConnectionKeepAliveStrategy());

        HttpComponentsClientHttpRequestFactory httpComponentsClientHttpRequestFactory = new HttpComponentsClientHttpRequestFactory(httpClient);

        // 链接超时配置 5秒
        httpComponentsClientHttpRequestFactory.setConnectTimeout(5000);
        // 连接读取超时配置
//        httpComponentsClientHttpRequestFactory.setReadTimeout(10000);
        // 连接池不够用时候等待时间长度设置，分词那边 500毫秒 ，我们这边设置成1秒
        httpComponentsClientHttpRequestFactory.setConnectionRequestTimeout(3000);

        // 缓冲请求数据，POST大量数据，可以设定为true 我们这边机器比较内存较大
        httpComponentsClientHttpRequestFactory.setBufferRequestBody(true);

        restTemplate = new RestTemplate();
        restTemplate.setRequestFactory(httpComponentsClientHttpRequestFactory);
        restTemplate.setErrorHandler(new DefaultResponseErrorHandler());
    }

    public static RestTemplate getRestTemplate(){
        return restTemplate;
    }

    private static int getMaxCpuCore(){
        int cpuCore = Runtime.getRuntime().availableProcessors();
        return  cpuCore;
    }
}

```

### 引用

[restTemplate 发送Https请求](<https://blog.csdn.net/justry_deng/article/details/82531306>)。里边有Https请求样例

[restTemplate 工具类](<https://blog.csdn.net/qq_43607700/article/details/91361029>)

[restTemplate 的几种实现](<https://blog.csdn.net/qq_35981283/article/details/82056285?utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~all~first_rank_v2~rank_v25-3-82056285.nonecase>)，(总结的很好)

[高并发情况下Socket InteAddress用尽问题](https://blog.csdn.net/qq_27217017/article/details/98601893?utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~all~sobaiduend~default-1-98601893.nonecase&utm_term=resttemplate%E6%98%AF%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%E7%9A%84%E5%90%97&spm=1000.2123.3001.4430)，使用连接池管理。默认情况下使用SimpleClientHttpRequestFactory，访问量不高的时候使用。



