## 스프링 데이터 JPA를 이용한 조회

### 스펙
검색 조건을 다양하게 조합해야 하는 경우 Spec을 사용하자
애그리거트가 특정 조건을 충족하는지를 검사할 때 사용하는 인터페이스다.

```java
public interface Specification<T> {
    public boolean isSatisfiedBy(T agg);
}
```

스펙을 리포지터리에 사용하면 agg는 애그리거트 루트가 되고,
DAO에 사용하면 agg는 검색 결과로 리턴할 데이터 객체가 된다.

repository가 스펙을 이용해서 검색 대상을 걸러주기에 특정 조건을 충족하는 
애그리거트를 찾고 싶으면 원하는 스펙을 생성해서 repository에 전달하면 된다.

```java
Specification<Order> orderSpec = new OrdererSpec("dcoding");

// repository에 전달
List<Order> orders = orderRepository.findAll(orderSpec);
```
스프링 데이터 JPA에서 스펙 사용하는 경우
```java
public class OrderIdSpec implements Specification<OrderSummary> {
	
	@Override
    public Predicate toPredicate(Root<OrderSummary> root,CriteriaQuery<?> query, CriteriaBuilder cb) {
		return cb.equal(root.get(OrderSummary_.ordererId), ordererId);
    }
}
```

`toPredicate()` 메서드를 오버라이드 하여 구현하고 이때 JPA 정적 메타모델 (OrderSummary_)를 사용하는 것이
문자열을 사용하는 것보다 안정적으로 사용가능
> JPA 정적 메타 모델의 경우  hibernate-jpamodelgen 의존성을 추가하면 자동으로 만들어 준다.

### 위와 같이 생성한 스펙을 조합해서 사용할 수 있다.
- `and()` 와 `or()` 두 메서드를 활용하여 조합할 수 있다.

    ```java
    Specification<OrderSummary> spec1 = OrderSummarySpecs.orderId("user1");
    Specification<OrderSummary> spec2 = OrderSummarySpecs.orderDateBetween(
        LocalDateTime.of(2022, 1, 1, 0, 0, 0),
        LocalDateTime.of(2022, 1, 2, 0, 0, 0));
    
    // 스펙 조합하기
    Specification<OrderSummary> spec3 = spec1.and(spec2);
    ```

- `not()`
  정적 메서드 조건을 반대로 적용하고 싶은 경우 사용
    ```java
    Specification<OrderSummary> spec = specification.not(OrderSummarySpecs.ordererId("user1"));
    ```
  
- `where()` 메서드를 활용하면 null값 제어를 편리하게 할 수 있다.


### 정렬 지정하기
- 메서드 이름에 OrderBy를 사용해서 정렬 기준 지정
- Sort를 인자로 전달

위 두가지 방식으로 Spring Data JPA에서 정렬을 지정할 수 있다.

> 메서드 이름을 사용하는 방식은 정렬조건이 한개인 경우 활용하면 개발 생산성을 높이는데 
> 좋은 것 같다. 하지만 여러 조건을 하면 메서드명이 길어지고 가독성이 떨어지는 것 같다.


### 페이징 처리하기
스프링 데이터 JPA는 페이징 처리를 위해 `Pageable` 타입을 이용한다.

`Page` 타입을 사용하면 데티어 목록뿐만 아니라 조건에 해당하는 전체 개수도 구할 수 있다.
`Pageable`을 사용하는 메서드의 리턴 타입이 `Page 일 경우 목록 조회 쿼리와 함께 COUNT 쿼리도 실행해서 
조건에 해당하는 데이터 개수를 구한다.

**`Pageable` 타입을 사용하더라도 리턴 타입이 `List`면 COUNT 쿼리를 실행하지 않는다.**
**하지만 스펙을 사용하는 findAll() 메서드에 `Pageable`타입을 사용하면 리턴 타입과 무관하게 COUNT 쿼리를 실행한다.**

- 조회 결과 목록
- 조건에 해당하는 전체 개수
- 전체 페이지 번호
- 현재 페이지 번호
- 조회 결과 개수
- 페이지 크기

와 같은 정보를 제공받을 수 있고 이또한 Spec과 함께 사용할 수 있다.

### 스펙 조합이 필요한 경우 스펙 빌더 클래스 사용하기
동적 쿼리를 스펙을 이용해서 구현하고 싶은 경우 빌더를 사용하면 코드량을 줄이고 가독성을 높일 수 있다.

```java
Specification<MemberData> spec = SpecBuilder.builder(MemberData.class)
    .ifTrue(searchRequest.isOnlyNotBlocked(),
		() -> MemberDataSpecs.nonBlocked())
    .ifHasText(searchRequest.getName(),
        name -> MemberDataSpecs.nameLike(searchRequest.getName()))
    .toSpec();

