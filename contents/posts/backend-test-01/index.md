---
title: "[Backend] 테스트 대역의 5가지 유형"
description: "벡엔드"
date: 2025-08-04
update: 2025-08-04
tags:
  - java
  - spring
  - switch-wiki
series: "Backend Engineering"
---

### 테스트와 격리

단위 테스트(Unit Test)의 핵심 원칙 중 하나는 **'격리(Isolation)'** 이다. 테스트 대상 시스템(SUT, System Under Test)을 외부 의존성으로부터 분리하여, 오직 SUT의 동작만을 독립적으로 검증하는 것을 목표로 한다. 하지만 대부분의 객체는 데이터베이스, 외부 API, 다른 서비스 등 다양한 의존 컴포넌트(DOC, Depended-on Component)와 상호작용한다.

이러한 외부 의존성은 테스트를 느리고, 비결정적이며, 복잡하게 만든다. 이 문제를 해결하기 위해 실제 의존 객체를 대체하는 가짜 객체를 사용하는데, 이를 **테스트 대역(Test Double)** 이라고 한다. 테스트 대역이라는 용어는 영화의 스턴트 더블(Stunt Double)에서 유래했으며, 테스트 환경에서 실제 객체를 대신하는 모든 종류의 가짜 객체를 통칭한다. 마틴 파울러는 이 테스트 대역을 목적과 구현 방식에 따라 5가지 유형으로 분류하였다.

### 1. Dummy (더미)

*   **정의**: 가장 간단하고 목적이 뚜렷한 대역이다. Dummy 객체는 아무런 동작도 하지 않으며, 오롯이 코드가 정상적으로 실행되도록 경로를 열어주는 역할만 한다.

  
* **목적**: 메서드 시그니처를 만족시키기 위해 인자로 전달되거나, 테스트의 관심사가 아닌 협력 객체의 자리를 채우기 위해 사용된다. `null`을 전달할 경우 `NullPointerException`이 발생할 수 있는 상황에서, 아무런 동작도 하지 않는 Dummy 객체를 전달하여 문법적 요구사항을 충족시키는 것이 주된 용도다.

### 2. Stub (스텁)

*   **정의**: 테스트 중에 만들어진 호출에 대해 미리 준비된 답변(Canned Answer)을 제공하도록 설정된 객체다. Stub은 실제 객체의 응답을 최대한 비슷하게 모방하여, 테스트가 원하는 방향으로 동작하도록 만든다.

  
* **목적**: 실제 의존 객체를 호출할 때 발생하는 네트워크 I/O, 데이터베이스 조회 같은 고비용(expensive) 작업을 피하고, 테스트가 특정 경로로 실행되도록 SUT의 상태를 제어하는 데 주 목적이 있다. Stub은 주로 **상태 검증(State Verification)**, 즉 메서드 실행 후 SUT의 상태가 기대하는 대로 변경되었는지 확인할 때 사용된다.

Mockito 프레임워크의 `when(...).thenReturn(...)` 구문은 Stub을 생성하는 대표적인 방법이다.

아래 `LostItemChatRoomInfoService` 테스트에서 `chatRoomInfoRepository`는 Stub으로 동작한다.

```java
// LostItemChatRoomInfoServiceTest.java
@Test
void 기존에_생성된_채팅방이_있으면_해당_채팅방_ID를_반환한다() {
    // given
    LostItemChatRoomInfoEntity chatRoomInfo = LostItemChatFixture.분실물_게시글_채팅방(articleId, 77, authorId, messageSenderId);

    // chatRoomInfoRepository.findByArticleIdAndMessageSenderId(...)가 호출될 경우,
    // 미리 정의된 Optional.of(chatRoomInfo)를 반환하도록 Stub을 설정한다.
    when(chatRoomInfoRepository.findByArticleIdAndMessageSenderId(articleId, messageSenderId))
        .thenReturn(Optional.of(chatRoomInfo));

    // when
    Integer chatRoomId = lostItemChatRoomInfoService.getOrCreateChatRoomId(articleId, messageSenderId);

    // then - 상태 검증 (State-based Verification)
    // getOrCreateChatRoomId 메서드의 반환 값(상태)이 Stub이 제공한 77인지 확인한다.
    assertThat(chatRoomId).isEqualTo(77);
}
```
이 테스트는 `getOrCreateChatRoomId` 메서드의 반환 값을 검증하는 것이 목적이다. 이를 위해 `chatRoomInfoRepository`가 특정 값을 반환하도록 Stub을 통해 테스트 대상 메서드의 실행 흐름을 제어한다.

Mockito를 사용하지 않는다면, 다음과 같이 구현할 수 있다.

