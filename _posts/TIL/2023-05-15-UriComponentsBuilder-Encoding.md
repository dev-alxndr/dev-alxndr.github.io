---
title: UriComponentsBuilder Encoding
tags: [TIL]
categories: [TIL]
date: 2023-05-15 18:39:43
---

A 서비스에서 B 서비스로 부터 특정 행동을 할 수 있는 URL을 받아서   
다시 B 서비스로 요청을 하는 로직을 개발하던 동료분의 에러를 찾다가 발견한 이슈..?

#### 예시
- B-서비스로 부터 받은 URL
	- `https://example.com?query=i-am-te%2Fster`

- 다시 요청한 URL
	- `https://example.com?query=i-am-te%252Fster`

예시는 짧은 URL이지만 실제 요청은 1482자로 이루어진 URL이였다.

요청을 보낼 때는 `UriComponentsBuilder`을 사용해서 URI를 만들었다.

```java
restTemplate.exchange(  
  builder.build().toUri(),  
  httpMethod,  
  httpEntity,  
  Void.class);
```

#### 문제 접근
받은 URL을 그대로 브라우저에 치면 정상적으로 동작하고 있었기 때문에 받은 URL과 요청한 URL 뭔가 달라서 문제가 발생하고 있다고 생각을 했다.   
그래서 URL을 비교해서 보기 시작했고 `%2F`와 `%252F`가 다르다는걸 확인했다.   
왜 저렇게 바꼈는지 코드를 확인해본다.   

- `UriComponentsBuilder::build`
![](Screenshot%202023-05-15%20at%2018.49.36.png)
	기존에 호출한 `.build()` 코드를 보면 `build(false)`를 호출하고 있다.

- `UriComponentsBuilder::build(boolean encoded)`
![](Screenshot%202023-05-15%20at%2018.50.08.png)
	주석을 읽어보면   
	`encoded – whether all the components set in this builder are encoded (true) or not (false)` 
	해당 URL이 Encoding이 되어있는지 안되어있는지를 알려주는 파라미터다.

#### 해결
문제는 이미 인코딩 되어있는 URL을 다시 한번 인코딩을 하면서 `%2F` -> %252F가 되어 버린 것이다.   
> `%`를 인코딩하면 `%25`가 된다.
> [URL Encoding Reference](https://www.w3schools.com/tags/ref_urlencode.ASP)

```java
  builder.build(true).toUri(),  
```
`.build(true)` 이렇게 수정해서 이미 인코딩된 URL임을 지정해줬더니 잘 됐다.
동료는 웃을 수 있었다.