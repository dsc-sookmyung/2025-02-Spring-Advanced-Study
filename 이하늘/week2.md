# 2장 테스트

## 2.1 UserDaoTest 다시보기
- 테스트는 코드가 예상대로 정확히 동작하는지 검증함.  
- 테스트 결과가 기대와 다르면 코드나 설계에 결함이 있음을 알려줘 디버깅 과정을 도와줌.  
- 테스트는 작은 단위로 분리해서 **자동화된 코드**로 작성해야 함. (단위 테스트)
  → 웹을 통한 DAO 테스트 처럼 진행하면 어느 레이어에서 잘못됐는지 한 눈에 확인하기 어려움.  
  → 작은 단위로 분리해 자동으로 수행시키면 테스트를 자주 반복 실행하여 기능이 안정적으로 유지되는지 빠르게 확인할 수 있음.  
- 지속적으로 개발할 때, 새로운 기능을 추가한 후에도 기존 기능이 영향을 받지 않는지 검증할 수 있음.  

```java
public class UserDaoTest {
    public static void main(String[] args) throws SQLException {
        ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");
        UserDao dao = context.getBean("userDao", UserDao.class);

        User user = new User();
        user.setId("user");
        user.setName("백기선");
        user.setPassword("married");
        dao.add(user);

        System.out.println(user.getId() + " 등록 성공");

        User user2 = dao.get(user.getId());
        System.out.println(user2.getName());
        System.out.println(user2.getPassword());
        System.out.println(user2.getId() + " 조회 성공");
    }
}
```

**문제점**  
- 위 방식은 `main()`으로 직접 결과를 확인해야 하므로 테스트가 완전히 자동이 아님.  
- DAO 수가 많아지면 main() 메소드를 여러 번 실행해야 하는 번거로움이 있음.  

---

## 2.2 UserDaoTest 개선
- **JUnit 전환**: 기존 main() 테스트를 JUnit 테스트 메소드로 전환함.  
  - `@Test` 애노테이션을 붙이고 `public void` 형식이어야 함.  
  - (JUnit5부터는 public이 아니어도 테스트로 동작함)  

- **테스트 자동화**: JUnit은 각 테스트 메소드 실행 시마다 새로운 테스트 클래스 인스턴스를 생성하고,  
  `@Before`/`@After`로 공통 준비·정리 작업을 자동 처리함.  

- **검증 코드**: `assertThat()` 등의 검증 메소드로 결과를 확인.  
  - 조건이 맞지 않으면 `AssertionError` 발생 → 테스트 실패.  
  - AssertJ 사용 시 가독성이 좋음:  
    ```java
    assertThat(actual).isEqualTo(expected);
    ```

```java
public class UserDaoTest {
    @Test
    public void addAndGet() throws Exception {
        ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");
        UserDao dao = context.getBean("userDao", UserDao.class);
    }
}
```

---

## 2.3 개발자를 위한 테스팅 프레임워크 JUnit
- **일관된 결과 보장**: 단위 테스트는 항상 동일한 결과를 보여야 함.  
  → DB나 외부 환경에 의존하지 않도록 설정 필요.  

- **포괄적 테스트**:  
  - 테스트 메소드는 **한 가지 동작만 검증**하도록 작성.  
  - 부정적 케이스도 포함해야 함.  
  - 예: 존재하지 않는 ID 조회 시 예외 발생 여부 확인.  

- **테스트 주도 개발 (TDD)**:  
  - 테스트 코드는 기능 정의서 역할을 함.  
  - **실패하는 테스트 → 코드 구현 → 테스트 통과** 순서로 개발.  

- **공통 준비/정리**  
  - `@BeforeEach`(`@Before`) : 각 테스트 실행 전 실행.  
  - `@BeforeAll`(`@BeforeClass`) : 전체 테스트 전에 한 번만 실행.  

- **JUnit 실행 흐름**:  
  - `@Test` 메소드 탐색 → 테스트 객체 생성 → `@Before → @Test → @After` 실행.  
  - 모든 테스트 실행 후 리포트 제공.  

---

## 2.4 스프링 테스트 적용
- **스프링 테스트 컨텍스트**:  
  - `@RunWith(SpringJUnit4ClassRunner.class)`  
  - `@ContextConfiguration`  
  → 애플리케이션 컨텍스트를 자동 생성·관리.  

- **컨텍스트 공유**:  
  - 같은 클래스의 모든 테스트는 **하나의 컨텍스트 공유**.  
  - 같은 설정 파일을 쓰는 다른 테스트 클래스도 공유.  
  - → 테스트 성능 대폭 향상.  

- **의존성 주입**:  
  - `@Autowired` 필드로 컨텍스트 내 빈 주입 가능.  
  - ApplicationContext도 DI 가능.  

- **인터페이스 DI 권장**:  
  - 구현체 대신 Mock이나 대체 빈 주입 가능.  
  - → 작고 독립적인 단위 테스트 작성 용이.  

- **테스트 전용 설정**:  
  - 운영 DB 대신 **메모리 DB** 같은 테스트용 DataSource 등록.  
  - → 빠르고 가벼운 테스트 가능.  

- **컨테이너 없이 테스트**:  
  - 스프링 API 의존이 없는 클래스는 **순수 자바 코드로 테스트**하는 게 가장 빠름.  

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="applicationContext.xml")
public class UserDaoTest {
    @Autowired
    private ApplicationContext context;
}
```

---

## 2.5 학습 테스트로 배우는 스프링
- **학습 테스트**:  
  - 자신이 직접 만들지 않은 외부 라이브러리/프레임워크 기능을 이해하기 위해 작성.  
  - 목적: 기능 동작을 익히고, 실제 동작 예제를 코드로 남김.  

- **장점**  
  - 다양한 조건에서 기능 동작을 빠르게 확인 가능.  
  - 예제 코드가 샘플 코드로 활용됨.  
  - 프레임워크 업그레이드 시 기존 학습 테스트를 돌려 문제 여부 확인.  
  - 테스트 작성 역량 및 학습 동기 상승.  

- **버그 테스트**:  
  - 오류 발생 시 그 오류를 재현하는 테스트 작성.  
  - 테스트가 통과하도록 코드 수정 → 버그 해결.  
  - 같은 문제 재발 시 빠른 추적 가능.  
  - 오류 원인 분석 및 검증 보완에 도움.  

---

## 요약
- 테스트는 **자동화**하고 **작은 단위**로 검증해야 함.  
- **일관성 있고 포괄적인 테스트**를 작성하여 기능 동작을 명확히 확인.  
- **테스트 주도 개발(TDD)**로 개발 효율 상승.  
- 스프링의 테스트 컨텍스트 활용 → 성능 향상 + 편리한 의존성 주입.  
- 새로운 기술 학습 → 학습 테스트 작성.  
- 버그 발견 → 버그 테스트 작성 후 해결.  
