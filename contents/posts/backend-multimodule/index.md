---
title: "[Backend] 스프링 멀티모듈 프로젝트"
description: "벡엔드"
date: 2024-12-21
update: 2024-12-21
tags:
  - java
  - spring
  - 단순 지식
series: "Backend Engineering"
---

## 자바 모듈 시스템
Java 9이전에는 패키지만으로 코드를 조직하고 관리했다. 자바 애플리케이션이 점점 커지고 복잡해 지면서 패키지만으로 코드를 관리하는데 한계가 드러나기 시작했다.

패키지 간의 의존성을 명확하게 정의하기 어려웠고 불필요한 클래스가 포함된 빌드파일로 인해 애플리케이션의 크기가 커지는 문제가 발생했다.

이런 문제점을 해결하기 위해 Java9부터 모듈 시스템([Java 9 Platform Module System](https://www.oracle.com/kr/corporate/features/understanding-java-9-modules.html))을 도입했다.

모듈 시스템은 애플리케이션을 작은 단위로 분리하여 각 모듈별로 의존성을 관리하는 방법을 제공했으며 성능과 보안을 강화하는 데 기여했다.


## 스프링 멀티 모듈 프로젝트



## 왜 멀티 모듈을 사용할까?

## 멀티 모듈 구현

### gradle.properties 를 이용해서 의존성 버전 관리하기

멀티 모듈 프로젝트는 여러개의 `build.gradle` 파일이 생성된다. 각 모듈별 버전 정보들이 파편화 되어 있으면 관리가 어려워질 것이다.

`gradle.properties` 파일을 통해 의존성 버전을 한곳에서 정의할 수 있다.

**gradle.properties**
```
applicationVersion = 0.0.1

projectGroup = com.devtab

springBootVersion = 3.4.1
springDependencyManagementVersion = 1.1.7
```

**settings.gradle**
```
pluginManagement {
    String springBootVersion = settings.springBootVersion
    String springDependencyManagementVersion = settings.springDependencyManagementVersion

    resolutionStrategy {
        eachPlugin {
            if (requested.id.id == 'org.springframework.boot') {
                useVersion(springBootVersion)
            }
            if (requested.id.id == 'io.spring.dependency-management') {
                useVersion(springDependencyManagementVersion)
            }
        }
    }
}
```

`settings.gradle` 파일에서 플러그인 버전을 관리할 수 있다. `pluginManagement` 블록 내에서 플러그인의 버전을 동적으로 설정하거나, 원격 저장소에서 플러그인을 검색하도록 구성할 수 있다.

**build.gradle**
```
plugins {
	id 'java'
	id 'org.springframework.boot'
	id 'io.spring.dependency-management'
}

allprojects {
	group = "${projectGroup}"
	version = "${applicationVersion}"
}
```

`gradle.properties` 파일에 정의된 변수는 `build.gradle` 파일에서 **"${변수명}"** 형식으로 참조 가능하다.

플러그인 버전에 대해선 `settings.gradle`에 정의된 값을 자동으로 불러오게 된다. `settings.gradle`을 사용하지 않는다면 다음과 설정하면 된다.
```
plugins {
	id 'java'
	id 'org.springframework.boot' version "${springBootVersion}"
	id 'io.spring.dependency-management' version "${springDependencyManagementVersion}"
}
```

## 참고
**멀티모듈 설계**
- [멀티모듈 설계 이야기 with Spring, Gradle, 권용근](https://techblog.woowahan.com/2637/)
- [우아한 멀티 모듈 by 우아한형제들 권용근님](https://www.youtube.com/watch?v=nH382BcycHc)
- [우아한 멀티 모듈 세미나 정리](https://hyeon9mak.github.io/woowahan-multi-module/)
- [실전! 멀티 모듈 프로젝트 구조와 설계 | 인프콘 2022](https://www.youtube.com/watch?v=ipDzLJK-7Kc)
- [우리는 이렇게 모듈을 나눴어요: 멀티 모듈을 설계하는 또 다른 관점 | 인프콘2023](https://www.youtube.com/watch?v=uvG-amw2u2s&t=762s)
- [왜 프로젝트를 멀티 모듈로 구성할까](https://www.youtube.com/watch?v=VPzg61njKxw&t=29s)
- [멀티모듈 쓰지 말자 | 멀티모듈 오용 사례와 모듈링 접근법](https://www.youtube.com/watch?v=6FiNarz3pOA)

**멀티모듈 적용**
- [Creating a Multi Module Project, Spring 문서](https://spring.io/guides/gs/multi-module)
- [Java 9 Modules 이해하기, 오라클 문서](https://www.oracle.com/kr/corporate/features/understanding-java-9-modules.html)
- [멀티 모듈 적용하기 with Gradle](https://tecoble.techcourse.co.kr/post/2021-09-06-multi-module/)
- [SpringBoot + Kotlin 멀티 모듈 구성 - 단일모듈에서 멀티모듈로 변경해보기 #1](https://www.youtube.com/watch?v=PdofVTuM-tE)
- [SpringBoot + Kotlin 멀티 모듈 구성 - 의존성 버전 관리 + 로깅 모듈 추가하기 #2](https://www.youtube.com/watch?v=rE89ppAmf_Y&t=59s)
- [SpringBoot + Kotlin 멀티 모듈 구성 - DB 모듈 추가하기 #3](https://www.youtube.com/watch?v=ODSFmLdecX0&t=7s)
- [SpringBoot + Kotlin 멀티 모듈 구성 - 도메인 모듈 분리 #4](https://www.youtube.com/watch?v=p5ZMF2bpE6A)
- [도메인 모듈 분리 시 Transaction + JPA 활용 방안](https://www.youtube.com/watch?v=18C2A56ialY)


