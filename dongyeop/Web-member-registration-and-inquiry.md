# 웹 계층 : 회원 기능 개발

<br/>

## 홈 화면 및 레이아웃
#### 홈 컨트롤러
- 코드
    ```java
    import lombok.extern.slf4j.Slf4j;
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    import org.springframework.stereotype.Controller;
    import org.springframework.web.bind.annotation.RequestMapping;

    @Controller
    @Slf4j //Logger 사용 가능
    public class HomeController {

        @RequestMapping("/")
        public String home() {
            log.info("home controller");
            return "home"; //home.html로 이동
        }
    }
    ```

<br/>

- 스프링 부트 타임리프 기본 설정 : `build.gradle`
```gradle
spring:
  thymeleaf:
    prefix: classpath:/templates/
    suffix: .html
```

<br/>

<br/>

## 회원 등록
- 폼 객체를 사용해서 화면 계층과 서비스 계층을 명확하게 분리한다.

<br/>

#### 회원 등록 폼 객체
- `MemberForm`
    ```java
    import jakarta.validation.constraints.NotEmpty;
    import lombok.Getter;
    import lombok.Setter;

    @Getter @Setter
    public class MemberForm {

        @NotEmpty(message = "회원 이름은 필수입니다.")
        private String name;

        private String city;
        private String street;
        private String zipcode;
    }
    ```

<br/>

#### 회원 등록 컨트롤러
- 알아두기
  - 홈에서 `회원 등록` 버튼을 누르면 `@GetMapping("/members/new")`으로 이동한다.
    - 이후, `createForm()`에서는 memberForm 객체를 `createMemberForm.html`로 싣어 보낸다.
  - 회원 등록 요청은 `@PostMapping("/members/new")`으로 들어온다.
    - `@Valid`를 이용해 MemberForm에서 에러가 없는지 검사한다. ex) `@NotEmpty` 등등
    - 회원 등록이 이상없이 마무리되면 `return "redirect:/";`을 통해 홈으로 이동한다.

<br/>

- `MemberController`
    ```java
    import jakarta.validation.Valid;
    import jpabook.jpashop.domain.Address;
    import jpabook.jpashop.domain.Member;
    import jpabook.jpashop.service.MemberService;
    import lombok.RequiredArgsConstructor;
    import org.springframework.stereotype.Controller;
    import org.springframework.ui.Model;
    import org.springframework.validation.BindingResult;
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.PostMapping;

    import java.util.List;

    @Controller
    @RequiredArgsConstructor
    public class MemberController {

        private final MemberService memberService;

        @GetMapping("/members/new")
        public String createForm(Model model) {
            model.addAttribute("memberForm", new MemberForm());
            return "members/createMemberForm"; //이 html로 이동
        }

        @PostMapping("/members/new")
        public String create(@Valid MemberForm form, BindingResult result) {    
            //@Valid -> @NotEmpty와 같은 기능을 검사

            if (result.hasErrors()) {
                return "members/createMemberForm";
            }

            Address address = new Address(form.getCity(), form.getStreet(), form.getZipcode());

            Member member = new Member();
            member.setName(form.getName());
            member.setAddress(address);

            memberService.join(member); //저장
            return "redirect:/"; //저장이 끝나면 홈으로 이동
        }
    }
    ```

<br/>

<br/>

## 회원 목록 조회
- 기능 설명
  - 조회한 상품을 뷰에 전달하기 위해 스프링 MVC가 제공하는 모델(`Model`) 객체에 보관한다.
  - 실행할 뷰 이름을 반환

<br/>

- 알아두기
  - `findMembers()`의 결과 리스트를 Member 타입으로 받고 있다.
  - 나중에 API를 작성할 땐, 절대 엔티티 타입을 외부로 반환하지말고, DTO를 이용하자.
    - 비밀번호나 중요 정보가 함께 노출될 수 있기 때문!

<br/>

- `MemberController`에 조회 로직 추가
    ```java
    @GetMapping("/members")
    public String list(Model model) {
        //사실 조회에도 아래처럼 Member 엔티티를 직접 사용하기보단
        //DTO를 사용하는게 좋음!
        //나중에 말하겠지만 API를 만들땐, 절대 엔티티를 외부로 반환하지 말것.
        List<Member> members = memberService.findMembers();
        model.addAttribute("members", members);
        return "members/memberList";
    }
    ```

<br/>

> ***실무에서 엔티티는 핵심 비즈니스 로직만 가지고 있고, 화면을 위한 로직은 없어야 한다.*** 
>
> 요구사항이 정말 단순할 때는 폼 객체(`MemberForm`)없이 엔티티(`Member`)를 직접 등록 및 수정 화면에서 사용해도 된다. 
>
> 하지만 화면 요구사항이 복잡해지기 시작하면, 엔티티에 화면을 처리하기 위한 기능이 점점 증가한다.
>
> → 결과적으로 엔티티는 점점 화면에 종속적으로 변하고, 이렇게 화면 기능 때문에 지저분해진 엔티티는 결국 유지보수하기 어려워진다.

<br/>

<br/>