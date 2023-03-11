# Spring/JPA
1. [요청과 응답으로 엔티티 대신 DTO 사용](#요청과-응답으로-엔티티-대신-DTO-사용)
2. [엔티티 수정은 병합(merge)보다 변경 감지(Dirty Checking)](#엔티티-수정은-병합merge보다-변경-감지dirty-checking)
3. [N + 1 문제](#n--1-문제)
4. [JPQL](#jpql)
   - [fetch join을 통해 N + 1 문제 해결](#fetch-join을-통해-n--1-문제-해결)
   - [fetch join의 distinct](#fetch-join의-distinct)
   - [fetch join vs 일반 join](#fetch-join-vs-일반-join)
   - [fetch join의 한계](#fetch-join의-한계)
5. [Hibernate의 Batch Fetching](#hibernate의-batch-fetching)
6. [적당한 Batch-Size와 Legacy 전략](#적당한-batch-size와-legacy-전략)

## 요청과 응답으로 엔티티 대신 DTO 사용

처음 JPA를 사용하여 API를 개발할 때 흔히 하는 실수가 있다. 

바로 엔티티를 외부 JSON과 바인딩(요청)하거나 외부에 노출(응답)하는 것이다.

```java
// MemberAPIController.java
@PostMapping("/api/v1/members")
public CreateMemberResponse joinV1(@RequestBody @Valid Member member) {
    Long id = memberService.join(member);
    return new CreateMemberResponse(id);
}

@GetMapping("/api/v1/members")
public List<Member> memberV1() {
    return memberService.findMembers();
}

@Data
@AllArgsConstructor
static class CreateMemberResponse {
    private Long id;
}
```

```java
// Member.java
@Entity
@Getter
public class Member {
    @Id @GeneratedValue
    @Column(name = "member_id")
    private Long id;

    @NotEmpty(message = "회원가입 시 이름은 반드시 입력해야 합니다.")
    private String name;

    @JsonIgnore
    private String password;
}
```
다음은 잘못된 예시이다.
Controller 첫 번째는 회원가입 API이고 엔티티를 파라미터로 받았다. 
두 번째는 회원 조회 API이고 엔티티 리스트를 return했다. 

이때 발생하는 문제점은 다음과 같다.

### 1. 엔티티 변경 시 API 스펙도 함께 변경된다.
API 스펙 변경은 협업 시 큰 문제이다.

### 2. 프레젠테이션 계층을 위한 로직이 엔티티에 추가되었다.   
서버는 요청에 대한 유효성을 검증해야 한다. 따라서 스프링에서 Validation 어노테이션을 사용하여 검증한다. 예시에서 회원가입 시 회원의 이름은 반드시 받기 위해 @NotEmpty를 붙였다.

그리고 엔티티를 그대로 전달할 시 엔티티의 모든 정보가 노출된다. 그래서 예시에서 password 같은 민감한 정보를 감추고 싶어 @JsonIgnore 어노테이션을 사용했다. 

엔티티는 **최대한 순수하게** 유지되어야 한다. 하지만 Validation(@NotEmpty), @JsonIgnore과 같이 프레젠테이션 계층을 위한 로직이 엔티티에 추가되었다. 

### 3. 하나의 엔티티가 모든 API의 요구사항을 담고 있다.
예시에서 Member 엔티티 안에 회원가입 API와 회원 조회 API에 대한 요구사항이 모두 담겼다. 많은 요구사항이 담긴 엔티티를 바라봤을 때 API에서 어떤 값이 들어오고 나가는지 알 수 없다.

만약 Member 엔티티에 대한 API가 늘어난다고 하자. Member 엔티티가 더욱 다양해지는 요구사항을 담기는 어려울 것이다.

### 4. 양방향 의존관계
웹 애플리케이션 아키텍처는 의존관계가 단방향으로 흐른다.

presentation → domain → data access

엔티티는 domain 계층에 해당한다.
이때 persentation 계층이 엔티티에 의존할 수는 있지만 엔티티는 presentation 계층에 의존해선 안된다.

앞선 예시에서 엔티티에 presentation 계층을 위한 로직이 존재한다. 따라서 엔티티는 presentation에 의존하고 있고 결국 양방향 의존관계를 이루고 있다.

### 5. 컬렉션을 응답으로 반환했다.
List<Member\>와 같이 컬렉션을 응답으로 반환한다면 문제가 발생한다. 바로 **API 스펙을 확장하기 어렵다**는 점이다.  

컬렉션을 응답으로 반환할 시 배열이 반환된다. 이때 배열의 길이를 응답에 추가해 달라는 요청이 들어오면 유연하게 대응하기 어렵다.

하지만 다음과 같이 반환한다면 유연하게 API 스펙 확장에 유연하게 대응할 수 있다.
```json
// bad👎
[
    {
        "name" : "park",
        "age" : 12
    }.
    {
        "name" : "kim",
        "age" : 40
    }
]
// good👍
{
    "count" : 2,
    "data" : [
        {
            "name" : "park",
            "age" : 12
        },
        {
            "name" : "kim",
            "age" : 40
        }
    ]

}
```

위와 같은 문제점을 해결하기 위해선 어떻게 해야할까? 결론은 _요청과 응답으로 엔티티 대신 DTO를 사용하라_ 다. API마다 DTO를 생성해서 요청과 응답으로 사용하면 DTO만 보고 API 스펙을 직관적으로 알 수 있다.

```java
// MemberAPIController.java
@PostMapping("/api/v2/members")
public CreateMemberResponse joinV2(@RequestBody @Valid CreateMemberRequest createMemberRequest) {
    Member member = new Member();
    member.setName(createMemberRequest.getName());
    Long id = memberService.join(member);
    
    return new CreateMemberResponse(id);
}

@Data
@AllArgsConstructor
static class CreateMemberResponse {
    private Long id;
}

@Data
@AllArgsConstructor
static class CreateMemberRequest {
    @NotEmpty
    private String name;
}

@GetMapping("/api/v2/members")
public Result membersV2() {
    List<Member> members = memberService.findMembers();
    // 엔티티 -> Dto
    List<MemberDto> memberDtos = members.stream()
                                        .map(i -> new MemberDto(i.getName()))
                                        .collect(Collectors.toList());
    return new Result(memberDtos);
}

@Data
@AllArgsConstructor
static class Result<T> {
    private T data;
}

@Data
@AllArgsConstructor
static class MemberDto {
    private String name;
    private Integer age;
}
```

## 엔티티 수정은 병합(merge)보다 변경 감지(Dirty Checking)

엔티티의 값이 변경되었을 때 JPA는 어떻게 처리할까?

병합(merge)과 변경 감지(Dirty Checking) 두 방법이 존재하고 이 둘을 비교해보겠다. 

먼저 병합(merge)을 사용하는 코드이다.
```java
// ItemRepository.java
public void save(Item item) {
    if (item.getId() == null) {
        em.persist(item);
    } else {
        Item mergeItem = em.merge(item);
    }
}
```
Item 엔티티에 식별자가 없다면 DB에 저장되지 않은 새로운 값으므로 영속화(persist)한다.
하지만 식별자가 존재한다면 병합(merge)한다. 병합은 다음과 같이 동작한다.

먼저 파라미터로 들어오는 식별자가 있는 엔티티는 준영속 상태이다. 
준영속이란 영속성 컨텍스트(JPA)가 더 이상 관리하지 않는 엔티티이다.
반대로 영속이란 영속선 컨텍스트(JPA)가 관리하는 엔티티이다.

merge가 호출되면 파라미터로 넘어온 준영속 엔티티(item)를 영속 엔티티로 만든다. 
그리고 준영속 엔티티의 값을 영속 엔티티에 모두 밀어넣는다.
마지막으로 영속 상태인 엔티티를 반환한다. 위에서 mergeItem이 영속 엔티티이다.

하지만 병합은 단점이 존재한다. **엔티티의 모든 값을 변경한다는 점**이다. 
예를 들어 Item 엔티티에는 식별자, 이름, 가격, 재고 속성을 가지고 있다. 
하지만 우리는 가격만 수정하고 싶어 나머지는 null값이고 변경된 가격 값만 객체에 넣어 전송했다.
이때 모든 속성값이 변경되어 null로 수정된다.

반대로 변경 감지(Dirty Checking)을 사용한 코드이다.
```java
// ItemService.java
@Transactional
public void updateItem(Long itemId, UpdateItemDto param) {
    Item findItem = itemRepository.findOne(itemId);
    
    findItem.setName(param.getName());
    findItem.setPrice(param.getPrice());
    findItem.setStockQuantity(param.getStockQuantity());
}
```
먼저 파라미터로 식별자(itemId)와 변경할 데이터(param)가 DTO로 넘어왔다.
컨트롤러 계층에서 엔티티를 생성해 넘기는 것보다 **식별자만 서비스 계층으로 넘기는 것**이 좋다. 

컨트롤러에서 생성한 엔티티는 준영속 상태이다. 
하지만 **Transaction이 있는 서비스 계층**에서 식별자를 이용해 영속 상태의 엔티티(findItem)를 조회할 수 있다.
영속 상태의 엔티티의 값을 변경하고 트랜잭션이 commit되면 JPA는 변경 감지(Dirty Checking)를 실행한다.
그리고 JPA는 DB로 update 쿼리를 날린다.

엔티티를 수정할 때 모든 값을 수정하는 경우보단 일부만 수정하는 경우가 훨씬 많다.
따라서 _엔티티를 수정할 땐 병합보다 변경 감지를 사용_ 하는 것이 좋다

## N + 1 문제
N + 1 문제는 지연 로딩의 엔티티를 조회할 때 발생한다. 

다음과 같이 엔티티가 양방향 연관관계에 있다.

![ER Diagram](https://user-images.githubusercontent.com/83766322/222296083-d4191c66-af67-4c5a-be20-ec1a9029890d.png)

Order 엔티티는 다음과 같고 XToOne은 default가 EAGER(즉시 로딩)이므로 LAZY(지연 로딩)으로 설정한다.
```java
import static javax.persistence.FetchType.*;

@Entity
@Table(name = "orders")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Order {
	
    @Id @GeneratedValue
    @Column(name = "order_id")
    private Long id;
	
    @ManyToOne(fatch = LAZY)
    @JoinColumn(name = "member_id")
    private Member member;
	
    @OneToOne(fetch = LAZY)
    @JoinColumn(name = "delivery_id")
    private Delivery delivery;
	
}
```

이제 Order를 모두 조회하는 API를 개발하자. 그리고 Order과 연관된 Member과 Delivery도 함께 조회할 것이다.

```java
@GetMapping("/api/simple-orders")
public List<SimpleOrderDto> orders() {
    List<Order> orders = orderService.searchOrder(new OrderSearch()); // 조회 조건이 비었으므로 order 전체 조회
    List<SimpleOrderDto> orderDto = orders.stream().map(SimpleOrderDto::new).collect(toList());
    return orderDto;
}

@Data
static class SimpleOrderDto {
    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;
    
    public SimpleOrderDto(Order o) {
        orderId = o.getId();
        name = o.getMember().getName(); // Lazy 강제 초기화
        orderDate = o.getOrderDate(); 
        orderStatus = o.getStatus();
        address = o.getDelivery().getAddress(); // Lazy 강제 초기화
    }
}
```
코드를 살펴보면 엔티티가 아닌 Dto로 반환했다. 
그리고 지금 Dto는 엔티티에 의존하고 있다. 
토픽과 다른 얘기지만, Dto는 엔티티를 의존해도 되지만 엔티티는 Dto를 의존해선 안된다.
Dto는 하나의 API에서 사용되지만 엔티티는 정말 많은 곳에서 사용된다. 
따라서 다른 글에서 말했듯이 엔티티는 최대한 순수해야 하므로 Dto를 의존해선 안된다.

그리고 LAZY를 강제 초기화하여 Order과 연관된 Member와 Delivery를 조회한다.
이때 N + 1문제가 발생한다. Order를 조회했을 때 결과의 수가 N이라고 하자.
Member와 Delivery를 조회하는 쿼리가 N번만큼 실행된다. 

결국 최악의 경우 Order를 조회하는 쿼리 1번, Member를 조회하는 쿼리 N번, Delivery를 조회하는 쿼리 N번이 실행된다.

왜 최악의 경우일까?
만약 서로 다른 주문 A, B는 동일한 회원 C의 주문이다. 
A 주문에 대하여 회원 C을 조회했다면 이제 회원 C는 영속성 컨텍스트가 관리하는 엔티티이다.  

다음으로 B 주문에 대하여 회원 C를 조회하는데 이미 영속성 컨텍스트에 있으므로 DB가 아닌 영속성 컨텍스트에서 가져온다.
이러한 경우에 Member를 조회하는 쿼리가 N - 1번 실행된다.

그렇다면 이러한 N + 1 문제를 어떻게 해결할까? fetch join을 사용하여 조회하면 해결된다.
따라서 [fetch join을 토픽으로 새로운 글을 작성](#jpql)하려고 한다.

# JPQL

JPQL에 대한 설명이 길어질 것 같은데 폴더를 따로 옮겨야 할지 모르겠다.

## fetch join을 통해 N + 1 문제 해결

fetch join을 통해서 앞서 말한 N + 1 문제를 해결할 수 있다.

아래는 [예시](#N-1-문제)에서 발생한 문제를 fetch join으로 해결하는 코드이다.

```java
public List<Order> findAllWithMemberDelivery() {
    return em.createQuery(
            "select o from Order o"
            + " join fetch o.member m"
            + " join fetch o.delivery d", Order.class)
            .getResultList();
}
```
fetch join을 사용하여 **단 한 번의 쿼리로** Order과 연관된 Member와 Delivery 엔티티를 가져올 수 있다.
지연 로딩인 엔티티 중 원하는 엔티티만 함께 조회하면 된다. 

이때 각 엔티티의 속성을 모두 조회한다. 
예를 들어 Member의 속성이 이름, 이메일, 전화번호, 주소라면 우리는 Member의 이름만 원했음에도 Member의 속성들을 모두 조회한다.
따라서 select 절이 매우 길어지고 퍼올리는 데이터 양이 크므로 네트워크 용량이 커진다.

이에 대한 대안으로 아래와 같이 처음부터 Dto로 조회하는 방법이 있다.
```java
public List<SimpleOrderQueryDto> findOrderDtos() {
    return em.createQuery(
            "select new com.study.jpaproject.dto.SimpleOrderQueryDto(o.id, m.name, d.address) "
                    + "from Order o"
                    + " join fetch o.member m"
                    + " join fetch  o.delivery d", SimpleOrderQueryDto.class
    ).getResultList();
}

@Data
public class SimpleOrderQueryDto {
	
	private Long orderId;
	private String name;
	private Address address;

	public SimpleOrderQueryDto(Long orderId, String name, Address address) {
		this.orderId = orderId;
		this.name = name;
		this.address = address;
	}
	
}
```
참고로 다른 예제처럼 생성자에 엔티티만 넘긴다고 Order의 o를 넘기면 안된다.
엔티티가 아닌 식별자(Long)만 넘어간다.

처음부터 원하는 속성만 조회했으므로 쿼리의 select 절이 짧아졌다. 
따라서 네트워크 용량이 최적화 되었다.
하지만 Repository에서 Dto를 조회한다는 점은 여러 문제가 발생한다.

- Repository가 Presentation에 의존한다.
  Repository는 Data Access 계층이다. 따라서 Presentation 계층, API 스펙에 의존한다는 것은 계층이 깨졌다는 것이다.
- Repository 재사용성이 떨어진다. 
  Controller가 Repository로부터 엔티티를 받아서 Dto로 변환하는 것은 재사용성이 높다.
  하지만 처음부터 Dto로 받는 메서드가 있다면 다른 API에서 사용하지 못할 것이다.
- 네트워크 용량의 최적화가 미비하다. 

결론은 트래픽이 매우 많이 들어오는 API라면 고려할 필요가 있다.  
그리고 Dto를 반환하는 메서드를 작성했다면 이러한 메서드를 위한 Repository를 따로 생성하는 것이 좋다. 

## fetch join의 distinct

XToMany 관계의 엔티티를 fetch join을 사용하여 조회하면 N + 1 문제를 해결할 수 있지만 또 다른 문제가 발생한다.

다음과 같이 엔티티가 양방향 연관관계에 있다. 이번 글에서 OrderItem과 Item에 대한 코드는 생략한다.

![ER Diagram](https://user-images.githubusercontent.com/83766322/222725344-177b7393-6d0c-4eca-9f52-8068ebdc6b96.png)

Order 엔티티는 다음과 같다. XToMany 관계에서는 지연 로딩(LAZY)이 default이므로 따로 설정은 하지 않는다.

```java
@Entity
@Table(name = "orders")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Order {
	
    @Id @GeneratedValue
    @Column(name = "order_id")
    private Long id;
	
    @OneToMany(mappedBy = "order")
	private List<OrderItem> orderItems = new ArrayList<>();

}
```
그리고 Order를 모두 조회하는 API를 개발한다. 이때 Order과 연관된 OrderItem 리스트도 함께 조회한다.

```java
@GetMapping("/api/orders")
public List<OrderDto> orders() {
    List<Order> orders = orderRepository.findWithItems();
    List<OrderDto> orderDtos = orders.stream()
                                    .map(OrderDto::new)
                                    .collect(toList());
    
    return orderDtos;
}

@Data
static class OrderDto {
    private List<OrderItemDto> orderItems;
    
    public OrderDto(Order o) {
        orderItemDto = o.getOrderItems()
                        .stream()
                        .map(OrderItemDto::new)
                        .collect(toList());
    }
}

@Data
static class OrderItemDto {
    private int totalPrice;

    public OrderItemDto(OrderItem oi) {
      totalPrice = oi.getTotalPrice(); // Lazy 강제 초기화
    }
}
```
코드를 살펴보면 Order 안의 OrderItem 모두를 Dto로 변환하여 반환했다.
응답으로 내보낼 때 엔티티와의 의존을 모두 끊어준 뒤 내보내야 한다.

마지막으로 OrderRepository에 findWithItems 메서드이다.
```java
public List<Order> findWithItems() {
    return em.createQuery(
                    "select o from Order o" +
                    " join fetch o.orderItems oi", Order.class)
            .getResultList();
}
```
fetch join을 사용해서 양방향 연관관계의 OrderItem을 조회할 때 발생하는 N + 1문제를 해결했지만 또 다른 문제가 발생했다. 

Order A가 OrderItem B와 OrderItem C를 참조하고 있을 때 Order A가 두 번 조회되는 현상이 발생했다.
즉, **Order이 참조하고 있는 OrderItem 개수 N만큼 조회된다**는 문제가 발생했다.

먼저 fetch join을 사용하면 데이터베이스에 inner join 쿼리가 전달된다.
따라서 데이터베이스는 다음과 같은 결과를 보여준다.

| order_id | order_item_id |
|----------|---------------|
| A        | B             |
| A        | C             |

JPA는 데이터베이스의 row 수만큼 컬렉션을 만든다. 
결국은 동일한 Order 엔티티가 두 번 조회된다. 표의 간략화를 위해 식별자만 나타냈지만 실제로는 **Order에 대한 모든 정보를 가져오게 되므로** 컬럼 수가 매우 늘어날 것이고 중복되는 데이터가 많다. 

동일한 엔티티라는 뜻은 참조하는 메모리가 같다는 뜻이다. 

```java
public void orders() {
    orderRepository.findWithItems().stream().forEach(System.out::println)
}
/**
 * com.study.jpaproject.domain.Order@52091390
 * com.study.jpaproject.domain.Order@52091390 
 */
```

fetch join의 **distinct**를 사용하면 위와 같은 문제를 해결할 수 있다.
OrderRepository의 findWithItems 메서드를 고쳐보자.
```java
public List<Order> findWithItems() {
    return em.createQuery(
                    "select distinct o from Order o" +
                    " join fetch o.orderItems oi", Order.class)
            .getResultList();
}
```
데이터베이스에 distinct 쿼리가 전달되었지만 여전히 row 수는 동일하다.
하지만 컬렉션에 Order가 중복되는 문제는 사라졌다.

데이터베이스는 모든 컬럼의 값이 같아야 동일한 데이터라고 판단한다.
반면에 JPA는 식별자(order_id)가 같으면 동일한 엔티티라고 판단한다. 
따라서 distinct를 사용하면 JPA가 동일한 엔티티에 대하여 중복을 막아준다.

## fetch join vs 일반 join

fetch join은 결국 inner join과 다름 없는것 같다. 그런데 왜 연관된 엔티티를 조회할 때 일반 join이 아닌 fetch join을 사용하는 것일까?

먼저 일반 join은 실행 시 **연관된 엔티티를 함께 조회하지 않는다.** 아래와 같이 Member를 join했다면 select 절에 지정한 엔티티, 즉 Order만 조회한다.
```java
public List<Order> ordinaryJoin() {
    return em.createQuery(
            "select o from Order o" +
            " join o.member m", Order.class
    ).getResultList();
}
```
반대로 fetch join은 **연관된 엔티티도 함께 조회**, 즉 즉시 로딩을 수행한다.

## fetch join의 한계

fetch join을 공부하면서 가장 어려웠던 부분이다. 하지만 이해한 것을 바탕으로 최대한 적어보려고 한다.

### 1. fetch join 대상에는 별칭을 줄 수 없다.
fetch join 대상에는 별칭을 줄 수 없지만 몇 가지 예외가 있다.
- **fetch join을 여러 단계로 수행할 경우**
  ![ER Diagram](https://user-images.githubusercontent.com/83766322/222725344-177b7393-6d0c-4eca-9f52-8068ebdc6b96.png)
  
  다음과 같이 엔티티가 관계를 맺고 있을 때 Order에서 OrderItem을 그리고 OrderItem에서 Item 조회하는 코드는 아래와 같다. 
  
  ```java
  List<Order> orders = em.createQuery(
                "select o from Order o" +
                " join fetch o.orderItems oi" +
                " join fetch oi.item i", Order.class)
                .getResultList();
  ```
  
  이렇게 fetch join 대상의 별칭을 이용해 여러 단계의 fetch join을 수행할 수 있다.

- **데이터 무결성을 해치지 않는 경우**
  
  먼저 데이터 무결성을 해치는 경우를 살펴보자.

  Order입장에서 OrderItem과 @OneToMany 관계이다. 즉, Order은 OrderItem에 대한 리스트를 가지고 있다.
  
  Order A가 OrderItem B와 C랑 관계를 맺고 있고 Order를 통해 OrderItem을 조회하려고 한다. 따라서 다음 코드를 실행했을 때 데이터베이스에서 결과가 어떻게 나타나는지 확인하자.

  ![Order & OrderItem](https://user-images.githubusercontent.com/83766322/222947913-06eead93-a8fc-4069-9cd1-8a3bce0c965c.png)
  
  ```java
  List<Order> orders1 = em.createQuery(
                "select o from Order o" +
                " join fetch o.orderItems oi", Order.class)
                .getResultList();
  ```
  
  [fetch join의 distinct](#fetch-join의-distinct)에서 말했다싶이 다음과 같이 나타난다. 
  
  | order_id | order_item_id | order_item_price |
  |----------|---------------|------------------|
  | A        | B             | 10000            |
  | A        | C             | 20000            |
  
  그런데 fetch join 대상인 OrderItem에 대하여 필터링을 한다고 하자. OrderItem의 price가 10000원 초과라는 조건문을 where절에 넣어준다. 그리고 다음 코드를 실행했을 때 데이터베이스의 결과는 아래와 같다. 
  
  ```java
  List<Order> orders2 = em.createQuery(
            "select o from Order o" +
            " join fetch o.orderItems oi" +
            " where oi.orderPrice > 10000", Order.class)
            .getResultList();
  ```

  | order_id | order_item_id | order_item_price |
  |----------|---------------|------------------|
  | A        | C             | 20000            |
  
  식별자가 A인 Order 엔티티는 식별자가 B와 C인 OrderItem 엔티티를 가지고 있는게 맞다. 그런데 orders2 리스트의 Order A는 OrderItem C만 가지게 된다. 따라서 **데이터 무결성이 깨졌다.**
 
  반대로 데이터 무결성이 깨지지 않는 경우이다.

  Order 입장에서 Member과 @ManyToOne 관계이다. Order A와 Member C 그리고 Order B와 Member D와 관계를 맺고 있다고 하자. Order를 통해 Member를 조회하려고 한다. 아래는 코드와 코드를 실행했을 때 데이터베이스의 결과이다.

  ```java
  List<Order> orders3 = em.createQuery(
				"select o from Order o" +
				" join fetch o.member m", Order.class)
                .getResultList();
  ```
  | order_id | member_id | member_name |
  |----------|-----------|-------------|
  | A        | C         | memberC     |
  | B        | D         | memberD     |
 
  fetch join의 대상인 Member에 대하여 이름이 memberC인 사람만 조회하면 다음과 같다.

  ```java
  String memberName = "memberC";
  List<Order> orders4 = em.createQuery(
            "select o from Order o" +
            " join fetch o.member m" +
            " where m.name =: memberName", Order.class)
            .setParameter("memberName", memberName)
            .getResultList();
  ```
  | order_id | member_id | member_name |
  |----------|-----------|-------------|
  | A        | C         | memberC     |

  orders4 리스트에서 식별자가 A인 Order 엔티티는 여전히 Member C와 관계를 맺고 있고 **데이터 무결성이 깨지지 않았다.** 이와 같이 데이터가 일관성을 유지하는 경우 fetch join 대상에 별칭을 줄 수 있다.

### 2. 둘 이상의 컬렉션은 fetch join 할 수 없다.
다음과 같은 관계일 때 Member를 통해 Order를 그리고 Order를 통해 OrderItem을 조회한다고 하자.

![ER Diagram](https://user-images.githubusercontent.com/83766322/222956495-1c1efd43-e8b8-43c0-acfd-d1b4489c6b1f.png)

컬렉션 조회 시 Many에 맞춰 데이터가 증가된다. 예를 들어 Member 하나가 Order N개와 연관되어 있고
각 Order은 OrderItem M개와 연관되어 있다고 가정하자. 겨우 Member 하나만 조회하는 데에 데이터베이스의 row는 N * M만큼이 늘어날 것이고 엄청난 데이터 중복이 발생한다.

### 3. 컬렉션 fetch join 시 페이징이 불가하다.

XToOne과 같이 단일 값 연관 필드와 fetch join을 해도 페이징할 수 있다. 
하지만 XToMany와 같이 컬렉션 연관 필드와 fetch join을 하면 페이징이 불가능하다.

Order과 OrderItem 관계는 생략한다. Order를 통해 OrderItem을 조회하면서 Order를 기준으로 페이징을 하고 싶다. 따라서 다음과 같은 코드를 작성했다.

```java
List<Order> orders = em.createQuery(
        "select distinct o from Order o" +
        " join fetch o.orderItems oi", Order.class)
        .setFirstResult(0) // 페이징 메서드
        .setMaxResults(1)
        .getResultList();
```

이 코드는 원하는 대로 리스트에 첫번째 Order만 담아있다. 하지만 하이버네이트는 다음과 같은 경고 로그를 남긴다.

> HHH000104: firstResult/maxResults specified with collection fetch; applying in memory!

페이징 쿼리를 전달하지 않고 **데이터베이스에서 모든 데이터를 가져온 다음 메모리에서 페이징을 한다**는 경고이다. Order의 데이터가 적을 경우 괜찮지만 데이터가 많아진다면 문제가 발생할 수 있음을 알 수 있다.

컬렉션 조회 시 Order은 OrderItem 개수에 맞춰 중복 조회가 발생한다. 따라서 Order를 기준으로 페이징하기 어려우니 JPA는 위와 같은 전략을 통해 페이징을 한다.

그렇다면 컬렉션 fetch join 시 어떻게 페이징을 할 수 있을까? [하이버네이트의 batch-fetch size](#hibernate의-batch-fetching과-legacy-전략)를 지정하면 된다.

## Hibernate의 Batch Fetching

fetch join의 한계에서 XToMany 관계는 fetch join 시 페이징이 어렵다고 했다. 

따라서 hibernate.default_batch_fetch_size를 설정해 이러한 한계를 돌파할 수 있다.

```yml
# application.yml
jpa :
    hibernate :
        default_batch_fetch_size : 100
```

이 전략을 사용했을 때 어떻게 동작하는지 예제로 보여주겠다.

다음과 같이 Order이 Mebmer와 OrderItem과 관계를 맺고 있다.

![ER Diagram](https://user-images.githubusercontent.com/83766322/222956495-1c1efd43-e8b8-43c0-acfd-d1b4489c6b1f.png)

그리고 데이터는 다음과 같이 들어있다. 데이터가 한 개 뿐이지만 전략을 설명하는 데에 큰 무리가 없을 것이다.

![image](https://user-images.githubusercontent.com/83766322/223011961-8a78ced2-0746-4c83-941a-845381f3ae2f.png)

Order를 조회하면서 Member와 OrderItem을 함께 조회하려고 한다. 또한 Order을 기준으로 페이징하려고 한다.

먼저 XToOne 관계는 fetch join을 한 뒤, 페이징을 한다. Order이 하나뿐 이므로 만약 offset이 0이고 limit가 100일 때, 하나의 Order만 검색된다. 

```java
public List<Order> findWithMember(int offset, int limit) {
		return em.createQuery(
				"select o from Order o" +
                " join fetch o.member m", Order.class)
                .setFirstResult(offset)
                .setMaxResults(limit)
                .getResultList();
}
```
| order_id | member_id |
|----------|-----------|
| A        | B         |

그리고 Order를 Dto로 변환할 때 지연 로딩(Lazy)인 컬렉션(OrderItem)을 강제 초기화 한다. [이 글](#fetch-join의-distinct)을 참고하자.

컬렉션이 강제 초기화될 때 아래와 같은 쿼리가 실행된다. findWithMember로 조회된 Order의 아이디를 IN 절에 넣는다. 그리고 OrderItem 테이블에서 Order 리스트와 연관된 데이터만 가져온다. 

> select * from order_item where order_id in (A);

| order_item_id | order_id |
|---------------|----------|
| C             | A        |
| D             | A        |

처음에 default_batch_fetch_size를 100으로 설정했다. 이 숫자는 IN절의 파라미터 수를 100으로 제한한다는 뜻이다.

이 전략의 장점은 무엇일까?

### 1. 데이터 중복이 없다.

먼저 컬렉션을 fetch join 했을 때 데이터 중복이 발생한다고 설명했다. Order에 대한 모든 정보가 OrderItem의 개수만큼 늘어난다면 데이터베이스에서 전송되는 데이터 양이 엄청날 것이다.

반대로 IN 절을 사용해 Order과 연관되어 있는 모든 OrderItem의 정보만을 가져올 뿐, Order의 정보는 전송되지 않는다. 따라서 데이터베이스에서 전송되는 양이 훨씬 줄어든다. 

### 2. N + 1 문제를 해결한다. 
만약 batch_fetch_size를 지정하지 않았다면 OrderItem을 가져올 때 Order 리스트의 사이즈(N)만큼 쿼리를 전송할 것이다.

하지만 100으로 지정함으로써 1 + 1번의 쿼리를 전송했다. 첫 번째는 Order과 Member를 inner join하고 페이징하는 쿼리이다. 두 번째는 IN절을 사용해 Order과 연관된 OrderItem을 조회하는 쿼리이다.

만약 Order 리스트의 사이즈가 200이라면 IN절의 파라미터를 100으로 제한했으므로 3번의 쿼리가 전송된다.

Member와 OrderItem 둘 다 fetch join을 한다면 단 하나의 쿼리가 전송될 것이다. 하지만 데이터 중복이 발생하므로 DB에서 전송되는 데이터 양이 크다.

반대로 Member만 fetch join을 하고 OrderItem은 IN절로 가져오다면 전송되는 쿼리 수는 늘어난다. 하지만 데이터 중복이 발생하지 않으므로 DB에서 전송되는 데이터 양이 적다.

## 적당한 Batch-Size와 Legacy 전략 

[이 글](#hibernate의-batch-fetching)과 연결되는 글이다.

이번엔 default_batch_fetch_size을 몇으로 지정하면 좋을지에 대해서 이야기 해보자. 

데이터베이스에 따라 IN절 파라미터를 1000으로 제한하기도 한다. 1000이면 데이터베이스에서 한 번에 당겨오는 데이터가 엄청나므로 순간 부하가 발생할 수 있다. 따라서 애플리케이션이 순간 부하를 얼만큼 견딜 수 있는지에 따라 적당히 조절해야 한다.

다음으로 Batch Fetching 전략에 대해 설명한다.

하이버네이트(Hibernate)는 세 개의 서로 다른 Batch Fetching 전략을 제공하는데 Legacy 전략이 default이다.

먼저 default_batch_fetch_size를 15로 두었다. 그리고 offset을 0, limit를 12으로 두어 12개의 Order를 조회했다.

IN절의 파라미터가 최대 15라고 설정했으므로 우리는 쿼리가 두 번 수행되기를 기대하고 있다. 
하지만 쿼리가 세 번 수행되었다.

> select * from order_item where order_id in (?, ?, ?, ?, ?, ?, ?, ?, ?, ?) 
>
> select * from order_item where order_id in (?, ?, ?)

IN절의 파라미터가 10번, 3번으로 나눠졌다. 왜 이렇게 동작할까?

보통 관계형 데이터베이스(RDB)는 미리 쿼리를 파싱(parsing 구문 분석)한 다음 캐싱(caching 저장)한다. 이를 PreparedStatement라고 한다.

default_batch_fetch_size를 100으로 정한다면 다음과 같이 최대 100의 PreparedStatement를 캐싱해야 한다.

select * from table where in (?)<br>
select * from table where in (?, ?)<br>
...<br>
select * from table where in (?, * 100)

이러한 방식은 데이터베이스 입장에서 너무 많은 쿼리를 캐싱하므로 성능이 떨어진다. 따라서 하이버네이트는 Legacy 전략으로 최적화를 시도한다. 

10을 초과하는 크기를 지정했을 때 확실히 이해할 수 있다. 
크기가 25라면 (25, 12, 10, 9, 8 ... 1) 이렇게 batch size를 나눈다.

batch size를 25로 두고 Order 리스트의 크기가 20이라면 12, 8 크기로 나눠진 뒤 2개의 쿼리가 실행될 것이다.

그 외의 전략에 대한 설명은 생략한다.

## OSIV

JPA를 사용하면 애플리케이션 시작 시점에 다음과 같은 경고 로그가 남겨진다.

> spring.jpa.open-in-view is enabled by default. Therefore, database queries may be performed during view rendering. Explicitly configure spring.jpa.open-in-view to disable this warning

Open Session In View(이하 OSIV) 전략을 실행한다는 뜻이다. OSIV는 enable이 default 이다. 

OSIV 전략은 Service 계층에서 Transaction을 시작할 때 커넥션 풀(Connection Poll) 데이터베이스 커넥션을 가져온 뒤 
API 응답이 끝날 때까지 영속성 컨텍스트와 데이터베이스 커넥션을 유지한다.
즉, Controller에서 외부 API를 호출하면 응답 대기 시간만큼 데이터베이스 커넥션 자원을 반환하지 않는다.

따라서 Controller나 View 계층에서 지연 로딩이 가능하다는 장점이 있다. 
하지만 트래픽이 많은 API에서 OSIV 전략을 사용하는 것은 큰 문제가 발생한다.

이 전략은 너무 오랜시간 데이터베이스 커넥션 자원을 사용한다. 
따라서 트래픽이 많으면 커넥션 풀에서 데이터베이스 커넥션 자원이 부족해지고 이것은 결국 장애로 이어진다.

```yml
jpa:
    open-in-view: false
```

반대로 OSIV 전략을 사용하지 않는다면 트랜잭션을 종료하면 영속성 컨텍스트를 닫고 데이터베이스 커넥션을 반환한다.
따라서 데이터베이스 커넥션 자원을 낭비하지 않게 된다. 하지만 Controller나 View 계층에서 지연 로딩이 불가능하다.
따라서 트랜잭션 내에서 모든 지연 로딩을 처리해야 한다.

앞서 지연 로딩을 처리하는 코드는 복잡했다. 
따라서 OSIV를 끈 상태로 복잡성을 관리하기 위해 커맨드와 쿼리를 분리한다.

### back / [up](#springjpa)

## 후기
아직 웹 계층 아키텍처에 대해 정확히 모르겠다.
영속성 컨텍스트에 대해 좀 더 자세히 알아보자.

---

## 참조
- https://jojoldu.tistory.com/603
- https://docs.jboss.org/hibernate/orm/4.2/manual/en-US/html/ch20.html#performance-fetching-batch

