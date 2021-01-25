---
layout:     post
title:      "Reflect"
subtitle:   " \" Java Reflect\""
date:       2021-01-25 12:00:00
author:     "ysbao"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Java
---

> 千里之行，始于足下

### 简介

经期，由于工作的需要，根据对应的模板内容解析对应的属性值。并将获取的属性名和值持久化保存到数据库中。

研究后，可以采取三种方法进行对应的JavaBean的实例初始化。

* 利用java 的反射机制。
* ReflectASM，高性能的反射。
* 通过fastjson等工具，将对应的map,转成对应javaBean。（推荐）

### 什么是反射

反射(Reflection)是 Java 程序开发语言的特征之一，它允许运行中的 Java 程序获取自身的信息，并且可以操作类或对象的内部属性。

通过反射机制，可以在运行时访问 Java 对象的属性，方法，构造方法等。

### 反射的应用场景

反射的主要应用场景有：

- 开发通用框架 - 反射最重要的用途就是开发各种通用框架。很多框架（比如 Spring）都是配置化的（比如通过 XML 文件配置 JavaBean、Filter 等），为了保证框架的通用性，它们可能需要根据配置文件加载不同的对象或类，调用不同的方法，这个时候就必须用到反射——运行时动态加载需要加载的对象。
- 动态代理 - 在切面编程（AOP）中，需要拦截特定的方法，通常，会选择动态代理方式。这时，就需要反射技术来实现了。
- 注解 - 注解本身仅仅是起到标记作用，它需要利用反射机制，根据注解标记去调用注解解释器，执行行为。如果没有反射机制，注解并不比注释更有用。
- 可扩展××× - 应用程序可以通过使用完全限定名称创建可扩展性对象实例来使用外部的用户定义类。

### 反射的缺点

- 性能开销 - 由于反射涉及动态解析的类型，因此无法执行某些 Java 虚拟机优化。因此，反射操作的性能要比非反射操作的性能要差，应该在性能敏感的应用程序中频繁调用的代码段中避免。
- 破坏封装性 - 反射调用方法时可以忽略权限检查，因此可能会破坏封装性而导致安全问题。
- 内部曝光 - 由于反射允许代码执行在非反射代码中非法的操作，例如访问私有字段和方法，所以反射的使用可能会导致意想不到的副作用，这可能会导致代码功能失常并可能破坏可移植性。反射代码打破了抽象，因此可能会随着平台的升级而改变行为。

### ReflectASM，高性能的反射

ReflectASM 是一个非常小的 Java 类库，通过代码生成来提供高性能的反射处理，自动为 get/set 字段提供访问类，访问类使用字节码操作而不是 Java 的反射技术，因此非常快。

###  简单使用

