# 회원 도메인 개발

## 회원 리포지토리 개발
```java
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import jpabook.jpashop.domain.Member;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public class MemberRepository {

    @PersistenceContext
    private EntityManager em;

    public void save(Member member) {
        em.persist(member);
    }

    public Member findOne(Long id) {
        return em.find(Member.class, id);
    }

    public List<Member> findAll() {
        return em.createQuery("select m from Member m", Member.class).getResultList();
    }

    public List<Member> findByName(String name) {
        return em.createQuery("select m from Member m where m.name = :name", Member.class)
                .setParameter("name", name)
                .getResultList();
    }
}
```

</br>

> **기술 설명**

- `@Repository` : 스프링 빈으로 등록, JPA 예외를 스프링 기반 예외로 예외 변환
- `@PersistenceContext` : 엔티티 매니저(`EntityManager`) 주입
- `@PersistenceUnit` : 엔티티 메니터 팩토리(`EntityManagerFactory`) 주입

</br>

> **참고하자**

- `persist()` : 영속성 컨텍스트에 객체를 넣고, 나중에 트랜잭션이 커밋될 때 DB에 반영된다.
- `find()` : 한 건을 조회할 때 사용하고, `(타입, PK)`를 인자로 갖는다.
- JPQL : SQL과 문법이 다르고, `FROM`의 대상이 테이블이 아닌 엔티티이다.
- 여러 건을 조회할 때는 쿼리를 이용하자!

</br>

## 회원 서비스 개발
```java
import jpabook.jpashop.domain.Member;
import jpabook.jpashop.repository.MemberRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class MemberService {

    private final MemberRepository memberRepository;

    /**
     * 회원 가입
     */
    @Transactional
    public Long join(Member member) {
        validateDuplicateMember(member);
        memberRepository.save(member);
        return member.getId();
    }

    private void validateDuplicateMember(Member member) {
        // EXCEPTION
        List<Member> findMembers = memberRepository.findByName(member.getName());
        if (!findMembers.isEmpty()) {
            throw new IllegalStateException("이미 존재하는 회원입니다.");
        }
    }

    // 회원 전체 조회
    public List<Member> findMembers() {
        return memberRepository.findAll();
    }

    public Member findOne(Long memberId) {
        return memberRepository.findOne(memberId);
    }
}
```

</br>

> **기술 설명**

- `@Transactional` : 트랜잭션, 영속성 컨텍스트
    - `readOnly=true` : 데이터의 변경이 없는 읽기 전용 메서드에 사용하고, 약간의 성능 향상이 있을 수 있다.
    - 읽기 전용이 아닌 메서드에는 `@Transactional`을 다시 따로 추가하면 된다.
- `@Autowired`
    - 생성자 주입 많이 사용, 생성자가 하나면 생략 가능

</br>

> **참고하자**

- 스프링 데이터 JPA를 사용하면 `EntityManager` 도 `@Autowired` 로 주입 가능하다.
- 실무에서는 검증 로직이 있어도 멀티 쓰레드 상황을 고려해서 회원 테이블의 회원명 컬럼에 유니크
제약 조건을 추가하는 것이 안전하다.
- 필드 주입 대신, 생성자 주입을 사용하자!

</br>

## 회원 기능 테스트
```java
import jpabook.jpashop.domain.Member;
import jpabook.jpashop.repository.MemberRepository;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.transaction.annotation.Transactional;

import static org.junit.jupiter.api.Assertions.*;

@RunWith(SpringRunner.class)
@SpringBootTest
@Transactional
public class MemberServiceTest {

    @Autowired MemberService memberService;
    @Autowired MemberRepository memberRepository;

    @Test
    public void 회원가입() throws Exception {
        // given
        Member member = new Member();
        member.setName("kim");

        // when
        Long saveId = memberService.join(member);

        // then
        assertEquals(member, memberRepository.findOne(saveId));
    }

    @Test(expected = IllegalStateException.class)
    public void 중복_회원_예외() throws Exception {
        // given
        Member member1 = new Member();
        member1.setName("kim");

        Member member2 = new Member();
        member2.setName("kim");

        // when
        memberService.join(member1);
        memberService.join(member2);

        // then
        fail("예외가 발생해야 한다.");
    }
}
```

</br>

> **기술 설명**

- `@RunWith(SpringRunner.class)` : 스프링과 테스트 통합
- `@SpringBootTest` : 스프링 부트 띄우고 테스트(이게 없으면 `@Autowired` 다 실패)
- `@Transactional` : 반복 가능한 테스트 지원, 각각의 테스트를 실행할 때마다 트랜잭션을 시작하고, **테스트가 끝나면 트랜잭션을 강제로 롤백** (이 어노테이션이 테스트 케이스에서 사용될 때만 롤백)

</br>

> **참고하자**
- `@Test(expected = IllegalStateException.class)` : `try-catch`문 대신 사용이 가능하다!
- `fail()` : 최후의 방지!

</br>

## 테스트 케이스에 대한 추가 정보

</br>

> **테스트 케이스를 위한 설정**
 
- 테스트는 케이스 격리된 환경에서 실행하고, 끝나면 데이터를 초기화하는 것이 좋다.
    - 그런 면에서 메모리 DB를 사용하는 것이 가장 이상적이다.

- 추가로 테스트 케이스를 위한 스프링 환경과, 일반적으로 애플리케이션을 실행하는 환경은 보통 다르므로 설정 파일을 다르게 사용하면 좋다.

- 테스트 디렉토리에 추가적으로 설정 파일을 생성하면 된다.

</br>

-> 스프링 부트는 datasource 설정이 없으면, 기본적으로 메모리 DB를 사용하고, driver-class도 현재 등록된 라이브러리를 보고 찾아주므로 별도의 추가 설정을 하지 않아도 된다.