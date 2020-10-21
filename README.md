# Table of contents

- [장비대여](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
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
    - [configmap](#신규-개발-조직의-추가)

# 서비스 시나리오



기능적 요구사항
1.	Manager는 장비를 등록할 수 있다.
2.	Employee가 장비를 오더하면 승인 요청이 생성되며 자동 승인된다
3.	요청이 승인되면 오더의 상태가 APPROVED로 변경되며, 장비의 재고가 오더된 만큼 감소한다.
4.	Employee는 장비 오더를 취소할 수 있다.
5.	오더가 취소되면 승인 상태가 CANCELLED으로 변경되고 취소된 오더만큼 장비의 재고가 증가한다.
6.	Employee는 장비 오더의 상태를 확인할 수 있다.





비기능적 요구사항
1. 트랜잭션
    1. 장비 오더는 승인이 되지 않는 경우 성립불가해야한다.  Sync 호출
    1. 오더와 승인은 동시에 일어난다.  Sync 호출

1. 장애격리
    1. 승인 모듈이 멈추더라도 오더는365일 24시간 받을 수 있어야 한다. Async 호출 (event-driven)
    1. 오더 시스템이 과중되면 승인을 받지 않고 오더취소를 잠시후에 하도록 유도한다  Circuit breaker, fallback

1. 성능
    1. Employee는 오더의 상태를 확인할 수 있다.  CQRS
    1. 오더가 승인되면 오더의 상태도 변경된다.  Async 호출






# 분석/설계


## AS-IS 조직 (Horizontally-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684144-2a893200-826a-11ea-9a01-79927d3a0107.png)

## TO-BE 조직 (Vertically-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684159-3543c700-826a-11ea-8d5f-a3fc0c4cad87.png)


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과
![image](https://user-images.githubusercontent.com/70302894/96408508-9f8bf300-121e-11eb-8f13-5696543089c7.JPG)

### 이벤트 도출
![image](https://user-images.githubusercontent.com/70302894/96408026-9d756480-121d-11eb-8518-a90e32816873.JPG)



### 어그리게잇으로 묶기 / 액터, 커맨드 부착하여 읽기 좋게
![image](https://user-images.githubusercontent.com/70302894/96409240-ec23fe00-121f-11eb-9ccd-66b428a4047f.JPG)

    - 장비, 승인,  등은 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 묶어준다.

### 바운디드 컨텍스트로 묶기

![image](https://user-images.githubusercontent.com/70302894/96409243-ed552b00-121f-11eb-8f12-d704f1d81447.JPG)

    - 도메인 서열 분리 
        - 오더 : 오더 오류를 최소화 한다. (Core)
        - 승인 : 승인 오류를 최소화 한다. (Supporting)
        - 장비 : 장비 재고 관리 오류를 최소화 한다. (Supporting)

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

![image](https://user-images.githubusercontent.com/70302894/96409245-ededc180-121f-11eb-9389-90bc8bd86a5e.JPG)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![image](https://user-images.githubusercontent.com/70302894/96409246-ededc180-121f-11eb-882b-9a5dc9ccc0fd.JPG)

### 완성된 1차 모형

![image](https://user-images.githubusercontent.com/70302894/96409248-ee865800-121f-11eb-9f83-d0c20f596095.jpg)



### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

![image](https://user-images.githubusercontent.com/70302894/96409249-ef1eee80-121f-11eb-987e-a1af6fbb00d9.jpg)

    - Employee가 오더의 상태를 확인한다.(?)
    - Employee가 장비를 선택해 오더한다. (OK)
    - 오더 시 자동 승인된다. (OK)
    - 승인 시 장비의 재고가 감소한다. (OK)
    - 승인 시 오더의 상태가 APPROVED로 변경된다. (OK)
    






![image](https://user-images.githubusercontent.com/70302894/96409250-efb78500-121f-11eb-8d85-83d1ddb234b8.jpg)


    - Employee가 오더를 취소한다. (OK)
    - 오더 취소 시 자동 승인 취소된다. (OK)
    - 승인이 취소되면 장비의 재고가 증가한다. (OK)





### 모델 수정

![image](https://user-images.githubusercontent.com/70302894/96394207-5d51ba00-11fc-11eb-80d9-1d5bb4356b1a.JPG)
    
    - View Model 추가 (CQRS 적용)
    - 수정된 모델은 모든 요구사항을 커버함.


### 비기능 요구사항에 대한 검증

![image](https://user-images.githubusercontent.com/70302894/96408042-a1a18200-121d-11eb-93cb-c21d735967df.JPG)


    - 1. 장비등록 서비스를 오더 / 승인 서비스와 격리되어 오더 / 승인 모듈 장애 시에도 장비 등록 가능
    - 2. 오더 승인 시 오더의 상태도 
 









## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/70302894/96408043-a23a1880-121d-11eb-96a8-4936ba2a46b7.JPG)


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd equipment
mvn spring-boot:run

cd order
mvn spring-boot:run 

cd approval
mvn spring-boot:run  

cd mis
mvn spring-boot:run  
```

gateway의 application.yml에 spirng, docker 환경별 uri이 설정되어있다.

```

        - id: order
          uri: http://localhost:8081
          predicates:
            - Path=/orders/** 
        - id: approval
          uri: http://localhost:8082
          predicates:
            - Path=/approvals/** 
        - id: equipment
          uri: http://localhost:8083
          predicates:
            - Path=/equipment/** 
        - id: mis
          uri: http://localhost:8084
          predicates:
            - Path= /mis/** 
	    

        - id: order
          uri: http://order:8080
          predicates:
            - Path=/orders/** 
        - id: approval
          uri: http://approval:8080
          predicates:
            - Path=/approvals/** 
        - id: equipment
          uri: http://equipment:8080
          predicates:
            - Path=/equipment/** 
        - id: mis
          uri: http://mis:8080
          predicates:
            - Path= /mis/**    
```



## DDD 의 적용

- 각 서비스내에 도출된 Aggregate Root 객체를 Entity 로 선언하였다. (Approval)
- Approval모듈은 각 연동 서비스의 key를 가지고 있어 연동 시 어떠한 요청 건인지 구별 가능하다. (Correlation-key)

```
package equipmgnt;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Approval_table")
public class Approval {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long equipmentId;
    private Long orderId;
    private Integer qty;
    private String status;
    
    ....중략

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Long getEquipmentId() {
        return equipmentId;
    }

    public void setEquipmentId(Long equipmentId) {
        this.equipmentId = equipmentId;
    }
    public Long getOrderId() {
        return orderId;
    }

    public void setOrderId(Long orderId) {
        this.orderId = orderId;
    }
    public Integer getQty() {
        return qty;
    }
    ....중략
}


```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package equipmgnt;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface OrderRepository extends PagingAndSortingRepository<Order, Long>{


}
```
- 적용 후 REST API 의 테스트
```
# equipment 서비스의 장비등록
http http://localhost:8083/equipments stock=10 

# order 서비스의 오더
http http://localhost:8081/orders qty=2 equipmentId=1 status=ORDERED

# approval 서비스의 승인 상태 확인
http http://localhost:8082/approvals

# order 서비스의 오더 상태 확인
http http://localhost:8081/orders/1

# equipment 서비스의 장비 상태 확인
http http://localhost:8083/equipments/1

```


## 폴리글랏 퍼시스턴스

각 마이크로서비스는 별도의 H2 DB를 가지고 있으며 CQRS를 위한 MIS 서비스에서는 H2가 아닌 HSQLDB를 적용하였다.

```
# MIS의 pom.xml에 dependency 추가
<!-- 
		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>
 -->
		<dependency>
		    <groupId>org.hsqldb</groupId>
		    <artifactId>hsqldb</artifactId>
		    <version>2.4.0</version>
		    <scope>runtime</scope>
		</dependency>


```




## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 오더(order)->승인(approval) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 승인서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
# Order서비스의 ApprovalService.java


package equipmgnt.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="approval", url="${api.approval.url}")
public interface ApprovalService {

    @RequestMapping(method= RequestMethod.POST, path="/approvals")
    public void requestapprove(@RequestBody Approval approval);

}

# Order서비스 생성 시 승인 내역의 Approval객체를 같이 던진다. (자동승인)
   @PostPersist
    public void onPostPersist(){

        try {
            Thread.currentThread().sleep((long) (800 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        ordered.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.


        equipmgnt.external.Approval approval = new equipmgnt.external.Approval();

        approval.setOrderId(ordered.getId());
        approval.setQty(ordered.getQty());
        approval.setEquipmentId(ordered.getEquipmentId());
        approval.setStatus("APPROVED");
        // mappings goes here
        OrderApplication.applicationContext.getBean(equipmgnt.external.ApprovalService.class)
            .requestapprove(approval);


    }


```



- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 승인 시스템이 장애가 나면 오더도 못받는다는 것을 확인:


```
# 승인 (approval) 서비스를 잠시 내려놓음 (ctrl+c)

#오더취소처리
http http://localhost:8081/orders qty=2 equipmentId=1 status=ORDERED   #Fail
http http://localhost:8081/orders qty=3 equipmentId=1 status=ORDERED   #Fail

#승인서비스 재기동
cd approval
mvn spring-boot:run

#오더취소처리
http http://localhost:8081/orders qty=2 equipmentId=1 status=ORDERED   #Success
http http://localhost:8081/orders qty=3 equipmentId=1 status=ORDERED   #Success
```

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)




## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


승인이 이루어진 후에 장비 모듈에 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 장비 시스템의 처리가 블로킹 되지 않도록 처리한다.
 
- 이를 위하여 승인이력에 기록을 남긴 후에 곧바로 승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
    @PostPersist
    public void onPostPersist(){

        ApprovalObtained approvalObtained = new ApprovalObtained();
        BeanUtils.copyProperties(this, approvalObtained);
        approvalObtained.publishAfterCommit();


    }
```
- 장비 서비스에서는 승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
   @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }
    @Autowired
    EquipmentRepository equipmentRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverApprovalObtained_Decrease(@Payload ApprovalObtained approvalObtained){

        if(approvalObtained.isMe()){
            System.out.println("##### listener Decrease : " + approvalObtained.toJson());

            Optional<Equipment> equipmentOptional = equipmentRepository.findById(approvalObtained.getEquipmentId());

            Equipment equipment = equipmentOptional.get();
            equipment.setStock(equipment.getStock()-approvalObtained.getQty());


            equipmentRepository.save(equipment);
        }
    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverCancelRequested_Increased(@Payload CancelRequested cancelRequested){

        if(cancelRequested.isMe()){
            System.out.println("##### listener Increased : " + cancelRequested.toJson());
            Optional<Equipment> equipmentOptional = equipmentRepository.findById(cancelRequested.getEquipmentId());

            Equipment equipment = equipmentOptional.get();
            equipment.setStock(equipment.getStock()+cancelRequested.getQty());

            equipmentRepository.save(equipment);
        }
    }
  
```



- 오더-승인 시스템은 비동기로 처리되므로 승인 시스템이 유지보수로 인해 잠시 내려간 상태라도 오더를 받는덴 문제가 없다:
 
```
# 승인 서비스 (approval) 를 잠시 내려놓음 (ctrl+c)


#오더처리
http http://localhost:8081/orders qty=2 equipmentId=1 status=ORDERED   #Success

#오더상태 확인
http localhost:8081/orders     # APPROVED로 안바뀜 확인

#승인서비스 기동
cd approval
mvn spring-boot:run

#오더상태 확인
http localhost:8081/orders     # 주문의 상태가 "APPROVED"으로 확인
```


# 운영

## CI/CD 설정


각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AWS CodeBuild를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 buildspec.yml 에 포함되었다.

## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹은 istio destination 룰 적용하여 구현한다.

시나리오는 오더-->승인 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.



- 피호출 서비스(승인)의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# Approval.java (Entity)

    @PostPersist
    public void onPostPersist(){
        try {
            Thread.currentThread().sleep((long) (800 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        ApprovalObtained approvalObtained = new ApprovalObtained();
        BeanUtils.copyProperties(this, approvalObtained);
        approvalObtained.publishAfterCommit();


    }
```

서킷브레이킹 미적용 시 100%임을 확인

![서킷브레이킹미적용](https://user-images.githubusercontent.com/70302894/96725059-766c8d80-13eb-11eb-9dd0-3e2fa9a797e4.JPG)


데스티네이션 룰 적용
```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-approval
  namespace: istio-cb-ns
spec:
  host: skccuser24-approval
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1024           # 목적지로 가는 HTTP, TCP connection 최대 값. (Default 1024)
      http:
        http1MaxPendingRequests: 1  # 연결을 기다리는 request 수를 1개로 제한 (Default 
        maxRequestsPerConnection: 1 # keep alive 기능 disable
        maxRetries: 3               # 기다리는 동안 최대 재시도 수(Default 1024)
    outlierDetection:
      consecutiveErrors: 2          # 5xx 에러가 5번 발생하면
      interval: 1s                  # 1초마다 스캔 하여
      baseEjectionTime: 30s         # 30 초 동안 circuit breaking 처리   
      maxEjectionPercent: 7       # 100% 로 차단
EOF
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 20명
- 20초 동안 실시

```
siege -c20 -t20S -v  --content-type "application/json" 'http://skccuser24-approval:8080/approvals POST {"orderId":"1","equipmentId":"1"}'
```

![서킷브레이킹적용](https://user-images.githubusercontent.com/70302894/96725060-766c8d80-13eb-11eb-8fb1-4086c7360a75.JPG)

- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 

### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy skccuser24-approval --min=1 --max=10 --cpu-percent=10 -n istio-cb-ns
kubectl autoscale deployment.apps/skccuser24-approval --cpu-percent=1 --min=1 --max=10 -n istio-cb-ns
```
- metrics-server 설치 및 확인
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
kubectl get deployment metrics-server -n kube-system
```

- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
siege -c20 -t120S -v  --content-type "application/json" 'http://skccuser24-approval:8080/approvals POST {"orderId":"1","equipmentId":"1"}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy skccuser24-approval -w
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:

![오토스케일링](https://user-images.githubusercontent.com/70302894/96725067-779dba80-13eb-11eb-8fb0-1d6b935d5e38.JPG)

- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 

![서킷브레이킹 리트라이룰적용](https://user-images.githubusercontent.com/70302894/96725053-753b6080-13eb-11eb-9703-a111e9b3187a.JPG)


## 무정지 재배포 (Readiness Probe)

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 Readiness Probe 미설정 시 무정지 재배포 가능여부 확인을 위해 buildspec.yml의 Readiness Probe 설정을 제거함

- Readiness Probe 제거한 상태에서 부하를 주고 측정 시 Availability가 80%대로 떨어짐. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:
```
siege -c20 -t120S -v  --content-type "application/json" 'http://skccuser24-approval:8080/approvals POST {"orderId":"1","equipmentId":"1"}'
```
![무정지배포88](https://user-images.githubusercontent.com/70302894/96725049-74a2ca00-13eb-11eb-801d-5ffe64dd8460.JPG)



```
# deployment.yaml 의 readiness probe 의 설정:


kubectl apply -f kubernetes/deployment.yaml
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
![무정지배포100](https://user-images.githubusercontent.com/70302894/96725051-753b6080-13eb-11eb-86ff-6822b9321164.JPG)

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.








## Liveness Probe




## configmap







