# Rent the WaterPurifier

## Table of contents
  - [서비스 시나리오](#서비스-시나리오)
  - [분석/설계](#분석설계)
  - [구현](#구현)
    - [DDD의 적용](#DDD의-적용)
    - [Polyglot Persistent](#Polyglot-Persistent)
    - [Polyglot Programming](#Polyglot-Programming)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출과 Eventual Consistency](#비동기식-호출과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#CI/CD-설정)
    - [동기식 호출/서킷 브레이킹/장애격리](#동기식-호출/서킷-브레이킹/장애격리)
    - [AutoScale Out](#AutoScale-Out)
    - [무정지 재배포](#무정지-재배포)
    - [Self Healing](#Self-Healing)
    - [개발 운영 환경 분리](#개발-운영-환경-분리)

## 서비스 시나리오
    기능적 요구사항
        1. 관리자는 정수기 정보를 등록한다.
        2. 고객은 관리자가 등록한 정수기를 조회한 후 원하는 정수기를 렌탈 신청한다.
        3. 고객은 신청한 정수기의 렌탈비를 결제한다
        4. 렌탈 신청이 완료되면 정수기를 배송 시작한다
        5. 고객은 렌탈 신청을 취소할 수 있다.
        6. 렌탈 신청이 취소되면 결제가 취소된다.
        7. 결제 취소되면 배송이 취소된다.
        8. 배송 시작/취소되면 정수기 재고 수량을 조정한다. 
        9. 고객은 언제든지 렌탈/배송 현황을 조회한다. (CQRS-View)
       10. 렌탈 신청 상태,배송상태가 바뀔때마다 해당 고객에게 문자 메세지가 발송된다. 

    비기능적 요구사항
        1. 트랜잭션 
            - 렌탈비 결제가 완료되어야만 렌탈 신청을 완료할 수 있다. -> Sync 호출
        2. 장애격리
            - 상품관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다 -> Async(event-driven),Eventual Consistency
            - 결제시스템이 과중되면 렌탈 신청자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다 -> Circuit breaker,fallback
        3. 성능
            고객이 수시로 렌탈/배송현황을 MyPage에서 확인할 수 있어야 한다 -> CQRS

## 분석/설계
### Event Storming 결과
    MSAEz 로 모델링한 이벤트스토밍 결과
    http://labs.msaez.io/#/storming/fMnS97KBhKdTR73T20ep9sUs6kE2/69c4cd8c92b1f6f3b5d35ea823ab4921

#### 1.이벤트 도출
 ![image](https://user-images.githubusercontent.com/87048633/129510838-93083903-ff02-40aa-ac7d-2baaa87b8e57.png)

#### 2.부적격 이벤트 탈락
 ![image](https://user-images.githubusercontent.com/87048633/129510865-2c6a3cfe-f293-4f2f-99ac-c18b86d7f7cf.png)
  - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
   - '상품조회됨' :  UI 의 이벤트이지, 업무적인 의미의 이벤트가 아니라서 제외
   - '렌탈신청내역조회' : 별도 VIEW로 구현 예정이라 제외
   - '렌탈결제됨' : '결제승인됨'과 중복되어 제외
 
#### 3.Actor, Command 부착하여 읽기 좋게
 ![image](https://user-images.githubusercontent.com/87048633/129569046-6ba9b793-304d-4848-b20d-53d156c53c5c.png)
 
#### 4.Aggregate으로 묶기
 ![image](https://user-images.githubusercontent.com/87048633/129568675-d7c24316-bfb1-4f0f-a24a-bba7a2443f54.png)
 
#### 5.Bounded Context로 묶기
 ![image](https://user-images.githubusercontent.com/87048633/129569416-dd8adf83-8a7f-4ab0-8b79-8d02135c7fad.png)

#### 6.Policy 부착
 ![image](https://user-images.githubusercontent.com/87048633/129569892-3798600e-603f-42f6-be15-8dc1b0dc2103.png)
 
#### 7.Policy의 이동과 Context 매핑
 ![image](https://user-images.githubusercontent.com/87048633/129570393-c1cec300-5a84-406f-9c15-fecdd9866b3f.png)
 
#### 8.완성된 1차 모형
 ![image](https://user-images.githubusercontent.com/87048633/129510939-ba685a74-0be2-4143-aa9e-c704ab1f0fe9.png)
 
#### 9.1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증
 ![image](https://user-images.githubusercontent.com/87048633/129571570-a6d5aee5-3eaa-4b6e-90ab-c1addedec257.png)
 (O)  1. 관리자는 정수기 정보를 등록한다.</br>
 (O)  2. 고객은 관리자가 등록한 정수기를 조회한 후 원하는 정수기를 렌탈 신청한다.</br>
 (O)  3. 고객은 신청한 정수기의 렌탈비를 결제한다.</br>
 (O)  4. 렌탈 신청이 완료되면 정수기를 배송 시작한다.</br>
 (O)  5. 고객은 렌탈 신청을 취소할 수 있다.</br>
 (O)  6. 렌탈 신청이 취소되면 결제가 취소된다.</br>
 (O)  7. 결제 취소되면 배송이 취소된다.</br>
 (O)  8. 배송 시작/취소되면 정수기 재고 수량을 조정한다. </br>
 (O)  9. 고객은 언제든지 렌탈/배송 현황을 조회한다. (CQRS-View)</br>
 (O) 10. 렌탈 신청 상태,배송상태가 바뀔때마다 해당 고객에게 문자 메세지가 발송된다. </br>
 
#### 10.모델수정
 ![image](https://user-images.githubusercontent.com/87048633/129571664-25aa977a-cc1c-4a18-a127-b131a02a308f.png)

#### 11.비기능적 요구사항에 대한 검증
 ![image](https://user-images.githubusercontent.com/87048633/129572639-d7a2731b-afbe-4a58-a5a5-32c569e96714.png)
 (O) 1. 트랜잭션</br>
       - 렌탈비 결제가 완료되어야만 렌탈 신청을 완료할 수 있다. -> Sync 호출</br>
 (O) 2. 장애격리</br>
       - 상품관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다 -> Async(event-driven),Eventual Consistency</br>
       - 결제시스템이 과중되면 렌탈 신청자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다 -> Circuit breaker,fallback</br>
 (O) 3. 성능</br>
       - 고객이 수시로 렌탈/배송현황을 MyPage에서 확인할 수 있어야 한다 -> CQRS</br>
 
#### 12.Hexagonal Architecture Diagram 도출
 ![image](https://user-images.githubusercontent.com/87048624/130107446-bb0b79eb-220f-4c20-bab2-7901632b7a1d.png)

 
## 구현
- 분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 Spring Boot와 Java로 구현하였다.</br>
(각자의 포트넘버는 8081 ~ 808n 이다)</br>
```
cd rental
mvn spring-boot:run

cd payment
mvn spring-boot:run 

cd product
mvn spring-boot:run  

cd mypage
mvn spring-boot:run

cd delivery
mvn spring-boot:run

cd gateway
mvn spring-boot:run

```
![image](https://user-images.githubusercontent.com/87048633/130033039-cfb1d4d5-395d-47b7-9687-8979ab42040a.png)

- AWS 클라우드의 EKS 서비스 내에 서비스를 모두 빌드한다.
![image](https://user-images.githubusercontent.com/87048633/130029192-6520c94a-ffe2-4bc3-93c9-3f3d6498bfe1.png)
![image](https://user-images.githubusercontent.com/87048633/130029296-b2324bb8-08de-4749-ae77-8e4a9de4cfc5.png)

### DDD의 적용
- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다. 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다.
- Project 서비스 (Project.java)
```java
    package com.example.product;

    import javax.persistence.*;
    import org.springframework.beans.BeanUtils;
    import java.util.List;
    import java.util.Date;

    @Entity
    @Table(name="Product_table")
    public class Product {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private int amt;
    private int stock;

    @PostPersist
    public void onPostPersist(){
        ProductDecresed productDecresed = new ProductDecresed();
        BeanUtils.copyProperties(this, productDecresed);
        productDecresed.publishAfterCommit();


    }

    @PreRemove
    public void onPreRemove(){
        ProductIncresed productIncresed = new ProductIncresed();
        BeanUtils.copyProperties(this, productIncresed);
        productIncresed.publishAfterCommit();


    }

    public Long getId() {
        return this.id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public int getAmt() {
        return this.amt;
    }

    public void setAmt(int amt) {
        this.amt = amt;
    }

    public int getStock() {
        return this.stock;
    }

    public void setStock(int stock) {
        this.stock = stock;
    }

}
```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```java
package com.example.rental;

import org.springframework.data.repository.CrudRepository;

public interface RentalRepository extends CrudRepository<Rental, Long> {

}
```

- 적용 후 REST API 의 테스트
==>>>>> 없는 캡처들도 추가로 캡처 할건지 통합해서 할건지 확인...

  - 상품(정수기) 등록
  
  ![image](https://user-images.githubusercontent.com/87048624/130093629-001bf63d-51fa-48fd-b48f-6cb2ed7145af.png)


  - 렌탈 가능 정수기 조회

  - 렌탈 신청
  
  ![image](https://user-images.githubusercontent.com/87048624/130093337-76f4cc66-bde8-4fc1-93c7-d847c509184e.png)

  - 렌탈 신청 확인

  - 결제 승인 확인

  - 배송 시작 확인
   1) 서비스 직접 조회
   
    ![image](https://user-images.githubusercontent.com/87048624/130100882-d053b088-2e49-476b-8521-fbe29367e739.png)
   
   2) gateway로 조회
   
    ![image](https://user-images.githubusercontent.com/87048624/130101109-121b595c-aedf-49dc-bf57-fb658d2756da.png)


  - 상품(정수기) 재고 감소 확인

   ![image](https://user-images.githubusercontent.com/87048624/130100487-dad88728-d494-4761-9063-4db1b89d47e8.png)

  - 렌탈 취소 
   
   ![image](https://user-images.githubusercontent.com/87048624/130101401-06674612-6182-405d-afc6-381d4af52c90.png)


  - 렌탈 취소 확인
  
   ![image](https://user-images.githubusercontent.com/87048624/130101518-5ccd7977-a265-4fab-b50f-c2dd9f66732a.png)


  - 결제 취소 확인

  - 배송 취소 확인

  - 상품(정수기) 재고 증가 확인

  - My Page에서 렌탈 신청 여부/결제성공여부/배송상태확인 (CQRS)
  
    mypage 조회시rentalPlaced 이벤트까지만 수신내역 확인, 모든 이벤트 수신내역 확인
    
     ![image](https://user-images.githubusercontent.com/87048624/130083651-18549595-ec82-4bca-a5e7-3d23952872dc.png)




### Polyglot Persistent / Polyglot Programming
- Polyglot Persistent 조건을 만족하기 위해 기존 h2 DB를 hsqldb로 변경하여 동작시킨다.
```
<!--
		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>
		-->

		<!-- polyglot start -->
		<dependency>
			<groupId>org.hsqldb</groupId>
			<artifactId>hsqldb</artifactId>
			<scope>runtime</scope>
		</dependency>
		<!-- polyglot end -->
```

- pom.xml 파일 내 DB 정보 변경 및 재기동 후 렌탈 처리</br>
![image](https://user-images.githubusercontent.com/87048624/130165475-6efe2537-9941-4e0a-a689-a6de60a8ee67.png)
</br>


### 동기식 호출
- 분석단계에서의 조건 중 하나로 렌탈(rental) -> 결제(Payment)간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다.
- 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.
 ![image](https://user-images.githubusercontent.com/87048624/130165134-0539dc96-5779-410c-ac46-f16f1ade7658.png)
 ![image](https://user-images.githubusercontent.com/87048624/130165157-e4028d4c-6ecd-44aa-b21f-d52a1bfecd6a.png)

 ![image](https://user-images.githubusercontent.com/87048624/130095444-62dfc940-cecf-4791-907e-bd0136ce1b25.png)


### 비동기식 호출과 Eventual Consistency
- 결제가 이루어진 후에 배송 시스템으로 이를 알려주는 행위는 비동기식으로 처리하여 배송 시스템의 처리를 위하여 결제 주문이 블로킹 되지 않아도록 처리한다.
+ 이를 위하여 결제이력에 기록을 남긴 후에 곧바로 결제승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)

![image](https://user-images.githubusercontent.com/87048624/130095576-6a139cd6-67c3-4667-ab7d-a4a961114e10.png) </br></br>



## 운영
### CI/CD 설정
- 각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 aws codebuild를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 buildspec.yml 에 포함되었다.
  ![image](https://user-images.githubusercontent.com/87048633/130006493-f79b40dc-242d-4684-95eb-e4a305abb6ef.png)
  ![image](https://user-images.githubusercontent.com/87048633/130006762-19c4648c-0e27-461b-897f-59aeeddb2bc6.png)
  ![image](https://user-images.githubusercontent.com/87048633/130007397-522fdd2e-cd61-4364-86b4-276afff0248d.png)
  ![image](https://user-images.githubusercontent.com/87048633/130007020-ce217d04-844f-423b-8bae-b141bc377ec8.png)
  
### 동기식 호출/서킷 브레이킹/장애격리
- 서킷 브레이킹 프레임워크의 선택: FeignClient + hystrix </br>
- Hystrix 를 설정: 요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 Circuit Breaker 회로가 닫히도록 설정</br>
  ![image](https://user-images.githubusercontent.com/87048624/130067453-789251b4-e84a-4036-a1f6-2fd1154d8203.png)

- 피호출 서비스(결제:payment) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게 처리함 
  ![image](https://user-images.githubusercontent.com/87048624/130067490-0a3c5a7e-9cda-4143-b115-76e5ef9bb8ba.png)

- 서킷 브레이크 적용전

  ![image](https://user-images.githubusercontent.com/87048624/130067551-49d8804e-af47-4913-a727-27ec6d01945b.png)

- 서킷 브레이크 적용후 

  ![image](https://user-images.githubusercontent.com/87048624/130067583-be97a8b1-6080-4145-a681-f87b476f58d3.png)



### AutoScale Out
- replica를 동적으로 늘려서 HPA를 설정한다.</br>
- 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다.</br>
- 렌탈 신청 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다.</br>
- 렌탈 서비스의 buildspec.yaml에 resource 설정을 추가한다. </br>
 ![image](https://user-images.githubusercontent.com/87048624/130164837-cbf48e01-6f6f-4fea-99a1-84e0fce444a1.png)

- 오토스케일링 설정 (해당 deployment 컨테이너의 최대값/최소값 설정)
  >kubectl autoscale deployment wizard-product --cpu-percent=50 --min=1 --max=10   
 
- 워크로드를 110명, 30초 동안 부하를 걸어준다.</br>
 ![image](https://user-images.githubusercontent.com/87048624/130164937-5be87dce-e3dd-4f3e-b1cd-aa02610c13de.png)</br>
 
- AutoScale이 어떻게 되고 있는지 모니터링을 걸어둔다.</br>
 ![image](https://user-images.githubusercontent.com/87048633/130032165-431e8c28-1d82-4a9a-9391-ddda9cae46a7.png)</br>
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다. </br>
 ![image](https://user-images.githubusercontent.com/87048633/130032517-f53b5f92-acf0-4e68-908e-f60cdc2044ab.png)</br>
 ![image](https://user-images.githubusercontent.com/87048633/130032462-2dba6d76-6bcf-41f2-a391-936f9aa1e5d0.png)</br>
- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다.</br>
 ![image](https://user-images.githubusercontent.com/87048633/130032321-6b55498c-56d0-40e2-8b46-c83aa66cb951.png)</br>


### 무정지 재배포
==========>>>> readiness probe 구현 안하게되면 삭제!! 

- readiness probe 를 통해 이후 서비스가 활성 상태가 되면 유입을 진행시킨다.

### Self-Healing
- Liveness probe 를 통해 Pod의 상태를 체크하다가, Pod의 상태가 비정상인경우 재시작한다. 
  - rental 서비스 yaml파일에 Liveness probe 설정을 한다. </br>  
   . /tmp/test 파일이 존재하는지,  5초(periodSeconds 파라미터 값)마다 확인</br>
  - 파일이 존재하지 않을 경우, 정상 작동에 문제가 있다고 판단되어 자동으로 컨테이너가 재시작</br>
    ![image](https://user-images.githubusercontent.com/87048624/130088840-997e5103-14f7-47c0-8148-b2f1d7de99d1.png)

  - 해당 pod가 재시작하는걸 확인한다.   
    ![image](https://user-images.githubusercontent.com/87048624/130086524-d479788b-1023-4a95-906d-dad1a9cef1e5.png) 
    ![image](https://user-images.githubusercontent.com/87048624/130086552-2c59944d-6f71-4332-8c82-1ce5eff50a85.png)



### 개발 운영 환경 분리
=======>>>> ConfigMap 구현안하면 삭제!!
- ConfigMap 사용하여 운영과 개발 환경 분리
- kafka환경


