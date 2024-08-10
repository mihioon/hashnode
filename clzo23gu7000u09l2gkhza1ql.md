---
title: "Proxy와 Annotation"
seoDescription: "Understanding Proxy and Annotation usage in Spring, including JDK and CGLIB proxies, and methods for applying annotations internally"
datePublished: Sat Aug 10 2024 11:32:17 GMT+0000 (Coordinated Universal Time)
cuid: clzo23gu7000u09l2gkhza1ql
slug: proxy-annotation
tags: proxy, springboot, annotations, spring-aop

---

# Proxy

## 역할

> 어노테이션은 기능을 활성화하거나 설정, 프록시는 실제로 그 기능을 구현하는 역할

어노테이션이 적용된 메서드나 클래스에 대해 추가로직을 삽입하거나, 메서드 호출을 가로채 특정 기능을 수행

## 동작 방식

### **JDK 동적 프록시**

* 인터페이스 기반 : 인터페이스를 구현하는 프록시 객체를 생성
    
* `InvocationHandler` 인터페이스를 구현하여 메서드 호출을 가로챔
    
* 인터페이스가 없는 클래스는 적용 불가능
    

### **CGLIB 프록시**

* 클래스 기반 : 클래스 상속을 통해 프록시 객체 생성
    
* 원래 클래스의 서브클래스를 생성하고, 원래 클래스의 메서드를 오버라이드 하여 메서드 호출을 가로챔
    
* 오버라이드 된 메서드에서 추가로직을 수행한 후 원래 메서드를 호출
    
* final 메서드나 클래스에는 적용 불가능(오버라이드나 상속이 불가능하기 때문)
    

## 생성되는 경우

### AOP 어노테이션

`@Aspect`, `@Before`, `@After`, `@Around`

### 트랜잭션 관리 어노테이션

`@Transactional`

### 비동기 처리

`@Async`

# 클래스 내부에서 어노테이션이 작동하지 않는 이유

외부에서 메서드를 호출할 때, 적용된 어노테이션을 감지하고, 프록시 객체를 생성한 후 해당 어노테이션이 해야할 역할을 수행한다.

그런데 내부에서 메서드를 호출하면 프록시 객체를 거치지 않고 직접 메서드를 호출하기 때문에, 당연히 해당 작업이 불가능하다.

# 클래스 내부에서도 어노테이션을 적용하는 방법

프록시 객체를 거치지 않아 어노테이션 적용이 안 되는 거라면, 프록시 객체를 만들어주면 된다.

## 자기 자신을 주입받기

클래스 내부에서 호출해도 프록시 객체를 통해 호출될 수 있도록 외부에서 호출한 것처럼 자신을 주입받아 호출하는 방법이다.

스프링 컨테이너가 클래스의 프록시 객체를 생성하여 내부에 주입하고 관리한다.

스프링 컨테이너는 프록시 객체를 한 번만 생성하므로, 동일한 프록시 객체를 재사용한다.

```java
public class ThisClass {
    @Autowired
    private ThisClass this;

    public void method_A() {
        this.method_B(); // 프록시 객체를 통해 호출
    }

    @Transactional
    public void method_B() {
        // 트랜잭션이 적용됨
    }    
}
```

## applicationContext.getBean(클래스명.class)

애플리케이션 컨텍스트에서 특정 클래스 타입의 빈을 동적으로 가져와 사용하는 방법이다.

스프링 컨테이너가 등록해 둔 애플리케이션 컨텍스트를 통해 클래스의 프록시 객체를 동적으로 가져와 호출한다.

애플리케이션 컨텍스트를 주입받아야 사용가능하다.

```java
public class ThisClass {
    @Autowired
    private ApplicationContext applicationContext;

    public void method_A() {
        ThisClass this
            = applicationContext.getBean(ThisClass.class);

        this.method_B();
    }

    @Transactional
    public void method_B(){
        // 트랜잭션이 적용됨
    }
}
```

## AopContext.currentProxy()

현재 실행 중인 프록시 객체를 직접 가져오는 방법이다.

따로 주입 없이 스프링 AOP가 활성화 되어있기만 하면 사용 가능하다.

```java
public class ThisClass {
    public void method_A() {
        ThisClass this = (ThisClass) AopContext.currentProxy();

        this.method_B();
    }

    @Transactional
    public void method_B() {
        // 트랜잭션이 적용됨
    }
}
```

## **AOP 설정 변경**

AOP 설정을 변경하여 클래스 내부 호출에도 프록시가 적용되도록 한다.

<s>딱 봐도 몇몇 호출을 위해 전체 호출을 변경하는 것이 위험한 느낌이 든다.</s>

# 어떤 방식을 써야할까?

각각의 장단점이 존재해서 적절히 선택해서 사용하면 될 것 같다.

자기 자신을 주입하는 방식이 많이 선호된다고는 하는데, 코드가 명시적이고 스프링 컨테이너가 주입을 관리하여 일관성이 있기 때문이라고 한다.

개인적으로는 AopContext를 사용하는 방식이 코드가 가장 간결해서 제일 괜찮아보이는데...변수 명만 제대로 적어준다면 이해가 어렵지 않을 것 같기도 하고? 스프링 컨테이너의 관리라는 측면은 아직은 와닿지 않아서, 좀 더 생각해 볼 부분인 것 같다.