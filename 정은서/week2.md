# [정은서] 2025 GDG Spring Advanced Study - 2주차
# 토비의 스프링 3.1(vol1) - 2장 테스트 

## 스프링

- IoC 와 DI 를 이용해 객체지향 프로그래밍 언어의 근본과 가치를 손쉽게 적용하고 사용 가능
- 확장과 변화를 고려한 객체지향적 프로그래밍
- 만들어진 코드를 확신할 수 있게 해주고 변화에 유연하게 대처할 수 있는 테스트

## 2.1 UserDao Test 다시보기 

- main() 메소드를 통해 add(), get() 메소드 호출 및 출력 

```java
public class UserDaoTest {
	public static void main(String[] args) throws SQLException {
		ApplicationContext context = new GenericXmlApplicationContext("");
		
		UserDao Dao = context.getBean("UserDao", UserDao.class); // UserDao 직접 호출 
		
		User user= new User();
		user.setId("user");
		user.setName("정은서");
		user.setPassword("eunseo");
		
		dao.add(user);
		
		System.out.println(user.getId() + "등록 성공");
		
		User user2 = dao.get(user.getId());
		System.out.println(user2.getName());
		
		System.out.println(user2.getID() + "조회 성공");
	}
}
```

- 웹을 통한 DAO 테스트 방법의 문제점
    - UserDao 뿐만 아니라 서비스, MVC 까지 다 포함한 기능이 어느정도 있어야 테스트 가능
        - 이런 방법은 UserDao를 테스트하고 싶었던 건데 다른 계층의 코드에 영향을 줄 수 있음
        - 오류가 있을때 빠르게 대응하기 어려움

- 작은 단위의 테스트
    - 관심사의 분리
    - 단위 테스트(unit test) : 작은 단위의 코드에 대해 테스트를 수행한 것

- 자동수행 테스트 코드
    - UserDaoTest : 테스트할 데이터가 코드를 통해 제공되고 테스트 작업 역시 코드를 통해 자동으로 실행한다
    - main() 메소드를 실행해서 User 오브젝트를 만들어 적절한 값을 넣고 연결된 DB 에서 UserDao 오브젝트를 컨테이너에서 가져와서 add() 메소드 호출, get() 메소드 호출 하는 것까지 자동으로 진행
    - 테스트는 자동으로 수행되도록 코드로 만들어지는 것이 중요
    - 별도로 테스트용 클래스를 만들어서 테스트 코드를 넣는 편이 낫다

## 2.2 UserDaoTest 개선

### 2.2.1 테스트 검증의 자동화

- add() 에 전달한 User 오브젝트에 담긴 사용자 정보와 get() 을 통해 다시 가져온 User 정보가 정확히 일치하는지 확인해보자
    - 기존에는 콘솔에 get() 해서 가져온 정보를 직접 확인 가능하도록 출력되도록 함
    - 추가로 기대한 결과와 달라서 실패한 경우에는 실패 메시지 뜨도록 함

```java
//System.out.println(user.getId() + "등록 성공");
		
//User user2 = dao.get(user.getId());
//System.out.println(user2.getName());
		
//System.out.println(user2.getID() + "조회 성공");
		
		
if (!user.getName().equals(user2.getName())) {
    System.out.println("테스트 실패"); // 처음 add() 에 전달한 User와 get() 을 통해 가져오는 User의 값을 비교 
}
else if (!user.getPassword().equals(user2.getPassword())) {
	System.out.println("테스트 실패");
}
else {
	System.out.println("테스트 성공");
}
```

### 2.2.2 테스트의 효율적인 수행과 결과 관리

- main() 메소드를 이용한 테스트 작성 방법 만으로는 에플리케이션 규모가 커지고 테스트 개수가 많아지면 테스트를 수행하는 일이 점점 부담됨
- JUnit → 테스트 지원 도구, 자바 테스팅 프레임워크
    - 프레임 워크(개발자가 만든 클래스에 대한 제어 권한을 넘겨 받아 주도적으로 애플리케이션의 흐름 제어)
    - main() 메소드 필요없고 오브젝트 만들 필요도 없음

