---
title: springboot 管理系统 jwt 认证
date: 2019-06-10 20:41:55
tags: 
  - spring
  - 管理系统
categories: springboot-管理系统
---

> 在基于 springboot 的管理系统中，将 jwt 集成在系统中，并作为系统前后端分离的认证方式。

<!-- more -->

## 什么是 jwt

jwt 全称为 json web token，它是一个开放标准，它是一种基于 json 格式，传递可以被验证和信任的信息。

jwt 由三个部分组成，这三个部分为

- header
- payload (body)
- signature

### Header

header 部分是一个 json 对象，描述 jwt 的元数据，通常是下面的样子

```json
{
  "alg":"HS256",
  "typ": "JWT"
}
```

alg 属性表示签名的算法，默认是 HMAC SHA256，typ 标识这个令牌的类型，JWT 令牌统一写 JWT。最后，用 Base64 URL 算法转换为字符串。

### Payload

payload 也是一个 json 对象，用来存放实际需要传递的数据， JWT 规定了7个官方字段

- iss (issuer) 签发人
- exp (expiration time) 过期时间
- sub (subject) 主题
- aud (audience) 受众
- nbf (Not before) 生效时间
- iat (Issued At) 签发时间
- jti (JWT ID) JWT 编号

除了上述的官方字段，你还可以自定义一些私有字段，通常 payload 部分只用 Base64 进行编码，一般不加密，所以私有字段不要存放敏感信息。

### Signature

signature 是对前两个部分进行签名，防止篡改。

## Java JWT

[JJWT][jjwt] 是用于 Android 和 Java 平台的 JSON Web Token。本系统采用了该包作为 JWT 认证，验证工具。

在 maven 中加入下面的依赖，来使用 JJWT

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.10.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.10.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.10.5</version>
    <scope>runtime</scope>
</dependency>
```

### 封装工具类

参考 jjwt 的使用手册，对 jjwt 进行封装，主要包括，创建 token ，认证，获取 token 中信息等功能。

```java
package github.beginner.noname.util;

import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;

@Slf4j
@Component
public class JwtUtils {
  /**
     * 验证jwt
     * @param jwsString jwt经过加密签名后称作jws(JSON Web Signature)
     * @return true 如果验证通过 false 验证失败
     */
    public static boolean verifyJwt(String jwsString) {
        boolean res =  false;
        try {
            Jws<Claims> claims =  Jwts.parser()
                    .setSigningKey(key)
                    .parseClaimsJws(jwsString);
            // 追加过期时间
            claims.getBody().setExpiration(new Date(System.currentTimeMillis() + configProperty.jwtExpiration));
            res = true;
        } catch (JwtException e) {
            log.info("jwt verify failed with jws: {}", jwsString);
        }
        return res;
    }

    /**
     * 解析jwt
     * @param jwsString 待解析的 jws
     * @return 解析后的结果
     */
    public static Jws<Claims> parseJwt(String jwsString) {
        return Jwts.parser()
                .setSigningKey(key)
                .parseClaimsJws(jwsString);
    }

    public static String buildJws(String loginName, Long id) {
        return Jwts.builder()
                .setHeaderParam("userId", id)
                .setSubject(loginName)
                .setExpiration(new Date(System.currentTimeMillis() + 失效时间))
                .signWith(key)
                .compact();
    }
}
```

上述代码涉及其他依赖：

- lombok
- spring

## spring MVC 集成 jwt 认证 （无框架）

在这里，出于学习的目的，没有使用 shiro 或者 spring security 等框架，而是自己才用相对原生（原始）的手段，来集成 jwt 认证，以便加深对 jwt 认证的理解。

我理解的 jwt 认证的方式大致是如下述图所示。

![jwt 认证过程](https://user-images.githubusercontent.com/21177719/59336117-845aca80-8d30-11e9-88a3-30821d08a890.png)

认证的核心就是要对所有的请求进行验证，查看是否存在 token 并且要验证 token 是否合法。

核心步骤如下：

1. 对所有的请求进行验证，这里就需要用到 spring MVC 的 web 拦截器，对请求进行拦截，查看请求是否携带 token。
2. 判断该请求是否需要验证 token （存在一些无需认证的 URL）， 如果需要验证，执行第三步， 若不需要，进行放行。
3. 使用封装好的 jjwt 工具对 token 进行验证，验证通过，放行请求，不通过，返回对应的响应结果。

spring mvc 使用拦截器对所有请求进行拦截，代码如下：

```java
package github.beginner.noname.config.web.interceptor;

