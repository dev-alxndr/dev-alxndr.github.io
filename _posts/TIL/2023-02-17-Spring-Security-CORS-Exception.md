---
title: Spring Security CORS Exception
tags: [spring, CORS]
categories: [TIL]
date: 2023-02-17 17:38:01
---

서비스에서 QA 중 CORS 오류가 발생

### Exception
```plainText
java.lang.IllegalArgumentException: When allowCredentials is true, allowedOrigins cannot contain the special
value “*” since that cannot be set on the “Access-Control-Allow-Origin” response header.
To allow credentials to a set of origins, list them explicitly or consider using “allowedOriginPatterns” instead
```

Spring Security 5.3 Version부터 `allowCredentials` 가 true 면 `allowedOrigins = *`이 될 수 없다. 특정 도메인으로 설정해줘야한다. 
갑자기 안된 이유는 기존 Spring boot 2.3 -> 2.7.6 버전으로 올리면서 Security Version이 올라갔다.

`setAllowedOrigin(CorsConfiguration.ALL)` -> `corsConfig.addAllowedOriginPattern(CorsConfiguration.ALL)`

-   [setAllowedOrigins](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/cors/CorsConfiguration.html#setAllowedOrigins(java.util.List))
-   [setAllowedOriginPatterns](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/cors/CorsConfiguration.html#setAllowedOriginPatterns(java.util.List))