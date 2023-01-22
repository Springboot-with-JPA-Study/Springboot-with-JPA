# 웹 계층 : 상품 등록 및 조회, 수정

<br/>

## 상품 등록
#### 상품 등록 폼
- `BookForm`
    ```java
    import lombok.Getter;
    import lombok.Setter;

    @Getter @Setter
    public class BookForm {

        private Long id;

        //Item 공통 속성
        private String name;
        private int price;
        private int stockQuantity;

        //Book만 해당하는 속성
        private String author;
        private String isbn;
    }
    ```

<br/>

#### 상품 등록 컨트롤러
- 기능 설명
  - 상품 등록 폼에서 데이터를 입력하고 Submit 버튼을 클릭하면 `/items/new`를 `POST` 방식으로 요청한다.
  - 상품 저장이 끝나면 상품 목록 화면(`"redirect:/items"`)으로 리다이렉트한다.

<br/>

- 알아두기
  - 지금은 예제를 위해 Setter를 이용했지만, 꼭 별도의 생성메서드를 이용하자.

<br/>

- `ItemController`
    ```java
    import jpabook.jpashop.domain.item.Book;
    import jpabook.jpashop.service.ItemService;
    import lombok.RequiredArgsConstructor;
    import org.springframework.stereotype.Controller;
    import org.springframework.ui.Model;
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.PostMapping;

    @Controller
    @RequiredArgsConstructor
    public class ItemController {

        private final ItemService itemService;

        @GetMapping("/items/new")
        public String createForm(Model model) {
            model.addAttribute("form", new BookForm());
            return "items/createItemForm";
        }

        @PostMapping("/items/new")
        public String create(BookForm form) {
            //책 생성
            //아래처럼 setter를 만들지 말고, 생성메서드를 만들자.
            Book book = new Book();
            book.setName(form.getName());
            book.setPrice(form.getPrice());
            book.setStockQuantity(form.getStockQuantity());
            book.setAuthor(form.getAuthor());
            book.setIsbn(form.getIsbn());

            //저장
            itemService.saveItem(book);
            return "redirect:/";
        }
    }
    ```

<br/>

<br/>

## 상품 목록 조회
- 상품 컨트롤러에 조회 로직 추가
    ```java
    @GetMapping(value = "/items")
    public String list(Model model) {
        List<Item> items = itemService.findItems();
        model.addAttribute("items", items);
        return "items/itemList";
    }
    ```

<br/>

<br/>

## 상품 수정
- 기능 설명
    - 상품 수정 폼 이동
        1. 수정 버튼을 선택하면 `/items/{itemId}/edit` URL로 `GET` 방식으로 요청한다.
        2. 그 결과로 `updateItemForm()` 메서드를 실행하는데 이 메서드는 `itemService.findOne(itemId)`를 호출해서 수정할 상품을 조회한다.
        3. 조회 결과를 모델 객체에 담아서 뷰(`items/updateItemForm`)에 전달한다.
    
    <br/>

    - 상품 수정 실행
        1. 상품 수정 폼에서 정보를 수정하고 Submit 버튼을 선택
        2. `/items/{itemId}/edit` URL로 `POST` 방식으로 요청하고 `updateItem()` 메서드를 실행
        3. 이때 컨트롤러에 파라미터로 넘어온 item 엔티티 인스턴스는 현재 ***준영속 상태***다. 따라서 영속성 컨텍스트의 지원을 받을 수 없고 ***데이터를 수정해도 변경 감지 기능은 동작하지 않는다.***

<br/>

- 상품 컨트롤러에 수정 로직 추가
    ```java
    /**
    * 상품 수정 폼
    */
    @GetMapping(value = "/items/{itemId}/edit")
    public String updateItemForm(@PathVariable("itemId") Long itemId, Model model) {
        Book item = (Book) itemService.findOne(itemId);

        BookForm form = new BookForm();
        form.setId(item.getId());
        form.setName(item.getName());
        form.setPrice(item.getPrice());
        form.setStockQuantity(item.getStockQuantity());
        form.setAuthor(item.getAuthor());
        form.setIsbn(item.getIsbn());

        model.addAttribute("form", form);
        return "items/updateItemForm";
    }

    /**
    * 상품 수정
    */
    PostMapping(value = "/items/{itemId}/edit")
    public String updateItem(@ModelAttribute("form") BookForm form) {
        Book book = new Book();
        book.setId(form.getId());
        book.setName(form.getName());
        book.setPrice(form.getPrice());
        book.setStockQuantity(form.getStockQuantity());
        book.setAuthor(form.getAuthor());
        book.setIsbn(form.getIsbn());
        
        itemService.saveItem(book);
        return "redirect:/items";
    }
    ```
