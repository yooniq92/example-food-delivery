
# automechanicsmall
# Table of contents

- [automechanicsmall](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 서비스 시나리오

기능적 요구사항
1. 접수처에서 수리 신청을 받는다.
1. 수리처에서 접수처에 접수된 수리를 받고 접수상태를 변경한다.
1. 수리처에서 수리 완료한 후 접수상태를 변경한다.
1. 접수처에서 수리 완료된 접수에 대해 결제 요청을 한다.
1. 고객이 결제하고 접수상태를 변경한다.
1. 접수처에서 수리 취소를 요청한다.
1. 수리처에서 취소 요청된 수리를 취소하고 접수상태를 변경한다.
1. 수리처에서 접수처에 접수된 수리를 거절하고 접수상태를 변경한다.

비기능적 요구사항
1. 트랜잭션
    1. 수리가 완료된 수리접수건에 대해서만 결제가 성립되어야 한다 Sync 호출
1. 장애격리
    1. 수리처 기능이 수행되지 않더라도 접수은 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency
    1. 결제시스템에 문제가 발생했을 때 결제를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다 Circuit breaker, fallback
1. 성능
    1. 접수처에서 수시로 접수상태를 확인할 수 있어야 한다. CQRS
    1. 수리처에서 상태를 변경할 수 있어야 한다.
    1. 고객이 직접 결제할 수 있어야 한다.

# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/JCfIVLVOzfPg7DfOPWjEfNsoMa92/share/b3974b4a87da79a7e8333e2729bfe356/-MK7sQ_71ifl_gedWf2O

### 이벤트 도출
![image](https://user-images.githubusercontent.com/22365716/98182203-8a1a0700-1f48-11eb-964e-636005c3aa9c.png)

### 액터, 커맨드 부착하여 읽기 좋게
![image](https://user-images.githubusercontent.com/22365716/98183397-64423180-1f4b-11eb-86c6-2fd6434f676f.png)

### 어그리게잇으로 묶기
![image](https://user-images.githubusercontent.com/22365716/98183508-9a7fb100-1f4b-11eb-9843-a54d01660bec.png)

### 바운디드 컨텍스트로 묶기
![image](https://user-images.githubusercontent.com/22365716/98182436-19271f00-1f49-11eb-9bd9-7ed7a1958a3e.png)

    - 도메인 서열 분리 
        - Core Domain:  receipt : 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 배포주기는 receipt 의 경우 1개월 1회 미만
        - Supporting Domain:  repair : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
        - General Domain:   payment : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)
![image](https://user-images.githubusercontent.com/22365716/98183782-40cbb680-1f4c-11eb-8cfd-91c7ea098d87.png)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)
![image](https://user-images.githubusercontent.com/22365716/98183829-5b9e2b00-1f4c-11eb-8784-429e1dc55cf5.png)

### 완성된 1차 모형
![image](https://user-images.githubusercontent.com/22365716/98187334-06661780-1f54-11eb-82db-4351f052c5c9.png)


### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

![image](https://user-images.githubusercontent.com/46660236/98188908-3cf16180-1f57-11eb-80ef-86588ece7db9.png)

    - 접수처에서 수리 신청을 받는다. (ok)
    - 수리처에서 접수처에 접수된 수리를 받고 접수상태를 변경한다. (ok)
    - 수리처에서 수리 완료한 후 접수상태를 변경한다. (ok)
    - 접수처에서 수리 완료된 접수에 대해 결제 요청을 한다. (ok)
    - 고객이 결제하고 접수상태를 변경한다. (ok)

![image](https://user-images.githubusercontent.com/46660236/98188999-6c07d300-1f57-11eb-91e3-73521fa58498.png)

    - 접수처에서 수리 취소를 요청한다. (ok)
    - 수리처에서 취소 요청된 수리를 취소하고 접수상태를 변경한다. (ok)

![image](https://user-images.githubusercontent.com/46660236/98188993-6a3e0f80-1f57-11eb-90aa-053abdad6703.png)

    - 수리처에서 접수처에 접수된 수리를 거절하고 접수상태를 변경한다. (ok)


### 모델 수정

![image](https://user-images.githubusercontent.com/22365716/98186925-2ea14680-1f53-11eb-9ef5-4a5bbb3aabfc.png)
    
    - 수정된 모델은 모든 요구사항을 커버함.

### 비기능 요구사항에 대한 검증

![image](https://user-images.githubusercontent.com/22365716/98186925-2ea14680-1f53-11eb-9ef5-4a5bbb3aabfc.png)

    - 트랜잭션 처리
        - 수리 완료시 결제처리에 대해서는 Request-Response 방식 처리
    - 성능
        - 접수처, 수리처, 결제 뷰 생성


## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/22365716/98185109-54c4e780-1f4f-11eb-9544-7ea07a1365ad.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 리액트으로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)
```
cd receipt
mvn spring-boot:run
cd repair
mvn spring-boot:run
cd payment
mvn spring-boot:run
cd display
mvn spring-boot:run
cd car-react
npm start
```
## DDD 의 적용
- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 pay 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 하지만, 일부 구현에 있어서 영문이 아닌 경우는 실행이 불가능한 경우가 있기 때문에 계속 사용할 방법은 아닌것 같다. (Maven pom.xml, Kafka의 topic id, FeignClient 의 서비스 id 등은 한글로 식별자를 사용하는 경우 오류가 발생하는 것을 확인하였다)
```
package automechanicsmall;
import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
@Entity
@Table(name="Repair")
public class Repair {
    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String vehiNo;
    private String stat;
    private Integer repairAmt;
    private Long receiptId;
public Long getId() {
        return id;
    }
    public void setId(Long id) {
        this.id = id;
    }
    public String getVehiNo() {
        return vehiNo;
    }
    public void setVehiNo(String vehiNo) {
        this.vehiNo = vehiNo;
    }
    public String getStat() {
        return stat;
    }
    public void setStat(String stat) {
        this.stat = stat;
    }
    public Integer getRepairAmt() {
        return repairAmt;
    }
    public void setRepairAmt(Integer repairAmt) {
        this.repairAmt = repairAmt;
    }
    public Long getReceiptId() {
        return receiptId;
    }
    public void setReceiptId(Long receiptId) {
        this.receiptId = receiptId;
    }
}
```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package automechanicsmall;
import org.springframework.data.repository.PagingAndSortingRepository;
import java.util.List;
public interface RepairRepository extends PagingAndSortingRepository<Repair, Long>{
}
```
- 적용 후 REST API 의 테스트
```
# 접수 생성 & 수리 요청
http POST http://localhost:8088/receipts vehiNo=12가1234 stat=REQUESTREPAIR
# 수리 완료
http POST http://localhost:8088/receipts vehiNo=12가1234 stat=REQUESTREPAIR
# 비용 지불
http PATCH http://localhost:8088/receipts/1 stat=PAYMENTREQUESTED payAmt=3000
# 각 어그리게이트 확인 (접수, 수리, 비용지불)
http http://localhost:8088/receipts/1
http http://localhost:8088/repairs/1
http http://localhost:8088/payments/1
```
## 폴리글랏 퍼시스턴스
마이크로서비스의 폴리그랏 퍼시스턴스의 예로 데이터의 빈번한 입출력을 사용하는 부분은 Display(View)의 저장소는 Mongo DB(NO SQL)를 사용하고 그 외의 업무 도메인인 접수(Receipt), 수리(Repair), 결재(Payment)는 Maria DB(RDB)를 사용하였다.
application.yml의 간단한 설정을 통해 설정이 가능하다.
```
application.xml (receipt service)
spring:
  profiles: default
  jpa:
    database-platform: org.hibernate.dialect.MariaDB53Dialect
    hibernate:
      ddl-auto: create #create update none
    properties:
      hibernate:
        show_sql: true
        format_sql: true
  datasource:
    url: jdbc:mariadb://${payment.db.url}:3306/receipt?useSSL=true
    username: ${payment.db.name}
    password: ${payment.db.password}
    driverClassName: org.mariadb.jdbc.Driver
application.xml (display service)
spring:
  profiles: default
  data:
    mongodb:
      uri: mongodb://${display.db.url}:27017/display
      username: ${display.db.name}
      password: ${display.db.password}
  jpa:
    properties:
      hibernate:
        show_sql: true
        format_sql: true
```
## 동기식 호출 과 Fallback 처리
분석단계에서의 조건 중 하나로 접수(repair)->결제(payment) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.
- 결제서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현
```
# (receipt) PaymentService.java
package automechanicsmall.external;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.*;
import java.util.Date;
@FeignClient(name="payment", url="${api.payment.url}")
public interface PaymentService {
    @RequestMapping(method= RequestMethod.POST, path="/payments")
    void pay(@RequestBody Payment payment);
}
- 결재 요청을 받은 직후(@PostPersist) 정상적인 서비스일 경우 pub/sub 방식으로 완료 회신

# Payment.java (Entity)
    @PostPersist
    public void onPostPersist(){
        System.out.println("###payment.java - PrePersist###");
        System.out.println(this.getReceiptId());
        PaymentApproved paymentApproved = new PaymentApproved();
        BeanUtils.copyProperties(this, paymentApproved);
        paymentApproved.publishAfterCommit();
    }
```
- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 결재요청을 못받는다는 것을 확인:
```
# 결제 (pay) 서비스를 잠시 내려놓음 (ctrl+c)
#지불 요청
http PATCH http://localhost:8088/receipts/1 stat=PAYMENTREQUESTED   #Fail
http PATCH http://localhost:8088/receipts/2 stat=PAYMENTREQUESTED   #Fail
#payment 재기동
cd payment
mvn spring-boot:run
#주문처리
http PATCH http://localhost:8088/receipts/1 stat=PAYMENTREQUESTED   #Success
http PATCH http://localhost:8088/receipts/1 stat=PAYMENTREQUESTED   #Success
```
- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)
## 비동기식 호출 / 장애격리 / 최종 (Eventual) 일관성 테스트
수리 프로세스에 문제가 있더라도 접수는 계속 받을 수 있도록 비동기식 호출하여 처리 한다. (kafka)
추후에 수리 프로세스가 복구 완료되면 접수에서 정상적으로 수리 요청 되었다고 이벤트를 수신한다. (Polish Hanler 처리 및 kafka로 접수에 수리 요청 상태 변경 회신)
```
<receipt.java>
@PostPersist
public void onPostPersist() throws Exception {
    System.out.println("###Reservaton.java - onPrePersist###");
    try {
        // 수리 요청
        if(this.stat.equals("REQUESTREPAIR")) {
            Received received = new Received();
            BeanUtils.copyProperties(this, received);
            received.setReceiptId(this.getId());
            received.publishAfterCommit();
        }else{
            //Exception exception = new Exception();
            //throw exception; //예외 발생
        }
    }catch(Exception e) {
        throw new Exception("요청할 수 없는 상태 입니다.");
        //return;
    }finally {
    }
}
<Repair - PolicyHandler>
@Service
public class PolicyHandler{
    @Autowired
    RepairRepository repairRepository;
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverReceived_RequestRepair(@Payload Received received){
        if(received.isMe()){
            System.out.println("##### listener RequestRepair : " + received.toJson());
            Repair repair = new Repair();
            repair.setReceiptId(received.getReceiptId());
            repair.setVehiNo(received.getVehiNo());
            repair.setStat("REQUESTREPAIR");
            repairRepository.save(repair);
        }
    }
}
<Repair.java>
@PostPersist
public void onPostPersist(){
    System.out.println("###Repair.java - onPostPersist###");
    // 수리 요청
    if(this.getStat().equals("REQUESTREPAIR")) {
        RepairReceived repairReceived = new RepairReceived();
        BeanUtils.copyProperties(this, repairReceived);
        System.out.println("####"+this.getReceiptId()+"####");
        repairReceived.publishAfterCommit();
    }
}
```
수리 시스템은 접수/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 수리시스템이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다:
```
# 수리 서비스 (store) 를 잠시 내려놓음 (ctrl+c)
#처리
http POST http://localhost:8088/receipts vehiNo=1234 stat=REQUESTREPAIR   #Success
http POST http://localhost:8088/receipts vehiNo=4567 stat=REQUESTREPAIR   #Success
#주문상태 확인
http GET http://localhost:8088/receipts     # 접수 완료로 변경되지 않음
http GET http://localhost:8088/repairs      # 수리 정보 없음
#수리 서비스 기동
cd repair
mvn spring-boot:run
#수리상태 확인
http GET http://localhost:8088/receipts     # 접수 완료로 변경
http GET http://localhost:8088/repairs      # 수리 정보 생성
```

# 운영

## CI/CD 설정

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 azure를 사용하였으며 CI/CD는 아래와 같습니다.

![image](https://user-images.githubusercontent.com/46660236/98197484-3ae4ce00-1f6a-11eb-9085-10c4ce540ac1.png)

![image](https://user-images.githubusercontent.com/46660236/98197489-3cae9180-1f6a-11eb-9793-7d9099238007.png)


## 동기식 호출 / 서킷 브레이킹 / 장애격리
* 서킷 브레이킹 프레임워크의 선택: Istio Destination rule
MSA 의 각 서비스들의 에러가 전파되는 것을 막기 위해서 특정 서비스에서 에러가 발생할 경우 해당 서비스로의 연결을 차단하도록 구성하였습니다.
- Destination rule 를 설정:  5xx Error 가 5번 연속적으로 발생 시 해당 서비스로의 연결을 15분 동안 끊는다.
```
# kubectl -n automechanic get destinationrule
NAME         HOST      AGE
display-dr   display   18h
payment-dr   payment   19h
receipt-dr   receipt   19h
repair-dr    repair    19h
```
```
# payment-dr.yml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-dr
  namespace: automechanic
spec:
  host: payment
  trafficPolicy:
    outlierDetection:
      baseEjectionTime: 15m
      consecutive5xxErrors: 5
      interval: 5m
      maxEjectionPercent: 100
```
- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 63.55% 가 성공하였고, 46%가 실패했다는 것은 고객 사용성에 있어 좋지 않기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.

- Retry 의 설정 (istio)
- Availability 가 높아진 것을 확인 (siege)

### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 90프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy payment --min=1 --max=10 --cpu-percent=90
```
- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
siege -c100 -t120S -r10 'http://20.196.136.26:8080/payments GET
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy pay -w
```
- 어느정도 시간이 흐른 후 스케일 아웃이 벌어지는 것을 확인할 수 있다:
```
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
payment   1/1     1            1           45h
payment   1/2     1            1           45h
payment   1/2     1            1           45h
payment   1/2     1            1           45h
payment   1/2     2            1           45h
:
```

## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"item": "chicken"}'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
:

```

- 새버전으로의 배포 시작
```
kubectl set image ...
```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
```
Transactions:		        3078 hits
Availability:		       70.45 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02

```
배포기간중 Availability 가 평소 100%에서 70% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:

```
# deployment.yaml 의 readiness probe 의 설정:


kubectl apply -f kubernetes/deployment.yaml
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
```
Transactions:		        3078 hits
Availability:		       100 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02

```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.

# 신규 서비스 추가

# 서비스 시나리오

기능적 요구사항
1. 접수처에서 수리 신청을 받는다.
1. 수리처에서 접수처에 접수된 수리를 받고 접수상태를 변경한다.
1. 수리처에서 수리 완료한 후 접수상태를 변경한다.
1. 접수처에서 수리 완료된 접수에 대해 결제 요청을 한다.
1. 고객이 결제하고 접수상태를 변경한다.
1. 접수처에서 수리 취소를 요청한다.
1. 수리처에서 취소 요청된 수리를 취소하고 접수상태를 변경한다.
1. 수리처에서 접수처에 접수된 수리를 거절하고 접수상태를 변경한다.

2. 수리처에서 렌터처로 렌터카 접수를 요청한다.
2. 렌터처에서 수리처에서 접수된 렌터 요청을 받고 접수상태를 변경한다.
2. 접수처에서 렌터카 배정 완료된 접수에 대해 결제 요청을 한다.
2. 수리처에서 렌터처로 렌터카 취소 요청한다.
2. 렌터처에서 취소 요청된 렌터카 접수를 취소하고 접수상태를 변경한다.


비기능적 요구사항
1. 트랜잭션
    1. 수리가 완료된 수리접수건에 대해서만 결제가 성립되어야 한다 Sync 호출
    
    2. 수리 접수건에 대해서 렌터 카가 배정되어야한다.
1. 장애격리
    1. 수리처 기능이 수행되지 않더라도 접수은 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency
    1. 결제시스템에 문제가 발생했을 때 결제를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다 Circuit breaker, fallback
    
    2. 렌터 카 배정에 문제가 발생했을 때 결제를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다.
    
1. 성능
    1. 접수처에서 수시로 접수상태를 확인할 수 있어야 한다. CQRS
    1. 수리처에서 상태를 변경할 수 있어야 한다.
    1. 고객이 직접 결제할 수 있어야 한다.
    
    2. 렌터처에서 상태를 변경할 수 있어야 한다.

## 렌터카 업체의 추가
    - KPI: 렌터카를 제공함으로써 고객에 편리성 제공
    - 구현계획 마이크로 서비스: 수리처에 렌터카 필요하다고 판단될때 렌터카를 제공할 예정

## 이벤트 스토밍 
![image](https://user-images.githubusercontent.com/33418976/98324387-74cbd800-202f-11eb-95aa-4527fdb646c4.png)


## 헥사고날 아키텍처 변화 
![image](https://user-images.githubusercontent.com/33418976/98323950-5e714c80-202e-11eb-91f0-6777bc67dfb3.png)

# 구현:
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 리액트으로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)
```
cd receipt
mvn spring-boot:run
cd repair
mvn spring-boot:run
cd payment
mvn spring-boot:run
cd display
mvn spring-boot:run
cd rental
mvn spring-boot:run
cd car-react
npm start
```

## DDD 의 적용
- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 pay 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 하지만, 일부 구현에 있어서 영문이 아닌 경우는 실행이 불가능한 경우가 있기 때문에 계속 사용할 방법은 아닌것 같다. (Maven pom.xml, Kafka의 topic id, FeignClient 의 서비스 id 등은 한글로 식별자를 사용하는 경우 오류가 발생하는 것을 확인하였다)
```
package automechanicsmall;
import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
@Entity
@Table(name="Rental")
public class Rental {
    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String rentalvehiNo;
    private String rentalStat;
    private Integer repairRentalAmt;
   
public Long getId() {
        return id;
    }
    public void setId(Long id) {
        this.id = id;
    }
    public String getrentalvehiNo() {
        return vehiNo;
    }
    public void setrentalvehiNo(String rentalvehiNo) {
        this.rentalvehiNo = rentalvehiNo;
    }
    public String getrentalStat() {
        return rentalstat;
    }
    public void setrentalStat(String stat) {
        this.stat = stat;
    }
    public Integer getrepairRentalAmt() {
        return repairRentalAmt;
    }
    public void setrepairRentalAmt(Integer repairRentalAmt) {
        this.repairRentalAmt = repairRentalAmt;
    }
}
```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RentalRepository 를 적용하였다
```
package automechanicsmall;
import org.springframework.data.repository.PagingAndSortingRepository;
import java.util.List;
public interface RentalRepository extends PagingAndSortingRepository<Rental, Long>{
}
```
- 적용 후 REST API 의 테스트
```
# 접수 생성 & 렌트 요청
http POST http://localhost:8088/rentals rentalvehiNo=12허1234 stat=REQUESTRENTAL
# 렌트 완료
http POST http://localhost:8088/rentals rentalvehiNo=12허1234 stat=REQUESTRENTAL
# 렌트 지불
http PATCH http://localhost:8088/rentals/1 stat=PAYMENTREQUESTED repairRentalAmt=0
# 각 어그리게이트 확인 (접수, 수리, 비용지불,렌트)
http http://localhost:8088/receipts/1
http http://localhost:8088/repairs/1
http http://localhost:8088/payments/1
http http://localhost:8088/rentals/1
```

## 폴리글랏 퍼시스턴스
마이크로서비스의 폴리그랏 퍼시스턴스의 예로 데이터의 빈번한 입출력을 사용하는 부분은 Display(View)의 저장소는 Mongo DB(NO SQL)를 사용하고 그 외의 업무 도메인인 접수(Receipt), 수리(Repair), 결재(Payment), 렌트(Rental)는 Maria DB(RDB)를 사용하였다.
application.yml의 간단한 설정을 통해 설정이 가능하다.

```
# (rental)application.yml

---
spring:
  profiles: nosql
  data:
    mongodb:
      uri: mongodb://${mongodb.url}:27017/rental
...
---
spring:
  profiles: docker
...
  datasource:
    url: jdbc:mariadb://${datasource.url}:3306/rental?useSSL=true
    username: ${datasource.user}
    password: ${datasource.password}
    driverClassName: org.mariadb.jdbc.Driver

```
# 운영

## CI/CD 설정

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 azure를 사용하였으며 CI/CD는 아래와 같습니다.

![image](https://user-images.githubusercontent.com/33418976/98322148-194b1b80-202a-11eb-83c4-37d043a0bfbd.png)

![image](https://user-images.githubusercontent.com/33418976/98322177-2cf68200-202a-11eb-8462-9f7cef61c8e5.png)


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

 
- 이를 위하여 공간배치 기록을 남긴 후에 곧바로 결제승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
@Entity
@Table(name="Place")
public class Place {

 ...
    @PostPersist
    public void onPostPersist(){
        if(this.state.equals("COMPLETE")){
            AssignApproved assignApproved = new AssignApproved();
            assignApproved.setId(this.id);
            assignApproved.setPlaceNumber(this.placeNumber);
            assignApproved.setRepairId(this.repairId);
            BeanUtils.copyProperties(this, assignApproved);
            assignApproved.publishAfterCommit();
        }
        if(this.state.equals("FAILED")){
            AssignApproved assignApproved = new AssignApproved();
            assignApproved.setId(this.id);
            assignApproved.setPlaceNumber(this.placeNumber);
            assignApproved.setRepairId(this.repairId);
            BeanUtils.copyProperties(this, assignApproved);
            assignApproved.publishAfterCommit();
        }
    }

}
```
- 상점 서비스에서는 결제승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
package fooddelivery;

...

@Service
public class PolicyHandler{

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverAssignApproved_RequestRepair(@Payload AssignApproved assignApproved){

        if(assignApproved.isMe()){
            if(assignApproved.getRepairId()==null){
                System.out.println("ID is null.");
                return;
            }
            Optional<Repair> entity = repairRepository.findById(assignApproved.getRepairId());
            if(!entity.isPresent()) { // null
                System.out.println("There is no such Repair Log");
            } else{
                Repair repair = entity.get();
                if(assignApproved.getPlaceNumber().equals(0) ){ // place assign 실패
                    System.out.println("PLACE ASSIGNE 실패");
                    repair.setStat("ASSIGNEFAILED");
                } else{ // place assing
                    System.out.println("PLACE ASSIGNE 성공");
                    repair.setStat("PLACEASSIGNED");
                }
                repairRepository.save(repair);
            }
        }
    }

}

```
실제 구현을 하자면, 카톡 등으로 점주는 노티를 받고, 요리를 마친후, 주문 상태를 UI에 입력할테니, 우선 주문정보를 DB에 받아놓은 후, 이후 처리는 해당 Aggregate 내에서 하면 되겠다.:
  
```
  @Autowired 주문관리Repository 주문관리Repository;
  
  @StreamListener(KafkaProcessor.INPUT)
  public void whenever결제승인됨_주문정보받음(@Payload 결제승인됨 결제승인됨){

      if(결제승인됨.isMe()){
          카톡전송(" 주문이 왔어요! : " + 결제승인됨.toString(), 주문.getStoreId());

          주문관리 주문 = new 주문관리();
          주문.setId(결제승인됨.getOrderId());
          주문관리Repository.save(주문);
      }
  }

```

상점 시스템은 주문/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 상점시스템이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다:
```
# Place 를 잠시 내려놓음 (ctrl+c)

#주문 접수 
curl -X PATCH http://localhost:8088/repairs/1  -d '{"vehiNo":"0000", "stat":"PLACEREQUEST"}' -H 'Content-Type':'application/json' # 성공

#주문
curl -X GET http://localhost:8088/repairs     # 상태 안바뀜 확인

# Place 서비스 기동

#공간배정 상태 확인
curl -X GET http://localhost:8088/places     # 상태가 "PLACEASSIGNED"으로 확인
```


# 운영

## CI/CD 설정

## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Istio Destination rule MSA 의 각 서비스들의 에러가 전파되는 것을 막기 위해서 특정 서비스에서 에러가 발생할 경우 해당 서비스로의 연결을 차단하도록 구성하였습니다.

Place 서비스에 부하가 차거나 에러가 발생할 경우 연결 차단

- Destination rule 를 설정: http connection pool 이 1개가 차면 연결을 끊는다. 5xx Error 가 5번 연속적으로 발생 시 해당 서비스로의 연결을 5분 동안 끊는다.
```
# ❯ kubectl -n car get destinationrule rental-dr -o yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: rental-dr
  namespace: car
spec:
  host: rental
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      baseEjectionTime: 5m
      consecutive5xxErrors: 1
      interval: 5s
      maxEjectionPercent: 100

```
