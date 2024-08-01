---
title: "[E-commerce] Point Service"
seoTitle: "E-commerce Service Point"
seoDescription: "Simplify e-commerce service logic by optimizing transactions, validating user points, and using optimistic locking with retries"
datePublished: Thu Aug 01 2024 19:42:18 GMT+0000 (Coordinated Universal Time)
cuid: clzbomyu5000009l725ib4crh
slug: e-commerce-point-service
tags: ecommerce, lock, facade-pattern

---

# 포인트 충전 파사드?

```java
/**
 * Spring Data JPA에서 낙관적락 사용시
 * 버전 충돌 시 Hibernate는 StaleStateException을 발생시킴
 * Spring은 이를 OptimisticLockingFailureException으로 래핑하여 처리
 * */
@Slf4j
@Component
@RequiredArgsConstructor
public class ChargePointUseCase {
    private final CustomerService customerService;
    private final PointHistoryService pointHistoryService;

    @Transactional
    @Retryable(
            retryFor = {ObjectOptimisticLockingFailureException.class},
            maxAttempts = 3,
            backoff = @Backoff(100)
    )
    public void chargePoint(Long customerId, BigDecimal point){
        // 사용자 여부 확인
        User user = userService.findUser(userId);

        // 충전 내역 저장
        LocalDateTime dateTime = LocalDateTime.now();
        PointHistory pointHistory = new PointHistory(customerId, dateTime, PointType.CHARGE, point);
        pointHistoryService.savePointHistory(pointHistory);

        // 사용자 금액 업데이트
        customer.chargePoint(point);
        customerService.updateChargeCustomer(customer);
    }

    @Recover
    public void recover(ObjectOptimisticLockingFailureException e) {
        // 포인트 충전 실패 로직
        log.error("포인트 충전 실패: ", e);        
        throw 커스텀 에러;
    }
}
```

먼저 코드에 조의를 표한다...R.I.P...

애당초 파사드가 필요없는 간단한 로직을 파사드로 구현해야하는 줄 알고 제대로 이해도 못 한 상태로 억지로 쥐어짜내려니 쓸데없이 기능에 비해 복잡한 코드가 나왔다. <s>이딴거 다 필요없다.</s>

## 파사드는 독립적인 도메인 서비스들의 결합!

PointHistory? 포인트가 뭐지? 포인트는 애당초 User에 속한 도메인이다. User를 벗어날 수 없고, Point하면 무조건 User꺼다. 그러니까 point서비스에 user관련 레포지토리가 당연히 있어도 된다. 굳이 파사드로 복잡하게 구현할 이유가 없다.

## 싹 다 서비스레이어로 내릴게요...

내리는 김에 고민을 해보자. 포인트를 충전하거나 사용하고 금액 내역을 업데이트하는 떨어질 수 있는 로직인가? 포인트 저장이 실패하면 당연히 내역을 저장할 필요가 없다. 포인트 저장에 성공하면 당연히 내역을 저장해야한다. 둘 중 하나만 성공하는 경우는 없다.

### 절대로 떨어질 수 없는 기능은 하나의 기능이다.

그렇다면 이건 하나의 트랜잭션으로 묶여야 하는 기능이자, 어쩌면 레포지토리 단위로 내려가도 괜찮은 기능이다. 항상 같이 요청할 건데 따로 요청할 수 있도록 만들 이유가 당연히 없다.

### 트랜잭션은 가능한 한 하위 레이어에 위치해야한다.

트랜잭션이 짧을 수록 서버 리소스를 덜 사용하기 때문에, 트랜잭션은 짧을수록, 하위 레이어 메서드 단위에 위치할수록 효율적이다. 특히 여기선 낙관적 락을 채택했기 때문에 어차피 충돌이 일어나는 건 트랜잭션의 끝 부분이라 굳이 전체 로직을 하나의 단위로 묶지 않아도 된다.

해당 기능에서 실제로 DB의 데이터에 변동이 있는 부분은 포인트를 저장하고 히스토리를 저장하는 부분이다.

즉, 하나의 단위로 묶여 롤백되어야 하는 부분은 포인트 저장 및 히스토리 저장이고, 나머지 부분은 트랜잭션 단위에 포함되지 않아도 된다. 따라서 트랜잭션은 레포지토리 단위로 내려가도 무방하다.

