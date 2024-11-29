---
title: "[스프링] 의존관계 주입 - 권장하는 방법 찾기"
description: "스프링의 다양한 의존관계 주입 방법"
date: 2024-05-29
update: 2024-11-29
tags:
  - spring
  - 단순 지식
series: "스프링"
---

자바(스프링)에서는 의존관계를 주입하기 위한 여러가지 방법이 있다. 그중 코드가 가장 간결하게 보이는 방법이 필드 주입이다. 그러나 이 방식은 권장하는 방법이 아니라고 한다. 왜 그럴까?

### 테스트 코드 작성의 어려움

필드 주입의 가장 큰 문제점은 스프링과 같은 DI 프레임워크 없이 테스트 코드를 작성할 수 없다는 것이다.

```java
@Component
public class DBMemberRepository implements MemberRepository{...}
```

```java
@Component
public class SilveRoomPolicy implements RoomRankPolicy{...}
```

```java
@Component
public class ReservationServiceImpl implements ReservationService{

	@Autowired private MemberRepository memberRepository;
	@Autowired private RoomRankPolicy roomRankPolicy;
	
	public void discountReservation(Long memberID, Long reserveID){
		roomRankPolicy.findPrice(reserveID);
		...
	}
	...
}
```

스프링 부트를 통해 빌드해서 실행할 경우 위와 같이 필드 주입으로 의존성을 주입해도 이상없이 잘 동작할 것이다. 그러나 `ReservationServiceImpl` 에 대한 테스트 코드를 작성하려고 할 때 문제가 발생한다.

```java
public class ReservationServiceTest{
	@Test
	void goldRoomDiscountTest(){
		ReservationServiceImpl rs = new ReservationServiceImpl();
		rs.discountReservation();// NullPointerException
	}
}
```

스프링 컨테이너 밖 순수한 자바는 `@Autowried` 어노테이션을 인식하지 못한다. 따라서 일반 환경에서 ``goldRoomDiscountTest`` 단위 테스트를 실행할 경우 NullPointerException 오류가 발생한다.

또한, 테스트를 작성할 때 개발자가 원하는 의존관계를 직접 주입할 수 있는 방법이 없으므로 유연한 테스트 작성이 불가능하다.

이러한 이유로 스프링에선 생성자 주입이나 수정자 주입(setter 주입) 방식을 권장한다.

### 수정자 주입(Setter 주입)을 사용할까?

위의 예시 코드를 수정자 주입 방식으로 고쳤다.

``` java
@Component
public class ReservationServiceImpl implements ReservationService{

	private MemberRepository memberRepository;
	private RoomRankPolicy roomRankPolicy;
	
	@Autowired 
	public void setMemberRepository(MemberRepository memberRepository) {
		this.memberRepository = memberRepository;
	}
	
	@Autowired 
	public void setRoomRankPolicy(RoomRankPolicy roomRankPolicy) {
		this.roomRankPolicy = roomRankPolicy;
	}
	
	public void discountReservation(Long memberID, Long reserveID){
		roomRankPolicy.findPrice(reserveID);
		...
	}
	...
}
```

```java
public class ReservationServiceTest{
	@Test
	void goldRoomDiscountTest(){
		ReservationServiceImpl rs = new ReservationServiceImpl();
		
		rs.setMemberRepository(new TestMemberRepo());
		rs.setRoomRankPolicy(new TestRoomRankPolicy());
		
		rs.discountReservation();
	}
}
```

수정자 주입 방식을 사용하면, 스프링 컨테이너가 아니더라도 의존성 주입이 가능하므로 유연한 단위 테스트 작성이 가능하다. 필드 주입을 했을 때 일어날 수 있는 단점을 적절히 해결한 것 같이 보인다.

그럼 수정자 주입 방식이 스프링에서 가장 적합한 의존관계 주입 방법일까?

### 수정자 주입의 문제점

수정자 주입은 필드 주입에 비해 테스트 코드 작성에 용이하다는 장점이 있지만, 다음과 같은 단점 또한 존재한다.

#### 부작용 발생 가능
의존성 주입 시점이 명확하지 않아, 의존성 주입 전후로 객체의 상태가 달라질 수 있다. 스프링 컨테이너에서는 상관 없는 이야기 이지만, 테스트 코드 작성시에는 유의해야 한다.

```java
public class ReservationServiceTest{
	@Test
	void goldRoomDiscountTest(){
		ReservationServiceImpl rs = new ReservationServiceImpl();
		
		rs.discountReservation(); // NullPointerException 오류 발생 
	}
}
```

위 오류를 막기 위해선 ``discountReservation`` 메서드에 따로 의존성 주입 여부를 확인하는 추가 로직을 작성해야 한다.

```java
public void discountReservation(Long memberID, Long reserveID) {
        if (roomRankPolicy == null || memberRepository == null) {
            throw new IllegalStateException("Dependencies have not been set.");
        }
```


