# 2. 테스트

<aside>
<img src="/icons/book-closed_gray.svg" alt="/icons/book-closed_gray.svg" width="40px" />

- 테스트란
- 테스트의 가치와 장점, 활용 전략, 스프링과의 관계
- 대표적인 테스트 프레임워크, 이를 활용한 학습 전략
</aside>

# 2.1 UserDaoTest 다시 보기

## 2.1.1 테스트의 유용성

- 테스트
    - 예상하고 의도했던 대로 코드가 정확히 동작하는지를 확인 → 만든 코드를 확신
    - 코드나 설계에 결함이 있는지 확인 가능
    - 테스트가 성공하면 모든 결함 제거됐다는 확신

## 2.1.2 UserDaoTest의 특징

- 웹을 통한 DAO 테스트 방법의 문제점
    - 서비스 클래스, 컨트롤러, JSP 뷰 등 모든 레이어의 기능을 다 만들고 테스트 가능
    - 어디서 문제 발생했는지 찾아야 함 → 빠르고 정확하게 대응 힘듦

- 작은 단위의 테스트
    - 관심사의 분리 → 단위 테스트(unit test)
        - 일반적으로 단위는 작을수록 좋음
        - 개발자 테스트, 프로그래머 테스트
    - 과정을 묶어서 테스트할 필요도 있음(ex. 등록 - 로그인 - 기능 사용 - 로그아웃) : 단위 테스트 후에
- 자동수행 테스트 코드
    - 테스트는 자동으로 수행되어야 함
- 지속적인 개선과 점진적인 개발을 위한 테스트
    - 가장 단순한 기능(등록, 조회) 만들고 이를 테스트 → 기능 추가 후 테스트 함께 추가
    - 기존 기능이 새 기능에 영향 받지 않고 잘 동작하는지 확인 가능

## 2.1.3 UserDaoTest의 문제점

- 수동 확인 작업의 번거로움
    - 사람의 눈으로 확인하는 과정 : DB 등록 후 갖져왔을 때 값이 일치하는지 여부
- 실행 작업의 번거로움
    - `main()` 메소드보다 편리하고 체계적으로 실행 및 확인 방법 필요

# 2.2 UserDaoTest 개선

## 2.2.1 테스트 검증의 자동화

- 테스트의 결과
    - 성공
    - 실패
        - 테스트 에러 : 테스트가 진행되는 동안에 에러 발생
        - 테스트 실패 : 결과가 기대와 다르게 나오는 경우
- 테스트 실패 확인 자동화를 위한 코드 수정
    
    ```java
    if (!user.getName().equals(user2.getName())){
        System.out.println("테스트 실패 (name)");
    }
    else if (!user.getPassword().equals(user2.getPassword())){
        System.out.println("테스트 실패 (password)");
    }
    else {
        System.out.println("조회 테스트 성공");
    }
    ```
    
- 기존 코드 수정 확인 빠르게 가능

## 2.2.2 테스트의 효율적인 수행과 결과 관리

- `main()`  메소드를 이용한 테스트 → 규모가 커지고 테스트 개수 많아질 시 부담

### JUnit 테스트로 전환

- JUnit : 프레임워크 → main() 메소드, 오브젝트를 만들어서 실행하는 코드 필요없음

### 테스트 메소드 전환

- 테스트 메소드에 JUnit 프레임워크가 요구하는 조건
    - public으로 선언
    - `@Test`  애노테이션

```java
@Test
public void addAndGet() throws SQLException{
    ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");

    UserDao dao = context.getBean("userDao", UserDao.class);
}
```

### 검증 코드 전환

```java
asserThat(user1.getPassword(), is(user.getPassword()));
```

- `asserThat()`  : 첫 번째 파라미터 값을 뒤에 나오는 매처 조건으로 비교, 일치하면 다음으로 아니면 실패
    - `is()`  : 매처의 일종, `equals()` 로 비교

### JUnit 테스트 실행

- JUnit 실행 → `main()` 메소드 추가, 그 안에 JUnitCore 클래스의 main 메소드 호출하는 코드 넣기
    
    ```java
    import org.junit.runner.JUnitCore;
    
    public static void main(String[] args){
        JUnitCore.main("springbook.user.dao.UserDatTest");
    }
    ```
    
- 총 테스트 중 몇 개의 테스트 실패했는지, 실패 원인과 위치 확인 가능
- 테스트 실패 시 `AssertionError`

# 2.3 개발자를 위한 테스팅 프레임워크 JUnit

## 2.3.1 JUnit 테스트 실행 방법

- 자바 IDE 내장 JUnit 테스트 지원 도구 사용

