---
title: "[E-commerce] 주문 Facade EDA 개선"
seoTitle: "Facade EDA Enhancement"
seoDescription: "트랜잭션 분리와 Event Driven Architecture를 통해 주문 시스템의 효율성과 유지보수성을 개선합니다"
datePublished: Wed Aug 07 2024 17:34:22 GMT+0000 (Coordinated Universal Time)
cuid: clzk4pkb9000308lch5qu1pyl
slug: e-commerce-facade-eda
tags: event-driven-architecture

---

# 하나의 원자적 단위였을 때

`포인트 차감` ⇢ `재고 차감` ⇢ `주문 생성` ⇢ `주문 히스토리 저장` ⇢ `외부 결제`

```java
    public void checkOut(OrderRequest orderRequest) {
        Long orderSheetId = orderRequest.getOrderSheetId();

        // 포인트 차감
        PointCommand pointCommand = orderRequest.toPointCommand();
        pointService.usePoint(pointCommand);

        // 재고 차감
        OrderSheet orderSheet = orderSheetService.find(orderSheetId);
        List<StockCommand> stocks = toDomain(orderSheet.getOrderProducts());
        stockService.deductStocks(stocks);

        // 주문 생성
        Long orderId = orderService.createOrder(orderSheetId); // 주문생성

        // 결제
        Payment payment = orderRequest.toPayment();
        paymentService.pay(payment);

        // 주문완료 상태 업데이트
        orderService.updateState(orderId, Order.State.PAID);

        // 주문 히스토리 저장
        orderService.saveHistory(orderId);
    }
```

프로세스 중 하나라도 실패하면 모두 롤백되기 때문에,  
주문의 상태는 언제나 `생성되거나`, `생성되지 않거나`로 정해진다.

## 발생할 수 있는 문제점

그러나 이렇게 모든 로직을 하나의 트랜잭션으로 묶는다면 무분별한 비즈니스 로직과 트랜잭션의 규모에 따라 문제가 발생할 수 있다.

* 한 트랜잭션이 너무 많은 책임(로직)을 포함하여 복잡도 증가
    
* 한 트랜잭션 안에서 특정 작업만 느리거나 다른 요청을 대기시키게 됨(e.g. Lock)
    
* 트랜잭션 범위 내에서 DB 와 무관한 작업이 포함(e.g. 외부 API)
    

하나의 트랜잭션이 너무 많은 책임을 갖고있어 응답시간이 지연되거나, 유지보수가 어려워지고, 모든 로직이 성공해도 중요하지 않은 단 하나의 로직에 의해 실패할 수 있게 된다. 또한 외부서버의 문제로 인해 에러 및 지연이 내부서버로 전파되어 문제를 일으킬 수 있다.

따라서 이를 적절하게 분리하기 위해선 하나의 원자성을 가져야할 중요한 로직들과, 상대적으로 중요하지 않거나 별도의 관리가 필요하여 분리될 수 있는 로직을 구분해야 한다.

# 비즈니스 로직의 관심사 분리

| 핵심 로직 | 핵심 로직 | 비핵심 로직 |
| --- | --- | --- |
| 내부 서버 | 외부 서버 | 내부 서버 |
| `포인트 차감` , `재고 차감` , `주문 생성` | `외부 결제` | `주문 완료 상태 업데이트` , `주문 히스토리 저장` |

외부 결제가 처리되기를 기다린 후에 주문을 생성하는 방식은 트랜잭션을 분리하더라도 여전히 핵심 로직 끼리의 의존성을 가지기 때문에, 주문이 성공하거나, 혹은 결제를 따로 진행하는 경우등 효율적으로 주문 데이터를 관리하기 위해 주문의 상태(state)관리를 추가했다.

# **Event Driven Architecture**

## CheckOut Facade - 이벤트 발행

### 비즈니스 로직에서 처리하지 않은 이유

이벤트 발행을 서비스 로직에서 처리할 수도 있지만, 비즈니스 로직과 완전히 분리하는 방식을 선택했다.  
가장 큰 이유는 비즈니스 로직을 보호하기 위함이다.

* `관심사의 분리 & 단일 책임 원칙`: 서비스 로직은 해당 도메인의 로직만 담당
    

기존 비즈니스 로직을 수정하지 않기 때문에 테스트 코드를 유지하면서, 애플리케이션 레벨에서 이벤트 발행과 수신을 한 눈에 확인할 수 있어 가독성이 좋고 변경의 유연성이 뛰어나다.

```java
public void checkOut(CheckOutRequest checkOutRequest) {
    Long orderSheetId = checkOutRequest.getOrderSheetId();
    OrderSheet orderSheet = orderSheetService.find(orderSheetId);

    // 포인트 차감
    PointCommand pointCommand = checkOutRequest.toPointCommand();
    pointService.usePoint(pointCommand);

    // 재고 차감
    List<StockCommand> stockList = orderSheet.toStockList();
    stockService.deductStocks(stockList);

    // 주문 생성
    Order order = orderSheet.toOrder();
    Long orderId = orderService.createOrder(order);

    // 주문 생성 완료 이벤트 발행
    orderCreatedPublisher.publish(checkOutRequest);
}
```

