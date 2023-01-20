## 주문 도메인 개발

#### 주문 엔티티 개발

- 기능 설명
  - 생성 메서드(`createOrder()`): 주문 엔티티를 생성할 때 사용한다. 
    - 주문 회원, 배송정보, 주문상품의 정보를 받아서 실제 주문 엔티티를 생성한다.
  - 주문 취소(`cancel()`): 주문 취소시 사용한다. 
    - 주문 상태를 취소로 변경하고 주문상품에 주문 취소를 알린다. 
    - 만약 이미 배송을 완료한 상품이면 주문을 취소하지 못하도록 예외를 발생시킨다.
  - 전체 주문 가격 조회(`getTotalPrice()`): 주문 시 사용한 전체 주문 가격을 조회한다.
    - 전체 주문 가격을 알려면 각각의 주문상품 가격을 알아야 한다. 
    - 로직을 보면 연관된 주문상품들의 가격을 조회해서 더한 값을 반환한다. 
    - (실무에서는 주로 주문에 전체 주문 가격 필드를 두고 역정규화 한다.)

<br/>

- 알아두기
  - `XXXToOne`은 FetchType의 기본값이 `EAGER`이므로 직접 `LAZY`로 바꿔주기
    - 반대로 `XXXToMany`는 FetchType의 기본 값이 `Lazy`이므로 냅두자.
  - `@Enumerated(EnumType.STRING)` : 디폴트 값은 EnumType.ORDINARY인데 쓰지말자.
  - `setMember()`와 같은 연관관계 편의 메소드는 컨트롤하는 쪽이 들고 있는게 좋다!

<br/>

- 주문 엔티티 코드
    ```java
    import lombok.Getter;
    import lombok.Setter;
    import javax.persistence.*;
    import java.time.LocalDateTime;
    import java.util.ArrayList;
    import java.util.List;

    @Entity
    @Table(name = "orders")
    @Getter @Setter
    public class Order {
        @Id @GeneratedValue
        @Column(name = "order_id")
        private Long id;

        @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "member_id") private Member member; //주문 회원

        @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
        private List<OrderItem> orderItems = new ArrayList<>();

        @OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY) @JoinColumn(name = "delivery_id")  //주인
        private Delivery delivery; //배송정보

        private LocalDateTime orderDate; //주문시간 
        
        @Enumerated(EnumType.STRING) //EnumType.ORDINARY가 디폴트인데 절대 쓰지 말 것!! 중간에 다른거 끼면 골치아픔
        private OrderStatus status; //주문상태 [ORDER, CANCEL]

        //==연관관계 메서드==//
        public void setMember(Member member) {
            this.member = member;
            member.getOrders().add(this);
        }
        
        public void addOrderItem(OrderItem orderItem) {
            orderItems.add(orderItem);
            orderItem.setOrder(this);
        }

        public void setDelivery(Delivery delivery) {
            this.delivery = delivery;
            delivery.setOrder(this);
        }

        //==생성 메서드==//
        public static Order createOrder(Member member, Delivery delivery, OrderItem... orderItems) {
            Order order = new Order();
            order.setMember(member);
            order.setDelivery(delivery);
        
            for (OrderItem orderItem : orderItems) {
                order.addOrderItem(orderItem);
            }
            order.setStatus(OrderStatus.ORDER);
            order.setOrderDate(LocalDateTime.now());
            return order;
        }

        //==비즈니스 로직==//
        /** 주문 취소 */
        public void cancel() {
            if (delivery.getStatus() == DeliveryStatus.COMP) {
                throw new IllegalStateException("이미 배송완료된 상품은 취소가 불가능합니다."); 
            }
            this.setStatus(OrderStatus.CANCEL);
            for (OrderItem orderItem : orderItems) {
                orderItem.cancel();
            }
        }

        //==조회 로직==//
        /**전체 주문 가격 조회*/ 
        public int getTotalPrice() {
            int totalPrice = 0;
            for (OrderItem orderItem : orderItems) {
                totalPrice += orderItem.getTotalPrice();
            }
            return totalPrice;
        }
    }
    ```

<br/>

<br/>

#### 주문상품 엔티티 개발

- 기능 설명
  - 생성 메서드(`createOrderItem()`)
    - 주문 상품, 가격, 수량 정보를 사용해서 주문상품 엔티티를 생성한다. 
    - 그리고 `item.removeStock(count)`를 호출해서 주문한 수량만큼 상품의 재고를 줄인다.
  - 주문 취소(`cancel()`)
    - `getItem().addStock(count)`를 호출해서 취소한 주문 수량만큼 상품의 재고를 증가시킨다.
  - 주문 가격 조회(`getTotalPrice()`)
    - 주문 가격에 수량을 곱한 값을 반환한다.