### IDE

- 이클립스 기준 설명
- `@Test`  들어 있는 클래스 선택 → run 메뉴 → Run As → JUnit Test 선택
    - JUnitCore를 이용할 때처럼 main 메소드 만들지 않아도 됨
- 테스트의 총 수행시간, 실행한 테스트의 수, 테스트 에러의 수, 테스트 실패의 수, 어떤 테스트 클래스 실행했는지 등 확인 가능
- 한 번에 여러 테스트 클래스 동시에 실행 가능
    - 특정 패키지 선택 → 컨텍스트 → Run As → JUnit Test

### 빌드 툴

- 빌드 툴에서 제공하는 플러그인이나 태스크 이용
- 실행 결과는 옵션에 따라 HTML이나 텍스트 파일 형태로 가능
- 여러 개발자가 만든 코드 통합할 경우, 서버에서 빌드 → 빌드 스크립트를 통해 테스트 실행 → 메일을 통해 결과 통보 방식

## 2.3.2 테스트 결과의 일관성

- 코드에 변경사항이 없다면 테스트는 항상 동일한 결과

### deleteAll()의 getCount() 추가

- UserDao에 기능 추가
- `deleteAll()`
    - USER 테이블의 모든 레코드 삭제
- `getCount()`
    - USER 테이블의 레코드 개수 반환

### deleteAll()과 getCount()의 테스트

- `deleteAll()` 을 이용해 테이블 내용 삭제, `getCount()` 로 검증

```java
dao.deleteAll()
assertThat(dao.getCount(), is(0))
// 유저 추가 코드
assertThat(dao.getCount(), is(1))
// 검증 코드 
```

### 동일한 결과를 보장하는 테스트

- 테스트 전 상태로 만들어
- 단위 테스트는 항상 일관성 있는 결과가 보장되어야 함
    - DB 데이터 등 외부 환경에 영향을 받지 말아야 함
    - 테스트 순서 변경해도 동일한 결과

## 2.3.2 포괄적인 테스트

### getCount() 테스트

- 세 개의 오브젝트 생성 후 하나씩 추가하면서 결과 확인
- 테스트 실행 순서에 독립적으로 항상 동일한 결과 낼 수 있도록

### addAndGet() 테스트 보완

- 두 개의 User add 후 id로 정보 가져오기 → 해당하는 사용자 가져온 것인지 확인 가능

### get() 예외조건에 대한 테스트

- id 값에 해당하는 사용자가 없을 경우 → null과 같은 특별한 값 리턴 / id에 해당하는 정보를 찾을 수 없다고 예외
    - 후자의 경우 → 일단 미리 정의한 예외 `EmptyResultDataAccessException`  사용
- 메소드 예외상황에 대한 테스트
    - 특정 예외가 던져지면 테스트 성공, 던져지지 않을 경우 테스트 실패
    
    ```java
    @Test(expected=EmptyResultDataAccessException.class)
    public void getUserFailure() throws SQLException {
        ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");
    
        UserDao dao = context.getBean("userDao", UserDao.class);
        dao.deleteAll()
        assertThat(dao.getCount(), is(0));
    
        dao.get("unknown_id");
    }
    ```
    
    - `@Test` 의 expected 엘리먼트 : 테스트 메소드 실행 중에 발생하리라 기대되는 예외 클래스

### 테스트를 성공시키기 위한 코드의 수정

- 현재 예상한 예외가 아닌 `SQLException`  발생으로 실패 → 아래처럼 수정 필요

```java
if (user==null) throw new EmptyResultDataAccessException(1);
```

### 포괄적인 테스트

- 다양한 상황과 입력 값을 고려하는 포괄적인 테스트 작성 필요
- 네거티브 테스트부터 만들기

## 2.3.4 테스트가 이끄는 개발

### 기능설계를 위한 테스트

- 추가하고 싶은 기능을 테스트 코드로 표현 → 코드 작성 → 테스트 빠르게 검증

### 테스트 주도 개발

- 테스트 주도 개발(TDD, Test Driven Development) : 테스트 코드를 먼저 만들고, 테스트를 성공하게 해주는 코드를 작성하는 방식의 개발 방법
    - 테스트 우선 개발
- 테스트를 빼먹지 않고 꼼꼼하게 만들어낼 수 있음
- 테스트를 작성하는 시간 - 애플리케이션 코드 작성 시간 간격 짧아짐 → 빠른 피드백 가능
- 테스트 작성 - 코드 작성 주기를 최대한 짧게 가져가도록 권장

## 2.3.5 테스트 코드 개선

### @Before, @After