List<MemberData> result = memberDataDao.findAll(spec, PageRequest.of(0, 5)); 
```

- and()
- ifHasText()
- ifTrue()

등의 메서드는 SpringData JPA에서 기본으로 SpecBuilder를 제공하는 것이 아니기에 
필요에 따라 Builder 클래스를 만들어서 사용하고 내부 메서드를 정의해서 구현해서 사용하자

### 조회 전용 모델을 만드는 이유
스프링부트를 기반으로 개발하면 디폴트로 Jackson을 사용하게되고 직렬화 책임을 수행한다.

하지만 VO를 사용하게 되면 표현계층에서 의도와 다르게 JSON 으로 변환된다.

```java
public class Product {
    private String name;
    private Money price;  // 밸류 타입
}
```

위 Product를 JSON으로 응답하는 아래와 같이 응답을 제공하기를 바란다.
```json
{
  "name": "노트북",
  "price": 1500000
}
```

하지만 현실은 아래와 같은 형태가 된다.

```json
{
  "name": "노트북",
  "price": {
    "value": 1500000
  }
}
```

`Money`는 객체이기 때문에 Jackson이 객체의 필드를 그대로 풀어서 출력한다.

따라서 표현계층에서 전용으로 사용하는 조회 전용 모델(DTO)를 따로 만들어서 관리하자
또는 Jackson에 `Money` 전용 시리얼라이저를 등록하는 방법도 있다.

### @Subselect??

`@Subselect`는 조회 쿼리를 값으로 갖는다.
이 어노테이션을 사용하면 쿼리 실행 결과를 매핑할 테이블처럼 사용할 수 있다.

주로 여러 테이블을 조인해서 조회한 결과를 한 테이블 처럼 보여주기 위한 용도로 사용한다.
> 이 어노테이션을 알기전까지 복잡한 통계성 데이터를 제공해야하는 경우 연관된 모든 애그리거트의 루트로 
> 데이터를 조회하고 여기서 현재 응답으로 제공할 필드들만 재조합해서 사용해야하는 건가?
> 라는 의문이 있었는데 이를 해결할 수 있는 좋은 방법인 것 같다.

`@Subselect`로 조회한 `@Entity`는 수정할 수 없다. 
실수로 `@Subselect`를 이용해 조회한 엔티티의 필드를 수정하면 하이버에니트는 변경 내역을 반영하기 위해 update 쿼리를 실행할 것이다.
하지만 실제로 매핑되는 테이블이 존재하지 않아 오류가 발생할 것이다.

이런 문제를 방지하기 위해 `@Immutable`을 사용한다.
하이버네이트는 `@Immutable`가 붙어 잇는 엔티티의 매핑 필드/프로퍼티가 변경되도 DB에 반영하지 않고 무시한다.

이외에도 아래와 같은 업데이트와 조회를 동시에 하는 경우 주의사항이 있다.

```java
Order order = orderRepository.findById(orderNumber);
order.change(newInfo); // 상태 변경

// 위에서 상태 변경을 했지만 아직 반영되지 않은 상태에서 조회 쿼리 실행
List<OrderSummary> summaries = orderSummaryRepository.findByOrdererId(userId);
```

위 코드는 Order의 상태를 변경하고 OrderSummary를 조회한다.
하이버네이트는 특별한 설정을 하지 않은 경우 상태 변경을 트랜잭션을 커밋하는 시점에 반영하는데

OrderSummary 조회 시 Order의 변경 내역이 반영되지 않는다.

이런 문제를 `@Synchronize` 를 사용하여 방지할 수 있다.
`@Synchronize`는 해당 엔티티와 관련된 테이블 목록을 명시하여 사용한다.

명신된 엔티티에서서 데이터를 불러오기전에 지정한 테이블과 관련된 변경이 발생하면 
flush를 먼저 수행하여 변경 내용을 반영하고 최신 정보를 조회할 수 있도록 한다.
