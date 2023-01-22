# 변경 감지와 병합

<br/>

## 준영속 엔티티란?
- ***영속성 컨텍스트가 더는 관리하지 않는 엔티티***를 말한다.
    - 아래 코드(`ItemController.java`)에서 Book 객체에 해당 
        ```java
        @PostMapping("/items/{itemId}/edit")
        public String updateItem(@PathVariable Long itemId, @ModelAttribute("form") BookForm form) {

            Book book = new Book();
            //book은 준영속 상태이므로 변경이 저장이 안됨
            book.setId(form.getId());
            book.setName(form.getName());
            book.setPrice(form.getPrice());
            book.setStockQuantity(form.getStockQuantity());
            book.setAuthor(form.getAuthor());
            book.setIsbn(form.getIsbn());

            //따라서 직접 save 호출해야함.
            itemService.saveItem(book);
            return "redirect:/items";
        }
        ```

<br/>

<br/>

## 준영속 엔티티를 수정하는 2가지 방법
#### 1. 변경 감지 기능(`dirty checking`)사용
- `ItemService.java`
    ```java
    @Transactional
    public Item updateItem(Long itemId, Book param) {
        Item findItem = itemRepository.findOne(itemId);
        findItem.setPrice(param.getPrice());
        findItem.setName(param.getName());
        findItem.setStockQuantity(param.getStockQuantity());
        //기타 등등 나머지 필드도 채웠다고 가정

        return findItem;
    }
    ```

<br/>

- 기능 설명
    - 매개변수로 넘어온 `param`은 준영속 엔티티이다.
    - 따라서 ***트랜잭션 안에서 엔티티를 다시 조회하고, 변경할 값 선택하자!***
    - repo에서 조회를 통해 가져온 Item이므로 트랜잭션 커밋 시점에 변경 감지(Dirty Checking)이 동작한다 → 데이터베이스에 UPDATE SQL 실행

<br/>

- 알아두기
    - `findItem` 자체가 repo에서 찾아온 영속성 객체이다.
    - 따라서 `itemRepository.save(findItem);`를 안해도 반영이 된다.
    - `@Transactional`이 commit을 한다. (변경사항들을 flush)


<br/>

<br/>

#### 2. 병합(`merge`) 사용
- 병합은 준영속 상태의 엔티티를 영속 상태로 변경할 때 사용하는 기능이다.
- `ItemRepository.java`
    ```java
    @Transactional
    void update(Item itemParam) { 
        //itemParam: 파리미터로 넘어온 준영속 상태의 엔티티 
        Item mergeItem = em.merge(itemParam);
    }
    ```

<br/>

#### 병합 동작 방식

<img src="https://github.com/Springboot-with-JPA-Study/Springboot-with-JPA/blob/main/dongyeop/image/2.png" width = 500/>

1. `merge()`를실행한다.
2. 파라미터로 넘어온 준영속 엔티티의 식별자 값으로 1차 캐시에서 엔티티를 조회한다.
    - 만약 1차 캐시에 엔티티가 없으면 데이터베이스에서 엔티티를 조회하고, 1차 캐시에 저장한다.
3. 조회한 영속 엔티티(`mergeMember`)에 member 엔티티의 값을 채워 넣는다. 
    - ***member 엔티티의 모든 값을 mergeMember에 밀어 넣는다.*** 이때 mergeMember의 “회원1”이라는 이름이 “회원명변경”으로 바뀐다.)
4. 영속 상태인 mergeMember를 반환한다.

<br/>

> 병합시 동작 방식을 간단히 정리 
>1. 준영속 엔티티의 식별자 값으로 영속 엔티티를 조회한다.
>2. 영속 엔티티의 값을 준영속 엔티티의 값으로 모두 교체한다.(병합한다.)
>3. 트랜잭션 커밋 시점에 변경 감지 기능이 동작해서 데이터베이스에 UPDATE SQL이 실행

