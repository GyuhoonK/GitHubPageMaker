---
layout: post
current: post
cover: assets/built/images/spring/injection.jpg
navigation: True
title: DIP(Dependency Inversion Principle),의존관계 역전 원칙
date: 2024-01-05 22:30:00 +0900
tags: [spring]
class: post-template
subclass: 'post tag-spring'
author: GyuhoonK
---

의존관계 역전 원칙과 이를 지키기 위한 의존관계 주입(Depdendency Injection, DI)에 대해서 알아봅니다

## DIP란?

> 객체 지향 프로그래밍에서 의존관계 역전 원칙은 소프트웨어 모듈들을 분리하는 특정 형식을 지칭한다. 이 원칙을 따르면, 상위 계층(정책 결정)이 하위 계층(세부 사항)에 의존하는 전통적인 의존관계를 반전(역전)시킴으로써 상위 계층이 하위 계층의 구현으로부터 독립되게 할 수 있다. 이 원칙은 다음과 같은 내용을 담고 있다.  
첫째, 상위 모듈은 하위 모듈에 의존해서는 안된다. 상위 모듈과 하위 모듈 모두 추상화에 의존해야 한다.  
둘째, 추상화는 세부 사항에 의존해서는 안된다. 세부사항이 추상화에 의존해야 한다.  
이 원칙은 '상위와 하위 객체 모두가 동일한 추상화에 의존해야 한다'는 객체 지향적 설계의 대원칙을 제공한다. [출처](https://ko.wikipedia.org/wiki/%EC%9D%98%EC%A1%B4%EA%B4%80%EA%B3%84_%EC%97%AD%EC%A0%84_%EC%9B%90%EC%B9%99)

DIP, 의존관계 역전 원칙은 객체지향 설계를 위해 적용되어야는 원칙들 중 하나입니다.  `역전(inversion)`이란 단어를 사용한 이유는 전통적인 의존관계를 역전시켰기 때문입니다. 이러한 역전은 상위 모듈과 하위 모듈은 독립될 수 있고, 구체(구현체 클래스)에 의존하지 않고 추상화(인터페이스)에만 의존하도록 합니다.


## 전통적인 의존 관계

DIP에서 말하는 전통적인 의존 관계란 상위 클래스와 하위 클래스 간의 의존관계가 구현체(구체 클래스)에 의해 결정되는 경우입니다.  
예를 들어, 상위 클래스가 `OrderService`(주문 서비스)이고, 하위 클래스가 `DiscountPolicy`(할인 정책)라고 가정합니다.

하위 클래스 `DiscountPolicy`는 비율 할인과 정액 할인이 존재합니다. 비율 할인과 정액 할인은 각각 구체 클래스로 아래와 같이 작성했습니다.

```java
/* discount/FixDiscountPolicy.java */
public class FixDiscountPolicy implements DiscountPolicy{
    private int discountFixAmount = 1000; // 1000원 정액 할인
    @Override
    public int discount(Member member, int price) {
        if (member.getGrade() == Grade.VIP){
            return discountFixAmount;
        } else
            return 0;
    }
}
```
```java
/* discount/RateDiscountPolicy.java */
public class RateDiscountPolicy implements DiscountPolicy {
    private int discountPercent = 10;
    @Override
    public int discount(Member member, int price) { // cmd + shift + t
        if (member.getGrade() == Grade.VIP){
            return price * discountPercent / 100;
        }
        return 0;
    }
}
```

전통적인 의존관계에서는 `OrderService`의 구현체 작성 시에 하위 구체 클래스를 직접 사용합니다. 정액 할인 정책을 적용하여 주문 서비스를 구현한다면 아래와 같이 작성할 것입니다

```java
/* order.OrderServiceImpl.java */
public class OrderServiceImpl implements OrderService {
   private final MemberRepository memberRepository = new MemoryMemberRepository();
   private final DiscountPolicy discountPolicy = new FixDiscountPolicy();

    @Override
    public Order createOrder(Long memberId, String itemName, int itemPrice) {
        Member member = memberRepository.findById(memberId);
        int discountPrice = discountPolicy.discount(member, itemPrice);
        return new Order(memberId, itemName, itemPrice, discountPrice);
    }
}
```
상위 계층의 구현(`OrderServiceImpl`) 시에 하위 계층인 할인 정책의 구현체(`FixDiscountPolicy`)를 직접 참조하고 있습니다. 만약 할인 정책을 비율 할인으로 변경하고 싶다면, 상위 계층 구현체(`OrderServiceImpl`)를 직접 수정해야합니다.

```java
/* order.OrderServiceImpl.java */
public class OrderServiceImpl implements OrderService {
   private final MemberRepository memberRepository = new MemoryMemberRepository();
//    private final DiscountPolicy discountPolicy = new FixDiscountPolicy();
   private final DiscountPolicy discountPolicy = new RateDiscountPolicy();

    @Override
    public Order createOrder(Long memberId, String itemName, int itemPrice) {
        Member member = memberRepository.findById(memberId);
        int discountPrice = discountPolicy.discount(member, itemPrice);
        return new Order(memberId, itemName, itemPrice, discountPrice);
    }
}
```

위처럼 할인 정책 변경을 위해 상위 구현체를 직접 수정하는 접근 방식은 객체 지향 프로그래밍의 원칙에 어긋납니다. 

전통적인 의존관계에서 `OrderServiceImpl`의 역할은 2가지입니다.
- 주문 서비스의 구현: OrderServiceImpl을 작성한 본래 목적이며 가장 중요한 역할입니다
- 하위 계층(할인 정책)의 선택: 이는 OOP의 관점에서 볼 때 불필요한 역할입니다

객체 지향 프로그래밍은 클래스 간의 독립적 디자인을 중요시합니다. 따라서 `OrderServiceImpl`의 클래스 디자인에서 다른 클래스를 참조하는 것은 객체 지향 프로그래밍의 원칙을 위배하고 있으므로 제거해야할 역할입니다. 

## 해결책: 의존성 주입(Dependency Injection)

먼저, 하위 계층(구체 클래스)을 직접 참조하지 않기 위해 인터페이스 간의 의존관계만을 정의합니다. 

```java
/* order.OrderServiceImpl.java */
public class OrderServiceImpl implements OrderService {
   private final MemberRepository memberRepository;
   private final DiscountPolicy discountPolicy;

    @Override
    public Order createOrder(Long memberId, String itemName, int itemPrice) {
        Member member = memberRepository.findById(memberId);
        int discountPrice = discountPolicy.discount(member, itemPrice);
        return new Order(memberId, itemName, itemPrice, discountPrice);
    }
}
```

위 코드에서 구체 클래스가 등장하지 않습니다. 인터페이스에 대한 의존관계만을 정의합니다. 이런 경우 실제 구체 클래스는 어떻게 생성해주어야할까요? 외부에서 구체 클래스를 선택하도록 구현해야합니다. `AppConfig`로 의존관계를 작성하겠습니다.

```java
public class AppConfig {
    public OrderService orderService() {
        return new OrderServiceImpl(new MemoryMemberRepository(), new FixDiscountPolicy());
    }
}
```

`AppConfig`는 어떤 구체 클래스를 적용할지 결정하는 역할을 합니다. `OrderServiceImpl`에 생성자(constructor)를 추가해주겠습니다.

```java
/* order.OrderServiceImpl.java */
public class OrderServiceImpl implements OrderService {
   private final MemberRepository memberRepository;
   private final DiscountPolicy discountPolicy;

    @Override
    public Order createOrder(Long memberId, String itemName, int itemPrice) {
        Member member = memberRepository.findById(memberId);
        int discountPrice = discountPolicy.discount(member, itemPrice);
        return new Order(memberId, itemName, itemPrice, discountPrice);
    }
    // 생성자 추가
    public OrderServiceImpl(MemberRepository memberRepository, DiscountPolicy discountPolicy) {
        this.memberRepository = memberRepository;
        this.discountPolicy = discountPolicy;
    }
}
```

`AppConfig`에서 하위 계층(할인 정책,`FixDiscountPolicy`)을 생성하고, 이를 이용하여 상위 계층(주문 서비스, `OrderServiceImpl`)을 생성합니다.  
구체 클래스 간의 의존관계(전통적인 의존관계)는 존재하지 않으며, 오직 인터페이스(`DiscountPolicy, OrderService`)간의 의존관계만이 존재합니다.

이처럼 인터페이스 간의 의존관계만을 이용하고, 구체 클래스의 선택을 클래스 내부가 아니라 외부에서 작성해두는 방법을 의존관계 주입(Dependency Injection, DI)이라고 부릅니다.

 DI를 이용하는 경우 DIP를 준수할 수 있게 됩니다. 즉, 상위 계층이 하위 계층에 의존하지 않을 수 있습니다.

## Spring에서의 사용 방법(Spring Container)

Spring은 작성된 `AppConfig`를 어디서든 불러서 사용할 수 있게끔 관리해줍니다. Spring Container를 생성하고, AppConfig에 작성해두었던 구체 클래스의 의존 관계를 저장해둡니다.

```java
@Configuration
public class AppConfig {

    @Bean
    public MemberRepository memberRepository() {
        return new MemoryMemberRepository();
    }

    @Bean
    public DiscountPolicy discountPolicy(){
       return new FixDiscountPolicy();
    }

    @Bean
    public OrderService orderService() {
        return new OrderServiceImpl(memberRepository(), discountPolicy());
    }
}
```
이렇게 작성해두면 스프링 컨테이너 내부에서 구체 클래스를 생성한 뒤, 구체 클래스 간 의존 관계를 자동으로 설정합니다.  
구체적으로 설명하자면, `MemoryMemberRepository`, `FixDiscountPolicy`를 먼저 생성한 뒤에 이를 이용하여 `OrderService`의 생성해 사용합니다.

이렇게 생성된 Bean들은 아래와 같이 호출할 수 있습니다. 

```java
AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(SameBeanConfig.class);

MemberRepository memberRepository = ac.getBean("memberRepository", MemberRepository.class);
DiscountPolicy discountPolicy = ac.getBean("discountPolicy", DiscountPolicy.class);
OrderService orderService = ac.getBean("orderService", OrderService.class);
```



## 정리
- 전통적인 의존관계 정의는 OOP 원칙을 준수할 수 없다
- DIP는 인터페이스 간의 의존관계만을 정의한다
- 구체 클래스의 선택을 외부에 위임한다

[참고]  
[[inflearn]스프링 핵심 원리 - 기본편](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B8%B0%EB%B3%B8%ED%8E%B8/dashboard)  
[[wiki]의존관계 역전 원칙](https://ko.wikipedia.org/wiki/%EC%9D%98%EC%A1%B4%EA%B4%80%EA%B3%84_%EC%97%AD%EC%A0%84_%EC%9B%90%EC%B9%99)  
[[wiki]객체 지향 프로그래밍](https://ko.wikipedia.org/wiki/%EA%B0%9D%EC%B2%B4_%EC%A7%80%ED%96%A5_%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D)  