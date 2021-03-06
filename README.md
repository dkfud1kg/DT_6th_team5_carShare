# DT_6th_team5_carShare (자동차 공유 서비스)

5팀 자동차 공유 서비스 CNA개발 실습을 위한 프로젝트

# 구현 Repository
 1. 접수관리 : https://github.com/dkfud1kg/carShareOrder.git
 1. 결제관리 : https://github.com/dkfud1kg/carSharePayment.git
 1. 배송관리 : https://github.com/dkfud1kg/carShareDelivery.git
 1. 고객페이지 : https://github.com/dkfud1kg/carShareStatusview.git
 1. 게이트웨이 : https://github.com/dkfud1kg/carShareGateway.git
 1. (추가) 알림 : https://github.com/dkfud1kg/carShareAlarm.git

# Table of contents

- [서비스 시나리오](#서비스-시나리오)
  - [시나리오 테스트결과](#시나리오-테스트결과)
- [분석/설계](#분석설계)
- [구현](#구현)
  - [DDD 의 적용](#ddd-의-적용)
  - [Gateway 적용](#Gateway-적용)
  - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
  - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
  - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
- [운영](#운영)
  - [CI/CD 설정](#cicd설정)
  - [서킷 브레이킹 / 장애격리](#서킷-브레이킹-/-장애격리)
  - [오토스케일 아웃](#오토스케일-아웃)
  - [무정지 재배포](#무정지-재배포)
  

# 서비스 시나리오

## 기능적 요구사항
1. 고객이 공유차를 선택하여 렌탈 주문한다.
1. 고객이 결제하면 접수된다.
1. 업체가 공유차를 고객위치로 가져다놓는다.
1. 고객이 렌탈 주문을 취소할 수 있다.
1. 렌탈 주문 취소 시 배송과 결제가 취소된다.
1. 고객이 자신의 렌탈 정보를 조회한다.
1. (추가) 주문이 접수되면 알림을 발송한다. (Async)
1. (추가) 주문 취소는 고객에게 알림을 발송한 후 취소한다. (Sync)


## 비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 주문건은 아예 접수가 성립되지 않아야 한다(Sync)
    1. (추가) 주문취소 알림이 발송 되지않은 건은 주문취소가 완료되지 않는다.(Sync) 
1. 장애격리
    1. 배송관리 기능이 수행되지 않더라도 접수는 정상적으로 처리 가능하다(Async(event-driven), Eventual Consistency)
    1. 접수시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다(Circuit breaker, fallback)
1. 성능
    1. 고객이 본인의 렌탈 상태 및 이력을 접수시스템에서 확인할 수 있어야 한다(CQRS)
    1. (추가) 알림 발송 상태를 고객 상태뷰에 업데이트한다.(CQRS)



# 분석/설계

## 이벤트스토밍
* MSAEz 로 모델링한 이벤트스토밍 결과:  
![image](https://user-images.githubusercontent.com/70302900/96704954-9db56180-13cf-11eb-96ec-7b8292b26324.png)

* alarm 서비스를 추가한 이벤트스토밍 결과:  
![image](https://user-images.githubusercontent.com/70302900/96686874-14476480-13ba-11eb-8f48-9817ae55596e.png)


## 헥사고날 아키텍처 다이어그램 도출
![image](https://user-images.githubusercontent.com/70302900/96705114-d35a4a80-13cf-11eb-9b8a-0bd8af1be18b.png)

* alarm 서비스를 추가하여 다이어그램 도출
![image](https://user-images.githubusercontent.com/70302900/96705325-187e7c80-13d0-11eb-95c3-441e1f32e96e.png)


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐

   
# 구현

## 시나리오 테스트결과
| 기능 | 이벤트 Payload |
|---|:---:|
| 1. 고객이 공유차 렌탈 주문한다.</br>2. 결제가 정상적으로 완료되면 주문이 완료된다. (Sync)</br>3. 주문이 완료되면 배송이 시작된다. (Async)</br>4. 배송이 시작되면 접수정보의 상태를 변경한다. (Async)</br>5. (추가) 주문 및 배송이 시작되면, 알림이 발송된다. (Async)|![image](https://user-images.githubusercontent.com/70302900/96770950-00344f00-141c-11eb-86cb-d168b13e6a71.png)|
| 6. 고객이 공유차 렌탈 주문을 취소한다.</br>7. 배송 취소가 정상적으로 완료되면 결제 취소가 진행된다. (Sync)</br>8. (추가) 주문 취소 알림이 발송 완료되면 주문이 최종적으로 취소된다. (Sync) |![image](https://user-images.githubusercontent.com/70302900/96770959-032f3f80-141c-11eb-98ec-77bf03cb6b9e.png)|
| 9.고객이 접수 상태를 조회한다.|![제목없음21](https://user-images.githubusercontent.com/42608068/96581350-a5640000-1314-11eb-8336-0474e2d1716b.png)|

## DDD 의 적용
분석/설계 단계에서 도출된 MSA는 총 5개로 아래와 같다.
* alarm 서비스 추가 (알림 발송 상태값을 status view에 업데이트하여 CQRS 추가구현)

| MSA | 기능 | port | 조회 API | Gateway 사용시 |
|---|:----:|:---:|---|---|
| order | 접수 관리 | 8081 | http://localhost:8081/orders |http://carshareorder:8080/orders |
| delivery | 배송 관리 | 8082 | http://localhost:8082/deliveries | http://carsharedelivery:8080/deliveries |
| customerpage | 상태 조회 | 8083 | http://localhost:8083/customerpages | http://carsharestatusview:8080/customerpages |
| payment | 결제 관리 | 8084 | http://localhost:8084/payments | http://carsharepayment:8080/payments |
| alarm | 알림 관리 | 8085 | http://localhost:8085/alarms | http://carsharealarm:8080/alarms |

## Gateway 적용

```
spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: order
          uri: http://carshareorder:8080
          predicates:
            - Path=/orders/** 
        - id: delivery
          uri: http://carsharedelivery:8080
          predicates:
            - Path=/deliveries/**,/cancellations/**
        - id: statusview
          uri: http://carsharestatusview:8080
          predicates:
            - Path= /customerpages/**
        - id: payment
          uri: http://carsharepayment:8080
          predicates:
            - Path=/payments/**,/paymentCancellations/**
        - id: alarm
          uri: http://carsharealarm:8080
          predicates:
            - Path=/alarms/**  
```


## 폴리글랏 퍼시스턴스

신규 마이크로 서비스를 위한 alarm 서비스의 DB를 분리 적용. 인메모리DB인 hsqldb 사용

```
pom.xml 에 적용
<!-- 
		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>
 -->	
		<dependency>
			<groupId>com.hsqldb</groupId>
			<artifactId>hsqldb</artifactId>
			<scope>runtime</scope>
		</dependency>
```


## 동기식 호출 과 Fallback 처리

접수(order) > 알림(alarm) 간의 일부 프로세스(주문취소)호출을 동기식 일관성을 유지하는 트랜잭션으로 처리하도록 했다. 
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.
- FeignClient 서비스 구현

```
# AlarmService.java

@FeignClient(name="alarm", contextId ="alarm", url="${api.alarm.url}", fallback = AlarmServiceFallback.class)
public interface AlarmtService {

    @RequestMapping(method= RequestMethod.POST, path="/alarms")
    public void pay(@RequestBody Alarm alarm);

}
```
- 접수취소요청을 받은 직후(@PostPersist) 알림을 발송하도록 처리
```
# Order.java (Entity)

    @PostPersist
    public void onPostPersist(){
        Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        ordered.publishAfterCommit();

        carshare.external.Payment payment = new carshare.external.Payment();
        payment.setOrderId(this.getId());
        payment.setProductId(this.getProductId());
        payment.setQty(this.getQty());
        payment.setStatus("OrderApproved");
        OrderApplication.applicationContext.getBean(carshare.external.PaymentService.class)
            .pay(payment);
    }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 알림 서비스가 장애가 주문취소를 못한는다는 것을 확인


```
#알림(alarm) 서비스를 잠시 내려놓음 (ctrl+c)

#주문취소 처리 (주문 취소 수행되지 않음)
http DELETE localhost:8081/orders productId=1001 qty=1 status="order"   #Fail
http DELETElocalhost:8081/orders productId=1002 qty=3 status="order"   #Fail

#알림 서비스 재기동
cd carsharealarm
mvn spring-boot:run

#주문취소요청 처리 성공
http DELETE localhost:8081/orders productId=1001 qty=1 status="order"   #Success
http DELETE localhost:8081/orders productId=1002 qty=3 status="order"   #Success
```

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, Fallback 처리는 운영단계에서 설명한다.)


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

주문이 이루어진 후에 알림 서비스로 이를 알려주는 행위는 동기식이 아니라 비동기식으로 처리하여  시스템의 처리를 위해 결제가 블로킹되지 않도록 처리한다.
 
- 이를 위하여 주문 기록을 남긴 후에 곧바로 주문이 완료 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
package carshare;

@Entity
@Table(name="Order_table")
public class Order {

 ...
    @PostPersist
    public void onPostPersist(){
        Paid Ordered = new Ordered();
        BeanUtils.copyProperties(this, Ordered);
        Ordered.publishAfterCommit();    
    }
}
```
- 알림 서비스에서는 주문완료 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다

```
package carshare;

...

@Service
public class PolicyHandler{

   @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaid_Ship(@Payload Ordered Ordered){

        if(Ordered.isMe()){
            Alarm alarm = new alarm();
            alarm.setOrderId(Ordered.getOrderId());
            alarm.setReciever(Ordered.getId());
            alarm.setMessage("Sent");

            deliveryRepository.save(delivery) ;
        }
    }

}

```

알림 서비스는 주문 서비스와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 알림 서비스가 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데에는 문제가 없다.

```
#알림(alarm) 서비스를 잠시 내려놓음 (ctrl+c)

#접수요청 처리 (주문접수는 문제없이 처리 됨)
http localhost:8081/orders productId=1003 qty=2 status="ordered"   #Success
http localhost:8081/orders productId=1004 qty=4 status="ordered"   #Success

#알림 발송 상태 확인
http localhost:8081/alarms     # 알림 발송되지않음

#알림 서비스 기동
cd carsharealarm
mvn spring-boot:run

#알림 발송 상태 확인
http localhost:8081/alarms     # 알림 발송 성공 됨
```

# 운영

## CI/CD 설정

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AWS CodeBuild를 사용하였다.
![image](https://user-images.githubusercontent.com/70302900/96768068-599a7f00-1418-11eb-85fc-ae226e33c5fb.png)


Git Hook 설정으로 연결된 GitHub의 소스 변경 발생 시 자동 배포된다.
$$$ 배포화면 캡쳐 추후 추가&&&


## 서킷 브레이킹 / 장애격리

### 서킷 브레이킹 istio-injection + DestinationRule

* istio-injection 적용 (기 적용완료)
```
kubectl label namespace carshare istio-injection=enabled 
```
* 서킷 브레이킹을 위한 DestinationRule 적용
```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: alarms
  namespace: carshare
spec:
  host: carsharealarm
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      interval: 1s
      consecutiveErrors: 2
      baseEjectionTime: 10s
      maxEjectionPercent: 100
```
* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 200명
- 20초 동안 실시
```
siege -c200 -t20S -v 'http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms  POST {"orderId": "1001", "reciver":"SKCC"}

HTTP/1.1 200    0.02 secs:    6273  bytes ==> GET   /alarms
HTTP/1.1 200    0.03 secs:    6273  bytes ==> GET   /alarms
HTTP/1.1 200    0.05 secs:    6273  bytes ==> GET   /alarms
HTTP/1.1 200    0.11 secs:    6273  bytes ==> GET   /alarms
HTTP/1.1 503    0.08 secs:      81  bytes ==> GET   /alarms
HTTP/1.1 200    0.10 secs:    6273  bytes ==> GET   /alarms
HTTP/1.1 200    0.08 secs:    6273  bytes ==> GET   /alarms
HTTP/1.1 503    0.02 secs:      81  bytes ==> GET   /alarms
HTTP/1.1 200    0.11 secs:    6273  bytes ==> GET   /alarms
HTTP/1.1 503    0.05 secs:      81  bytes ==> GET   /alarms
HTTP/1.1 503    0.03 secs:      81  bytes ==> GET   /alarms
:
Transactions:                   432  hits
Availability:                 84.54  %
Elapsed time:                  4.23  secs
Data transferred:              0.28  MB
Response time:                 0.06  secs
Transaction rate:             78.60  trans/sec
Throughput:                    0.21  MB/sec
Concurrency:                  21.16
Successful transactions:        432
Failed transactions:             79
Longest transaction:           1.44
Shortest transaction:          0.01

```

DestinationRule 적용 제거 후 다시 부하 발생하여 정상 처리 확인
```
siege -c200 -t20S -v 'http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms  POST {"orderId": "1001", "reciver":"SKCC"}

HTTP/1.1 200    0.04secs:     6273  bytes ==> GET   /alarms
HTTP/1.1 200    0.02 secs:    6273  bytes ==> GET   /alarms
HTTP/1.1 200    0.12 secs:    6273  bytes ==> GET   /alarms
HTTP/1.1 200    0.03 secs:    6273  bytes ==> GET   /alarms
HTTP/1.1 200    0.02 secs:    6273  bytes ==> GET   /alarms
HTTP/1.1 200    0.02 secs:    6273  bytes ==> GET   /alarms
:
Transactions:                   981  hits
Availability:                100.00  %
Elapsed time:                  2.87  secs
Data transferred:              0.00  MB
Response time:                 0.04  secs
Transaction rate:            103.21  trans/sec
Throughput:                    0.00  MB/sec
Concurrency:                  20.56
Successful transactions:        981
Failed transactions:             0
Longest transaction:           2.13
Shortest transaction:          0.01

```

### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 

- Deployment 배포시 resource 설정 적용
```
spec:
      containers:
          ...
          resources:
            limits:
              cpu: 500m 
            requests:
              cpu: 200m 
```

- replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 5프로를 넘어서면 replica 를 10개까지 늘려준다
```
kubectl autoscale deploy carsharealarm -n carshare --min=1 --max=10 --cpu-percent=5
```

- 오토스케일이 어떻게 되고 있는지 HPA 모니터링을 걸어둔다, 어느정도 시간이 흐른 후, 스케일 아웃이 벌어지는 것을 확인할 수 있다
```
$ kubectl get deploy carsharealarm -n carshare -w 

NAME             READY   UP-TO-DATE   AVAILABLE   AGE
carsharealarm     1/4     4            1           3h
carsharealarm     2/4     4            2           3h
carsharealarm     3/4     4            3           3h
carsharealarm     4/4     4            4           3h
carsharealarm     4/8     4            4           3h
carsharealarm     4/8     4            4           3h
carsharealarm     4/8     4            4           3h
carsharealarm     4/8     8            4           3h
carsharealarm     4/10    8            4           3h
carsharealarm     4/10    8            4           3h
carsharealarm     4/10    8            4           3h
carsharealarm     4/10    10           4           3h

```

- kubectl get으로 HPA을 확인하면 CPU 사용률이 132%로 증가됐다.
```
NAME                                                 REFERENCE                   TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/alarm         Deployment/alarm               132%/15%   1         10          5       3h
```

## 무정지 재배포

먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler, CB 설정을 제거함

- Readiness Probe 미설정 시 무정지 재배포 불가 테스트
Readiness Probe 미설정 시 무정지 재배포 가/불가 여부 확인을 위해 buildspec.yml의 Readiness Probe 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
~$ siege -v -c1 -t240S --content-type "application/json" 'http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms POST {"orderId": "1001", "reciver":"SKCC"}'

** SIEGE 4.0.4
** Preparing 1 concurrent users for battle.
The server is now under siege...
HTTP/1.1 200     1.03 secs:       0 bytes ==> POST http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms
HTTP/1.1 200     0.41 secs:       0 bytes ==> POST http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms
:

```

- CI/CD 파이프라인을 통해 새버전으로 재배포 작업함
Git hook 연동 설정되어 Github의 소스 변경 발생 시 자동 빌드 배포됨
재배포 작업 중 서비스 중단됨을 확인 (500 오류 발생 확인됨)
```
HTTP/1.1 200     0.34 secs:       0 bytes ==> POST http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms
HTTP/1.1 200     0.42 secs:       0 bytes ==> POST http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms
HTTP/1.1 500     0.38 secs:     205 bytes ==> POST http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms
HTTP/1.1 500     0.41 secs:     205 bytes ==> POST http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms
HTTP/1.1 500     0.42 secs:     205 bytes ==> POST http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms
HTTP/1.1 500     0.37 secs:     205 bytes ==> POST http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms
HTTP/1.1 500     0.39 secs:     205 bytes ==> POST http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms
HTTP/1.1 500     0.39 secs:     205 bytes ==> POST http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms
HTTP/1.1 500     0.36 secs:     205 bytes ==> POST http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms
HTTP/1.1 500     0.35 secs:     205 bytes ==> POST http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms
HTTP/1.1 500     0.36 secs:     205 bytes ==> POST http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms
HTTP/1.1 500     0.37 secs:     205 bytes ==> POST http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms
HTTP/1.1 500     0.38 secs:     205 bytes ==> POST http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms
HTTP/1.1 500     0.38 secs:     205 bytes ==> POST http://a13bace79d588418ba102b6880b1fb46-68406260.ap-south-1.elb.amazonaws.com:8080/alarms
:

```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
```
Transactions:                    224 hits
Availability:                  83.27 %
Elapsed time:                 128.90 secs
Data transferred:               0.01 MB
Response time:                  0.34 secs
Transaction rate:               1.51 trans/sec
Throughput:                     0.00 MB/sec
Concurrency:                    1.00
Successful transactions:         224
Failed transactions:              45
Longest transaction:            1.51
Shortest transaction:           0.25

```
- 배포기간중 Availability 가 평소 100%에서 83% 대로 떨어지는 것을 확인. 
원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문으로 판단됨. 

- Readiness Probe 설정 시 무정지 재배포 가능 테스트
Readiness Probe 를 설정함 (buildspec.yml의 Readiness Probe 설정)
```
# buildspec.yaml 의 Readiness probe 의 설정:
- CI/CD 파이프라인을 통해 새버전으로 재배포 작업함

readinessProbe:
    httpGet:
      path: '/actuator/health'
      port: 8080
    initialDelaySeconds: 30
    timeoutSeconds: 2
    periodSeconds: 5
    failureThreshold: 10
    
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
```
Transactions:                    215 hits
Availability:                 100.00 %
Elapsed time:                  87.52 secs
Data transferred:               0.00 MB
Response time:                  0.41 secs
Transaction rate:               2.44 trans/sec
Throughput:                     0.00 MB/sec
Concurrency:                    1.00
Successful transactions:         215
Failed transactions:               0
Longest transaction:            1.26
Shortest transaction:           0.28

```

배포기간 동안 Availability 100% 유지되어 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.

## ConfigMap 사용

시스템별로 또는 운영중에 동적으로 변경 가능성이 있는 설정들을 ConfigMap을 사용하여 관리합니다.
Application에서 특정 도메일 URL을 ConfigMap 으로 설정하여 운영/개발등 목적에 맞게 변경가능합니다.  

* my-config.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
  namespace: carshare
data:
  api.payment.url: http://carsharepayment:8080
```
my-config라는 ConfigMap을 생성하고 key값에 도메인 url을 등록한다. 

* carsharealarm/buildsepc.yaml (configmap 사용)
```
 cat  <<EOF | kubectl apply -f -
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: $_PROJECT_NAME
          namespace: $_NAMESPACE
          labels:
            app: $_PROJECT_NAME
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: $_PROJECT_NAME
          template:
            metadata:
              labels:
                app: $_PROJECT_NAME
            spec:
              containers:
                - name: $_PROJECT_NAME
                  image: $AWS_ACCOUNT_ID.dkr.ecr.$_AWS_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
                  ports:
                    - containerPort: 8080
                  env:
                    - name: api.payment.url
                      valueFrom:
                        configMapKeyRef:
                          name: my-config
                          key: api.payment.url
                  imagePullPolicy: Always
                
        EOF
```
Deployment yaml에 해단 configMap 적용

* PaymentService.java
```
@FeignClient(name="payment", contextId = "payment", url="${api.payment.url}")
public interface PaymentService {

    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void pay(@RequestBody Payment payment);

}
```
url에 configMap 적용

* kubectl describe pod carsharealarm-bdd8c8c4c-l52h6  -n carshare
```
Containers:
  carsharealarm:
    Container ID:   docker://f3c983b12a4478f3b4a7ee5d7fea308638903eb62e0941edd33a3bce5f5f6513
    Image:          496278789073.dkr.ecr.ap-southeast-2.amazonaws.com/carshareorder:9289bba10d5b0758ae9f6279d56ff77b818b8b63
    Image ID:       docker-pullable://879772956301.dkr.ecr.ap-south-1.amazonaws.com/carsharealarm@sha256:95395c95d1bc19ceae8eb5cc0b288b38dc439359a084610f328407dacd694a81
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 21 Oct 2020 02:13:01 +0000
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:  500m
    Requests:
      cpu:        200m
    Liveness:     http-get http://:8080/ delay=120s timeout=2s period=5s #success=1 #failure=5
    Readiness:    http-get http://:8080/ delay=30s timeout=2s period=5s #success=1 #failure=10
    Environment:
      api.payment.url:  <set to the key 'api.payment.url' of config map 'my-config'>  Optional: false
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-5gx6w (ro)

```
kubectl describe 명령으로 컨테이너에 configMap 적용여부를 알 수 있다. 



