# 상품 주문, 검색 및 취소

<br/>

## 상품 주문
#### 상품 주문 컨트롤러
- 주문 컨트롤러 코드
    ```java
    import jpabook.jpashop.domain.Member;
    import jpabook.jpashop.domain.item.Item;
    import jpabook.jpashop.service.ItemService;
    import jpabook.jpashop.service.MemberService;
    import jpabook.jpashop.service.OrderService;
    import lombok.RequiredArgsConstructor;
    import org.springframework.stereotype.Controller;
    import org.springframework.ui.Model;
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.PostMapping;
    import org.springframework.web.bind.annotation.RequestParam;

    import java.util.List;

    @Controller
    @RequiredArgsConstructor
    public class OrderController {

        private final OrderService orderService;
        private final MemberService memberService;
        private final ItemService itemService;

        @GetMapping("/order")
        public String createForm(Model model) {

            List<Member> members = memberService.findMembers();
            List<Item> items = itemService.findItems();

            model.addAttribute("members", members);
            model.addAttribute("items", items);

            return "order/orderForm";
        }

        @PostMapping("/order")
        public String order(@RequestParam("memberId") Long memberId,
                            @RequestParam("itemId") Long itemId,
                            @RequestParam("count") int count) {

            orderService.order(memberId, itemId, count);
            return "redirect:/orders";
        }
    }
    ```

<br/>

<br/>

## 주문 목록 검색 및 취소
- `OrderController`에 기능 추가
    ```java
    /**
    * 주문 검색
    */
    @GetMapping("/orders")
    public String orderList(
        @ModelAttribute("orderSearch") OrderSearch orderSearch,
        Model model) {

            List<Order> orders = orderService.findOrders(orderSearch);
            model.addAttribute("orders", orders);

            return "order/orderList";
        }

    @PostMapping("/orders/{orderId}/cancel")
    public String cancelOrder(@PathVariable("orderId") Long orderId) {
        orderService.cancelOrder(orderId);
        return "redirect:/orders";
    }

    /**
    * 주문 취소
    */
    @PostMapping(value = "/orders/{orderId}/cancel")
    public String cancelOrder(@PathVariable("orderId") Long orderId) {
        orderService.cancelOrder(orderId);
        return "redirect:/orders";
    }
    ```