import github.beginner.noname.annotation.NotCheckJwt;
import github.beginner.noname.constant.CommonConstant;
import github.beginner.noname.exception.JwtVerifyException;
import github.beginner.noname.util.JwtUtils;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.servlet.HandlerInterceptor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

/**
 * token 验证拦截器
 * @author zyp on 2019/2/19
 */
@Slf4j
@Component
public class JwtInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        // 如果不是映射到方法，直接通过，比如类级别的映射，按道理来说是错误地址
        if (!(handler instanceof HandlerMethod)) {
            return true;
        }
        HandlerMethod handlerMethod = (HandlerMethod) handler;
        // 获得class 级别的注解
        NotCheckJwt notCheckJwt = handlerMethod.getMethod().getAnnotation(NotCheckJwt.class);
        // 存在@NotCheckJwt注解，不需要验证jwt
        if (notCheckJwt != null) {
            return true;
        }

        String jws = request.getHeader(CommonConstant.USER_TOKEN);
        if (jws == null) {
            throw new JwtVerifyException("token is empty");
        }
        boolean res = JwtUtils.verifyJwt(jws);
        if (res) {
            return true;
        } else {
            throw new JwtVerifyException("access-deny");
        }
    }
}
```

上述代码，首先请求的路径上是否有 @NotCheckJwt 注解。

```java
package github.beginner.noname.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 加入此注解的method，无需验证jwt
 * @author zyp on 2019/2/19
 */
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface NotCheckJwt {}
```

如果存在该注解，则直接放行，说明对应的 URL 不需要验证 jwt。其次，从 header 中获取对应的 token 信息 （我这里把 token 放在了 header中），如果为空，抛出异常，如果不为空，进行验证，验证失败的话，抛出异常。这里抛出的异常交给了 spring mvc 的 HandlerExceptionResolver 来进行处理。

```java
package github.beginner.noname.config.web.exception;

import com.alibaba.fastjson.JSON;
import github.beginner.noname.constant.CodeConstant;
import github.beginner.noname.domain.dto.common.ResponseMsg;
import github.beginner.noname.exception.JwtVerifyException;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerExceptionResolver;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

/**
 * @author zyp on 2019/2/19
 */
@Slf4j
@Component
public class WebExceptionResolver implements HandlerExceptionResolver {
    @Override
    public ModelAndView resolveException(HttpServletRequest request,
                                         HttpServletResponse response, Object handler, Exception ex) {
        ResponseMsg responseMsg = new ResponseMsg();
        if (ex instanceof JwtVerifyException) {
            resolverJwtException(ex, responseMsg);
        } else {
            log.debug("请求时发生异常 {} ", ex.getMessage(), ex);
            ex.printStackTrace();
            resolverOtherException(ex, responseMsg);
        }

        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding("UTF-8");
        response.setHeader("Cache-Control", "no-cache, must-revalidate");
        try {
            response.getWriter().write(JSON.toJSONString(responseMsg));
        } catch (IOException e) {
            log.error("与客户端通讯异常：{}", e.getMessage(), e);
            e.printStackTrace();
        }
        return new ModelAndView();
    }

    private void resolverJwtException(Exception ex, ResponseMsg resMsg) {
        JwtVerifyException jwtVerifyException = (JwtVerifyException) ex;
        resMsg.setCode(CodeConstant.JWT_VERIFY_CODE);
        resMsg.setMsg(jwtVerifyException.getMsg());
    }

    private void resolverOtherException(Exception ex, ResponseMsg resMsg) {
        resMsg.setCode(CodeConstant.FAIL_CODE);
        resMsg.setMsg(ex.getMessage());
    }
}

```

这一步完成后，所以访问的 URL ，如果没有 @NotCheckJwt 注解，都会进行 token 验证。

[jjwt]: https://github.com/jwtk/jjwt
