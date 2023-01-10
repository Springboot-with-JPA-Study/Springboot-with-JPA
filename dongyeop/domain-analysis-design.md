# 도메인 분석 설계

## 💡 요구사항 분석 및 도메인 모델과 테이블 설계
<img src="https://github.com/Springboot-with-JPA-Study/Springboot-with-JPA/blob/main/dongyeop/image/1.png" width = 500/>

- 회원, 주문, 상품의 관계
    - 회원은 여러 주문을 할 수 있다.
    - 한 주문에는 여러 상품이 들어갈 수 있다. → 다대다 관계

<br/>

<br/>

> 이러한 ***다대다 관계는 관계형 데이터베이스는 물론이고 엔티티에서도 거의 사용하지 않는다!*** 
>
> 따라서 주문상품이라는 엔티티를 추가해서 다대다 관계를 일대다, 다대일 관계로 풀어내자!

<br/>

<br/>

### 💡 외래 키가 있는 곳을 연관관계의 주인으로 정해라.
> 연관관계의 주인은 단순히 외래 키를 누가 관리하냐의 문제이지 비즈니스상 우위에 있다고 주인으로 정하면 안된다!
>
> 예를 들어서 자동차와 바퀴가 있으면, 일대다 관계에서 항상 다쪽에 외래 키가 있으므로 외래 키가 있는 바퀴를 연관관계의 주인으로 정하면 된다. 
>
> 물론 자동차를 연관관계의 주인으로 정하는 것이 불가능 한 것은 아니지만, 자동차를 연관관계의 주인으로 정하면 자동차가 관리하지 않는 바퀴 테이블의 외래 키 값이 업데이트 되므로 관리와 유지보수가 어렵고, 추가적으로 별도의 업데이트 쿼리가 발생하는 성능 문제도 있다.

<br/>

<br/>

## 💡 엔티티 클래스 개발

> 이론적으로 `Getter`, `Setter` 모두 제공하지 않고, 꼭 필요한 별도의 메서드를 제공하는게 가장 이상적이다. 
>
> 하지만 실무에서 엔티티의 데이터는 조회할 일이 너무 많으므로, Getter의 경우 모두 열어두는 것이 편리하다. 
>
> → Getter는 아무리 호출해도 호출 하는 것 만으로 어떤 일이 발생하지는 않는다! 
>
> → 하지만 Setter는 문제가 다르다. Setter를 호출하면 데이터가 변한다. Setter를 막 열어두면 가까운 미래에 엔티티에가 도대체 왜 변경되는지 추적하기 점점 힘들어진다. 
>
>
> → 엔티티를 변경할 때는 ***Setter 대신에 변경 지점이 명확하도록 변경을 위한 비즈니스 메서드를 별도로 제공***해야 한다.

<br/>

<br/>

회원 엔티티 예제
```java
@Entity
@Getter @Setter
public class Member {
    @Id @GeneratedValue
    @Column(name = "member_id")
    private Long id;
    
    private String name;
    
    @Embedded
    private Address address;

    @OneToMany(mappedBy = "member")
    private List<Order> orders = new ArrayList<>();
}
```

- 엔티티의 식별자는 id 를 사용하고 PK 컬럼명은 `member_id` 를 사용했다. 
- 엔티티는 타입(여기서는 Member)이 있으므로 id 필드만으로 쉽게 구분할 수 있다. 
- 하지만 테이블은 타입이 없으므로 구분이 어렵다! 
  - 그리고 테이블은 관례상 `테이블명 + id` 를 많이 사용한다. 
  - 참고로 객체에서 id 대신에 memberId 를 사용해도 된다. ***중요한 것은 일관성***이다.

<br/>

<br/>

### 💡 실무에서는 `@ManyToMany` 를 사용하지 말자

> `@ManyToMany`는 편리한 것 같지만 아래와 같은 단점을 가진다.
> - 중간 테이블( CATEGORY_ITEM )에 컬럼을 추가할 수 없다.
> - 세밀하게 쿼리를 실행하기 어렵기 때문에 실무에서 사용하기에는 한계가 있다. 
>
> → 따라서 중간 엔티티(CategoryItem)를 만들고 `@ManyToOne` , `@OneToMany`로 매핑해서 사용하자!
>
> → ***다시 말해, 대다대 매핑을 일대다, 다대일 매핑으로 풀어내서 사용하자.***

<br/>

<br/>

주소 값 타입 예제
```java
@Embeddable
@Getter
public class Address {
    private String city;
    private String street;
    private String zipcode;
    
    protected Address() {}

    public Address(String city, String street, String zipcode) {
        this.city = city;
        this.street = street;
        this.zipcode = zipcode;
    } 
}
```
- 값 타입은 변경 불가능하게 설계해야 한다.
  - `@Setter`를 제거하고, 생성자에서 값을 모두 초기화해서 변경 불가능한 클래스를 만들자! 
- JPA 스펙상 엔티티나 임베디드 타입(`@Embeddable`)은 자바 기본 생성자(default constructor)를 public 또는 protected 로 설정해야 한다. 
  - public 으로 두는 것 보다는 protected 로 설정하는 것이 그나마 더 안전하다.
  - JPA가 이런 제약을 두는 이유는 JPA 구현 라이브러리가 객체를 생성할 때 **리플랙션 같은 기술을 사용할 수 있도록 지원**해야 하기 때문이다.

<br/>

<br/>
