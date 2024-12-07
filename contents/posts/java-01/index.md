---
title: "[Java] 계약 기반 프로그래밍과 Template을 구현하는 Interface"
description: "범용 메소드의 강건성 지키기: Template의 등장 배경"
date: 2024-12-07
update: 2024-12-07
tags:
  - Java
  - 단순 지식
series: "Java"
---

# 범용 메서드의 강건성
## Object 클래스 이용
자바에서 범용 메서드의 강건성을 지키기 위해서 어떤 방법을 사용하고 있을까? 
인자 두 개를 받아서 우선순위가 높은 것을 반환하는 간단한 범용 메서드를 구현해보자. 
자바에서는 Object 클래스가 있으므로 다양한 타입의 인자를 받기 위해 다음과 같이 범용 메서드를 정의할 수 있다.

```java
public static Objcet max(Object a, Object b) {}
```

이렇게 정의하면 이 함수는 실제로 원시 타입을 제외한 모든 타입의 객체를 받을 수 있다. 하지만 몸체를 정의하다 보면 이 방식에 문제가 있다는 사실을 알 수 있다.

데이터를 비교하기 위해선 어떤 데이터가 더 앞에 와야 하는지를 판단할 수 있어야 한다. 원시 타입의 경우 연산자를 활용할 수 있지만 객체타입은 메소드를 사용해야 한다. 매개 변수의 타입이 Object 이므로 Object의 메소드 만을 사용해야 한다.

**Object 가 제공하는 public 메소드(스레드 관련 제외)** : `equals(), hashCode(), toString()`

불행히도 Object 타입에는 비교 메서드가 정의되어 있지 않다. 비교가 아닌 다른 역할의 범용 메소드를 정의할 경우에도 유사한 문제가 발생한다.

## 인터페이스 이용 - 계약 기반 프로그래밍
이 문제는 거꾸로 접근하여 해결할 수 있다. 즉, 비교 메소드가 선언되어 있는 객체들만 인자로 받으면 되는 것이다. 이를 위해 인터페이스를 사용할 수 있다.

```java
public static Comparable max(Comparable a, Comparable b) {}
```