#### 불변성 위반
생성자 주입이 객체가 생성됨과 동시에 의존성이 주입되는 것에 반해, 수정자 주입은 객체가 생성된 후에 의존성이 주입되므로, 의존성이 변경될 수 있어 불변성이 깨질 수 있다.

``` java
@Component
public class ReservationServiceImpl implements ReservationService{

	private MemberRepository memberRepository;
	private RoomRankPolicy roomRankPolicy;
	
	@Autowired 
	public void setMemberRepository(MemberRepository memberRepository) {
		this.memberRepository = memberRepository;
	}
	
	@Autowired 
	public void setRoomRankPolicy(RoomRankPolicy roomRankPolicy) {
		this.roomRankPolicy = roomRankPolicy;
	}
```

``@Autowried`` 덕분에 스프링 컨테이너 안에선 스프링이 알아서 의존 관계를 주입해 줄 것이다. 문제는 수정자 주입 방식 때문에 ``setMemberRepository`` 와 ``setRoomRankPolicy`` 메서드가 존재하고, 이 메서드들이 ``public``으로 열려 있다는 점이다.

만약 어떤 개발자가 자동으로 의존성이 주입된 객체에 ``setMemberRepository`` 를 호출하여 의존 관계를 변경한 경우 프로그램 동작에 문제가 발생할 수 있다. 물론 이를 막기 위해 다음과 같이 추가로 예외를 작성 할 수도 있다.

```java
public void setMemberRepository(MemberRepository memberRepository) {
    if (this.memberRepository != null) {
        throw new IllegalStateException("MemberRepository has already been set.");
    }
    this.memberRepository = memberRepository;
}

public void setRoomRankPolicy(RoomRankPolicy roomRankPolicy) {
    if (this.roomRankPolicy != null) {
        throw new IllegalStateException("RoomRankPolicy has already been set.");
    }
    this.roomRankPolicy = roomRankPolicy;
}
```

코드가 점점 길어지고 복잡해 지고 있다.

#### 가독성 저하
수정자 주입 방식은 의존 관계 주입을 위해 각각 setter 를 작성해 줘야 하고, 발생 가능한 예외 상황 처리 로직을 추가해야 하므로 전체적인 코드의 가독성이 떨어진다.

### 결론 : 생성자 주입을 사용하자

```java
@Component
public class ReservationServiceImpl implements ReservationService {

    private final MemberRepository memberRepository;
    private final RoomRankPolicy roomRankPolicy;

    @Autowired
    public ReservationServiceImpl(MemberRepository memberRepository, RoomRankPolicy roomRankPolicy) {
        this.memberRepository = memberRepository;
        this.roomRankPolicy = roomRankPolicy;
    }

    public void discountReservation(Long memberID, Long reserveID) {
        // ...
    }
    // ...
}
```

```java
public class ReservationServiceTest {
    @Test
    void goldRoomDiscountTest() {
        MemberRepository testMemberRepo = new TestMemberRepo();
        RoomRankPolicy testRoomRankPolicy = new TestRoomRankPolicy();
        
        ReservationServiceImpl rs = new ReservationServiceImpl(testMemberRepo, testRoomRankPolicy);
        
        rs.discountReservation();
        //...
    }
}
```

생성자 주입 방식을 사용하면 수정자 주입 방식과 같이 테스트 코드에서 의존성을 직접 주입할 수 있어 유연한 테스트 작성이 가능하다. 또한 클래스 내부에 별도의 setter 메서드가 존재하지 않으므로 의존관계가 변경될 가능성이 적고, 객체가 생성되면서 동시에 의존성이 주입되므로 의존성 주입 시점 때문에 발생할 수 있는 오류가 발생하지 않는다.

### 결론

의존관계를 주입하는 다양한 방식에 대해 공부하면서 생성자 주입이 가장 권장되는 방식임을 알게 되었다. 각 방식마다 장단점이 있지만, 생성자 주입 방식이 단점이 가장 적은 것으로 보인다.

생성자 주입 방식은 객체 생성 시점에 의존성을 주입하므로, 수정자 주입과는 다르게 테스트 코드를 작성할 때 의존성 주입 여부를 확인할 필요가 없다. NPE(NullPointerException) 오류 발생 가능성을 원천적으로 제거하는 것이다. 또한, 객체가 생성된 이후에는 의존성을 변경할 수 없기 때문에 불변성(Immutability)이 보장된다.

소규모 프로젝트나 개인 프로젝트에서 테스트 코드를 작성하지 않는 경우 필드 주입이나 수정자 주입을 사용해도 큰 문제가 없을 수 있다. 그러나 유지보수 및 테스트의 용이성을 고려해야 하는 큰 프로젝트를 진행하는 경우,  비록 코드 길이가 다소 길어지더라도 안정성 측면에서 생성자 주입이 가장 바람직한 방식일 것이다.

다음에는 @Autowired의 구체적인 동작 원리에 대해서 알아보려 한다.