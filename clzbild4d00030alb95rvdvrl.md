---
title: "[트러블 슈팅] @Retryable과 @Recover"
seoDescription: "Exploring solutions for handling @Retryable and @Recover annotations in Spring Retry, focusing on consistent error codes and class segregation"
datePublished: Thu Aug 01 2024 16:53:05 GMT+0000 (Coordinated Universal Time)
cuid: clzbild4d00030alb95rvdvrl
slug: retryable-recover
tags: springboot, annotations

---

# Recover 애너테이션

`@Retryable` 애너테이션을 통해 정의된 메서드에서 예외가 발생했을 때, 지정된 횟수만큼 재시도하고, 여전히 예외가 발생하는 경우 `@Recover` 애너테이션이 붙은 메서드가 호출된다.

## 문제 상황

Spring Retry는 `@Recover` 메서드의 파라미터 목록을 통해 어떤 메서드에서 발생한 예외를 처리할지 결정한다. `@Recover` 메서드는 예외 타입 외에도, `@Retryable` 메서드의 파라미터 타입과 일치하는 추가 파라미터를 가질 수 있다.

그런데...

```java
// 포인트 충전
@Retryable(
        retryFor = {ObjectOptimisticLockingFailureException.class},
        maxAttempts = 1, // 재시도 횟수
        backoff = @Backoff(100) // 재시도 간격
)
public void chargePoint(PointCommand pointCommand){
    ...
}

// 포인트 사용
@Retryable(
        retryFor = {ObjectOptimisticLockingFailureException.class},
        maxAttempts = 1, // 재시도 횟수
        backoff = @Backoff(100) // 재시도 간격
)
public void usePoint(PointCommand pointCommand){
    ...
}
```

비극이다... 발생하는 오류도 메서드의 파라미터 타입도 동일하다... 그래서 실제로 별도로 `@Recover` 메서드를 둬도 동일한 `@Recover` 메서드를 실행한다.

## 해결 방법

### 동일한 에러코드 사용하기

충전이나 실패에 상관없이 그냥 동시성 문제로 실패했다는 에러만 반환한다.

```java
@Recover
public void recover(ObjectOptimisticLockingFailureException e) {
    // 포인트 로직 실패 에러
    throw new InputValidationException(POINT_CONCURRENCY_FAILURE.getMessage());
}
```

### 클래스 분리하기

하나의 클래스에 예외와 파라미터 타입이 동일한 `@Retryable` 로직이 있어서 발생하는 문제기 때문에, 분리하면 간단하게 해결할 수 있다. 그러나 이런 식이면 낙관적 락을 구현하고 `@Retryable` 메서드를 실행해야하는 일이 있을 때 마다 클래스를 분리해야한다.

### AOP 설정

<s>돌고돌아 에.오.피...</s>

이렇게 하면 모든 Retryable의 ObjectOptimisticLockingFailureException 예외 처리를 한 번에 할 수 있다.

```java
@Aspect
@Component
public class ConcurrencyRecoverAspect {

    @Around("@annotation(org.springframework.retry.annotation.Retryable)")
    public Object handleRetryableMethod(ProceedingJoinPoint joinPoint) throws Throwable {
        try {
            return joinPoint.proceed();
        } catch (ObjectOptimisticLockingFailureException ex) {
            String methodName = joinPoint.getSignature().getName();
            throw new ConcurrencyException(POINT_CONCURRENCY_FAILURE.getMessage() + methodName);
        }
    }
}
```

## 결국 동일한 에러코드 사용했다...

어쨌든 현재 낙관적 락은 포인트 로직에서만 사용하고 있어서 AOP까지 설정할 필요는 없었고, 포인트 동시성 처리 관련 에러는 발생 빈도가 낮을 것으로 예상되기 때문이다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1722531260236/0dbf0296-68af-4810-8b99-9d65e6accf8f.png align="center")