Java의 Comparable 인터페이스엔 compareTo() 메서드가 선언되어 있다. 특정 타입의 객체를 비교하고 싶으면 해당 객체의 클래스에 Comparable 인터페이스를 구현하면 된다.
이와 같은 방식의 프로그래밍을 [계약에 의한 설계(Design by contract, DbC)](https://ko.wikipedia.org/wiki/%EA%B3%84%EC%95%BD%EC%97%90_%EC%9D%98%ED%95%9C_%EC%84%A4%EA%B3%84), 계약 기반 프로그래밍이라 한다.

그런데 위와 같이 범용 함수를 만들면 두가지 문제점이 발생한다.

### 1. a와 b의 타입 문제
```java
public class Dog impelments Comparable<Dog> {}
Dog dog = new Dog();

public class Cat impelments Comparable<Cat> {}
Cat cat = new Cat();

var result = max(dog, cat); // ?
```

범용 함수 max는 같은 종류의 두 객체를 비교하기 위한 것인데, 지금과 같은 방식은 a와 b에 전달된 객체가 같은 타입이라는 것을 보장해 주지 못한다.

### 2. 반환 타입의 문제
```java
Dog dogA = new Dog();
Dog dogB = new Dog();

Dog bigDog = (Dog)max(dogA, dogB);
```
범용 함수의 반환 타입이 인터페이스와 같은 타입이다. 따라서 이 메소드를 사용할 경우 반환 값을 의도된 원래 객체의 타입으로 타입 변환 해줘야 하는 번거로움이 있다.

## Template 이용
이 두가지 문제점은 더 유연한 범용 프로그래밍을 지원하기 위해 자바 5에서 추가된 template을 이용하여 해결할 수 있다. 하지만 template 기능을 사용한다고 해서 인터페이스의 역할이 사라지는 것은 아니다.

```java
public static <T> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}
```

이와 같이 범용 함수를 정의하면 타입 인자로 지정한 타입만을 객체로 전달할 수 있고, 반환 타입의 값이 전달된 인자의 타입과 같아지므로 타입 변환이 필요 없다.
하지만 어떤 타입 인자를 사용하는지 알 수 없기 때문에 타입 매개 변수 T를 이용할 경우 compareTo 메소드를 사용할 수 없다.
이 문제를 해결하기 위해 Java는 범용 메소드를 최종적으로 다음과 같이 정의한다.

```java
public static <T extends Comparable<? suprer T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}
```

타입 매개변수 T는 `Comparable<? suprer T>`를 구체화한 타입으로 제한하고 있기 때문에 compareTo 메소드를 사용할 수 있다.
`? suprer T`가 의미하는 것을 뭘까?

### PECS 규칙 (Producer-Extends Consumer-Super)
제네릭 타입을 사용할 때 적용되는 중요한 규칙이 바로 PECS 규칙이다. 이는 상한 경계 와일드카드(Upper Bounded Wildcard)와 하한 경계 와일드카드(Lower Bounded Wildcard)를 효과적으로 사용하는 방법을 정의한다.

#### Producer-Extends
이 규칙에 따르면 데이터를 생산(읽기)만 하는 메서드의 경우 extends 키워드를 사용한다.
```java
public static <T extends Comparable<? super T>> void printList(List<? extends T> list) {
    for (T element : list) {
        System.out.println(element);
    }
}
```

이 메서드에서 List<? extends T>는 T 타입이나 T의 하위 타입의 리스트를 받을 수 있다. 이는 리스트로부터 데이터를 읽어올 수 있지만, 리스트에 새로운 요소를 추가할 수는 없다.

#### Consumer-Super
데이터를 소비(쓰기)하는 메서드의 경우 super 키워드를 사용한다. 
```java
public static <T> void copyList(List<? super T> dest, List<? extends T> src) {
    dest.addAll(src);
}
```
여기서 List<? super T>는 T 타입이나 T의 상위 타입의 리스트를 의미한다. 이를 통해 리스트에 요소를 추가할 수 있지만, 리스트로부터 특정 타입으로 데이터를 읽어오는 것은 제한된다.

PECS 규칙을 사용하면 제네릭 메서드의 유연성을 크게 높일 수 있다. 앞서 살펴본 max() 메서드의 예시를 다시 한번 살펴보자
```java
public static <T extends Comparable<? suprer T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}
```

여기서 범용 타입으로 허용되는 클래스는 Comparable 인터페이스를 구현한 클래스뿐만 아니라, 해당 클래스의 상위 클래스에서 compareTo() 메서드를 구현한 경우도 포함된다. 이는 기존 클래스 계층 구조에서 더 유연한 비교 로직을 구현할 수 있게 해준다.

### `Comparable<? suprer T>`의 의미
```java
class Animal implements Comparable<Animal> {
    private int age;

    public int compareTo(Animal other) {
        return Integer.compare(this.age, other.age);
    }
}

class Dog extends Animal {
    private String breed;
}

public class Main {
    public static void main(String[] args) {
        Dog dog1 = new Dog();
        Dog dog2 = new Dog();

        Dog maxDog = max(dog1, dog2);
    }
}
```

만약 Dog 클래스에서 compareTo를 재정의 해야 할 경우 어떻게 해야 할까 ?

```java
// X
@Override
public int compareTo(Dog other)

// O
@Override
public int compareTo(Animal other)
```
compareTo가 구현되어 있는 클래스를 상속하면서 compareTo를 재정의 하고 싶으면 재정의 문법에 따라서 재정의 메소드의 매개변수가 부모 타입이 된다.
따라서 이 경우 인자로 받은 Animal 타입을 Dog 타입으로 형변환 해줘야 한다.

```java
@Override
public int compareTo(Animal other) {
    Dog dog = (Dog)other;
    // ...
}
```

```java
public static <T extends Comparable<? suprer T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}
```

이 때문에 위 범용 메소드에서 `? super T`의 사용이 필요한 것이다. 이 경우 Dog 클래스가 구현하고 있는 인터페이스는 Comparable<Dog>가 아니라 Comparable<Animal> 이다.

