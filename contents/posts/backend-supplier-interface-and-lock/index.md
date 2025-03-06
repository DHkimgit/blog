---
title: "[Backend] Supplier 인터페이스를 이용한 Lazy Evaluation과 분산락 구현"
description: "벡엔드"
date: 2025-03-06
update: 2025-03-06
tags:
  - java
  - spring
  - 단순 지식
series: "Backend Engineering"
---

자바는 8버전부터 여러 유용한 함수형 인터페이스를 제공하며, 이를 이용해서 유연하고 간결한 코드를 작성할 수 있다. 그러나 이러한 함수형 인터페이스들은 존재를 알고 있어도 실제로 자주 사용해보지 않는다면, 개발 과정에서 적재적소에 활용하기 어렵다고 느꼈다. 

그래서 이번 글에서는 Supplier 인터페이스가 어떤 상황에서 유용하게 사용될 수 있는지, 특히 Lazy Evaluation과 분산락 구현에 어떻게 적용할 수 있는지를 연습하고 소개하려고 한다.

## Supplier 인터페이스란 ?
Supplier는 Java 8에서 도입된 함수형 인터페이스로, 인자를 받지 않고 결과만 반환하는 메서드를 가진다. 간단히 말해 () -> T 형태의 람다식으로 표현할 수 있으며, 호출 시점에 값을 계산하는 지연 평가를 구현하기에 적합하다.

```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```

Supplier를 이용할 수 있는 간단한 사례를 소개하려고 한다.

```java
public static void logic(int number, String value) {
	if (number >= 0) {
		System.out.println(value);
	} else{
		System.out.println("lol");
	}
}
```

number의 수에 따라 출력되는 값이 달라지는 간단한 메서드다. 이 메서드의 value에 연산 과정이 복잡한, '고비용' 함수의 결과값이 제공된다고 하자.

```java
private static String expensiveValue(Long id) {...}

logic(10, expensiveValue(3l));

// 굳이 expensiveValue를 호출할 필요가 없다.
logic(-5, expensiveValue(4l));
```

number 값이 0 미만이라면, 두번째 인자로 제공되는 value는 사용되지 않는다. 즉, 고비용의 expensiveValue() 함수가 실행 될 필요가 없다는 것이다.

Supplier 인터페이스는 이런 상황에서 Lazy Evaluation을 적용하기 위해 사용될 수 있다.

```java
public static void logic(int number, Supplier<String> value) {
	if (number >= 0) {
		System.out.println(value.get());
	} else{
		System.out.println("lol");
	}
}

logic(10, () -> expensiveValue())
```

Supplier 인터페이스가 제공하는 유일한 추상 메서드인 get()을 호출하여 value 값이 필요할 때만 expensiveValue() 함수가 실행되도록 처리할 수 있다.

## 분산 락과 Lazy Evaluation
분산 시스템에서는 여러 서버 인스턴스가 동일한 리소스에 동시에 접근하는 상황이 발생한다. 이를 테면 예약 시스템에선 같은 시간, 같은 테이블에 대한 중복 예약을 방지하기 위해 분산 락이 필요하다.

스프링에서 분산 락은 다양한 형태로 구현되는데 일반적으로 AOP를 이용해서 어노테이션으로 구현하는 경우가 대부분 인 것 같다. 이 방식은 락 로직을 비즈니스 로직과 분리하고, 여러 메서드에서 일관적으로 사용할 수 있다는 장점이 있다.

Supplier 인터페이스를 이용해서도 분산 락을 구현할 수 있다. AOP와 비교했을때 내가 느끼는 가장 큰 장점은 락의 획득과 해제 과정이 코드 상에서 명확하게 드러나서 이해하기 쉽다는 점이다. 

```java
@Component
@RequiredArgsConstructor
public class DistributedLockManager {
    
    private final RedissonClient redissonClient;
    
    public <T> T executeWithLock(String lockName, long waitTime, long leaseTime, Supplier<T> supplier) {
        RLock lock = redissonClient.getLock(lockName);
        
        try {
            boolean isLocked = lock.tryLock(waitTime, leaseTime, TimeUnit.SECONDS);
            
            if (!isLocked) {
                throw new RuntimeException("락 획득 실패: " + lockName);
            }
            
            // Supplier.get() 호출 시점에 비즈니스 로직 실행 (Lazy Evaluation)
            return supplier.get();
        }
        /*...*/
```

락이 필요한 비즈니스 로직에서 다음과 같이 사용될 수 있다.

```java
    @Transactional
    public ReservationResponse reserve(Long memberId, ReservationRequest request) {
        String lockName = generateLockName(request.getDate(), request.getTableId());
        
        return lockManager.executeWithLock(lockName, LOCK_WAIT_TIME, LOCK_LEASE_TIME, () -> {
            // 락 획득 후 실행되는 비즈니스 로직 (Lazy Evaluation)
            Table table = tableRepository.findById(request.getTableId())
                    .orElseThrow(() -> new EntityNotFoundException("테이블을 찾을 수 없습니다."));
            
            if (!isTableAvailable(request.getDate(), request.getTableId())) {
                throw new IllegalStateException("이미 예약된 테이블입니다.");
            }
            
            Reservation reservation = Reservation.builder()
                    .memberId(memberId)
                    .tableId(request.getTableId())
                    .date(request.getDate())
                    .startTime(request.getStartTime())
                    .endTime(request.getEndTime())
                    .guestCount(request.getGuestCount())
                    .status(ReservationStatus.CONFIRMED)
                    .build();
            
            Reservation savedReservation = reservationRepository.save(reservation);
            
            return ReservationResponse.from(savedReservation);
        });
    }
```

Supplier 방식은 AOP에 비해 서비스 단의 코드가 더 복잡하지만, 락 획득 전후로 복잡한 로직이 존재하거나 메소드 내부 상태에 따라 락을 동적으로 생성해야 할 경우 유리하다.
