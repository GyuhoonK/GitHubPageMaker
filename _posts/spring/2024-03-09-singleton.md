---
layout: post
current: post
cover: assets/built/images/spring/singleton.jpg
navigation: True
title: Spring Singleton Container
date: 2024-03-09 22:30:00 +0900
tags: [spring]
class: post-template
subclass: 'post tag-spring'
author: GyuhoonK
---

Spring Container에서 지원하는 싱글톤 컨테이너에 대해서 알아봅니다.

## Spring Container는 기본적으로 Singleton이다
- Bean을 미리 생성하고 제공한다
- ApplicationContext가 Singleton Registry의 역할을 한다
- Singleton Registry 덕분에 싱글톤 패턴의 단점(안티패턴)을 극복할 수 있다
- 싱글톤 패턴만 지원하는 것은 아니다, 요청 시 마다 새로운 객체(Bean)를 생성하는 기능도 제공한다.

[참고] 싱글톤 패턴의 문제점
- 구현 코드 자체가 길어진다
- 의존 관게상 클라이언트가 구체 클래스에 의존한다 ⇒ DIP를 위반한다
- 클라이언트가 구체 클래스에 의존하기 때문에 OCP 원칙을 위반할 가능성이 높다
- 테스트가 어렵다
- 내부 속성을 변경하거나 초기화하기 어렵다
- private 생성자로 자식 클래스를 만들기 어렵다
- 결론적으로, 유연성이 떨어진다 
- 위와 같은 문제점 때문에 **안티패턴**이라고도 불린다


## 싱글톤 방식 구현 시 주의점: Stateless
- 싱글톤 패턴은 상태를 유지(stateful)하게 설계해선 안되고, 무상태(stateless)로 설계해야한다
    - 특정 클라이언트에 의존적인 field가 있어선 안된다
    - 특정 클라이언트가 값을 변경할 수 있어서는 안된다
    - 가급적 읽기만 가능해야한다
    - 필드 대신에 자바에서 공유되지 않는 지역변수, 파라미터, threadlocal 등을 사용한다

## 싱글톤 패턴을 위한 Annotation: `@Configuration`
- AppConfig.java: Bean 생성 시 마다 새로운 객체를 생성하는 것처럼 보인다
  ```java
  @Configuration
  public class AppConfig {
      @Bean
      public MemberService memberService() {
          System.out.println("AppConfig.memberService");
          return new MemberServiceImpl(memberRepository()); // cmd + opt + m (리팩토링)

      }
      @Bean
      public OrderService orderService() {
          System.out.println("AppConfig.orderService");
          return new OrderServiceImpl(memberRepository(), discountPolicy());
      }
  }
  ```
  `memberRepository`는 2번 생성되는 것처럼 보이지만, 실제로 생성된 Bean을 조회해보면 1회만 생성된다(생성자 자체가 1회만 호출된다)
- `@Configuration`은 CGLIB이라는 바이트 코드 조작 라이브러리를 이용하여 AppConfig를 상속받은 임의의 다른 클래스를 Spring Bean으로 등록한다.
  - `class hello.demo.AppConfig$$SpringCGLIB$$0`: Spring 내에서 클래스로 등록되는 것을 확인할 수 있다. 이는 CGLIB에 의해 AppConfig를 상속받은 임의의 다른 클래스이다.
  - 아마도 위 클래스 내에 이미 해당 클래스가 생성되었다면 추가로 생성하지 않는 방어로직이 있을 것이다 (코드를 더 깊게 보지는 않았다)
- `@Configuration`을 사용하지 않으면, Bean 객체가 순수한 java 클래스이다
  - 객체 생성 시마다 Bean을 생성한다 ⇒ 동일한 객체가 N번 생성되므로 싱글톤 패턴이 깨진다
- `@Bean` 대신에 클래스 작성 시에 `@Autowired`를 이용할 수도 있다
  ```java
  @Autowired
      public MemberServiceImpl(MemberRepository memberRepository) {
          this.memberRepository = memberRepository;
      }
  ```

## 정리
- Spring은 기본적으로 싱글톤 패턴 디자인을 지원한다.
- 싱글톤 패턴 디자인에서는 객체가 stateless하도록 설계해야한다.
- `@Configuration` 어노테이션을 사용하면, 의존성에 따라 생성이 필요한 객체(Bean)을 1회만 생성한다.

  
[참고]  
[[inflearn]스프링 핵심 원리 - 기본편](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B8%B0%EB%B3%B8%ED%8E%B8/dashboard)  
[[Spring]Bean Overview
](https://docs.spring.io/spring-framework/reference/core/beans/definition.html)