- 테스트 메소드 전환
    - 기존의 main() 메소드 테스트는 프레임워크에 적용하기엔 적합하지 않음
    - main() 메소드에 있던 테스트 코드를 일반 메소드로 옮겨야 됨
    - 조건
    1. 메소드가 public으로 선언
    2. @Test 애노테이션 붙이기 
    - if (!user.getName().equals(user2.getName())) → assertThat(user2.getName(), is(user.getName()));
        - 비교해서 일치하면 다음으로 넘어가고 아니면 실패
        - equals 기능

```java
public class UserDaoTest {

	@Test
	public void addAndGet() throws SQLExpection {
		ApplicationContext context = new ClassPathXmlApplicationContext("");
		
		UserDao dao = context.getBean("userDao", UserDao.class);
		User user = new User();
		
		user.setId("eunseo");
		user.setName("정은서");
		user.setPassword("030903");
		
		dao.add(user);
		
		User user2 = dao.get(user.getId());
		
		assertThat(user2.getName(), is(user.getName()));
		assertThat(user2.getPassword(), is(user.getPassword()));
		
	}
}
```
## 2.3 개발자를 위한 테스팅 프레임워크 JUnit

- 테스트 없이는 스프링도 의미가 없음

### 2.3.1 JUnit 테스트 방법 

- 테스트 수가 많아지면 관리하기가 힘들어짐
- 자바 IDE 에 내장된 JUnit 테스트 지원 도구 사용
- IDE, 빌드 툴, 빌드 스크립트

### 2.3.2 테스트 결과의 일관성

- UserDaoTest 를 할 때 실행하기 전에 DB의 User 테이블을 모두 삭제해야 하는 불편함
- 해결책은 addAndGet() 테스트를 마치고 등록한 사용자 정보를 모두 삭제하는 것 → deleteAll() 의 getCount() 추가

- deleteAll()
    - User 테이블의 모든 데이터 삭제

```java
public void deleteAll() {
	Connection c = dataSource.getConnection();
	
	PreparedStatement ps = c.prepareStatement("delete from users");
	
	ps.executeUpdate();
	ps.close();
	c.close();
}
```
- getCount()
    - User 테이블의 레코드 개수를 반환

```java
public int getCount() {
	Connection c = dataSource.getConnection();
	
	PreparedStatement ps = c.prepareStatement("select count(*) from users");
	
	ResultSet rs = ps.executeQuery();
	re.next();
	int count = rs.getInt();
	
	return count;
}
```
- add() 메소드 실행 후 getCount() 메소드 실행하면 1로 바뀌는 것 검증 추가 

```java
@Test
public void addAndGet() {
	...
	
	dao.deleteAll();
	assertThat(dao.getCount(), is(0));

    User user = new User();
	user.setId("eunseo");
	user.setName("정은서");
	user.setPassword("030903");
	
	dao.add(user);
	assertThat(dao.getCount(), is(1));
	
	User user2 = dao.get(user.getId());
	
	assertThat(user2.getName(), is(user.getName()));
	assertThat(user2.getPassword(), is(user.getPassword()));
}
```

→ 이렇게 하면 테스트 하기 전에 매번 직접 DB 에서 데이터를 삭제하지 않아도 됨 

- 테스트 하기 전에 테스트 실행에 문제가 되지 않는 상태를 만들어주기
- 단위 테스트는 항상 일관성 있는 결과가 보장돼야 한다 + 외부 환경에 영향을 받지 않아야 함

### 2.3.3 포괄적인 테스트

- 추가적인 getCount() 테스트
    - 여러 User 를 등록해보며 count 가 잘 반환되는지 보자
    - 테스트 메소드는 한 번에 한 가지 검증 목적에만 충실한 것이 좋다 → getCount 를 위한 테스트 메소드를 따로 만들어보자

