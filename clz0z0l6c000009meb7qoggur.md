---
title: "Transaction Log 출력하기"
seoTitle: "Transaction Log AOP"
seoDescription: "포인트컷을 통한 트랜잭션 로그 출력 방법과 AOP 개념 설명"
datePublished: Thu Jul 25 2024 07:47:21 GMT+0000 (Coordinated Universal Time)
cuid: clz0z0l6c000009meb7qoggur
slug: transaction-log
tags: logging, spring-aop, aop, transactional

---

# Transaction Logging AOP

## 포인트 컷(Pointcut)

* 조인 포인트 중에서 어드바이스가 적용될 위치를 선별하는 기능
    
* 어떤 메소드에 어드바이스를 적용할지를 결정
    
* 스프링 AOP는 프록시 방식을 사용하므로 메소드 실행 지점만 포인트 컷으로 선별 가능  
    ➔ 메소드 호출 시점에만 어드바이스가 적용될 수 있음
    

## 포인트컷 표현식(AspectJ pointcut expression)

```java
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

@Slf4j
@Aspect // 해당 어노테이션이 제공
@Component
public class TransactionLogger {
    
    @Pointcut("execution(* 주소.domain.product.StockService.*(..)) && @annotation(org.springframework.transaction.annotation.Transactional)")
    public void transactionalMethods() {}

    @Before("transactionalMethods()")
    public void beforeTransaction() {
        log.info("💚========> Transaction started");
    }

    @After("transactionalMethods()")
    public void afterTransaction() {
        log.info("❤️========> Transaction completed");
    }
}
```

## Console log

귀.엽.다!

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1721892485976/bfc9df11-990c-4eae-956e-b0c7081da3fb.png align="center")

#### 참고자료

[https://velog.io/@gwichanlee/AOP-Pointcut](https://velog.io/@gwichanlee/AOP-Pointcut)