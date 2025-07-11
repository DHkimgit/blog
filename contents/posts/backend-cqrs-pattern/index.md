---
title: "[Backend] CQRS 패턴의 소개와 간단한 적용 사례"
description: "벡엔드"
date: 2025-07-10
update: 2025-07-10
tags:
  - java
  - spring
  - switch-wiki
series: "Backend Engineering"
---

## CQRS (Command Query Responsibility Segregation)

모든 소프트웨어 시스템은 상태를 변경하는 작업과 상태를 조회하는 작업으로 나눌 수 있다. 단순한 시스템에서는 이 두 가지 작업을 하나의 모델과 데이터 구조로 처리하지만, 도메인이 복잡해지면 상태를 변경할 때 필요한 데이터와 조회할 때 필요한 데이터가 달라지기 시작한다.

하나의 모델로 명령(Command)과 조회(Query)를 모두 처리하려고 하면, 모델은 양쪽의 요구사항을 모두 수용해야 하므로 점차 비대해지고 복잡해진다. 예를 들어, 상태를 변경하는 로직을 분석하는데 조회에만 사용되는 필드들이 섞여 있다면 코드를 이해하기 어렵고 유지보수 비용을 증가시킨다.

**CQRS (Command Query Responsibility Segregation)** 패턴은 이런 문제를 해결하기 위해 시스템의 상태를 변경하는 **명령 모델**과 상태를 조회하는 **조회 모델**을 명시적으로 분리하는 접근 방식이다.

### 이점
CQRS를 적용하면 각 모델을 해당 기능에 최적화할 수 있다. 명령 모델은 데이터의 일관성과 비즈니스 규칙을 적용하는 데 집중하고, 조회 모델은 데이터를 효율적으로 읽어오는 데 집중한다. 이로써 한쪽의 변경이 다른 쪽에 미치는 영향을 최소화할 수 있다.

또한, 조회 기능의 성능을 독립적으로 최적화하기 용이하다. 이 프로젝트에서는 조회 전용 DTO를 사용하고, 필요에 따라 캐시를 적용하거나 읽기 전용 복제 데이터베이스를 사용하는 등의 전략을 추가로 고려할 수 있다.

### 단점
모델을 분리하면 작성해야 할 코드의 양이 늘어난다. 따라서 CQRS는 모델이 충분히 복잡하여 코드 분리로 얻는 이점이 추가 작업량을 상회할 때 가장 효과적이다. 또한, 조회 모델을 위해 별도의 기술을 도입하면 시스템 간 데이터 동기화를 위한 추가적인 구현이 필요할 수 있다.

## CQRS in switch-wiki

이 프로젝트는 `SwitchRating` 도메인에 CQRS 패턴을 적용하여 명령과 조회의 책임을 분리했다.

**1. 명령 (Command) 측면**

상태를 변경하는 책임은 `SwitchRatingService`가 담당한다. 이 서비스는 스위치 평점을 생성(`create`), 수정(`update`), 삭제(`delete`)하는 기능을 제공하며, 내부적으로 각 비즈니스 로직을 처리하는 UseCase를 호출한다.

```kotlin
// src/main/kotlin/wiki/devtab/switches/domain/rating/service/SwitchRatingService.kt
@Service
class SwitchRatingService (
    private val switchRatingCreateUseCase: SwitchRatingCreateUseCase,
    private val switchRatingUpdateUseCase: SwitchRatingUpdateUseCase,
    private val switchRatingDeleteUseCase: SwitchRatingDeleteUseCase
) {
    // ...
}
```

**2. 조회 (Query) 측면**

상태를 조회하는 책임은 `SwitchRatingQueryService`가 담당한다. 이 서비스는 `@Transactional(readOnly = true)` 속성을 통해 읽기 전용 트랜잭션으로 동작하며, 데이터를 조회하는 기능만을 제공한다.

```kotlin
// src/main/kotlin/wiki/devtab/switches/domain/rating/service/SwitchRatingQueryService.kt
@Service
class SwitchRatingQueryService (
    private val switchRatingRepository: SwitchRatingRepository,
) {
    @Transactional(readOnly = true)
    fun readUserRating(switchId: Long, userId: Long): UserSwitchRatingResponse? { ... }

    @Transactional(readOnly = true)
    fun readRatingAvg(switchId: Long): SwitchRatingResponse? { ... }
}
```

특히, `readRatingAvg` 메서드는 복잡한 집계 쿼리의 결과를 처리하기 위해 별도의 읽기 전용 모델인 `SwitchRatingScore` DTO를 사용한다.

```kotlin
// src/main/kotlin/wiki/devtab/switches/domain/rating/repository/SwitchRatingScore.kt
data class SwitchRatingScore(
    val totalCount: Long?,
    val averageContactRating: Double?,
    // ...
)
```

이 DTO는 `JDSLSwitchRatingRepositoryImpl`에서 Kotlin JDSL을 통해 직접 계산된 집계 결과(평균, 개수 등)를 담기 위해 설계되었다.

영속성 컨텍스트가 관리하는 `SwitchRating` 엔티티를 모두 불러와 애플리케이션 레벨에서 계산하는 대신, 데이터베이스에서 모든 계산을 완료하고 그 결과만을 DTO로 받아오는 방식을 사용함으로써 조회 성능을 최적화 했다.

```kotlin
// src/main/kotlin/wiki/devtab/switches/domain/rating/repository/JDSLSwitchRatingRepositoryImpl.kt
override fun findSwitchRatingScore(switchId: Long): SwitchRatingScore? {
    return kotlinJdslJpqlExecutor.findAll {
        select<SwitchRatingScore>(
            count(entity(SwitchRating::class)),
            avg(path(SwitchRating::contactRating)),
            // ...
        ).from(
            entity((SwitchRating::class))
        ).where(
            path(SwitchRating::switch)(Switch::id).eq(switchId)
        )
    }.singleOrNull()
}
```