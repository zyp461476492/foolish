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

[jjwt]: https://github.com/jwtk/jjwt
