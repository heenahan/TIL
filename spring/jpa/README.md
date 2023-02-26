# Spring/JPA
1. [요청과 응답으로 엔티티 대신 DTO 사용](#요청과-응답으로-엔티티-대신-DTO-사용)
2. [엔티티 수정은 병합(merge)보다 변경 감지(Dirty Checking))](#엔티티-수정은-병합(merge)보다-변경-감지(Dirty Checking))

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

## 후기
아직 웹 계층 아키텍처에 대해 정확히 모르겠다.
영속성 컨텍스트에 대해 좀 더 자세히 알아보자.

---

## 참조
- https://jojoldu.tistory.com/603


