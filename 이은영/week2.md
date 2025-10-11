# 2장. 테스트

스프링이 개발자에게 제공하는 가장 중요한 가치 → **객체지향과** **테스트** **!**

## 2.1 UserDaoTest 다시보기

**👁️‍🗨️  웹을 통한 DAO 테스트 방법의 문제점**

**웹을 통한 테스트란,** 웹 화면을 통해 값을 입력하고, 기능을 수행하고, 결과를 확인하는 방법

→ 하나의 테스트를 수행하는 데 참여하는 클래스와 코드가 너무 많기 때문에 에러가 날때 어디에서 문제가 발생했는지 찾기가 어렵다. ( 테스트하고 싶은건 UserDao뿐이지만 다른 계층의 코드, 컴포넌트, 서버 설정 상태까지 모두 테스트에 영향을 줄 수 있다 )

→  한번에 테스트하면 테스트 수행 과정이 복잡해지며, 오류 발생시 정확한 원인을 찾기 힘들어지기 때문에 테스트는 **작은 단위**로 쪼개서 실시해야한다 !

**👁️‍🗨️ 작은단위 테스트**

**단위테스트란,** 작은 단위의 코드에 대해 테스트를 수행하는 것으로 단위는 충분히 하나의 관심에만 집중해서 효율적으로 테스트할 만한 범위를 의미한다.

DB를 사용하더라도 상태를 테스트 코드가 제어할 수 있다면 단위 테스트로 볼 수 있으며, 작은 단위의 테스트는 코드의 안정성을 높이고, 개발 속도를 빠르게 유지하는 중요한 방법이다.

**👁️‍🗨️ 자동수행 테스트 코드**

**🌟 테스트는 자동으로 수행되도록 코드로 만들어져야 한다 !**

**👁️‍🗨️ UserDaoTest 문제점**

- 수동확인작업의 번거로움
  : add()에서 User 정보를 DB에 등록하고, 이를 다시 get()을 이용해 가져왔을 때 입 력한 값과 가져온 값이 일치하는지를 테스트 코드는 확인해주지 않는다.
- 실행 작업의 번거로움
  : DAO가 수백 개가 되고 그에 대한 main() 메소드도 그만큼 만들어진다면 전체 기능을 테스트해보기 위해 main() 메소드를 수백 번 실행하는 수고가 필요하다. 또한 그 결과를 눈으로 확인해서 기록하고, 이를 종합해서 전체 기능 을 모두 테스트한 결과를 정리하려면 이것도 제법 큰 작업이 된다.
  → main() 메 소드를 이용하는 방법보다 좀 더 편리하고 체계적으로 테스트를 실행하고 그 결과를 확인하는 방법이 필요하다.

## 2.2 UserDaoTest 개선

빠르게 실행 가능하고 스스로 테스트 수행과 기대하는 결과에 대한 확인까지 해주는 코드로 된 자동화된 테스트를 만들어 두자 !

¹ 일정한 패턴을 가진 테스트를 만들 수 있고, ² 많은 테스트를 간단히 실행시킬 수 있으며,
³ 테스트 결과를 종합해서 볼 수 있고, ⁴ 테스트가 실패한 곳을 빠르게 찾을 수 있는 기능을 갖춘 테스트 지원 도구, 그에 맞는 테스트 작성 방법이 필요 → JUnit 프레임워크를 사용해보자 !

**👁️‍🗨️ JUnit 테스트로 전환**

- main()에 있는 테스트 코드를 일반 메소드로 이동 ( 테스트가 main() 메소드로 만들어졌다는 건 제어권을 직접 갖는다는 의미 )
  **@Test** 어노테이션 + 테스트메소드 **public**으로 선언
- 검증 코드 전환
  **assertThat()** 메소드는 첫 번째 파라미터의 값을 뒤에 나오는 매처(matcher)라고 불리는 조건으로 비교해서 일치하면 다음으로 넘어가고, 아니면 테스트가 실패하도록 만들어준다.

## 2.3 개발자를 위한 테스팅 프레임워크 JUnit

**👁️‍🗨️ 단위 테스트**는 항상 **일관성 있는 결과가 보장**돼야 한다.

→ DB에 남아 있는 데이터와 같은 외부 환경에 영향을 받지 말아야 한다.
→ 모든 테스트는 실행 순서에 상관없이 독립적으로 항상 동일한 결과가 보장되도록 만들어야 한다. ( JUnit은 특정한 테스트 메소드의 실행 순서를 보장X )

**👁️‍🗨️ get() 예외조건에 대한 테스트**

🧐 get() 메소드에 전달된 id 값에 해당하는 사용자 정보가 없다면 ?

→ JUnit은 **예외조건 테스트**를 위한 특별한 방법을 제공

```java
@Test(expected=EmptyResultDataAccessException.class) // 발생할것으로 기대하는 예외 클래스를 지정
public void getUserFailure() throws SQLException {
		ApplicationContext context = new GenericXmlApplicationContext (
			"applicationContext.xml");
		UserDao dao = context.getBean ("userDao", UserDao.class);
		dao.deleteAll();
		assertThat(dao.getCount(), is(0));
		dao.get("unknown_id");
}
```

@Test에 expected를 추가하면,
정상적으로 테스트 메소드를 마치면 테스트가 실패, expected에서 지정한 예외가 던져지면 테스트가 성공

개발자가 테스트를 직접 만들 때 성공하는 테스트만 골라서 만드는 경향이 있다.

→ 항상 네거티브 테스트를 먼저 만들자 ! 🌟

**👁️‍🗨️ 테스트 주도 개발(TDD)**