<br/>

- 알아두기
  - `@NoArgsConstructor(access = AccessLevel.PROTECTED)`
    - protected OrderItem() {}와 같은 의미로, 기본 생성자가 아닌 생성 메서드를 이용하라는 의미이다.

<br/>

- 주문상품 엔티티 코드
    ```java
    import lombok.Getter;
    import lombok.Setter;
    import jpabook.jpashop.domain.item.Item;
    import javax.persistence.*;

    @Entity
    @Table(name = "order_item")
    @Getter @Setter
    @NoArgsConstructor(access = AccessLevel.PROTECTED) //protected OrderItem() {}와 같음
    public class OrderItem {
        @Id @GeneratedValue
        @Column(name = "order_item_id")
        private Long id;

        @ManyToOne(fetch = FetchType.LAZY) 
        @JoinColumn(name = "item_id") 
        private Item item; //주문 상품

        @ManyToOne(fetch = FetchType.LAZY) 
        @JoinColumn(name = "order_id") 
        private Order order; //주문

        private int orderPrice; //주문 가격
        private int count; //주문 수량

        //==생성 메서드==//
        public static OrderItem createOrderItem(Item item, int orderPrice, int count) {
            OrderItem orderItem = new OrderItem();
            orderItem.setItem(item);
            orderItem.setOrderPrice(orderPrice);
            orderItem.setCount(count);
            item.removeStock(count);
            return orderItem;
        }

        //==비즈니스 로직==//
        /** 주문 취소 */
        public void cancel() {
            getItem().addStock(count);
        }

        //==조회 로직==//
        /** 주문상품 전체 가격 조회 */ 
        public int getTotalPrice() {
            return getOrderPrice() * getCount();
        }
    }
    ```

<br/>

<br/>

#### 주문 리포지토리 개발

- 주문 리포지토리 코드
```java
import jakarta.persistence.EntityManager;
import jpabook.jpashop.domain.Order;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
@RequiredArgsConstructor
public class OrderRepository {

    private final EntityManager em;

    public void save(Order order) {
        em.persist(order);
    }

    public Order findOne(Long id) {
        return em.find(Order.class, id);
    }

    /**
     * 검색 기능 - 추후에 생성 예정
     */
//    public List<Order> findAll(OrderSerach orderSerach) {
//
//    }
}
```

<br/>

<br/>

#### 주문 서비스 개발
- 설명
  - 주문 서비스는 주문 엔티티와 주문 상품 엔티티의 비즈니스 로직을 활용해서 주문, 주문 취소, 주문 내역 검색 기능을 제공한다.
  - 예제를 단순화하려고 한 번에 하나의 상품만 주문할 수 있다고 가정했다.

<br/>

- 기능 설명
  - 주문(`order()`)
    - 주문하는 회원 식별자, 상품 식별자, 주문 수량 정보를 받아서 실제 주문 엔티티를 생성한 후 저장한다.
  - 주문 취소(`cancelOrder()`)
    - 주문 식별자를 받아서 주문 엔티티를 조회한 후 주문 엔티티에 주문 취소를 요청한다.
  - 주문 검색(`findOrders()`)
    - OrderSearch 라는 검색 조건을 가진 객체로 주문 엔티티를 검색한다. 

<br/>

- 알아두기
  - Order 내에 orderItems, delivery가 조인 속성이 `CascadeType.ALL`이다.
  - 따라서, order만 save해도 Item, Delivery가 자동으로 save된다!
  - 단, ***Cascade는 다른 곳에서도 참조를 한다면 ALL로 하지 말자!!*** 

<br/>

- 주문 서비스 코드
    ```java
    import jpabook.jpashop.domain.Delivery;
    import jpabook.jpashop.domain.Member;
    import jpabook.jpashop.domain.Order;
    import jpabook.jpashop.domain.OrderItem;
    import jpabook.jpashop.domain.item.Item;
    import jpabook.jpashop.repository.ItemRepository;
    import jpabook.jpashop.repository.MemberRepository;
    import jpabook.jpashop.repository.OrderRepository;
    import lombok.RequiredArgsConstructor;
    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Transactional;

    import java.util.List;

    @Service
    @Transactional(readOnly = true)
    @RequiredArgsConstructor
    public class OrderService {

        private final OrderRepository orderRepository;
        private final MemberRepository memberRepository;
        private final ItemRepository itemRepository;

        /**
        * 주문
        */
        @Transactional
        public Long order(Long memberId, Long itemId, int count) {

            //엔티티 조회
            Member member = memberRepository.findOne(memberId);
            Item item = itemRepository.findOne(itemId);

            //배송정보 생성
            Delivery delivery = new Delivery();
            delivery.setAddress(member.getAddress());

            //주문상품 생성
            OrderItem orderItem = OrderItem.createOrderItem(item, item.getPrice(), count);

            //주문 생성
            Order order = Order.createOrder(member, delivery, orderItem);

            //주문 저장
            orderRepository.save(order);
            return order.getId();
        }

        /**
        * 주문 취소
        */
        @Transactional
        public void cancelOrder(Long orderId) {
            //주문 엔티티 조회
            Order order = orderRepository.findOne(orderId);

            //주문 취소
            order.cancel();
        }

        /**
        * 주문 검색 : 추후에 완성 예정
        */
    //    public List<Order> findOrders(OrderSearch orderSearch) {
    //        return orderRepository.findAll(orderSearch);
    //    }
    }
    ```