```java
// StubChatRoomInfoRepository.java
class StubChatRoomInfoRepository implement ChatRoomInfoRepository {
    
    public Optional<LostItemChatRoomInfoEntity> findByArticleIdAndMessageSenderId(Integer articleId, Integer messageSenderId) {
        return Optional.of(
            LostItemChatFixture.분실물_게시글_채팅방(articleId, 77, authorId, messageSenderId);
        )
    }
}

// LostItemChatRoomInfoServiceTest.java
@Test
void 기존에_생성된_채팅방이_있으면_해당_채팅방_ID를_반환한다() {
    // when
    LostItemChatRoomInfoService lostItemChatRoomInfoService = LostItemChatRoomInfoService.builder()
        .chatRoomInfoRepository(new StubChatRoomInfoRepository())
        .build();
    
    Integer chatRoomId = lostItemChatRoomInfoService.getOrCreateChatRoomId(articleId, messageSenderId);

    // then - 상태 검증 (State-based Verification)
    // getOrCreateChatRoomId 메서드의 반환 값(상태)이 Stub이 제공한 77인지 확인한다.
    assertThat(chatRoomId).isEqualTo(77);
}
```

### 3. Fake (페이크)

* **정의**: 단순히 값을 반환하는 Stub에서 한 단계 더 나아가, 실제 로직을 가지고 동작하는 구현체다. 물론 프로덕션 코드만큼 복잡하지는 않으며, 데이터베이스를 `HashMap`으로 대체하는 In-Memory Repository처럼 테스트 목적에 맞게 핵심 기능이 간소화되어 있다.

  
* **목적**:여러 테스트 케이스에서 재사용 가능한, 실제와 유사한 동작을 하는 경량 의존성을 제공하는 것이다. Fake 객체를 사용하면 테스트마다 Stub을 설정하는 번거로움을 줄이고 테스트 코드 자체를 간결하게 유지할 수 있다.


* **상세**: Fake는 보통 특정 인터페이스를 구현하는 별도의 클래스로 직접 작성된다. 이는 여러 테스트에서 일관된 동작을 보장하고 재사용성을 높인다. 예를 들어, `FakeUserRepository`는 `save` 시 ID를 부여하고, `findById` 시 내부 `Map`에서 데이터를 조회하는 등 실제 DB의 동작을 흉내 낸다.

```java
// Fake는 실제 인터페이스를 구현하는 클래스로 만든다.
public class FakeUserRepository implements UserRepository {
    private final Map<Long, User> data = new ConcurrentHashMap<>();
    private final AtomicLong sequence = new AtomicLong(0);

    @Override
    public User save(User user) {
        if (user.getId() == null) {
            user.setId(sequence.incrementAndGet());
        }
        data.put(user.getId(), user);
        return user;
    }

    @Override
    public Optional<User> findById(Long id) {
        return Optional.ofNullable(data.get(id));
    }
    // ...
}
```

### 4. Mock (목)

*   **정의**: Mock 객체는 호출에 대한 기대 명세를 프로그래밍하고, SUT로부터의 호출을 기록하여 테스트가 종료된 후 이를 검증하는 객체다. Mock을 제대로 이해하려면 테스트 검증의 두 가지 접근법인 **상태 기반 검증**과 **행위 기반 검증**을 먼저 알아야 한다.

#### 상태 기반 검증 (State-based Verification)
상태 기반 검증은 메서드 실행 후 객체의 상태가 기대하는 대로 변경되었는지 확인하는 방식이다. 앞서 Stub 예제에서 `getOrCreateChatRoomId` 메서드를 호출한 뒤, 반환된 `chatRoomId`가 `77`인지 `assertThat`으로 검증한 것이 바로 상태 기반 검증이다. SUT의 최종 '결과물(상태)'에 집중한다.

#### 행위 기반 검증 (Behavior-based Verification)
반면 행위 기반 검증은 SUT가 작업을 수행하는 '과정'에 집중한다. 즉, SUT가 협력 객체와 어떤 식으로 **상호작용(Interaction)** 했는지를 검증한다. 여기서 **상호작용이란 객체 간의 메서드 호출 그 자체**를 의미한다. Mock 객체는 바로 이 행위 기반 검증을 위해 사용된다.

*   **목적**: SUT가 협력 객체의 올바른 메서드를, 올바른 인자와 함께, 정확한 횟수만큼 호출했는지를 확인하는 **행위 검증**이 Mock의 핵심 목적이다. 이는 반환 값만으로는 확인할 수 없는 부수 효과(Side Effect, 예: 이메일 발송, 로깅, DB 저장 호출)를 검증할 때 특히 유용하다.

Mockito 프레임워크의 `verify()` 구문이 Mock 객체의 행위를 검증하는 데 사용된다.