만들고자 하는 기능의 내용을 담고 있으면서 만들어진 코드를 검증도 해줄 수 있도록 테스트 코드를 먼저 만들고, 테스트를 성공하게 해주는 코드를 작성하는 방식의 개발

**👁️‍🗨️ JUnit이 하나의 테스트 클래스를 가져와 테스트를 수행하는 방식**

1. 테스트 클래스에서 8Test가 붙은 public이고 void형이며 파라미터가 없는 테스트 메소드를 모두 찾는다.
2. 테스트 클래스의 오브젝트를 하나 만든다.
3. @Before가 붙은 메소드가 있으면 실행한다.
4. @Test가 붙은 메소드를 하나 호출하고 테스트 결과를 저장해둔다.
5. @After가 붙은 메소드가 있으면 실행한다.
6. 나머지 테스트 메소드에 대해 2~5번을 반복한다.
7. 모든 테스트의 결과를 종합해서 돌려준다.

→ 각 테스트 메소드를 실행할 때마다 테스트 클래스의 오브젝트를 새로 만든다 (각 테스트가 서로 영향을 주지 않고 독립적으로 실행됨을 확실히 보장 )

**픽스처
→** 테스트를 수행하는 데 필요한 정보나 오브젝트로, 일반적으로 여러 테스트에서 반복적으로 사용하기 때문에 @Before 메소드를 이용해 생성해두면 편리

## 2.4 스프링 테스트 적용

**👁️‍🗨️ 스프링 테스트 컨텍스트 프레임워크 적용**

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="/applicationContext.xml")
public class UserDaoTest {
  @Autowired
  private ApplicationContext context;

  @Before
  public void setUp() {
    this.dao = context.getBean("userDao", UserDao.class);

    this.user1 = new User("gyumee", "박성철", "springno1");
    this.user2 = new User("leegw700", "이길원", "springno2");
    this.user3 = new User("bumjin", "박범진", "springno3");
  }
}
```

@RunWith → JUnit 프레임워크의 테스트 실행 방법을 확장할 때 사용하는 애노테이션
@ContextConfiguration → 자동으로 만들어줄 애플리케이션 컨텍스트의 설정파일 위치를 지정

@Autowired → 스프링의 DI에 사용되는 특별한 애노테이션.
변수에 할당 가능한 타입을 가진 빈을 자동으로 찾으며, 두개 이상으로 설정되어 타입으로 가져올 수 없는 경우에는 변수의 이름과 같은 이름의 빈으로 가져온다.

**👁️‍🗨️ DI와 테스트**

**1️⃣ 테스트 코드에 의한 DI**

테스트 코드 내에서 테스트용 DataSource를 사용해 DAO의 의존성을 수동으로 변경한다.

→ 애플리케이션 컨텍스트의 상태를 변경하면, 나머지 모든 테스트를 수행 하는 동안 변경된 애플리케이션 컨텍스트가 계속 사용된다.  
→ `@DirtiesContext` 애노테이션을 사용해 스프링의 테스트 컨텍스트 프레임워크에게 해당 클래스의 테스트에서 애플리케이션 컨텍스트의 상태를 변경한다는 것을 알려준다.  
→테스트 컨텍스트는 이 애노테이션이 붙은 테스트 클래스에는 애플리케이션 컨텍스트 공유를 허용하지 않으며, 테스트 메소드를 수행하고 나면 매번 새로운 애플리케이션 컨텍스트를 만들어서 다음 테스트 가 사용하게 해준다.

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="/applicationContext.xml")
@DirtiesContext
public class UserDaoTest {
  @Autowired
  private ApplicationContext context;

  @Before
  public void setUp() {
    ...
    DataSource dataSource = new SingleConnectionDataSource(
      "jdbc:mysql://localhost/testdb", "spring", "book", true
    );

  }
}
```

**2️⃣ 테스트를 위한 별도의 DI 설정**

테스트 전용 설정파일을 만들어 적용한다.

→ 수동 DI 하는 코드나 @DirtiesContex 필요X

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="/test-applicationContext.xml")
public class UserDaoTest {
  ...
}
```

**3️⃣ 컨테이너 없는 DI 테스트**

아예 스프링 컨테이너를 사용하지 않고, 테스트 코드에서 직접 오브젝트를 만들고 DI해서 사용한다.

```java
public class UserDaoTest {
  // @Autowired 사용 X
  UserDao dao;

  ...

  @Before
  public void setUp() {
    ...
    dao = new UserDao();
    DataSource dataSource = new SingleConnectionDataSource(
      "jdbc:mysql://localhost/testdb", "spring", "book", true
    );
    dao.setDataSource(dataSource);
    // 오브젝트 생성, 관계설정 등을 모두 직접 해준다.
  }
}

```

**DI를 이용한 테스트 방법 선택**

→ 항상 스프링 컨테이너 없이 테스트할 수 있는 방법을 우선적으로 고려한다. ( 테스트 수행 속도 빠르고 간결 )
여러 오브젝트와 복잡한 의존관계가 있는 오브젝트라면, 스프링의 설정을 이용한 DI의 테스트 방식을 이용한다.
테스트 설정을 따로 만들었다고 하더라도 예외적인 의존관계를 강제로 구성해서 태스트해야 하는 경우에는 DI 받은 오브젝트에 다시 테스트 코드로 수동 DI하는 방식 이용한다.

## 2.5 학습 테스트로 배우는 스프링

**학습 테스트란,** 자신이 사용할 API나 프레임워크의 기능을 테스트 보면서 사용 방법을 익히는 것.

장점

- 다양한 조건에 따른 기능을 손쉽게 확인해볼 수 있다.
- 학습 테스트 코드를 개발 중에 참고할 수 있다.
- 프레임워크나 제품을 업그레이드할 때 호환성 검증을 도와준다
