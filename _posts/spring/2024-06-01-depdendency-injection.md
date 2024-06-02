---
layout: post
current: post
cover: assets/built/images/spring/spring-3-logo.png
navigation: True
title: Spring Dependency Injection, Component Scan
date: 2024-06-01 22:30:00 +0900
tags: [spring]
class: post-template
subclass: 'post tag-spring'
author: GyuhoonK
---

Spring의 의존관계 주입 방법과 Component Scan에 대해서 알아봅니다

## 의존관계 자동 주입
- AS-IS: `AppConfig.java`에서 `@Bean`에 의존하거나 XML의 `<bean>`을 사용
- TO-BE: java 코드 작성과 동시에 의존 관계를 주입하고 싶다 (AppConfig, XML과 같이 의존관계를 따로 작성하고 싶지 않다)

### `@Component` 
- Bean 이름은 클래스명을 사용하되 맨 앞 글자만 소문자로 바꿔서 사용한다
- annotation 사용 시에 Bean name을 지정해줄 수도 있다 (잘 안씀)

`@Component` 이외에도 아래 annotation들을 추가하면 Bean으로 등록하고 의존관계를 자동으로 주입한다. 아래 annotation들은 `Component`가 이미 추가되어있다.

- Controller: Spring MVC Controller로 인식
- Repository: 데이터 접근 계층으로 인식하고, 데이터 계층의 예외를 Spring 예외로 변환
- Configuration: Spring 설정 정보로 인식하고 Bean이 싱글톤을 유지하도록 추가처리
- Service: 특별한 처리는 없음, 핵심 비즈니스 로직을 기술함

참고로, java annotation은 상속 관계가 없기 때문에 위 4가지 annotation들은 java 문법 상으로는 `Component`와 관련이 없는 annotation이다. Spring 내부에서 위 annotation들이 `Component`의 속성을 상속받아 작성되었다.

```java
@Component
public class OrderServiceImpl implements OrderService {
    private final MemberRepository memberRepository;
    private final DiscountPolicy discountPolicy;
}
```

### `@Autowired` 

- 스프링 컨테이너가 자동으로 해당 스프링 Bean을 찾아서 주입한다
- 타입이 같은 Bean을 찾아서 주입한다고 보면 된다

```java
    @Autowired
    public OrderServiceImpl(MemberRepository memberRepository, @MainDiscountPolicy DiscountPolicy discountPolicy) {
        this.memberRepository = memberRepository;
        this.discountPolicy = discountPolicy;
    }
```

## Component Scan

### 스캔 범위
Spring은 `Component`로 등록된 Bean을 자동으로 검색하고 추가한다. 그렇다면 Spring 프로젝트 전체를 스캔하는 것일까? 
일반적으로는 맞다. 따로 지정하지 않으면 `@SpringBootAplication`으로부터 Component Scan을 시작하는데, `@SpringBootAplication`는 Spring 프로젝트의 최상단에 위치하고 있기 때문이다.

```java
@SpringBootApplication
public class DemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}

}
```

참고로 `SpringBootApplication`은 아래와 같은 옵션으로 Component Scan을 실행한다.

```java
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication 
```

### 스캔 범위 조정

`@ComponentScan` 추가 시 아래와 파라미터들을 통해 스캔 범위를 조정할 수 있다.

```java
@Configuration
@ComponentScan(
        basePackages = "hello.demo.member", // 해당 패키지부터만 @Component를 
        basePackageClasses = AutoAppConfig.class, // 지정한 클래스 패키지를 탐색 
        excludeFilters = @ComponentScan.Filter(type=FilterType.ANNOTATION, classes = Configuration.class) // 수동으로 작성된 AppConfig.java는 등록에서 제외함
)
public class AutoAppConfig {
}
```

[참고] Filter 옵션
- ANNOTATION: 기본값, annotation을 인식해서 동작함
- ASSIGNABLE_TYPE: 지정한 타입과 자식 타입을 인식해서 동작함
- ASPECTJ: AspectJ 패턴을 사용함
- REGEX: 정규표현식
- CUSTOM: `TypeFilter`라는 인터페이스를 구현해서  처리함

### 권장하는 스캔 설정
위처럼 패키지 위치를 지정하지 않고, 설정 정보 클래스의 위치를 프로젝트 최상단에 두는 것을 권장한다. 

예를 들어, 프로젝트 구조가 아래와 같다면
```
com.hello
├── service
└── repository
```

`com.hello`가 프로젝트의 시작 루트이므로 `AppConfig`와 같은 메인 설정 정보를 두고 `@ComponentScan`annotation을 추가한다. 이렇게 설정하면 `com.hello` 이하의 모든 Component가 자동으로 스캔된다. 


## 중복 등록과  충돌
ComponentScan에 의해 *자동*으로 등록되는 Bean과 `@Bean`을 사용하여 *수동*으로 등록된 Bean이 동일한 이름을 갖는다면 어떻게 될까?

일반적으로 중복 Bean name은 `BeanDefinitionStoreException`를 발생시키지만, 수동 등록 Bean과 자동 등록 Bean은 충돌 시 수동 등록 Bean이 우선권을 갖고 자동 등록 Bean을 override하도록 설정할 수 있다.

이를 위해서는 `application.properties`에 아래 옵션을 추가해야한다

```properties
main.allow-bean-definition-overriding=true
```

옵션을 추가하고나면, Bean name 충돌에 대해서 Spring이 아래처럼 처리하는 로그를 확인할 수 있다.

```
Overriding bean definition for bean 'memoryMemberRepository' with a different definition
```

일반적으로 이는 권장하지 않는 방법이며, 중복 Bean name을 사용하지 않는 것이 가장 좋다.


[참고]  
[[inflearn]스프링 핵심 원리 - 기본편](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B8%B0%EB%B3%B8%ED%8E%B8/dashboard)  