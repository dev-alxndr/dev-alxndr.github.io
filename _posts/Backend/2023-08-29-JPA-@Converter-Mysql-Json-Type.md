---
title: JPA @Converter Mysql Json Type
tags: [JPA, Converter, Mysql]
categories: [Backend]
date: 2023-08-29 21:08:06
---

Mysql Json 컬럼을 JPA Entity Field에서 객체로 변환하는 방법 

찾아보니 대표적으로 2가지 방법
1. [GitHub - vladmihalcea/hypersistence-utils](https://github.com/vladmihalcea/hypersistence-utils)
2. `@Converter`

`@Converter`로 하면 의존성 추가 없이 가능하고 Json 형식 자체는 고정이기 때문에 2번째 방법으로 진행

> 첫번째 방법은 Maven Repository에 취약점 경고?가 있어서 괜히 쓰기싫은것도 있음

Mysql에는 Json Data Type을
JPA Entity에 맵핑할 때 String으로 맵핑하지 않고 객체로 맵핑하는 방법이다.

1. `AttributeConverter` 구현
```java
@Slf4j  
@Converter  
@RequiredArgsConstructor  
public class ExampleConverter implements AttributeConverter<Map<String, String>, String> {  // Map<String, String> = 전환하고자 하는 객체

	// jackson
	private final ObjectMapper objectMapper;  

	// Object to Json String
	@Override
	public String convertToDatabaseColumn(final List<Method> attribute) {  
		if (CollectionUtils.isEmpty(attribute)) {  
			return null;  
		} 
		
		try {  
			return objectMapper.writeValueAsString(attribute);  
		} catch (JsonProcessingException e) {  
			log.error("DB 컬럼 변환 중 에러");  
			throw new RuntimeException(e);  
		}  
	}  

	// DB Data to Object
	@Override  
	public Map<String, String> convertToEntityAttribute(final String dbData) {  
  
		if (!StringUtils.hasText(dbData)) {  
		    return null;  
		}
		try {  
			return objectMapper.readValue(dbData, new TypeReference<>() {});  
		} catch (JsonProcessingException e) {  
		    log.error("Entity 변환 중 에러");  
		    throw new RuntimeException(e);  
		}  
   }  
}
```

2. Entity Field Converter 등록
```java
@Entity  
public class Example { 

	@Id
	private Long id;

	@Convert(conveter = ExampleConverter.class)
	@Column(name = "json_field")
	private Map<String, String> jsonField;
}
```

JPA Select를 하게되면 자동으로 Json을 원하는 타입으로 컨버팅해주게 된다.

> The Convert annotation should not be used to specify conversion of the following: Id attributes, version attributes, relationship attributes, and attributes explicitly denoted as Enumerated or Temporal
> Convert 주석은 Id 속성, 버전 속성, 관계 속성 및 명시적으로 열거 또는 임시로 표시된 속성의 변환을 지정하는 데 사용되어서는 안 됩니다. 이러한 변환을 지정하는 애플리케이션은 이식 가능하지 않습니다.


---
### References
1. [엔티티의 필드 타입으로 JSON 사용하기](https://pika-chu.tistory.com/1296)
2. [JPA @Converter](https://cherrypick.co.kr/jpa-converter/) / [JPA에 JSON 컬럼을 제네릭으로 유연하게 매핑하기](https://velog.io/@hong1008/jpa-json-converter)