```java
public User(String id, String name, String password) {
	this.id = id;
	this.name = name;
	this.password = password;
}

public User() {
} // 자바빈의 규약에 따르는 클래스에 생성자를 명시적으로 추가했을 때는 
// 파라미터가 없는 디퐅트 생성자도 함께 정의해야 한다. 

@Test
public void count() {
	ApplicationContext context = new GenericXmlApplicationContext("");
	
	UserDao dao = context.getBean("UserDao", UserDao.class);
	// 먼저 3개의 오브젝트를 준비해두고
	User user1 = new User("1", "name1", "001");
	User user2 = new User("2", "name2", "002");
	User user3 = new User("3". "name3", "003");
	
	// 모두 삭제 
	dao.deleteAll();
	assertThat(dao.getCount(), is(0));
	
	// user 추가
	dao.add(user1);
	assertThat(dao.getCount(), is(1));
	
	dao.add(user2);
	assertThat(dao.getCount(), is(2));
	
	dao.add(user3);
	assertThat(dao.getCount(), is(3));
}
```
- addAndGet() 테스트 보완
    - add() 후에 레코드 개수 확인 , get 으로 읽어온 값 비교하는 검증은 충분히 괜찮다
    - 그러나 get() 의 id 에 해당하는 사용자를 가져온 것인지 아무거나 가져온 것인지 추가 검증 필요

```java
@Test
public void addAndGet() {
	UserDao dao = context.getBean("userDao", UserDao.class);
	
	User user1 = new User("1", "name1", "001");
	User user2 = new User("2", "name2", "002");
	
	dao.deleteAll();
	assertThat(dao.getCount(), is(0));
	
	dao.add(user1);
	dao.add(user2);
	assertThat(dao.getCount(), is(2));
	
	// 첫번째 유저 아이디로 get 하면 첫 번째 유저의 값을 가진 오브젝트 반환하는지 확인
	User userget1 = dao.get(user1.getId());
	assertThat(userget1.getName(), is(user1.getName()));
	assertThat(userget1.getPassword(), is(user1.getPassword()));
	
	...
```
- get() 메소드에 전달된 id 값에 해당하는 유저 정보가 없다면?
    - null or 예외처리
    - 이 테스트의 경우에는 테스트 진행 중에 특정 예외가 던져지면 테스트가 성공한 것이고 예외가 던져지지 않고 정상적으로 작업을 마치면 테스트가 실패했다고 판단해야 함
    - EmptyResultDataAccessExpection 이 던져지면 성공

```java
// expected 는 예외가 던져지면 테스트가 성공함 
// 예외가 반드시 발생해야 하는 경우를 테스트하고 싶을 때 유용
@Test(expected=EmptyResultDataAccessExpection.class)
public void getUserFailure() {
	ApplicationContext context = new ClassPathXmlApplicationContext("");
		
		UserDao dao = context.getBean("userDao", UserDao.class);
		// 모든 데이터 지우고
		dao.deleteAll();
		assertThat(dao.getCount(), is(0));
		
		//없는 데이터로 get 실행해서 예외 발생하면 성공 
		dao.get("unknown_id");
	}
```
- 성공하면 getUserFailure(), addAndGet(), get() 이 모두 성공
- 포괄적인 테스트
    - 예외적인 상황도 빠뜨리지 않는 꼼꼼한 개발이 필요
    - 네거티브 테스트를 먼저 만들어라

### 2.3.4 테스트가 이끄는 개발 

- 테스트 주도 개발(테스트 우선 개발)
    - 만들고자 하는 기능의 내용을 담고 있으면서 만들어진 코드를 검증도 해줄 수 있도록 테스트 코드를 먼저 만들고 테스트를 성공하게 해주는 코드를 작성하는 방식의 개발 방법

### 2.3.5 테스트 코드 개선

- 필요하다면 테스트 코드도 언제든지 내부구조와 설계를 개선해서 좀 더 깔끔하고 변경이 용이한 코드로 만들 필요가 있음
- 중복 코드를 제거한 UserDaoTest