```java
package com.tcl.tms.order.common;

import com.gexin.fastjson.JSONObject;
import com.tcl.tms.order.vo.PersonTest;

import java.lang.reflect.Method;
import java.math.BigDecimal;
import java.util.HashMap;
import java.util.Map;

public class ReflectUtil {

    /**
     * 根据属性，拿到set方法，并把值set到对象中
     *
     * @param obj       对象
     * @param clazz     对象的class
     * @param typeClass
     * @param value
     */
    public static void setValue(Object obj, Class<?> clazz, String filedName, Class<?> typeClass, Object value) {
        filedName = removeLine(filedName);
        String methodName = "set" + filedName.substring(0, 1).toUpperCase() + filedName.substring(1);
        try {
            Method method = clazz.getDeclaredMethod(methodName, new Class[]{typeClass});
            method.invoke(obj, new Object[]{getClassTypeValue(typeClass, value)});
        } catch (Exception ex) {
            ex.printStackTrace();
        }
    }

    /**
     * 通过class类型获取获取对应类型的值
     *
     * @param typeClass class类型
     * @param value     值
     * @return Object
     */
    private static Object getClassTypeValue(Class<?> typeClass, Object value) {
        if (typeClass == int.class || value instanceof Integer) {
            if (null == value) {
                return 0;
            }
            return value;
        } else if (typeClass == short.class) {
            if (null == value) {
                return 0;
            }
            return value;
        } else if (typeClass == byte.class) {
            if (null == value) {
                return 0;
            }
            return value;
        } else if (typeClass == double.class) {
            if (null == value) {
                return 0;
            }
            return value;
        } else if (typeClass == long.class) {
            if (null == value) {
                return 0;
            }
            return value;
        } else if (typeClass == String.class) {
            if (null == value) {
                return "";
            }
            return value;
        } else if (typeClass == boolean.class) {
            if (null == value) {
                return true;
            }
            return value;
        } else if (typeClass == BigDecimal.class) {
            if (null == value) {
                return new BigDecimal(0);
            }
            return new BigDecimal(value + "");
        } else {
            return typeClass.cast(value);
        }
    }

    /**
     * 处理字符串  如：  abc_dex ---> abcDex
     *
     * @param str
     * @return
     */
    public static String removeLine(String str) {
        if (null != str && str.contains("_")) {
            int i = str.indexOf("_");
            char ch = str.charAt(i + 1);
            char newCh = (ch + "").substring(0, 1).toUpperCase().toCharArray()[0];
            String newStr = str.replace(str.charAt(i + 1), newCh);
            String newStr2 = newStr.replace("_", "");
            return newStr2;
        }
        return str;
    }

    /**
     * 根据属性，获取get方法
     *
     * @param ob   对象
     * @param name 属性名
     * @return
     * @throws Exception
     */
    public static Object getGetMethod(Object ob, String name) throws Exception {
        Method[] m = ob.getClass().getMethods();
        for (int i = 0; i < m.length; i++) {
            if (("get" + name).toLowerCase().equals(m[i].getName().toLowerCase())) {
                return m[i].invoke(ob);
            }
        }
        return null;
    }


    /**
     * 根据属性，获取get方法
     *
     * @param ob   对象
     * @param name 属性名
     * @return
     * @throws Exception
     */
    public static Object getAsmMethod(Object ob,MethodAccess access,Object args,String name) throws Exception {
        String methodName = ("get" + name).toLowerCase();
        Object result = access.invoke(ob, "getName", args);
        return result;
    }

    /**
     * 根据属性，设置set方法
     *
     * @param ob   对象
     * @param name 属性名
     * @return
     * @throws Exception
     */
    public static Object setAsmMethod( MethodAccess access, String name) throws Exception {
        Method[] m = ob.getClass().getMethods();
        for (int i = 0; i < m.length; i++) {
            if (("get" + name).toLowerCase().equals(m[i].getName().toLowerCase())) {
                return m[i].invoke(ob);
            }
        }
        return null;
    }


    public static void main(String[] args) throws Exception {

        /**
         * 1、使用java 反射
         */
        //test removeLine
        System.out.println(removeLine("abg_dwr"));
        System.out.println(removeLine("ab_wr"));
        System.out.println(removeLine("abgwr"));

        PersonTest person1 = new PersonTest();
        person1.setSchool("惠州市一中");
        person1.setName("旺旺");
        Object ob = getGetMethod(person1, "name");
        System.out.println(ob);

        //test set
        PersonTest person2 = new PersonTest();
        String field2 = "name";
        setValue(person2, person2.getClass(), field2, PersonTest.class.getDeclaredField(field2).getType(), "汪汪");
        System.out.println(person2.getName());
        //获取某个属性的类型
        System.out.println(PersonTest.class.getDeclaredField("name").getType());

        /**
         * 2、使用ReflectASM 方法 java 反射的优化
         */
        PersonTest someObject = new PersonTest();
        MethodAccess access = MethodAccess.get(PersonTest.class);
        access.invoke(someObject, "setName","526");
        //invoke getName方法 获得值
        String name = (String)access.invoke(someObject, "getName", null);
        //String name = (String)getAsmMethod(someObject,access,null,"name");
        System.out.println(name);


        /**
         * 3、map 转javabean 这个快
         */
        Map<String, Object> map= new HashMap<>();
        map.put("name", "tom");
        map.put("age", 23);
       // map.put("school", "华罗庚中学");
        PersonTest result = JSONObject.parseObject(JSONObject.toJSONString(map), PersonTest.class);
        System.out.println(result.getSchool());
        System.out.println(result.getSky());
    }
}

```

### 引用

[深入理解反射](<https://www.jianshu.com/p/efe1fea3e2f7>)。总结的很好

[ReflectASM的官网](https://www.oschina.net/p/reflectasm)