## OrderCreatedPublisher - 이벤트 발행

```java
public void publish(CheckOutRequest checkOutRequest) {
    OrderCreatedEvent event = new OrderCreatedEvent("OrderCreated", checkOutRequest);
    log.info("이벤트 발행: " + event.getMessage());
    publisher.publishEvent(event);
}
```

이렇게 하면 checkOut의 역할은 끝이 난다. 이후의 로직은 고려할 필요가 없다.

## OrderCreatedListener - 이벤트 수신 및 발행

```java
@Async
@EventListener
public void handleEvent(OrderCreatedEvent event) {
    log.info("이벤트 수신: " + event.getMessage());
    CheckOutRequest checkOutRequest = event.getCheckOutRequest();
    // 외부 연동 결제
    Payment payment = checkOutRequest.toPayment();
    boolean paySuccess = paymentService.pay(payment);

    if (paySuccess) {
        payCompletedPublisher.success(checkOutRequest);
    } else {
        payCompletedPublisher.fail(checkOutRequest);
    }
} 
```

## PayCompletedPublisher - 이벤트 발행

```java
public void success(CheckOutRequest checkOutRequest) {
    PayCompleteEvent event = new PayCompleteEvent("PayCompleted", checkOutRequest);
    log.info("이벤트 발행: " + event.getMessage());
    publisher.publishEvent(event);
}

public void fail(CheckOutRequest checkOutRequest) {
    PayFailedEvent event = new PayFailedEvent("PayFailed", checkOutRequest);
    log.info("이벤트 발행: " + event.getMessage());
    publisher.publishEvent(event);
}
```

이렇게 OrderCreatedListener에서 결제 로직을 수행하고 PayCompletedPublisher로 결과를 반환하는 것으로 끝났다.

## PayCompletedListener - 이벤트 수신

```java
@Async
@EventListener
public void handleEvent(PayFailedEvent event) {
    log.info("이벤트 수신: " + event.getMessage());

    CheckOutRequest checkOutRequest = event.getCheckOutRequest();
    // 보상 트랜잭션
    PayCompletedListener proxy = (PayCompletedListener) AopContext.currentProxy();
    proxy.rollbackCheckOut(checkOutRequest);
}

@Transactional
public void rollbackCheckOut(CheckOutRequest checkOutRequest) {
    // 포인트 롤백
    PointCommand pointCommand = checkOutRequest.toPointCommand();
    pointService.chargePoint(pointCommand);

    // 재고 롤백
    OrderSheet orderSheet = orderService.findOrderSheet(checkOutRequest.getOrderSheetId());
    List<StockCommand> stockList = orderSheet.toStockList();
    stockService.addStocks(stockList);

    // 주문 삭제
    Long orderId = checkOutRequest.getOrderId();
    orderService.updateState(orderId, Order.State.FAILED);
}
```

# 트랜잭션 분리로 발생한 문제점

## Entity 중복 조회 문제

> **초기 설계 의도**

기존 로직은 하나의 트랜잭션이라 1차 캐시를 사용한다는 가정 하에 파라미터로 간단하게 ID를 넘기고 각 로직에서 동일한 엔티티를 조회하는 방식으로 처리했기 때문에 문제가 없었다.

트랜잭션 분리 과정에서 동일한 엔티티를 별도로 재조회하게 되어 불필요한 작업이 추가됐다.

## orderSheet 도메인 책임 훼손

> **초기 설계 의도**

주문 생성 및 결제 과정에서 프론트단에서 변조되지 않은 검증된 데이터를 사용하기 위해 orderSheet를 별도로 생성하고, 해당 도메인이 검증로직을 수행하도록 하여 도메인의 책임을 분리했다.

그런데 트랜잭션이 분리되면서 다시 request객체를 활용하다 보니 orderSheet의 활용이 애매해졌다.

## **설계 의도를 유지하기 위한 변경 사항**

* request객체가 아닌 orderSheetId를 요청하는 방식으로 변경
    
* orderSheet를 별도의 도메인으로 분리하고 order와 payment가 참조할 수 있게 변경
    

# **트랜잭션 분리와 EDA의 장단점**

* **장점**
    
    시스템의 유지보수성과 확장성을 높임
    
* **단점**  
    책임 분리를 위해 이벤트 별로 이벤트 객체와 발행 클래스, 수신 클래스가 생성되기 때문에 복잡성이 훨씬 증가
    

따라서 트랜잭션을 분리할 필요가 있는지 충분히 고려하고, 비즈니스 로직의 복잡성과 원자성, 에러 처리를 위한 보상 트랜잭션, 성능 요구 사항 등을 종합적으로 판단하여 결정하는 것이 중요하다.