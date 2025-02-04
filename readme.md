## 프로젝트 소개
음식 배달 서비스 cover<br>
해당 서비스 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전 단계를 커버할 수 있도록 구성한 예제로 제시되는 평가포인트에 맞춰 개발 진행되었습니다.
 - 평가포인트 : https://workflowy.com/s/c10811dfdb67/UhOZB2crKOhNPUYp#/fc30a50ea43b
## Table of contents
- [서비스 시나리오](#서비스-시나리오)
- [분석/설계](#분석/설계)
---
### 서비스 시나리오
배달의 민족 커버하기 - https://1sung.tistory.com/106

- 기능적 요구사항
1. 고객이 메뉴를 선택하여 주문한다
2. 고객이 결제한다
3. 주문이 되면 주문 내역이 입점상점주인에게 전달된다
4. 상점주인이 확인하여 요리해서 배달 출발한다
5. 고객이 주문을 취소할 수 있다
6. 주문이 취소되면 배달이 취소된다
7. 고객이 주문상태를 중간중간 조회한다
8. 주문상태가 바뀔 때 마다 카톡으로 알림을 보낸다

- 비기능적 요구사항
1. 트랜잭션<br>
결제가 되지 않은 주문건은 아예 거래가 성립되지 않아야 한다 Sync 호출
2. 장애격리<br>
상점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency
결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다 Circuit breaker, fallback
3. 성능<br>
고객이 자주 상점관리에서 확인할 수 있는 배달상태를 주문시스템(프론트엔드)에서 확인할 수 있어야 한다 CQRS
배달상태가 바뀔때마다 카톡 등으로 알림을 줄 수 있어야 한다 Event driven

---
### 분석/설계
---

#### Saga(Pub/Sub)

SAGA 패턴이란 마이크로서비스들끼리 이벤트를 주고 받아 특정 마이크로서비스에서의 작업이 실패하면 이전까지의 작업이 완료된 마이크서비스들에게 보상 (complemetary) 이벤트를 소싱함으로써 분산 환경에서 원자성(atomicity)을 보장하는 패턴

- 주문이 발생할 경우(Order Service - ordered(event)) publish -> 주문 목록이 업데이트 된다. (Store - UpdateOrderList(Polish)) Subscribe

- 주문이 취소될 경우(Order Service - ordercanceled(event)) publish -> 주문결제가 취소된다. (Payment - CancelPayment(Polish)) Subscribe

<br>

- 주문 생성

<img width="960" src="https://user-images.githubusercontent.com/115772322/197435954-fa78862f-d1d2-4182-b40e-ba3b2b681d26.png">

<br>

- 주문 취소

<img width="665" src="https://user-images.githubusercontent.com/115772322/197435970-62094945-e7f8-41b9-a25a-ae1c16983e1f.png">

<br>

- Kafka에 접속하여 이벤트 확인

<img width="1186" src="https://user-images.githubusercontent.com/115772322/197435566-e66e34ec-7267-4c46-a110-c1bc52610b90.png">

 

---

#### CQRS

주문 발생, 결제 승인, 결제 취소, 배달 시작, 배달 취소 시 고객은 OrderInfo(ReadModel)을 통해 주문상태를 중간중간 조회할 수 있다.

<br>

비동기식으로 처리되어 이벤트 기반의 Kafka를 통해 처리되어 별도 Table에 관리한다.

<br>

- Read Model CRUD 상세설계

![image](https://user-images.githubusercontent.com/115772322/197447342-0e106402-97d8-4603-a94c-475aa67ee924.png)

```
    @StreamListener(KafkaProcessor.INPUT)
    public void whenOrdered_then_CREATE_1 (@Payload Ordered ordered) {
        try {
            if (!ordered.validate()) return;
            // view 객체 생성
            OrderInfo orderInfo = new OrderInfo();
            // view 객체에 이벤트의 Value 를 set 함
            orderInfo.setId(ordered.getId());
            orderInfo.setOrderStatus("Ordered");
            // view 레파지 토리에 save
            orderInfoRepository.save(orderInfo);
            
        }catch (Exception e){
            e.printStackTrace();
        }
    }
```

```
    @StreamListener(KafkaProcessor.INPUT)
    public void whenOrderCanceled_then_UPDATE_1(@Payload OrderCanceled orderCanceled) {
        try {
            if (!orderCanceled.validate()) return;
                // view 객체 조회
            Optional<OrderInfo> orderInfoOptional = orderInfoRepository.findById(orderCanceled.getId()); 
            if( orderInfoOptional.isPresent()) {
                 OrderInfo orderInfo = orderInfoOptional.get();
            // view 객체에 이벤트의 eventDirectValue 를 set 함
                orderInfo.setOrderStatus("OrderCanceled");   
                // view 레파지 토리에 save
                 orderInfoRepository.save(orderInfo);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
```   

```
    @StreamListener(KafkaProcessor.INPUT)
    public void whenPaymentCanceled_then_UPDATE_2(@Payload PaymentCanceled paymentCanceled) {
        try {
            if (!paymentCanceled.validate()) return;
                // view 객체 조회
            Optional<OrderInfo> orderInfoOptional = orderInfoRepository.findById(Long.valueOf(paymentCanceled.getOrderId()));
            if( orderInfoOptional.isPresent()) {
                 OrderInfo orderInfo = orderInfoOptional.get();
            // view 객체에 이벤트의 eventDirectValue 를 set 함
                orderInfo.setOrderStatus("PaymentCanceled");   
                // view 레파지 토리에 save
                 orderInfoRepository.save(orderInfo);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
```

```
    @StreamListener(KafkaProcessor.INPUT)
    public void whenPaymentApproved_then_UPDATE_4(@Payload PaymentApproved paymentApproved) {
        try {
            if (!paymentApproved.validate()) return;
                // view 객체 조회
            Optional<OrderInfo> orderInfoOptional = orderInfoRepository.findById(Long.valueOf(paymentApproved.getOrderId()));
            if( orderInfoOptional.isPresent()) {
                 OrderInfo orderInfo = orderInfoOptional.get();
            // view 객체에 이벤트의 eventDirectValue 를 set 함
                orderInfo.setOrderStatus("PaymentApproved");   
                // view 레파지 토리에 save
                 orderInfoRepository.save(orderInfo);
                }
        }catch (Exception e){
            e.printStackTrace();
        }
    }
```

```
    @StreamListener(KafkaProcessor.INPUT)
    public void whenDeliveryStarted_then_UPDATE_5(@Payload DeliveryStarted deliveryStarted) {
        try {
            if (!deliveryStarted.validate()) return;
                // view 객체 조회
            Optional<OrderInfo> orderInfoOptional = orderInfoRepository.findById(Long.valueOf(deliveryStarted.getOrderId()));
            if( orderInfoOptional.isPresent()) {
                 OrderInfo orderInfo = orderInfoOptional.get();
            // view 객체에 이벤트의 eventDirectValue 를 set 함
                orderInfo.setOrderStatus("DeliveryStarted");   
                // view 레파지 토리에 save
                 orderInfoRepository.save(orderInfo);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
```

---

#### Compensation / Correlation

주문에 대한 복구 / 취소가 가능해야 한다. 해당 취소건에 대해서 필터링이 되어야 한다.

<br>

- 주문 취소

<img width="665" src="https://github.com/hyomin/hw6_msa-capstone-project-main/blob/main/capture/kor/%EC%A3%BC%EB%AC%B8%EC%B7%A8%EC%86%8C.png">

<br>

- 주문내역삭제(Correlation _ Compensation)

<img width="665" src="https://github.com/hyomin/hw6_msa-capstone-project-main/blob/main/capture/kor/%EC%A3%BC%EB%AC%B8%EB%82%B4%EC%97%AD%EC%82%AD%EC%A0%9C(Correlation%20_%20Compensation).png">


---

#### Request / Response

order > inventory 재고량을 조회

<br>

- 주문 취소

<img width="665" src="">


---

#### Circuit Breaker
(비기능적 요구사항)
장애격리
상점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다 Circuit breaker, fallback


Circuit Breaker 패턴이란 하나의 컴포넌트가 느려지거나 장애가 나면 그 장애가난 컴포넌트를 호출하는 종속된 컴포넌트까지 장애가 전파될때, 장애가 전파되지 않도록 회로 차단기 개념으로 장애 발생시 전파가 되지 않도록 하는 패턴을 말한다. 즉, Service A 와 B간의 통신중, B에 오류가 발생 할 경우 Circuit Breaker 가 작동하여 B를 호출하는 A 쓰레드들을 모두 차단하여 A가 더이상 B 서비스를 기다리지 않도록 하는 개념이다.

본 프로젝트에서는 Hystrix 라이브러리를 사용하였고 아래와 같이
요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정하였다.

<img width="450" alt="CircuitBreaker설정Yml" src="https://user-images.githubusercontent.com/5373350/197447628-5eb3dbeb-672a-42b2-be9b-e312aa7b7ca4.png">

아래는 siege 를 활용하여 시스템 부하를 통해 Circuit Breaker가 동작하는 모습이다.

![image](https://user-images.githubusercontent.com/5373350/197448196-4fdfdd30-2cd1-4a3c-a67a-0fa6537fe35d.png)


#### Fallback

Circuit Breaker 가 작동하여 서비스가 호출이 되지 않을 경우 이에 대해 서비스가 호출 되지 않는다 라는 응답이 필요한 경우 이를 설정 하는 부분이다.

우선, feign를 사용하였고, 아래와 같이 order -> payment 가 동기 호출(Req,Res) 인데, 이때, Payment 호출이 되지 않을 경우
PaymentServiceFallback 을 호출 한다는 의미이다.

<img width="752" alt="image" src="https://user-images.githubusercontent.com/5373350/197451487-a4575fde-c1ed-4eaa-9413-5e2438dfd896.png">

그리고 아래는 실제 Payment 서비스를 종료 시키고 order 서비스만 동작한뒤, 주문을 하였을때, 실제 fallback 이 실행되는 부분이다.

1. fallback 로직 구현
<img width="576" alt="image" src="https://user-images.githubusercontent.com/5373350/197452187-060a54a3-420d-4ccd-b529-24ffa145f1c7.png">


2. fallback 로직 실행
![image](https://user-images.githubusercontent.com/5373350/197452125-a56bfc59-58bb-4f63-8471-c8b9dbf26bad.png)


---

#### Gateway / Ingress

여러 개의 클라이언트가 여러 개의 서버 서비스를 각각 호출하게 된다면 매우 복잡한 호출 관계가 생성 되는데, 이를 해결하기 위해 단일 진입점을 만드는 것이 게이트웨이다. 아래 그림과 같이 다양한 클라이언트가 다양한 서비스에 액세스하기 위해서는 단일 진입점을 만들어 놓으면 여러모로 효율적이다. 다른 유형의 클라이언트에게 서로 다른 API조합을 제공할 수도 있고 각 서비스 접근 시 필요한 인증/인가 기능을 한 번에 처리할 수도 있다. 또 정상적으로 동작하던 서비스에 문제가 발생하여 서비스 요청에 대한 응답 지연이 발생하면 정상적인 다른 서비스로 요청 경로를 변경하는 기능이 작동되게 할 수도 있다.

<img width="584" alt="MSA2 16" src="https://user-images.githubusercontent.com/5373350/197448502-b216e83b-3461-48bd-b84d-ce76f505b704.png">

아래는 게이트 웨이의 진입점 포트(8088)와 연결되어있는 서비스들에 대한 설정 부분이고

<img width="373" alt="image" src="https://user-images.githubusercontent.com/5373350/197448604-29049486-5ec4-46f7-b7fb-4021d1423ec9.png">

아래 두개 화면은 8088 포트로 주문과 주문확인을 진행 하였을때, 내부적으로 어떻게 호출하는지에 대한 부분이다.


1. 8088 포트로 주문시 8082 포트로 올려진 order서비스 호출됨

<img width="776" alt="gateway서비스를 통한 주문" src="https://user-images.githubusercontent.com/5373350/197448995-420c66dc-9e4d-4408-92a3-4b08c968b355.png">

2. 주문확인

<img width="549" alt="gateway서비스를 통한 주문내역확인" src="https://user-images.githubusercontent.com/5373350/197449086-ff6700fa-0aaf-40ea-9e9d-9bd2587d22b6.png">
<br>


---

#### Deploy / Pipeline

kubernetes yaml 파일을 이용한 서비스 기동 확인
<br>
<img width="669" alt="deployment" src="https://user-images.githubusercontent.com/5373350/197651330-da10fce9-e54a-4f1b-8f59-de7874b8a921.png">



---

#### Autoscale (HPA)

order pod의 cpu사용률이 50% 가 넘어갈 경우 autoscale 되도록 설정

<img width="901" alt="스크린샷 2022-10-25 오후 1 21 23" src="https://user-images.githubusercontent.com/5373350/197681933-56fe0404-62f5-4e53-bb4a-1fd058b5ad09.png">




---

#### Zero-downtime deploy (Readiness probe)

서비스 재 배포시, 중단시간 없는 배포를 위해 yaml파일에 아래와 같이 설정 

<img width="289" alt="스크린샷 2022-10-25 오전 9 48 46" src="https://user-images.githubusercontent.com/5373350/197656953-f63215ec-c941-42d1-962d-2fc1e7499d5f.png">

재배포시 서비스 중단 발생 확인을 위해 siege로 부하 설정 및 서비스 무중단 확인

<img width="466" alt="무정지배포 siege" src="https://user-images.githubusercontent.com/5373350/197657042-414a9146-64aa-47cc-90d7-eb3c406beb85.png">

<img width="659" alt="스크린샷 2022-10-25 오전 9 50 49" src="https://user-images.githubusercontent.com/5373350/197656997-21fdef4c-1c12-4723-bbce-232b9330b693.png">


---

#### Persistence Volume/ConfigMap/Secret

내용내용

<br>

- 설명설명(이미지)

<img width="665" src="">


---

#### Polyglot

내용내용

<br>

- 설명설명(이미지)

<img width="665" src="">


---

#### Self-healing (liveness probe)

내용내용

<br>

- 설명설명(이미지)

<img width="665" src="">