<br/>

<br/>

> 주문 서비스의 주문과 주문 취소 메서드를 보면 비즈니스 로직 대부분이 엔티티에 있다. 
> 
> 서비스 계층은 단순히 엔티티에 필요한 요청을 위임하는 역할을 한다. 
> 이처럼 엔티티가 비즈니스 로직을 가지고 객체 지향의 특성을 적극 활용하는 것을 [도메인 모델 패턴](http://martinfowler.com/eaaCatalog/domainModel.html)이라 한다. 
>
>반대로 엔티티에는 비즈니스 로직이 거의 없고 서비스 계층에서 대부분의 비즈니스 로직을 처리하는 것을 [트랜잭션 스크립트 패턴](http://martinfowler.com/eaaCatalog/transactionScript.html)이라 한다.

<br/>

<br/>

## 주문 도메인 테스트

### 주문 기능 테스트

- 테스트 요구 사항
  - 상품 주문이 성공해야 한다.
  - 상품을 주문할 때 재고 수량을 초과하면 안 된다. 
  - 주문 취소가 성공해야 한다.

<br/>

<br/>

- 상품 주문 테스트 코드
  ```java
  import jakarta.persistence.EntityManager;
  import jpabook.jpashop.domain.Address;
  import jpabook.jpashop.domain.Member;
  import jpabook.jpashop.domain.Order;
  import jpabook.jpashop.domain.OrderStatus;
  import jpabook.jpashop.domain.item.Book;
  import jpabook.jpashop.domain.item.Item;
  import jpabook.jpashop.exception.NotEnoughStockException;
  import jpabook.jpashop.repository.OrderRepository;
  import org.junit.Test;
  import org.junit.runner.RunWith;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.boot.test.context.SpringBootTest;
  import org.springframework.test.context.junit4.SpringRunner;
  import org.springframework.transaction.annotation.Transactional;

  import static org.junit.Assert.*;

  @RunWith(SpringRunner.class)
  @SpringBootTest
  @Transactional
  public class OrderServiceTest {

      @Autowired
      EntityManager em;
      @Autowired
      OrderService orderService;
      @Autowired
      OrderRepository orderRepository;

      @Test
      public void 상품주문() throws Exception {
          //given -- 조건
          Member member = createMember();

          Book book = createBook("시골 JPA", 10000, 10);

          int orderCount = 2;

          //when -- 동작
          Long orderId = orderService.order(member.getId(), book.getId(), orderCount);

          //then -- 검증
          Order getOrder = orderRepository.findOne(orderId);

          assertEquals("상품 주문시 상태는 ORDER", OrderStatus.ORDER, getOrder.getStatus());
          assertEquals("주문한 상품 종류 수가 정확해야 한다.", 1, getOrder.getOrderItems().size());
          assertEquals("주문 가격은 가격 * 수량이다.", 10000 * orderCount, getOrder.getTotalPrice());
          assertEquals("주문 수량만큼 재고가 줄어야한다.", 8, book.getStockQuantity());

      }
  }
  ```

<br/>

> 상품주문이 정상 동작하는지 확인하는 테스트다. 
>
> - Given 절에서 테스트를 위한 회원과 상품을 만들고,
>  - When 절에서 실제 상품을 주문하고,
> - Then 절에서 주문 가격이 올바른지, 주문 후 재고 수량이 정확히 줄었는지 검증한다.

<br/>

<br/>

### 재고 수량 초과 테스트

- 재고 수량 초과 테스트 코드 : `NotEnoughStockException`이 발생해야 한다.
  ```java
  @Test(expected = NotEnoughStockException.class) 
  public void 상품주문_재고수량초과() throws Exception {
      //Given
      Member member = createMember();
      Item item = createBook("시골 JPA", 10000, 10); //이름, 가격, 재고
      int orderCount = 11; //재고보다 많은 수량
    
      //When
      orderService.order(member.getId(), item.getId(), orderCount);
      
      //Then
      fail("재고 수량 부족 예외가 발생해야 한다."); 
  }
  ```

<br/>

<br/>

### 주문 취소 테스트

- 주문 취소 테스트 코드 : 주문을 취소하면 그만큼 재고가 증가해야 한다.
```java
@Test
public void 주문취소() {
    //Given
    Member member = createMember();
    Item item = createBook("시골 JPA", 10000, 10); //이름, 가격, 재고
    int orderCount = 2;
    Long orderId = orderService.order(member.getId(), item.getId(),
  orderCount);

    //When
    orderService.cancelOrder(orderId);
    
    //Then
    Order getOrder = orderRepository.findOne(orderId); assertEquals("주문 취소시 상태는 CANCEL 이다.",OrderStatus.CANCEL, getOrder.getStatus());

    assertEquals("주문이 취소된 상품은 그만큼 재고가 증가해야 한다.", 10,
    item.getStockQuantity());
}
```

<br/>

> 주문을 취소하려면 먼저 주문을 해야 한다. 
> - Given 절에서 주문하고, When 절에서 해당 주문을 취소했다. 
> - Then 절에서 주문상태가 주문 취소 상태인지(`CANCEL`), 취소한 만큼 재고가 증가했는지 검증한다.

<br/>

<br/>

## 주문 검색 기능 개발
- JPA에선 동적쿼리를 어떻게 해결해야 하는가?

<br/>

- 검색에 쓰이는 파라미터(`OrderSearch`)
  ```java
  public class OrderSearch {
      private String memberName; //회원 이름
      private OrderStatus orderStatus;//주문 상태[ORDER, CANCEL] 
      //Getter, Setter
  }
  ```

<br/>

<br/>

#### JPQL로 처리하기
- `findAllByString()`
  ```java
  /**
  * 동적 쿼리가 아닌 직접 쿼리 작성하는 방식 : 권장x
  */
  public List<Order> findAllByString(OrderSearch orderSearch) {

      String jpql = "select o from Order o join o.member m";
      boolean isFirstCondition = true;

      //주문 상태 검색
      if (orderSearch.getOrderStatus() != null) {
          if (isFirstCondition) {
              jpql += "where";
              isFirstCondition = false;
          } else {
              jpql += " and";
          }
          jpql += " o.status = :status";
      }

      //회원 이름 검색
      if (StringUtils.hasText(orderSearch.getMemberName())) {
          if (isFirstCondition) {
              jpql += " where";
              isFirstCondition = false;
          } else {
              jpql += " and";
          }
          jpql += " m.name like :name";
      }

      TypedQuery<Order> query = em.createQuery(jpql, Order.class)
                                  .setMaxResults(1000);

      if (orderSearch.getOrderStatus() != null) {
          query = query.setParameter("status", orderSearch.getOrderStatus());
      }
      if (StringUtils.hasText(orderSearch.getMemberName())) {
          query = query.setParameter("name", orderSearch.getMemberName());
      }

      return query.getResultList();
  }
  ```

<br/>

<br/>

#### JPA Criteria로 처리
- `findAllByCriteria()`
```java
/**
* JPA Criteria : 권장 방법x
* JPA가 제공하는 동적 쿼리를 작성할수 있는 방법
*/
public List<Order> findAllByCriteria(OrderSearch orderSearch) {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<Order> cq = cb.createQuery(Order.class);
    Root<Order> o = cq.from(Order.class);
    Join<Order, Member> m = o.join("member", JoinType.INNER); //회원과 조인
    List<Predicate> criteria = new ArrayList<>();

    //주문 상태 검색
    if (orderSearch.getOrderStatus() != null) {
        Predicate status = cb.equal(o.get("status"),
        orderSearch.getOrderStatus());
        criteria.add(status);
    }

    //회원 이름 검색
    if (StringUtils.hasText(orderSearch.getMemberName())) {
        Predicate name =
            cb.like(m.<String>get("name"), "%" +
                orderSearch.getMemberName() + "%");
        criteria.add(name);
    }

    cq.where(cb.and(criteria.toArray(new Predicate[criteria.size()])));
    TypedQuery<Order> query = em.createQuery(cq).setMaxResults(1000); //최대 1000건
    return query.getResultList();
}
```

<br/>

<br/>

> JPA Criteria에 대한 자세한 내용은 자바 ORM 표준 JPA 프로그래밍 책을 참고하자!
>
> 동적 쿼리를 해결하는 방법은 Querydsl이다. 이는 추후에 배워보자.