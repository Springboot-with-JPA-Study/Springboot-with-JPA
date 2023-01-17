## 회원 도메인 개발

#### 회원 리포지토리
- 기술 설명
    - `@Repository` : 스프링 빈으로 등록, JPA 예외를 스프링 기반 예외로 예외 변환 
    - `@PersistenceContext` : 엔티티 매니저(EntityManager) 주입 
    - `@PersistenceUnit` : 엔티티 매니저 팩토리(EntityManagerFactory) 주입

<br/>

- 알아두기
    - `findAll()` 과 같이 여러 결과를 검색하는 기능은 쿼리를 작성하기
    - SQL과 JPQL은 약간의 문법이 다르다
      - from의 대상이 테이블이 아닌 엔티티임!  JPA 책 참고하기.

<br/>

- 레포지토리 코드
    ```java
    import jpabook.jpashop.domain.Member;
    import org.springframework.stereotype.Repository;
    import javax.persistence.EntityManager;
    import javax.persistence.PersistenceContext;
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
            return em.createQuery("select m from Member m", Member.class)
                    .getResultList();
        }
        
        public List<Member> findByName(String name) {
            return em.createQuery("select m from Member m where m.name = :name", Member.class)
                    .setParameter("name", name)
                    .getResultList();
        } 
    }
    ```

<br/>

<br/>

#### 회원 서비스
- 기술 설명
    - `@Service`
    - `@Transactional` : 트랜잭션, 영속성 컨텍스트 
    - `readOnly=true` : 데이터의 변경이 없는 읽기 전용 메서드에 사용
      - 데이터베이스 드라이버가 지원하면 DB에서 성능 향상
    - `@Autowired` : 생성자 Injection 많이 사용, 생성자가 하나면 생략 가능
    - `@RequiredArgsConstructor` : `@AllArgsConstructor`와는 다르게 final 필드만 생성

<br/>

- 알아두기 
  - 이 예제 코드의 경우 읽기 작업이 많으므로 `Transactional`의 기본 값을 readOnly로 설정
  - `@Autowired`를 이용한 필드 주입 말고, 생성자 주입을 사용하자.
  - 실무에서는 검증 로직이 있어도 멀티 쓰레드 상황을 고려해서 회원 테이블의 회원명 컬럼에 유니크 제약 조건을 추가하는 것이 안전하다.


<br/>

- 서비스 코드
    ```java
    import jpabook.jpashop.domain.Member;
    import jpabook.jpashop.repository.MemberRepository;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Transactional;
    import java.util.List;

    @Service
    @Transactional(readOnly = true)
    @RequiredArgsConstructor
    public class MemberService {
        //@Autowired
        MemberRepository memberRepository;

        /**
        * 회원가입
        */
        @Transactional //변경
        public Long join(Member member) { 
            validateDuplicateMember(member); //중복 회원 검증
            memberRepository.save(member);
            return member.getId();
        }
        
        private void validateDuplicateMember(Member member) {
            List<Member> findMembers = memberRepository.findByName(member.getName());
            if (!findMembers.isEmpty()) {
                throw new IllegalStateException("이미 존재하는 회원입니다."); }
        }

        /**
        *전체 회원 조회
        */
        public List<Member> findMembers() {
            return memberRepository.findAll();
        }
            
        public Member findOne(Long memberId) {
            return memberRepository.findOne(memberId);
        } 
    }
    ```

<br/>

<br/>

#### 회원 기능 테스트
- 기술 설명
  - `@RunWith(SpringRunner.class)` : 스프링과 테스트 통합
  - `@SpringBootTest` : 스프링 부트 띄우고 테스트(이게 없으면 `@Autowired` 다 실패) 
  - `@Transactional` : 반복 가능한 테스트 지원, 각각의 테스트를 실행할 때마다 트랜잭션을 시작하고 **테스트가 끝나면 트랜잭션을 강제로 롤백** (이 어노테이션이 테스트 케이스에서 사용될 때만 롤백)

<br/>

- 알아두기
  - `@Test(expected = IllegalStateException.class)`
    - 이 예외를 기대한다(이 예외가 발생해야 옳은 테스트)는 의미이다.
  - `fail("예외가 발생해야 한다.");`
    - 여기에 도달하면 안된다라는 의미이다.


<br/>

- 테스트 코드
    ```java
    import jpabook.jpashop.domain.Member;
    import jpabook.jpashop.repository.MemberRepository;
    import org.junit.Test;
    import org.junit.runner.RunWith;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.boot.test.context.SpringBootTest;
    import org.springframework.test.context.junit4.SpringRunner;
    import org.springframework.transaction.annotation.Transactional;
    import static org.junit.Assert.assertEquals;
    import static org.junit.Assert.fail;

    @RunWith(SpringRunner.class)
    @SpringBootTest
    @Transactional
    public class MemberServiceTest {
        @Autowired MemberService memberService;
        @Autowired MemberRepository memberRepository;
        
        @Test
        public void 회원가입() throws Exception {
            //Given
            Member member = new Member();
            member.setName("kim");

            //When
            Long saveId = memberService.join(member);

            //Then
            assertEquals(member, memberRepository.findOne(saveId));
        }

        @Test(expected = IllegalStateException.class) public void 중복_회원_예외() throws Exception {
            //Given
            Member member1 = new Member();
            member1.setName("kim");
            Member member2 = new Member();
            member2.setName("kim");

            //When
            memberService.join(member1); memberService.join(member2); //예외가 발생해야 한다.

            //Then
            fail("예외가 발생해야 한다."); 
        }
    }
    ```