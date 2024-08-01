---
title: "Composed-annotation"
seoTitle: "Annotation Composition"
seoDescription: "Overview of custom annotations, meta-annotation, and composed-annotation in Java with practical examples and considerations"
datePublished: Thu Aug 01 2024 16:09:25 GMT+0000 (Coordinated Universal Time)
cuid: clzbh17fw000o08ih7vus7zf4
slug: composed-annotation
tags: springboot, annotations

---

# 합성 애너테이션

`@Recover` 애너테이션으로 씨름하다가 커스텀 애너테이션을 만들면 어떨까? 하고 관련 정보를 찾아봤다.

Java에서는 애너테이션 상속이 지원되지 않는데, 애너테이션 합성을 통해 유사한 기능을 구현할 수 있다. 애너테이션 합성을 사용하면 커스텀 애너테이션을 정의하고, 해당 애너테이션이 다른 애너테이션을 포함하도록 할 수 있다.

## 메타 애너테이션(Meta-annotation)

> 애너테이션 위에 붙은 애너테이션

## 합성 애너테이션(Composed-annotation)

> 메타 애너테이션을 한 개 이상을 적용해서 만든 애너테이션

합성 애너테이션을 만들기 위해선 반드시 메타 애너테이션 `@Target` 과 `@Retention`을 둬야한다.

`@Target`

애너테이션을 어디에 적용할지 결정(여러개 지정 가능)

* ElementType.TYPE: (ElementType.java 참고)
    

`@Retention`

애너테이션이 어느 시점까지 유지될지 결정

* RetentionPolicy.SOURCE: 소스코드에서만 유지, 컴파일 시에 삭제
    
* RetentionPolicy.CLASS: 기본값, 컴파일된 class file에 포함, 런타임엔 유지X
    
* RetentionPolicy.RUNTIME: 런타임까지 유지, 리플렉션을 통해 애너테이션 정보를 읽을 수 있음
    

### ElementType.java

```java
public enum ElementType {
    /** Class, interface (including annotation interface), enum, or record
     * declaration */
    TYPE,
    /** Field declaration (includes enum constants) */
    FIELD,
    /** Method declaration */
    METHOD,
    /** Formal parameter declaration */
    PARAMETER,
    /** Constructor declaration */
    CONSTRUCTOR,
    /** Local variable declaration */
    LOCAL_VARIABLE,
    /** Annotation interface declaration (Formerly known as an annotation type.) */
    ANNOTATION_TYPE,
    /** Package declaration */
    PACKAGE,
    /**
     * Type parameter declaration
     *
     * @since 1.8
     */
    TYPE_PARAMETER,
    /**
     * Use of a type
     *
     * @since 1.8
     */
    TYPE_USE,
    /**
     * Module declaration.
     *
     * @since 9
     */
    MODULE,
    /**
     * Record component
     *
     * @jls 8.10.3 Record Members
     * @jls 9.7.4 Where Annotations May Appear
     *
     * @since 16
     */
    RECORD_COMPONENT;
}
```

# e.g.

## 스프링제공 합성 애너테이션

이미 익숙하게 사용중인 애너테이션들 중에서도 합성 애너테이션이 존재한다.

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller    // Meta Annotation
@ResponseBody  // Meta Annotation
public @interface RestController {...}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Service {...}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Repository {...}
```

## 실제 구현 예

다음과 같이 추가 변수를 두는 등 커스텀하여 사용할 수 있다.

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Recover
public @interface MethodRecover {
    String name();
}
...
@MethodRecover("원하는 변수")
```

결국 원하던 기능은 아니라 AOP로 구현할 예정이지만...🥲

📘 **참고자료**

[https://velog.io/@gmtmoney2357/스프링-부트4-메타-애노테이션과-합성-애노테이션](https://velog.io/@gmtmoney2357/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%B6%80%ED%8A%B84-%EB%A9%94%ED%83%80-%EC%95%A0%EB%85%B8%ED%85%8C%EC%9D%B4%EC%85%98%EA%B3%BC-%ED%95%A9%EC%84%B1-%EC%95%A0%EB%85%B8%ED%85%8C%EC%9D%B4%EC%85%98)