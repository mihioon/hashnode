---
title: "[ 트러블 슈팅 ] 재고 관리 서비스의 동시성 테스트 및 문제 해결(2) - Redis Lock"
seoTitle: "재고 관리 동시성 테스트 방법 - Redis Lock"
seoDescription: "Redis를 사용한 재고 관리 서비스의 동시성 테스트 및 문제 해결 전략"
datePublished: Mon Jul 29 2024 12:16:02 GMT+0000 (Coordinated Universal Time)
cuid: clz6ydix4000e09jo0x9f1bi9
slug: 2-redis-lock
tags: redis, concurrency, lock, redisson

---

# Redis 설정

### Lettuce

> Netty기반(비동기 이벤트 지원) 성능↑ 지연시간↓ PipeLining지원(Redisson 보다 뛰어남)

* spring-data-redis의 기본 구현체
    
* 기본 Spin Lock 사용 - Lock을 대기하는 상황에서, Lock을 획득할 수 있는지 계속 요청
    
    따라서 Lock을 획득하려는 스레드가 많을 경우 Redis에 **부하집중**
    
* Lock에 대한 타임아웃이 없어, Unlock(잠금 해제) 호출을 하지 못한 경우 Dead Lock 유발 가능
    

### Redisson

> Netty기반(비동기 이벤트 지원) 난이도↑ Lock지원 PipeLining지원

* pub/sub 방식
    
    Lock을 당장 획득할 수 없으면 대기
    
    Lock이 획득 가능할 경우 Redis에서 클라이언트로 획득 가능함을 알림
    
* Lock의 lease time 설정이 가능하다.
    
    설정된 lease time이 지난 경우 자동으로 Lock의 소유권을 회수하여 Dead Lock을 방지
    

### Jedis

> 동기처리 PipeLining지원(동기처리이므로 성능이 뛰어나지 않음)

※파이프라이닝(PipeLining): 여러 개의 명령어를 하나의 Request로 묶어 처리하는 기능. Queue에 Request를 추가한 후 순서대로 처리함.

# Lettuce 구현

## Simple Lock & Spin Lock

### 의존성 추가 build.gradle - Lettuce

```java
dependencies { 
    // Redis
	implementation 'org.springframework.boot:spring-boot-starter-data-redis' // redisson에 포함
	//implementation 'org.springframework.boot:spring-boot-starter-data-redis-reactive'
}
```

### RedisConfig.java

레디스 설정 파일

```yaml
  redis:
    host:127.0.0.1 # 로컬 호스트
    port:6379 # 기본 포트
```

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.repository.configuration.EnableRedisRepositories;


@Configuration
@EnableRedisRepositories
public class RedisConfig {
    @Value("{spring.redis.host}")
    private String redisHost;
    @Value("{spring.redis.port}")
    private int redisPort;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory(redisHost, redisPort);
    }

    @Bean
    public RedisTemplate<?, ?> redisTemplate() {
        RedisTemplate<byte[], byte[]> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory());
        return redisTemplate;
    }
}
```

### RedisLockRepository

초기 설정

```java
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;

import java.time.Duration;

@Component
public class RedisLockRepository {
    private RedisTemplate<String, Integer> redisTemplate;