```java
public class UserDaoTest {
	private UserDao dao;
	
	@Before
	public void setUp() {
		ApplicationContext context = new GenericXmlApplicationContext("");
		this.dao = context.getBean("userDao", UserDao.class);
	}
	
	...
	
	@Test
	public void addAndGet() {
	... // 각 테스트 메소드에 반복적으로 나타났던 코드를 제거하고 별도의 메소드로 옮김 
	}
	
	@Test
	public void count() {
	...
	}
	
	@Test(expected=EmptyResultDataAccessExpection.class)
	public void getUserFailure() {
	...
	}
}
```

- JUnit 이 테스트하는 방법
    1. 테스트 클래스에서 @Test 가 붙은 public 이고 void 형이며 파라미터가 없는 테스트 메소드를 모두 갖는다.
    2. 테스트 클래스의 오브젝트를 하나 만든다.
    3. @Before 가 붙은 메소드가 있으면 실행한다.
    4. @Test 가 붙은 메소드를 하나 호출하고 테스트 결과를 저장한다.
    5. @After 가 붙은 메소드가 있으면 실행한다.
    6. 나머지 테스트 메소드에 대해 반복한다.
    7. 모든 테스트의 결과를 종합해서 반환한다.
    - 하나의 테스트 클래스 안에서 공통적인 준비 작업과 정리 작업이 필요한 경우에 @Before, @After 사용, 자동 실행
        - 서로 주고받은 정보나 오브젝트가 있다면 인스턴스 변수 사용
    - 각 테스트 메소드를 실행할 때마다 테스트 클래스의 오브젝트를 새로 만듦 → 한 번 사용 후 버려짐
        - 왜 테스트 클래스마다 하나의 오브젝트만 만들어놓고 사용하지 않나 → 각 테스트가 서로 영향을 주지 않고 독립적으로 실행됨 보장

- 픽스처
    - 테스트를 수행하는 데 필요한 정보나 오브젝트
    - @Before 을 이용해서 생성하면 편리함
    - UserDaoTest 에서 dao

```java
// 픽스처 적용 
public class UserDaoTest {
	private UserDao dao;
	private User user1;
	private User user2;
	private User user3;
	
	@Before
	public void setUp() {
		...
		this.user1 = new User("1", "name1", "001");
		this.user2 = new User("2", "name2", "002");
	    this.user3 = new User("3". "name3", "003");
    }
}
```
## 2.4 스프링 테스트 적용

- 아직도 @Before 메소드가 테스트 메소드 개수만큼 반복됨
- 애플리케이션 컨텍스트도 세 번 만들어짐
- 빈이 많아지면 비효율적
- 테스트는 가능한 한 독립적으로 매번 새로운 오브젝트를 만들어서 사용하는 것이 원칙이지만 애플리케이션 컨텍스트처럼 생성에 많은 시간과 자원이 소모되는 경우에는 테스트 전체가 공유하는 오브젝트를 만들기도 함
- @BeforeClass 스태틱 메소드 → 테스트 전체에 걸쳐 딱 한 번만 실행

### 2.4.1 테스트를 위한 애플리케이션 컨텍스트 관리

```java
// JUnit이 테스트를 진행하는 중에 테스트가 사용할 애플리케이션 컨텍스트를 만들고 관리
@RunWith(SpringJUnit4ClassRunner.class)
// 자동으로 만들어줄 애플리케이션 컨텍스트 설정 파일 위치 지정
@ContextConfiguration(locations="/.xml")
public class UserDaoTest {
	@Autowired
	private ApplicationContext context;
	
	@Before
	public void setUp() {
		this.dao = this.context.getBean("userDao", UserDao.class);
	}
}
```

- 하나의 애플리케이션 컨텍스트가 만들어져 모든 테스트 메소드에서 사용됨
- JUnit 확장기능은 테스트가 실행되지 전에 딱 한 번만 애플리케이션 컨텍스트를 만들고
- 테스트 오브젝트가 만들어질 때마다 특별한 방법을 이용해 애플리케이션 컨텍스트 자신을 테스트 오브젝트의 특정 필드에 주입하는 것
    - 테스트 개수에 관계없이 한 번만 만들어서 공유하게 해줌
