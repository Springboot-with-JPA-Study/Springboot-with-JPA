## 상품 도메인 개발

#### 상품 엔티티
- 알아두기
  - `@Inheritance(strategy = InheritanceType.SINGLE_TABLE)`
    - 상속관계 매핑 : SINGLE_TABLE로, 즉 한 테이블로 모두 관리함을 의미
  - `addStock()`이나 `removeStock()` 과 같은 메서드로 Setter를 대체할 것!

<br/>

- 엔티티 코드
    ```java
    import jpabook.jpashop.exception.NotEnoughStockException;
    import lombok.Getter;
    import lombok.Setter;
    import jpabook.jpashop.domain.Category;
    import javax.persistence.*;
    import java.util.ArrayList;
    import java.util.List;

    @Entity
    @Inheritance(strategy = InheritanceType.SINGLE_TABLE)
    @DiscriminatorColumn(name = "dtype")
    @Getter @Setter
    public abstract class Item {
        @Id @GeneratedValue
        @Column(name = "item_id")
        private Long id;

        private String name;
        private int price;
        private int stockQuantity;

        @ManyToMany(mappedBy = "items")
        private List<Category> categories = new ArrayList<Category>();

        //==비즈니스 로직==//
        public void addStock(int quantity) {
            this.stockQuantity += quantity;
        }
        
        public void removeStock(int quantity) {
            int restStock = this.stockQuantity - quantity;
            if (restStock < 0) {
                throw new NotEnoughStockException("need more stock");
            }
            this.stockQuantity = restStock;
        }
    }    
    ```

<br/>

<br/>

#### 상품 리포지토리
- 기능 설명
  - `save()`
    - id 가 없으면 신규로 보고 `persist()` 실행
    - id 가 있으면 이미 데이터베이스에 저장된 엔티티를 수정한다(`merge()` 를 실행)

<br/>

- 레포지토리 코드
```java
import jpabook.jpashop.domain.item.Item;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Repository;
import javax.persistence.EntityManager;
import java.util.List;

@Repository
@RequiredArgsConstructor
public class ItemRepository {
    private final EntityManager em;

    public void save(Item item) {
        if (item.getId() == null) { //아이디가 없다는건 새로 생성했다는 객체
            em.persist(item);
        } else {                    //아이디가 있으면 이미 DB에 등록된 객체를 가져옴
            em.merge(item);         //like update라고 생각하기!
        }
    }
    
    public Item findOne(Long id) {
        return em.find(Item.class, id);
    }
    
    public List<Item> findAll() {
        return em.createQuery("select i from Item i",Item.class)
                 .getResultList();
    }
}
```
