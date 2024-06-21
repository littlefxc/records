---
title: SpringBoot数据脱敏
status: Done
tags:
  - Java
  - Spring
createDate: 2023-06-29
---

---

## 1 前言

数据脱敏是一种常见的数据安全保护技术，可以在保护数据隐私的同时，保持数据的有效性和可用性。在 Spring Boot 中，我们可以使用注解的方式实现数据脱敏，使代码更加简洁、易于维护和扩展。

本文章将介绍使用注解的方式实现，在返回数据给前端的时候，根据给定的脱敏规则实现敏感数据脱敏操作，实现步骤非常简单。

主要利用的就是 SpringBoot 默认的序列化工具 Jackson。

## 2 依赖

Hutool是一个小而全的Java工具类库，内含信息脱敏工具-DesensitizedUtil，这里我们需要用到DesensitizedUtil。还不了解Hutool工具类的朋友们可以看看官方文档 [https://hutool.cn/docs](https://hutool.cn/docs/)

```xml
<!-- hutool工具类 -->
<dependency>
    <groupId>cn.hutool</groupId>
    <artifactId>hutool-all</artifactId>
    <version>5.8.16</version>
</dependency>
```

## 3 自定义注解

```java
import com.fasterxml.jackson.annotation.JacksonAnnotationsInside;
import com.fasterxml.jackson.databind.annotation.JsonSerialize;
import com.poi.desensitize.enums.SensitizeRuleEnums;
import com.poi.desensitize.serializer.SensitiveJsonSerializer;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * @author fxc
 * @description 数据脱敏注解
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@JacksonAnnotationsInside
@JsonSerialize(using = SensitiveJsonSerializer.class)
public @interface Sensitize {
  SensitizeRuleEnums rule();
}

```

## 4 脱敏规则枚举类

这里的部分脱敏规则都是依赖Hutool的DesensitizedUtil工具类，同时也可以自行定义脱敏规则

```java
import cn.hutool.core.util.DesensitizedUtil;
import java.util.function.Function;

/**
 * @author hzd
 * @description 数据脱敏枚举
 */
public enum SensitizeRuleEnums {
  /**
   * 用户id脱敏
   */
  USER_ID(s -> String.valueOf(DesensitizedUtil.userId())),

  /**
   * 中文姓名脱敏
   */
  CHINESE_NAME(DesensitizedUtil::chineseName),

  /**
   * 身份证脱敏
   */
  ID_CARD(s -> DesensitizedUtil.idCardNum(s, 3, 4)),

  /**
   * 固定电话
   */
  FIXED_PHONE(DesensitizedUtil::fixedPhone),

  /**
   * 手机号脱敏
   */
  MOBILE_PHONE(DesensitizedUtil::mobilePhone),

  /**
   * 地址脱敏
   */
  ADDRESS(s -> DesensitizedUtil.address(s, 8)),

  /**
   * 电子邮箱脱敏
   */
  EMAIL(DesensitizedUtil::email),

  /**
   * 密码脱敏
   */
  PASSWORD(DesensitizedUtil::password),

  /**
   * 中国车牌脱敏
   */
  CAR_LICENSE(DesensitizedUtil::carLicense),

  /**
   * 银行卡脱敏
   */
  BANK_CARD(DesensitizedUtil::bankCard);

  /**
   * 可自行添加其他脱敏策略.....
   */
  private final Function<String, String> sensitize;

  public Function<String, String> sensitize() {
	return sensitize;
  }

  SensitizeRuleEnums(Function<String, String> sensitize) {
    this.sensitize = sensitize;
  }
}

```

## 5 数据脱敏 JSON 序列化类

SpringBoot 中默认使用 jackson 作为 JSON 序列化工具，我们只要在数据返回前台的序列化过程中获取到我们自定义注解(Sensitize) ,对有该注解的字段实现脱敏操作，这样前台看到的数据就是脱敏的啦

```java

import cn.hutool.core.util.DesensitizedUtil;
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.databind.BeanProperty;
import com.fasterxml.jackson.databind.JsonMappingException;
import com.fasterxml.jackson.databind.JsonSerializer;
import com.fasterxml.jackson.databind.SerializerProvider;
import com.fasterxml.jackson.databind.ser.ContextualSerializer;
import com.poi.desensitize.annotation.Sensitize;
import com.poi.desensitize.enums.SensitizeRuleEnums;

import java.io.IOException;
import java.util.Objects;

/**
 * @author hzd
 * @description 序列化类作用：对返回前台的数据进行脱敏
 */
public class SensitiveJsonSerializer extends JsonSerializer<String> implements ContextualSerializer {

  private SensitizeRuleEnums rule;

  @Override
  public void serialize(String value, JsonGenerator jsonGenerator, SerializerProvider serializerProvider) throws IOException {
    jsonGenerator.writeString(rule.sensitize().apply(value));
  }

  @Override
  public JsonSerializer<?> createContextual(SerializerProvider serializerProvider, BeanProperty beanProperty) throws JsonMappingException {
    Sensitize annotation = beanProperty.getAnnotation(Sensitize.class);
    if (Objects.nonNull(annotation) && Objects.equals(String.class, beanProperty.getType().getRawClass())) {
      this.rule = annotation.rule();
      return this;
    }
    return null;
  }
} 

```