- 같은 애플리케이션 컨텍스트 설정 파일 사용하면 테스트 수행 중에 단 한 개의 애플리케이션 컨텍스트만 만들어져 사용됨 (공유)

- @Autowired
    - 스프링의 DI 에 사용되는 애노테이션
    - @Autowired 가 붙은 인스턴스 변수가 있으면, 테스트 컨텍스트 프레임 워크는 변수 타입과 일치하는 컨텍스트 내의 빈을 찾는다
        - 타입이 일치하는 빈이 있으면 인스턴스 변수에 주입
        - 생성자, 수정자 메소드 생략 가능 → 타입에 의한 자동 와이어링
        - 애플리케이션 컨텍스트가 갖고 있는 빈을 DI을 받을 수 있다면 굳이 getBean() 말고 아예 UserDao 빈을 직접 DI 받을 수 있음
        - 어떤 빈이든 다 가져올 수 있음
```java
public class UserDaoTest {
	@Autowired
	UserDao dao; // UserDao 타입 빈을 직접 DI 받음 
}
```
### 2.4.2 DI 와 테스트 

- 인터페이스를 두고 DI 적용
    1. 클래스 대신 인터페이스 사용, new 대신 DI 를 통해 주입 
    2. 인터페이스를 두고 DI 를 적용하면 다른 차원의 서비스 기능 도입 가능 
    3. 효율적인 테스트 가능 
- 테스트 코드에 의한 DI
    - 테스트를 위한 수동 DI를 적용한 UserDaoTest
- 테스트에서 사용될 datasource 클래스가 빈으로 정의된 테스트 전용 설정 파일을 따로 만들어도 됨
    - 두 가지 설정 파일(서버및운영, 테스트)
- **스프링 컨테이너를 사용하지 않고 DI 를 테스트에 이용 가능**
    - @RunWith, @AutoWired 사용 x
    - 항상 우선적으로 고려 → 이 방법이 테스트 수행 속도가 가장 빠르고 테스트 자체가 간결

- 침투적 기술과 비침투적 기술
    - 침투적 기술 : 기술을 적용했을 때 애플리케이션 코드에 기술 관련 API 가 등장하거나 특정 인터페이스나 클래스를 사용하도록 강제하는 기술
    - 비침투적 기술 : 애플리케이션 로직을 담은 코드에 아무런 영향을 주지 않고 적용이 가능
        - 스프링은 비침투적인 기술의 대표적 예

## 2.5 학습 테스트로 배우는 스프링

- 학습 테스트 : 자신이 만들지 않은 프레임 워크나 다른 개발팀에서 만들어서 제공한 라이브러리 등에 대해서 테스트 작성
    - 다양한 조건에 따른 기능을 손쉽게 확인 가능
    - 학습 테스트 코드를 개발 중에 참고할 수 있음
    - 프레임워크나 제품을 업그레이드할 때 호환성 검증을 도와줌
    - 테스트 작성에 대한 좋은 훈련이 됨
- 학습 테스트 예제(JUnit)

```java
public class JUnitTest {
	static JUnitTest testObject;
	
	@Test
	public void test1() {
	// 스태틱 변수에 담긴 오브젝트와 자신을 비교해서 같지 않다는 사실 확인 
		assertThat(this, is(not(samInstance(testObject))));
		testObject = this;
	}
	
	@Test
	public void test2() {
		assertThat(this, is(not(samInstance(testObject))));
		testObject = this;
	}
	
	@Test
	public void test3() {
		assertThat(this, is(not(samInstance(testObject))));
		testObject = this;
	}
}
```

- 버그 테스트
    - 코드에 오류가 있을 때 그 오류를 가장 잘 드러내줄 수 있는 테스트
    - 일단 실패하도록 해야 함
    - 테스트의 완성도를 높여주고 버그의 내용을 명확하게 분석, 기술적인 문제를 해결하는 데 도움

	