<br/>


> 주의: 병합을 사용하지 말자.
>
>변경 감지 기능을 사용하면 원하는 속성만 선택해서 변경할 수 있지만, **병합을 사용하면 모든 속성이 변경**된다. 
>
>***병합시 값이 없으면 `null` 로 업데이트 할 위험도 있다.***

<br/>

<br/>

## 가장 최적의 코드
> 엔티티를 변경할 때는 항상 변경 감지를 사용하자.
>1. 컨트롤러에서 어설프게 엔티티를 생성하지 마세요.
>2. 트랜잭션이 있는 서비스 계층에 식별자(id)와 변경할 데이터를 명확하게 전달하세요.(파라미터 or dto를 이용하기) 
>3. 트랜잭션이 있는 서비스 계층에서 영속 상태의 엔티티를 조회하고, 엔티티의 데이터를 직접 변경하세요. 
>4. 트랜잭션 커밋 시점에 변경 감지가 실행됩니다.

<br/>

- 기존 `ItemController`의 `update()`로직
    ```java
    @PostMapping("/items/{itemId}/edit")
    public String updateItem(@PathVariable Long itemId, @ModelAttribute("form") BookForm form) {

        Book book = new Book();
        book.setId(form.getId());
        book.setName(form.getName());
        book.setPrice(form.getPrice());
        book.setStockQuantity(form.getStockQuantity());
        book.setAuthor(form.getAuthor());
        book.setIsbn(form.getIsbn());

        //book은 준영속 상태이므로 변경이 저장이 안됨
        //따라서 직접 save 호출해야함.
        itemService.saveItem(book);
        return "redirect:/items";
    }    
    ```

<br/>

- 개선된 `ItemController`의 `update()`로직
    ```java
    @PostMapping("/items/{itemId}/edit")
    public String updateItem(@PathVariable Long itemId, @ModelAttribute("form") BookForm form) {

        //아래의 경우와 달리, 가져올 파라미터(값)가 많으면 Service계층에 DTO만들기
        itemService.updateItem(itemId, form.getName(), form.getPrice(), form.getStockQuantity());
    }    
    ```

<br/>

- 기존 `ItemService`의 수정 로직
    ```java
    @Transactional
    public Item updateItem(Long itemId, Book param) {
        Item findItem = itemRepository.findOne(itemId);
        findItem.setPrice(param.getPrice());
        findItem.setName(param.getName());
        findItem.setStockQuantity(param.getStockQuantity());
        //기타 등등 나머지 필드도 채웠다고 가정

        return findItem;
    }
    ```

<br/>

- 개선된 `ItemService`의 수정 로직
    ```java
    @Transactional
    public Item updateItem(Long itemId, String name, int price, int stockQuantity) {
        Item findItem = itemRepository.findOne(itemId);
        findItem.setName(name);
        findItem.setPrice(price);
        findItem.setStockQuantity(stockQuantity);
    }
    ```

<br/>

<br/>

## 기존의 Setter를 이용한 코드를 변경 메소드로 수정

- `ItemController`의 상품 생성 로직에 `change()` 메서드로 setter 제거
    ```java
    @PostMapping("/items/new")
    public String create(BookForm form) {
        //책 생성
        Book book = new Book();
        book.change(form);

        //저장
        itemService.saveItem(book);
        return "redirect:/";
    }
    ```

<br/>

- `BookForm`에 `change()` 메소드 정의
    ```java
    import jpabook.jpashop.domain.item.Book;
    import lombok.Getter;
    import lombok.Setter;

    public class BookForm {
        //Book만 해당하는 속성
        private String author;
        private String isbn;

        public void change(Book item) {
            this.id = item.getId();
            this.name = item.getName();
            this.price = item.getPrice();
            this.stockQuantity = item.getStockQuantity();
            this.author = item.getAuthor();
            this.isbn = item.getIsbn();
        }
    }
    ```

<br/>