    public RedisLockRepository(final RedisTemplate<String, Integer> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public Boolean lock(final Long key) {
        return redisTemplate
                .opsForValue()
                .setIfAbsent(String.valueOf(key), // key : setIfAbsent로 존재하지 않을 때만 락을 설정
                             1, // value : 더미데이터 아무거나 사용해도 무관
                             Duration.ofMillis(1000)); // 유효기간 3초
    }

    public Boolean unlock(final Long key) {
        return redisTemplate.delete(String.valueOf(key));
    }
}
```

## Simple Lock

> key 선점에 의한 lock 획득 실패 시, 비즈니스 로직을 수행하지 않는 방법

동시에 들어온 요청 중 가장 첫 번째 요청만 성공하고 나머지는 모두 실패하기 때문에, DB커넥션은 1회 요청에 필요한 리소스만 사용하게 된다. 낙관적 락과 유사하지만, 낙관적 락과는 달리 트랜잭션 시작 전에 메모리 기반으로 락을 획득하기 때문에 메모리 사용량이 상대적으로 높은 대신 처리 시간이 빠르다.

성능은 좋겠지만 현재 비즈니스의 요구사항(재고가 있다면 모두 차감할 것)을 충족하는 것은 아니기 때문에 구현하지 않았다.

## Spin Lock

```java
        int retryCnt = 0;
        int MaxCnt = 100;
        while(true){
            if(redisLockRepository.lock(eventId)){
                try {
                    stockService.deductStocks(stocks);
                } finally {
                    redisLockRepository.unlock(eventId);
                    // 락 해제
                }
                break;
            }else{
                retryCnt++;
                if (retryCnt == MaxCnt) throw new Error("");
                // Thread.sleep(100);
            }
        }
```

### 요청이 성공할 때 까지 재시도 간격 없이 무한 반복

`스레드 수 10` | `총 작업 시간: 170ms` | `lockTryCnt : 138`

트랜잭션을 잡고 있는 내내 충돌하기 때문에 상당히 무식한 방법이다. 이를 조정하기 위해 적당히 재시도 간격을 조정한다.

### Thread.sleep으로 적당한 시간을 대기시키기

```java
Thread.sleep(100);
```

`동시요청 : 10` | `총 작업 시간 : 766ms` | `lockTryCnt : 38`

충돌 횟수는 줄어들었지만 작업시간이 상당히 늘어났다(대기시간이 포함되므로). 하지만 이렇게 처리하더라도 만약 트랜잭션이 오래 걸린다면? 그래서 key의 TTL보다 트랜잭션이 길어져 중간에 key가 삭제된다면?

레디스는 I/O성능이 좋은데다 테스트중인 기능의 트랜잭션이 워낙 짧아서 해당 경우를 테스트하기 위해 조금 극단적으로 상황을 설정했다.

```java
int numThreads = 3000; //동시에 실행할 스레드 수
Duration.ofMillis(10) // 유효기간

Expected :10
Actual   :13
```

`동시요청 : 3000` | `총 작업 시간 : 28112ms` | `lockTryCnt : 271523`

그렇다면 트랜잭션이 아직 끝나지 않은 경우 해당 락의 키를 확인해서 TTL을 갱신하면 어떨까?

### 트랜잭션이 유지되는 동안 락 유효기간 주기적 갱신

`동시요청 : 10` | `총 작업 시간: 776ms` | `lockTryCnt : 42`

```java
private final Set<String> activeLocks = new HashSet<>();
--- // 락획득 성공
        if (success) {
            activeLocks.add(String.valueOf(key));
        }
--- // 락 삭제
    activeLocks.remove(String.valueOf(key));
--- // 스케줄링 실행
    @Scheduled(fixedRate = 2000) // 2초마다 실행
    public void renewLocks() {
        log.info("🌿==========Scheduling => 작업");
        for (String key : activeLocks) {
            log.info("🌿==========Scheduling => renewLocks / key :" + key);
            redisTemplate.expire(key, Duration.ofMillis(3000));
        }
    }
```

스케줄링 간격보다 트랜잭션 시간이 짧으면 스케줄링 확인이 되지 않으므로 서비스 로직에 `Thread.sleep(스케줄링 간격보다 긴 시간);` 을 추가하면 확인이 가능하다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1721906472794/ecc58a1a-3e0a-484c-9c9e-f157060a76d6.png align="center")

해당 방법을 추가해도 작업시간이나 락 획득 시도 횟수는 크게 차이가 없지만, 당연히 스케줄링을, 그것도 key 유효기간에 맞춰 짧은 간격으로 반복하면 리소스가 낭비된다.

심지어 이렇게 시도해도(스케줄링 테스트 TTL 10ms로 진행, 스케줄링 간격은 5ms초로 설정) 테스트는 가끔 실패한다. 운 나쁘게 스케줄링과 key의 유효기간 만료시간이 맞물리는 경우 당연히 key가 만료되고 다른 트랜잭션의 접근이 가능하기 때문이다.

다행히 Redisson을 사용하면 락을 자동으로 갱신해주는 등의 기능을 지원하므로 굳이 해당 기능이 필요하다면 Redisson을 이용하는 것이 낫다.

### 유의할 점

1. **스케줄링 간격**: 트랜잭션 시간이 스케줄링 간격보다 짧아야 한다. 그러나 웬만하면 사용 X
    
2. **락 유효기간**: 락의 유효기간이 충분히 길어야 한다.(일반적인 트랜잭션 길이보다 길 것)
    

로컬에서 테스트 시 레디스는 3000개 요청까지도 가볍게 처리하지만, 그렇기에 일반적인 재고관리(동시요청 10회 가량을 상정)에 굳이 메모리 비용을 추가로 쓰면서 까지 사용할 필요는 없다. Lettuce는 비동기 API를 제공하기 때문에 필요시 해당부분을 고려할 수 있다.

# Redisson 구현

## Pub/Sub

### 의존성 추가 build.gradle - Redisson

[https://helloworld.kurly.com/blog/distributed-redisson-lock/](https://helloworld.kurly.com/blog/distributed-redisson-lock/) 해당 글을 참고했다.

```java
dependencies { 
    // Redis
	implementation 'org.redisson:redisson-spring-boot-starter'
	//implementation 'org.springframework.boot:spring-boot-starter-data-redis' // redisson에 포함
	//implementation 'org.springframework.boot:spring-boot-starter-data-redis-reactive'
}
```

### RedissonConfig.java

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import org.redisson.Redisson;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;

/*
 * RedissonClient Configuration
 */
@Configuration
public class RedissonConfig { // Redisson 클라이언트 인스턴스를 생성하고 구성
    @Value("${spring.data.redis.host}")
    private String redisHost;
    @Value("${spring.data.redis.port}")
    private int redisPort;

    private static final String REDISSON_HOST_PREFIX = "redis://";

    @Bean
    public RedissonClient redissonClient() { // Redisson 라이브러리를 사용하여 Redis 서버와 통신하는 주요 객체
        Config config = new Config();
        config.useSingleServer().setAddress(REDISSON_HOST_PREFIX + redisHost + ":" + redisPort) // 주소설정
                .setRetryInterval(1500)  // 연결 실패 시 재시도 간격 : 기본 값 1.5초
                .setRetryAttempts(10);    // 연결 실패 시 재시도 횟수 : 기본값 10
        return Redisson.create(config);
    }
}
```

### RedissonLock.java Custom Annotation

```java
import java.lang.annotation.*;

@Target({ElementType.METHOD, ElementType.TYPE}) // 클래스 전체나 특정 메서드
@Retention(RetentionPolicy.RUNTIME) // 런타임에도 유지되어야 함 - 리플렉션을 통해 런타임에 어노테이션 정보를 조회
@Documented // API 문서(JavaDoc)에 어노테이션 정보가 포함
public @interface RedissonLock {
    String value(); // Lock의 이름 (고유값)
    long waitTime() default 5000L; // Lock획득을 시도하는 최대 시간 (ms)
    long leaseTime() default 2000L; // 락을 획득한 후, 점유하는 최대 시간 (ms)
}
```

### RedissonLockAspect.java

```java
@Slf4j
@Aspect // AOP 애스펙트
@Component
@RequiredArgsConstructor
public class RedissonLockAspect {

    private final RedissonClient redissonClient;

    @Around("@annotation(주소.infrastructure.redis.RedissonLock)")
    public void redissonLock(ProceedingJoinPoint joinPoint) throws Throwable {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        RedissonLock annotation = method.getAnnotation(RedissonLock.class);
        String lockKey = annotation.value();

        RLock lock = redissonClient.getLock(lockKey);

        TransactionLogger.plusLockTryCnt(); // 테스트용 코드

        boolean lockable = false;
        try {
            lockable = lock.tryLock(annotation.waitTime(), annotation.leaseTime(), TimeUnit.MILLISECONDS);
            if (!lockable) {
                log.info("Lock 획득 실패={}", lockKey);
                return;
            }
            log.info("로직 수행");
            joinPoint.proceed(); // @Around 어노테이션이 적용된 메서드 실행
        } catch (InterruptedException e) {
            log.info("에러 발생");
            throw e;
        } finally {
//            if (lockable) {  // 잠금이 성공적으로 획득된 경우에만 해제
                log.info("락 해제");
                lock.unlock();
//            }
        }
    }
}
```

## 락을 하나로 관리하기

`동시요청 : 10` | `총 작업 시간: 42ms` | `lockTryCnt : 9`

```java
@RedissonLock(value = "1")
```

## 락을 여러개로 관리하기

비관적 락 처럼 각 재고 물품에 락을 거는 형태로 구현할 수도 있다. try ~ catch 블록으로 스핀락에서 while문(Redisson이 지원하므로)을 없앤 정도의 형태가 되는데, 이미 트랜잭션 전체를 락으로 잡아도 충분히 빠르기 때문에 굳이...이 정도로 구현할 필요는 없다.

Redis를 익히기 위해 코드를 구현해보긴 했지만, Redis는 분산된 환경에서 여러 노드가 동시에 자원에 접근할 때 유용하다. 즉, 단일 서버 환경에서는 굳이 Redis를 사용하여 락을 구현할 필요가 없다.

재고 관리 시스템에서 대용량 요청이 발생하지 않는 상황을 가정하고 있기 때문에, 데이터베이스 락을 사용하여 충분히 구현할 수 있다. 특히, 전체 트랜잭션에 락을 거는 것보다는 물품별로 락을 거는 것이 더 효과적이다. 동시성 이슈가 발생하는 상황에서 재고 차감을 최대한 진행해야 하므로, 충돌 시 일정 시간 후 로직을 재실행하여 트랜잭션을 오래 유지하는 것보다는 비관적 락(pessimistic lock)을 사용하는 것이 더 적합하다고 최종적으로 판단했다.

#### 참고자료

[https://techblog.woowahan.com/17416/](https://techblog.woowahan.com/17416/)  
[https://helloworld.kurly.com/blog/distributed-redisson-lock/](https://helloworld.kurly.com/blog/distributed-redisson-lock/)  
[https://velog.io/@juhyeon1114/Spring-RedisRedisson-분산락을-활용하여-동시성-문제-해결하기](https://velog.io/@juhyeon1114/Spring-RedisRedisson-%EB%B6%84%EC%82%B0%EB%9D%BD%EC%9D%84-%ED%99%9C%EC%9A%A9%ED%95%98%EC%97%AC-%EB%8F%99%EC%8B%9C%EC%84%B1-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0)