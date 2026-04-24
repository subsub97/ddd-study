## JPA를 이용한 리포지터리 구현

- 객체를 DB에 그대로 담기 위해서는 객체-관계 매핑(ORM)이 필요하다.
- JPA를 사용하면 도메인 객체를 크게 훼손하지 않고 DB와 매핑할 수 있다.
- 도메인 영역에는 리포지터리 인터페이스가, 인프라 영역에는 JPA를 사용한 구현체가 위치한다.

> ORM을 소개하는 느낌이 커서 JPA가 낯선 사람이 읽으면 도움이 될 것 같다.
.
## 스프링 데이터 JPA를 이용한 리포지터리 구현

- `JpaRepository<엔티티, ID>` 인터페이스를 상속받기만 하면 기본 CRUD가 자동으로 제공된다.
- 애그리거트를 조회하는 기능의 이름을 지을 때 특별한 규칙은 없지만 많이 사용되는 규칙인 `findBy프로퍼티이름()`을 사용한다.

```java
public interface OrderRepository extends JpaRepository<Order, OrderNo> {
    List<Order> findByOrdererId(String ordererId);
}
```

> 실무에서 스프링 데이터 JPA를 쓸 때 네이밍 규칙만으로는 한계가 있어서 
> `@Query`, QueryDSL을 같이 쓰게 될 것 같다는 생각을 했는데 실무에서는 아직 경험해보지 못했다.

## 매핑 구현

JPA의 문법을 알려주는 느낌이 강했다. (객체 세상을 DB와 매핑하는걸 알려주는 느낌?)

> 실무에서 어떻게 활용하는지 궁금하다.

### 엔티티와 밸류 매핑 (`@Embeddable` / `@Embedded`)

- 밸류 타입은 별도 테이블이 아닌 엔티티 테이블에 임베드되는 경우가 많다.
- 밸류 타입은 `@Embeddable`을 붙이고, 이를 쓰는 엔티티에서는 `@Embedded`로 매핑한다.

```java
@Embeddable
public class Address {
    @Column(name = "zip_code")
    private String zipCode;
    @Column(name = "address1")
    private String address1;
}

@Entity
public class Member {
    @EmbeddedId
    private MemberId id;

    @Embedded
    private Address address;
}
```

### 기본 생성자와 `get/set` 캡슐화 조심

- JPA는 프록시와 리플렉션을 위해 **기본 생성자**가 필요하다. 다만 외부에서 호출되지 않도록 `protected`로 막아두자.

```java
@Embeddable
public class Money {
    @Column(name = "price")
    private int value;

    protected Money() {} // JPA 전용
    public Money(int value) { this.value = value; }

    public int getValue() { return value; }
    // setter 없음
}
```

> JPA 어노테이션을 붙이는 순간, 도메인의 순수성을 조금 양보하게 되는 것 같다.
> `protected` 기본 생성자를 그나마 타협점으로 정한거 같은데 불편한 점은 없는지 궁금하다.

### `AttributeConverter` — 한 프로퍼티를 한 컬럼으로

- 밸류의 프로퍼티가 여러 개지만 DB에는 한 컬럼으로 저장하고 싶을 때 사용한다.
- `AttributeConverter<도메인타입, DB타입>`을 구현하고 `@Convert`로 붙이거나 `autoApply = true`로 전역 적용할 수 있다.

```java
public class MoneyConverter implements AttributeConverter<Money, Integer> {
    @Override
    public Integer convertToDatabaseColumn(Money money) {
        return money == null ? null : money.getValue();
    }

    @Override
    public Money convertToEntityAttribute(Integer value) {
        return value == null ? null : new Money(value);
    }
}

@Entity
public class Order {
    @Convert(converter = MoneyConverter.class)
    private Money totalAmounts;
}
```

> `AttributeConverter`를 팀에서 얼마나 적극적으로 쓰는지 궁금.

### 밸류 컬렉션 — 별도 테이블 매핑 (`@ElementCollection`, `@CollectionTable`)

- 밸류 컬렉션을 별도 테이블로 매핑할 때 사용한다.
- 순서가 의미 있는 리스트라면 `@OrderColumn`으로 인덱스 컬럼을 지정한다.