- 중복되는 코드 분리
- 테스트 메소드를 실행하기 전에 먼저 실행

```java
private UserDao dao;

@Before
public void setUp(){
    ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");
    this.dao = context.getBean("userDao", UserDao.class);
}
```

- `@Test` 가 붙은 메소드 실행 전, 후에 각각 `@Before` 와 `@After` 가 붙은 메소드 자동 실행
- 서로 주고받을 정보나 오브젝트가 있다면 인스턴스 변수 사용 필요
- 테스트 메소드를 실행할 때마다 테스트 클래스의 오브젝트 새로 생성
    - 각 테스트가 서로 독립적으로 실행됨을 보장해주기 위함
- 일부 테스트 메소드에서만 공통적으로 사용 → 일반적인 메소드 추출 방법 사용

### 픽스처

- 픽스처 : 테스트를 수행하는 데 필요한 정보나 오브젝트
    - 일부 메소드에서 사용하지 않더라도 `@Before`  메소드 이용하여 생성하면 편리
    - user1, user2, user3

# 2.4 스프링 테스트 적용

- 애플리케이션 컨텍스트처럼 생성에 많은 시간과 자원이 소모되는 경우 테스트 전체가 공유하는 오브젝트 생성 → 스태틱 메소드  `@BeforeClass`  사용하여 변수에 저장해두고 사용

## 2.4.1 테스트를 위한 애플리케이션 컨텍스트 관리

- 테스트 컨텍스트 프레임워크 → 애노테이션 설정으로 필요한 애플리케이션 컨텍스트 생성 후 모든 테스트가 공유 가능

### 스프링 테스트 컨텍스트 프레임워크 적용

- `@Autowired` , `@RunWith` , `@ContextConfiguration`
    - `@Autowired` : 테스트 오브젝트가 만들어지고 나면 스프링 테스트 컨텍스트에 의해 자동으로 값 주입
    - `@RunWith`  : 스프링의 테스트 컨텍스트 프레임워크의 JUnit 확장기능 지정
        - `SpringJUnit4ClassRunner`  JUnit용 테스트 컨텍스트 프레임워크 확장 클래스 지정 시 JUnit 테스트 진행 중 테스트가 사용할 애플리케이션 컨텍스트를 만들고 관리하는 작업 진행
    - `@ContextConfiguration`  : 테스트 컨텍스트가 자동으로 만들어줄 애플리케이션 컨텍스트의 위치 지정
    
    ```java
    @RunWith(SpringJUnit4ClassRunner.class)
    @ContextConfiguration(locations="/applicationContext.xml")
    public class UserDaoTest {
        @Autowired
        private ApplicationContext context;
    
        @Before
        public void setUp() {
            this.dao = this.context.getBean("UserDao", UserDao.class);
        }
    }
    ```
    

### 테스트 메소드의 컨텍스트 공유

- 콘솔에 출력하여 확인 시 여러 UserDaoTest 오브젝트에서 하나의 애플리케이션 컨텍스트가 만들어져 사용됨
- 처음 실행 시 최초로 애플리케이션 컨텍스트가 만들어지고 재사용되기 때문에 첫 테스트 클래스에서 가장 오랜 시간 소모

### 테스트 클래스의 컨텍스트 공유

- 같은 설정파일을 공유할 경우 하나의 애플리케이션 컨텍스트 공유

### @Autowired

- `@Autowired` 가 붙은 인스턴스 변수가 있으면, 테스트 컨텍스트 프레임워크는 변수 타입과 일치하는 컨텍스트 빈을 찾아 주입
    - 별도의 DI 설정 없이 필드의 타입정보를 이용해 빈을 가져옴 → 타입에 의한 자동 와이어링
- 컨텍스트에서 getBean()하는 것이 아닌 UserDao 빈을 직접 DI 받을 수 있음
    
    ```java
    public class UserDaoTest {
        @Autowired
        UserDao dao;
    }
    ```
    
- 같은 타입의 빈이 두 개 이상 있는 경우 변수의 이름과 같은 이름의 빈이 있는지 확인

## 2.4.2 DI와 테스트

- 인터페이스를 두고 DI를 적용해야  하는 이유
- 소프트웨어 개발에서 절대 바뀌지 않는 것은 없음
- 클래스의 구현 방식은 바뀌지 않더라도 인터페이스를 두고 DI를 적용하게 해두면 다른 차원의 서비스 기능을 도입할 수 있음
    - 1장의 DB 커넥션 개수 카운팅 기능
- 테스트
    - 효율적인 테스트를 쉽게 만들 수 있음

### 테스트 코드에 의한 DI