```java
// LostItemChatRoomInfoServiceTest.java
@Test
void 첫_번째_채팅방_생성시_채팅방_ID가_1로_생성된다() {
    // given: Repository가 비어있다고 Stubbing
    when(chatRoomInfoRepository.findByArticleIdAndMessageSenderId(articleId, messageSenderId)).thenReturn(Optional.empty());
    when(lostItemArticleRepository.getByArticleId(articleId)).thenReturn(lostItemArticle);
    when(chatRoomInfoRepository.findByArticleId(articleId)).thenReturn(List.of());

    // when
    lostItemChatRoomInfoService.getOrCreateChatRoomId(articleId, messageSenderId);

    // then - 행위 검증
    // chatRoomInfoRepository의 save() 메서드가 '정확히 1번' 호출되었는지 검증한다.
    // 이는 '저장'이라는 행위가 발생했음을 보장한다.
    verify(chatRoomInfoRepository, times(1)).save(any(LostItemChatRoomInfoEntity.class));
}

@Test
void 기존에_생성된_채팅방이_있으면_해당_채팅방_ID를_반환한다() {
    // ... (given, when 부분은 Stub 예제와 동일)

    // then - 행위 검증
    // 이 시나리오에서는 새로운 채팅방을 만들지 않아야 한다.
    // 따라서 save() 메서드가 '절대 호출되지 않았음'을 검증하여 올바른 로직 분기를 확인한다.
    verify(chatRoomInfoRepository, never()).save(any(LostItemChatRoomInfoEntity.class));
}
```
*   **주의점**: 행위 기반 검증은 테스트 대상의 내부 구현에 강하게 결합된다. 만약 `save` 메서드 대신 다른 이름의 메서드(예: `persist`)로 리팩토링하면 기능은 동일해도 테스트는 실패한다. 이는 리팩토링을 방해하는 '취약한 테스트(Fragile Test)'가 될 수 있다. 따라서 상태 기반 검증으로 충분하다면 이를 우선하고, 행위 기반 검증은 상태로 확인할 수 없는 중요한 부수 효과를 검증할 때 신중하게 사용해야 한다.

### 5. Spy (스파이)

*   **정의**: 실제 객체를 래핑(wrapping)하여 만드는 테스트 대역으로, 실제 객체의 동작을 그대로 수행하면서 필요한 부분만 행위를 기록하거나 Stub으로 대체할 수 있는 유연한 대역이다.
  

* **목적**: 대부분의 로직은 실제 객체의 것을 그대로 사용하고, 특정 메서드의 호출 여부만 검증하거나 반환 값만 일시적으로 변경하고 싶을 때 유용하다.
  

* **상세**: Mock과의 결정적인 차이는 **실제 인스턴스의 사용 여부**다. Mock 객체의 메서드는 기본적으로 아무 동작도 하지 않지만, Spy 객체의 메서드는 별도 설정이 없으면 **실제 객체의 원본 메서드를 호출한다**. 이 때문에 Mock은 인터페이스만으로 생성할 수 있지만, Spy는 반드시 인스턴스화된 실제 객체를 필요로 한다. Spy는 상속이나 프록시 패턴을 통해 구현할 수 있다.

```java
// Mockito를 사용한 Spy 예시
@Test
void spyExample() {
    List<String> realList = new ArrayList<>();
    List<String> spyList = spy(realList); // 실제 ArrayList 인스턴스를 spy로 감싼다.

    // Spy의 메서드를 호출하면 실제 객체의 메서드가 호출된다.
    spyList.add("one");
    spyList.add("two");

    // 행위 검증이 가능하다.
    verify(spyList).add("one");

    // 실제 객체의 상태가 변경되었다.
    assertThat(spyList.size()).isEqualTo(2);

    // 특정 메서드만 Stub으로 동작을 변경할 수 있다.
    when(spyList.size()).thenReturn(100);
    assertThat(spyList.size()).isEqualTo(100);
}
```

### 결론: 테스트 의도에 맞는 대역 선택

테스트 대역은 격리 수준을 높이고 테스트의 의도를 명확하게 표현하는 필수적인 도구다. 각 대역의 역할과 목적을 명확히 이해하는 것은 견고하고 유지보수하기 쉬운 테스트 코드를 작성하는 기반이 된다.

| 유형 | 핵심 목적 | 검증 방식 | 실제 로직 포함 여부 |
| :--- | :--- | :--- | :--- |
| **Dummy** | 메서드 시그니처 충족 | - | X |
| **Stub** | SUT에 특정 상태 제공 | **상태 검증** | X |
| **Fake** | 경량화된 의존성 구현 | 상태 검증 | O (간소화된 로직) |
| **Mock** | SUT의 행위 검증 | **행위 검증** | X |
| **Spy** | 부분적인 행위 검증/Stubbing | 행위/상태 검증 | O (실제 객체 위임) |

테스트를 작성할 때 "메서드 실행 후 **상태**를 검증할 것인가?" 아니면 "올바른 **상호작용**이 발생했는지 검증할 것인가?"를 먼저 고려해야 한다. 이 질문에 대한 답이 Stub, Mock 등 가장 적절한 테스트 대역을 선택하는 명확한 기준이 될 것이다.