```java
@Entity
public class Order {
    @EmbeddedId
    private OrderNo number;

    @ElementCollection(fetch = FetchType.LAZY)
    @CollectionTable(
        name = "order_line",
        joinColumns = @JoinColumn(name = "order_number")
    )
    @OrderColumn(name = "line_idx")
    private List<OrderLine> orderLines;
}
```

> `@ElementCollection`은 편하지만, 컬렉션 전체를 삭제 후 재삽입 하는 방식이라 성능 이슈가 있다 한다.
> 직접 사용해본 경험은 없고, 언제 사용하면 좋을까?

## 애그리거트 로딩 전략과 영속성 전파

- 애그리거트는 **루트를 조회할 때 하위 구성요소까지 함께 온전한 상태**로 가져오는 것이 자연스럽다.
- 기본은 **지연 로딩(`LAZY`)**으로 두고, 한 번에 가져와야 하는 상황에서는 `fetch join`이나 `@EntityGraph`를 사용한다.
- `cascade`를 통한 영속성 전파는 애그리거트 내부 구성요소에 한해 적용한다. (예: 주문의 `OrderLine`)

```java
@OneToMany(
    cascade = { CascadeType.PERSIST, CascadeType.REMOVE },
    orphanRemoval = true
)
@JoinColumn(name = "order_number")
private List<OrderLine> orderLines;
```

> 읽으면서 이 소절은 JPA의 **N+1 문제**를 간접적으로 다루는 것 같다고 느꼈다.
> 실무에서 N+1을 어떻게 인지하는지 궁금하고,
> 해결할 때 `fetch join`으로 푸는지, `@EntityGraph`로 푸는지, 아예 QueryDSL이나 MyBatis로 쿼리를 직접 쓰는지 관련 정책이 있는지?
> 이와 별개로 처음 DDD, 애그리거트 개념을 접했을 때 애그리거트의 일부분에 속하는 정보만 조회하고 싶은 경우
> 항상 모든 애그리거트를 select 하는게 맞을까? 라는 고민을 했고 아니라면 그 기준을 어떻게 잡는게 좋을까?

## 식별자 생성 기능

식별자를 만드는 방식은 크게 네 가지로 정리된다.

- **사용자 직접 입력** (이메일, 아이디 같은 경우)
- **도메인 로직으로 생성** — 도메인 서비스나 팩토리에 생성 책임을 둔다
- **DB의 auto-increment / 시퀀스**
- **UUID, Snowflake 같은 별도 생성기**

도메인 로직으로 생성하는 경우, 생성 책임을 어디에 둘지가 중요하다. 단순히 "번호를 매겨주는 규칙"이라면 도메인 서비스로 뽑고, 특정 애그리거트의 상태에 의존한다면 그 애그리거트의 팩토리 메서드로 둔다.

> 주로 auto-increment를 사용했지만, 학습 목적으로 **Snowflake** 알고리즘을 구현해서 분산 환경을 고려한 ID 생성 전략을 적용해본 경험이 있다.
> 솔직히 내 경우엔 꼭 필요했다기 보다는 Snowflake는 학습 용도가 컸다.
>
> auto-increment는 단순하고, 자연스러운 정렬성을 주며, DB가 원자적으로 발급해주니 구현 부담이 없다고 생각한다.
> 다만 샤딩이나 분산 환경에선 중복/병목 문제가 생기고, 외부로 노출되면 ID를 순차적으로 유추당할 수 있다는 단점이 있다.
> Snowflake는 타임스탬프를 내포해서 분산 환경에서도 유일하고 정렬 가능한 ID를 만들 수 있지만, 노드 ID와 시계 동기화를 직접 관리해야 한다.

## 도메인 구현과 DIP

- 도메인이 JPA 어노테이션에 의존하는 순간, 엄밀한 DIP 관점에서는 깨진 것처럼 보인다.
- 저자는 이 부분을 현실적 타협으로 받아들인다. 도메인을 완전히 분리하면 순수성은 얻지만 매핑 코드가 배로 늘고 실제 얻는 이득이 크지 않을 때가 많기 때문이다.

> 저자의 말에 동의한다. 하지만 **어느 경우에 JPA와 도메인 클래스를 분리하는 게 좋을지** 그 기준을 어떻게 정하는게 좋을까?