- 테스트용 DataSource 생성 후 코드에 의한 수동 DI
- 설정파일 수정하지 않고도 코드를 통해 오브젝트 관계 재구성 가능 → 바람직하지 못한 방법 → `@DirtiesContext`
    - 스프링의 테스트 컨텍스트 프레임워크에게 해당 클래스의 테스트에서 애플리케이션 컨텍스트의 상태 변경을 알려줌
    - 테스트 컨텍스트는 이 애노테이션이 붙은 테스트 클래스에는 애플리케이션 컨텍스트 공유를 허용하지 않음

### 테스트를 위한 별도의 DI 설정

- 테스트에서 사용될 테스트 전용 설정파일을 따로 설정
- 기존 applicationContext.xml에서 사용할 DB만 변경하여 저장 → `@ContextConfiguration`의 locations 엘리먼트 값만 변경

### 컨테이너 없는 DI 테스트

- 애플리케이션 컨텍스트 사용하지 않고 테스트 코드에서 직접 오브젝트를 만들고 DI 해서 사용

### DI를 이용한 테스트 방법 선택

- 수동 설정 / 별도의 DI 설정 / 컨테이너 사용x
- 스프링 컨테이너 없이 테스트할 수 있는 방법 우선적으로 고려
- 여러 오브젝트와 복잡한 의존 관계 가진 오브젝트 테스트 → 스프링의 설정을 이용한 DI 방식
- 예외적인 의존관계를 강제로 구성해서 테스트 → 수동 DI 방식

# 2.5 학습 테스트로 배우는 스프링

- 학습 테스트 : 자신이 만들지 않은 프레임워크나 다른 개발팀에서 만들어서 제공한 라이브러리에 대한 테스트
    - 자신이 사용할 API나 프레임워크의 기능을 테스트로 보면서 사용방법 학습
    - 테스트 대상보다는 테스트 코드 자체에 관심을 가지고 작성

## 2.5.1 학습 테스트의 장점

- 다양한 조건에 따른 기능 손쉽게 확인 가능
- 학습 테스트 코드를 개발 중에 참고 가능
- 프레임워크나 제품을 업그레이드할 때 호환성 검증을 도와줌
- 테스트 작성에 대한 좋은 훈련
- 새로운 기술을 공부하는 과정이 즐거워짐
- 

## 2.5.2 학습 테스트 예제

### JUnit 테스트 오브젝트 테스트

- 스태틱 변수에 저장한 테스트 오브젝트와 매 테스트 메소드의 자신과 비교하여 같지 않다는 사실 확인 : `@sameInstance` , `not()`
- 스태틱 변수로 컬렉션 만들고 테스트마다 컬렉션 확인, 없으면 자기 자신 추가 : `hasItem()`

### 스프링 테스트 컨텍스트 테스트

- 같은 테스트 컨텍스트를 오브젝트인지 확인
    - `@RunWith`, `@ContextConfiguration`, `@Autowired` 로 애플리케이션 컨텍스트 주입 후 테스트 컨텍스트가 같은 애플리케이션 컨텍스트를 주입해주는지 확인

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("junit.xml")
public class JUnitTest {
	@Autowired ApplicationContext context;
	
	static Set<JUnitTest> testObjects = new HashSet<JUnitTest>();
	static ApplicationContext contextObject = null;
	
	@Test public void test1() {
		assertThat(testObjects, not(hasItem(this)));
		testObjects.add(this);
		
		assertThat(contextObject == null || contextObject == this.context, is(true));
		contextObject = this.context;
	}
	
	@Test public void test2() {
		assertThat(testObjects, not(hasItem(this)));
		testObjects.add(this);
		
		assertTrue(contextObject == null || contextObject == this.context);
		contextObject = this.context;
	}
	
	@Test public void test3() {
		assertThat(testObjects, not(hasItem(this)));
		testObjects.add(this);
		
		assertThat(contextObject, either(is(nullValue())).or(is(this.contextObject)));
		contextObject = this.context;
	}
}
```

- `@assertTrue()`, `nullValue()` , `either`

## 2.5.3 버그 테스트

- 버그 테스트 : 코드에 오류가 있을 때 그 오류를 가장 잘 드러내줄 수 있는 테스트
- 테스트의 완성도를 높임
- 버그의 내용을 명확하게 분석하게 해줌
- 기술적인 문제를 해결하는데 도움

- 동등분할
    - 같은 결과를 내는 값의 범위 구분 → 대표값으로 테스트
- 경계값 분석
    - 동등분할의 경계에서 에라 많이 발생 → 경계값으로 테스트 (0, 최대값, 최소값 등)