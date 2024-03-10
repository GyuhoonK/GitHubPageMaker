---
layout: post
current: post
cover: assets/built/images/spring/singleton.jpg
navigation: True
title: Spring Dependency Injection
date: 2024-03-10 22:30:00 +0900
tags: [spring]
class: post-template
subclass: 'post tag-spring'
author: GyuhoonK
---

Spring의 의존관계 주입 방법에 대해서 알아봅니다

## 의존관계 자동 주입
- AS-IS: `AppConfig.java`에서 `@Bean`에 의존하거나 XML의 `<bean>`을 사용
- TO-BE: java 코드 작성과 동시에 의존 관계를 주입하고 싶다 (AppConfig, XML과 같이 의존관계를 따로 작성하고 싶지 않다)

### `@Component` 

- Contolloer
- Service
- Repository
- Configuration

[참고] annotation에는 상속 관계가 없으나, Spring에서 구현한 기능이다 (Java의 기능이 아님)

### `@Autowired`

## 의존관계 주입 방법 

### [1] 생성자 주입

### [2] 수정자 주입(setter)

### [3] 필드 주입 

### [4] 일반 메소드 주입




## 싱글톤 방식 구현 시 주의점: Stateless


## 싱글톤 패턴을 위한 Annotation: `@Configuration`


## 정리


  
[참고]  
[[inflearn]스프링 핵심 원리 - 기본편](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B8%B0%EB%B3%B8%ED%8E%B8/dashboard)  
[[Spring]Bean Overview
](https://docs.spring.io/spring-framework/reference/core/beans/definition.html)