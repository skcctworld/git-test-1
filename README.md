# 서비스 시나리오

기능적 요구사항
1. 고객이 마스크를 주문한다
1. 주문이 되면 주문내역이 배송팀에 전달된다
1. 주문이 되면 주문내역이 재고팀에 전달된다
 (상품팀이 상품을 등록한다)
1. 고객이 주문을 취소할 수 있다
1. 주문이 취소되면 배송이 취소된다
1. 주문이 취소되면 재고가 변경된다
 (상품팀이 상품을 변경할 수 있다)
 (상품정보가 변경되면 재고정보가 변경된다)

비기능적 요구사항
1. 트랜잭션
    배송팀에 할당되지 않은 주문건은 주문이 성립되지 않아야 한다  Sync 호출 
1. 장애격리
    (.....)
1. 성능
    1. 고객이 MyPage에서 주문정보를 확인할 수 있어야 한다  CQRS
  

# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://msaez.io/#/storming/myQFPsZHyPTzv4dF1Rp3utSmqZ22/share/373ad0f49c8377342299860271bed963/-MDnu4FWQAjhi_RHUFmd


### 이벤트 도출
![image](https://github.com/yslim83/git-test/blob/master/report_images_1/eventstorming_team_cmd_evt.png)

### 어그리게잇과 바운디드 컨텍스트로 묶기
![image](https://github.com/yslim83/git-test/blob/master/report_images_1/eventstorming_team_aggr_bc.png)

    - 주문, 배송, 재고와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위끼리 묶어줌

    - 도메인 서열 분리 
        - Core Domain:  order(주문), delivery(배송)
        - Supporting Domain:  inventory(재고)
        - General Domain:   My Page

### 폴리시 부착 

![image](https://github.com/yslim83/git-test/blob/master/report_images_1/eventstorming_team_policy.png)

    - 재고의 event를 삭제

### 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![image](https://github.com/yslim83/git-test/blob/master/report_images_1/eventstorming_team.png)


### 기능적/비기능적 요구사항을 커버하는지 검증

![image](https://github.com/yslim83/git-test/blob/master/report_images_1/eventstorming_team_1_func.png)

    - 고객이 마스크를 주문한다
    - 배송팀에 주문내역이 전달된다
    - 재고수량에서 주문수량만큼 차감된다

![image](https://github.com/yslim83/git-test/blob/master/report_images_1/eventstorming_team_2_func.png)

    - 고객이 주문을 취소할 수 있다
    - 주문이 취소되면 배송 상태값이 변경된다
    - 주문이 취소되면 취소된 수량만큼 재고 수량이 증가한다 


### 비기능 요구사항에 대한 검증

![image](https://user-images.githubusercontent.com/487999/79684184-5c9a9400-826a-11ea-8d87-2ed1e44f4562.png)

    - 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
        - 주문완료 시 배송정보 생성에 대해서는 Request-Response 방식 처리
        - 배송정보 생성 완료 시 재고처리:  delivery 에서 inventory 마이크로서비스로 요청이 전달되는 과정에 있어서 inventory 마이크로 서비스가 별도의 배포주기를 가지기 때문에 Eventual Consistency 방식으로 트랜잭션 처리함.
        




## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://github.com/yslim83/git-test/blob/master/report_images_1/hexagonal.png)


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 파이선으로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 8085 이다)

```
cd order
mvn spring-boot:run

cd delivery
mvn spring-boot:run 

cd inventory
mvn spring-boot:run  

cd gateway
mvn spring-boot:run  

cd mypage
mvn spring-boot:run
```

DDD 의 적용
각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 order 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다.
package maskShop3;

@entity
@table(name="Order_table")
public class Order {

@Id
@GeneratedValue(strategy=GenerationType.AUTO)
private Long id;
private Long orderId;
private Long productId;
private Integer qty;
private String type;
public Long getId() {
return id;
}
public void setId(Long id) {
this.id = id;
}

public Long getOrderId() {
    return orderId;
}
public void setOrderId(Long orderId) {
    this.orderId = orderId;
}

public Long getProductId() {
    return productId;
}
public void setProductId(Long productId) {
    this.productId = productId;
}

public Integer getQty() {
    return qty;
}
public void setQty(Integer qty) {
    this.qty = qty;
}

public String getType() {
    return type;
}

public void setType(String type) {
    this.type = type;
}
}


Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다\
package maskShop3;
import org.springframework.data.repository.PagingAndSortingRepository;

    public interface OrderRepository extends PagingAndSortingRepository<Order, Long>{
}


적용 후 REST API 의 테스트

# order 서비스의 주문처리
http localhost:8081/orders orderId=1111 productId=1111 qty=10
# order 상태 확인
http localhost:8081/orders/1

# inventory 서비스의 재고처리
http localhost:8085/inventories productId=1111 invQty=100
# inventoryu 상태 확인
http localhost:8085/inventories/1


동기식 호출 처리

분석단계에서의 조건 중 하나로 주문(order) -> 배송(delivery) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.

delivery 서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현

# (order) deliveryService.java
package maskShop3.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import java.util.Date;

@FeignClient(name="delivery", url="http://localhost:8082")
public interface DeliveryService {
    @RequestMapping(method= RequestMethod.POST, path="/deliveries")
    public void update(@RequestBody Delivery delivery);
}


- 주문을 받은 직후(@PostPersist) delivery 서비스를 요청하도록 처리

Order.java (Entity)
@PostPersist
public void onPostPersist(){

    // order -> delivery create
    maskShop3.external.Delivery delivery = new maskShop3.external.Delivery();
    delivery.setOrderId(getOrderId());
    delivery.setStatus("ordered");
    delivery.setProductId(getProductId());
    delivery.setInvQty(getQty());
    delivery.setId(getId()+10000);
    OrderApplication.applicationContext.getBean(maskShop3.external.DeliveryService.class).update(delivery);

}

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, delivery 시스템이 장애가 나면 주문도 못받는다는 것을 확인:

delivery 서비스를 잠시 내려놓음 (ctrl+c)

#주문처리
http localhost:8081/orders orderId=1111 productId=1111 qty=10 #Fail
http localhost:8081/orders orderId=2222 productId=2222 qty=20 #Fail

#delivery 재기동
mvn spring-boot:run

#주문처리
http localhost:8081/orders orderId=1111 productId=1111 qty=10 #success
http localhost:8081/orders orderId=2222 productId=2222 qty=20 #success



## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


delivery 가 발생한 후 inventory 에  재고를 업데이트 하는 이벤트는 동기식이 아니라 비 동기식으로 처리하여 상점 시스템의 처리를 위하여 주문이 블로킹 되지 않아도록 처리한다.
 
- 이를 위하여 delivery 가 발생되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
package fooddelivery;

@entity
@table(name="Delivery_table")
public class Delivery {

@Id
@GeneratedValue(strategy=GenerationType.AUTO)
private Long id;
private Long orderId;
private String status;
private Long productId;
private Integer invQty;

@PostPersist
public void onPrePersist(){

    DeliveryRegisterd deliveryRegisterd = new DeliveryRegisterd();
    BeanUtils.copyProperties(this, deliveryRegisterd);
    deliveryRegisterd.publish();

 }
}

- inventory 에서는 deliveryRegister 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

package maskShop3;

import maskShop3.config.kafka.KafkaProcessor;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

@service
public class PolicyHandler{

@Autowired
InventoryRepository inventoryRepository;

@StreamListener(KafkaProcessor.INPUT)
public void wheneverDeliveryRegisterd_Change(@Payload DeliveryRegisterd deliveryRegisterd){

    if(deliveryRegisterd.isMe()){
    
            System.out.println("##### listener INVENTORY INSERT ======================");
            inventory.setProductId(deliveryRegisterd.getProductId());
            inventory.setInvQty(deliveryRegisterd.getInvQty());
            inventoryRepository.save(inventory);

    }
}
inventory 에서는 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 재고시스템이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다:

# inventory 서비스를 잠시 내려놓음 

#주문처리
http localhost:8081/orders orderId=1111 productId=1111 qty=10   #success
http localhost:8081/orders orderId=2222 productId=2222 qty=20   #success

#주문상태 확인
http localhost:8081/orders     # 주문 처리됨

#inventory 서비스 기동
mvn spring-boot:run

#inventory 상태 확인
http localhost:8085/inventories/1 재고변경 확인



운영

CI/CD 설정
각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 GCP를 사용하였다.

오토스케일 아웃
주문요청 폭주에 대비하여 자동화된 확장 기능을 적용하고자 한다.

order 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:

kubectl autoscale deploy order --min=1 --max=10 --cpu-percent=15

워크로드를 2분 동안 걸어준다.
siege -c100 -t120S -r10 --content-type "application/json" 'http://order:8080/orders POST {"productId":1, "orderId":"1", "qty":"1000"}'


오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
kubectl get deploy order -w

어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:

NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
order     1         1         1            1           17s
order    1         2         1            1           45s
order    1         4         1            1           1m
:
