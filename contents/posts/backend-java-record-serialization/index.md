---
title: "[Backend] 스프링 @Cacheable (Redis Cache) Java Record 역직렬화 문제 해결"
description: "벡엔드"
date: 2025-07-05
update: 2025-07-05
tags:
  - java
  - spring
series: "Backend Engineering"
---
> Could not read JSON:Unexpected token (START_OBJECT), expected START_ARRAY: need Array value to contain `As.WRAPPER_ARRAY` type information for class java.lang.Object

- record 를 이용해서 직렬화한 객체가 redis에 저장된 경우, 이를 역직렬화 하는 과정에서 위와 같은 오류 발생

- 역직렬화를 위한 타입 추론을 위해선, 직렬화시 해당하는 정보를 저장해 줘야 함 [참고](https://isaac1102.github.io/2021/04/28/jackson)

### DefaultTyping.OBJECT_AND_NON_CONCRETE
- jackson 의 디폴트 추론 정보 제공 옵션은 `ObjectMapper.DefaultTyping.OBJECT_AND_NON_CONCRETE` 이며, 이를 통해 직렬화된 캐시는 다음과 같은 형태.
```json
{
    "count": 10,
    "contents": [
        "java.util.ImmutableCollections$ListN",
        [
            {
                "content": "문 앞에 놔주세요 (벨 눌러주세요)"
            },
            {
                "content": "문 앞에 놔주세요 (노크해주세요)"
            },
        ]
    ]
}
```
- 이 정보로 역직렬화시 아래 오류가 발생
```
Could not read JSON:Unexpected token (START_OBJECT), expected START_ARRAY: need Array value to contain `As.WRAPPER_ARRAY` type information for class java.lang.Object
 at [Source: (byte[])"{"count":10,"contents":["java.util.ImmutableCollections$ListN",[{"content":"문 앞에 놔주세요 (벨 눌러주세요)"},{"content":"문 앞에 놔주세요 (노크해주세요)"},{"co"[truncated 82 bytes]; line: 1, column: 1]
 ```
#### 오류 발생 원인
```java
@JsonNaming(SnakeCaseStrategy.class)
public record RiderMessageResponse(
    @Schema(description = "요청 사항 수", example = "5")
    Integer count,
    @Schema(description = "배달 기사 요청 사항 목록")
    List<InnerRiderMessageResponse> contents)
{

    @JsonNaming(SnakeCaseStrategy.class)
    public record InnerRiderMessageResponse(
        @Schema(description = "배달 기사 요청 사항", example = "문 앞에 놔주세요 (노크해주세요)")
        String content
    ) {
        public static InnerRiderMessageResponse from(RiderMessage riderMessage) {
            return new InnerRiderMessageResponse(riderMessage.getContent());
        }
    }

    public static RiderMessageResponse from(List<RiderMessage> riderMessages) {
        return new RiderMessageResponse(riderMessages.size(),
            riderMessages.stream().map(InnerRiderMessageResponse::from).toList());
    }
}
```
- `OBJECT_AND_NON_CONCRETE`는 선언된 타입의 객체이거나 abstract 타입의 프로퍼티 정보를 제공하는데, Record는 final 클래스이면서 특별한 구조를 가지고 있어 이 정보 제공 범위에 포함되지 않음

- 저장된 JSON을 보면, contents 필드에는 java.util.ImmutableCollections$ListN 타입 정보만 포함되어 있고, 각 개별 Record 객체(InnerRiderMessageResponse)에 대한 타입 정보는 누락됨

- 이때문에 역직렬화 과정에서 Jackson은 List<InnerRiderMessageResponse>를 복원하려고 하지만, 개별 요소들의 정확한 타입 정보가 없어 Record 내부에서 생성되는 표준 생성자를 찾지 못함

### DefaultTyping.EVERYTHING
-  `ObjectMapper.DefaultTyping.EVERYTHING` 으로 설정하고 직렬화한 경우 다음과 같은 형태로 저장됨
```json
[
    "in.koreatech.koin.domain.order.delivery.dto.RiderMessageResponse",
    {
        "count": 10,
        "contents": [
            "java.util.ImmutableCollections$ListN",
            [
                [
                    "in.koreatech.koin.domain.order.delivery.dto.RiderMessageResponse$InnerRiderMessageResponse",
                    {
                        "content": "문 앞에 놔주세요 (벨 눌러주세요)"
                    }
                ],
                [
                    "in.koreatech.koin.domain.order.delivery.dto.RiderMessageResponse$InnerRiderMessageResponse",
                    {
                        "content": "문 앞에 놔주세요 (노크해주세요)"
                    }
                ]
            ]
        ]
    }
]
```
- 이 정보로 역직렬화시 문제가 발생하지 않음

#### 문제가 해결된 이유
- 모든 객체에 대해 정확한 타입 정보가 포함(최상위 객체 RiderMessageResponse, 내부 record InnerRiderMessageResponse)되서 Jackson이 Record의 표준 생성자를 올바르게 찾아서 사용 할 수 있는 것으로 보임

## 코드
  ```java
  @Configuration
@Profile("!test")
public class CacheConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        return RedisCacheManager.RedisCacheManagerBuilder.fromConnectionFactory(connectionFactory)
            .cacheDefaults(defaultConfiguration())
            .withInitialCacheConfigurations(customConfigurationMap())
            .build();
    }

    private RedisCacheConfiguration defaultConfiguration() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        mapper.disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);

        PolymorphicTypeValidator ptv = BasicPolymorphicTypeValidator.builder()
            .allowIfBaseType(Object.class)
            .build();
        mapper.activateDefaultTyping(ptv, ObjectMapper.DefaultTyping.EVERYTHING);

        return RedisCacheConfiguration.defaultCacheConfig()
            .serializeKeysWith(fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(fromSerializer(new GenericJackson2JsonRedisSerializer(mapper)))
            .entryTtl(Duration.ofMinutes(1));
    }

    private Map<String, RedisCacheConfiguration> customConfigurationMap() {
        Map<String, RedisCacheConfiguration> customConfigurationMap = new HashMap<>();

        customConfigurationMap.put(
            CacheKey.RIDER_MESSAGES.getCacheNames(),
            defaultConfiguration().entryTtl(Duration.ofMinutes(CacheKey.RIDER_MESSAGES.getTtl()))
        );

        return customConfigurationMap;
    }
}
  ```