# PointService

```java
    // 포인트 충전
    public void chargePoint(PointCommand pointCommand){
        // 포인트 유효성 검사
        pointCommand.validate();
        // 사용자 금액 업데이트 및 내역 저장
        userRepo.chargePointAndSaveHistory(pointCommand);
    }

    // 포인트 사용
    public void usePoint(PointCommand pointCommand){
        // 유저 검색
        User user = userRepo.findById(pointCommand.getCustomerId())
                .orElseThrow(() -> new InputValidationException(USER_IS_NOT_FOUND.getMessage()));
        // 포인트 사용 유효성 검사
        user.validPoint(pointCommand.getPoint());
        // 사용자 금액 업데이트 및 내역 저장
        userRepo.usePointAndSaveHistory(pointCommand);
    }
```

* **PointCommand.**validate  
    충전 시 포인트가 0이상이어야 하는 유효성 검사는 PointCommand의 역할이다.
    
* **User.**validPoint
    
    그러나 사용 시 사용할 포인트가 사용자의 포인트보다 작거나 같은지 비교하는 것은 User의 역할이라고 판단했다.
    

사용자 금액을 업데이트하고 포인트 내역을 저장하는 건 앞서 말했듯이 동시에 일어나는 작업이므로 레포지토리에서 한 번에 처리하도록 한다.

```java
private Long customerId;
private LocalDateTime dateTime;
private PointType type;
private BigDecimal point; /* 금액 */
```

```java
    /**
     * Point
     */
    // 포인트 사용 및 내역 저장
    @Override
    @Transactional
    public void usePointAndSaveHistory(PointCommand pointCommand) {
        // 엔티티 조회 및 포인트 추가
        UserEntity userEntity= userJpaRepo.findById(pointCommand.getUserId())
                .orElseThrow(() -> new NoSuchElementException("UserEntity not found"));
        userEntity.minusPoint(pointCommand.getPoint());
        // 포인트 내역 저장
        saveHistory(pointCommand, PointHistoryEntity.Type.CHARGE);
    }

    // 포인트 충전 및 내역 저장
    @Override
    @Transactional
    public void chargePointAndSaveHistory(PointCommand pointCommand) {
        // 엔티티 조회 및 포인트 차감
        UserEntity userEntity= userJpaRepo.findById(pointCommand.getUserId())
                .orElseThrow(() -> new NoSuchElementException("UserEntity not found"));
        userEntity.plusPoint(pointCommand.getPoint());
        // 포인트 내역 저장
        saveHistory(pointCommand, PointHistoryEntity.Type.USE);
    }
```

마이너스와 플러스같은 계산은 다 레포지토리에게 넘겨버리고 도메인은 검증만 하도록 구현했다. 포인트를 적절하게 가공해서 저장하는 기능은 userEntity의 영역이라고 생각하여 데이터를 수정할 메서드를 만들어주었다.

# 낙관적 락?

## 포인트 로직에서 낙관적 락은 적절한 선택

다수의 사용자가 같은 공유데이터에 접근하는 것이 아니라, 유저 별 각각의 포인트에 접근하는 것이기 때문에 충돌 가능성이 비교적 낮다. 일반적인 상황이라면 충전과 동시에 사용하는 정도라 낙관적락과 retry로 충분하다고 판단했다.

## 근데 Retry를 3회나 해?

3회도 많다! 3회까지 왜 가냐? 그 정도의 문제라면 비관적락을 써야 한다. 직접 테스트를 해보며 짧은 트랜잭션에서 retry가 얼마나 비효율적인지 깨달았다.

```java
@Retryable(
        retryFor = {ObjectOptimisticLockingFailureException.class},
        maxAttempts = 1, // 재시도 횟수
        backoff = @Backoff(100) // 재시도 간격
)
```

어차피 중복결제 오류는 별도의 요청 코드로 관리되어야하는 부분이고, 그 이상은 비정상적인 요청이나 악의적인 공격이다. 따라서 재시도 횟수는 1번으로 처리한다. 재시도 간격의 경우 우선은 100ms로 두고, 이후 조정해 볼 수 있을 것 같다.