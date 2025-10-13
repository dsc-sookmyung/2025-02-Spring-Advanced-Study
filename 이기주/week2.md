# 2장 테스트

## 2.1 UserDaoTest 다시보기
### 테스트의 유용성
코드를 짜면서 예상하고 의도한 대로 **정확히 동작하는지 확인**해서 확신을 얻을 수 있다. 테스트의 결과가 원하는 대로 나오지 않았으면 코드나 설계에 결함이 있음을 알 수 있다. 따라서 디버깅을 하며 테스트가 성공하면 실제 코드를 사용할 수 있다.
```java
public class UserDaoTest{
    public static void main(String[] args) throws SQLException{
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

### 웹으로 DAO 테스트
방법
1. DAO를 만들고, 서비스 계층, MVC까지 포함한 모든 입출력 기능을 대충 만든다.
2. 위에서 만든 테스트용 웹 애플리케이션을 서버에 배치 후, 웹에 띄워 폼을 열어 값을 입력해 등록 해본다.
    - 이를 위해 폼의 값을 파싱한 뒤 ```User``` 오브젝트로 만들어 ```UserDao```를 호출하는 기능이 있어야 한다.
3. 에러가 없으면 검색 폼 or 파라미터 지정 가능 URL을 사용해 입력한 값을 가져올 수 있는지 테스트해본다.
    - ```UserDao```가 돌려주는 결과를 화면에 출력하는 기능이 있어야 한다.
<br>

**모든 레이어의 기능을 다 만들고** 테스트를 할 수 있다는 점이 가장 큰 단점이다. 테스트 중 에러 발생이나 테스트 실패 시 어디에서 문제가 발생했는지 찾아내기 어렵다. ```UserDao```를 테스트하고 싶었지만 **많은 계층들이 테스트에 영향**을 줄 수 있기 때문에 번거롭고 대응하기 힘들다.

### 작은 단위의 테스트
"관심사의 분리"라는 원리는 여기에도 적용해야 한다. 이렇게 작은 단위의 코드에 대해 테스트 수행한 것을 **단위 테스트 unit test**라고 한다. ```UserDaoTest```는 ```DAO```라는 기능과 DB까지만 집중적으로 테스트할 수 있었다. (테스트 시 DB의 상태가 매번 달라지고, 테스트를 위해 DB를 특정 상태로 만들어줄 수 없다면 단위 테스트로서 가치가 없어진다.)

단위 테스트가 필요한 이유는 개발자가 만든 코드가 의도대로 동작하는지 빨리 확인하기 위해서다. 확인의 대상과 조건이 간단, 명확할수록 좋다.

### UserDaoTest의 문제점
1. 수동 확인 작업의 번거로움
    - ```add()```에서 ```User``` 정보를 DB에 등록하고, 다시 ```get()```을 이용해 가져왔을 때 입력값과 가져온 값이 일치하는지 **수동으로 확인해야 한다**.
    - 검증해야 하는 양이 많고 복잡해지거나 작은 차이가 있다면 발견하기 힘든 문제가 있다.
2. 실행 작업의 번거로움
    - 만약 ```DAO```가 많아진다면 그에 대한 ```main()``` 메소드를 **수백 번 실행**해야 한다.
    - 그 결과를 눈으로 확인해서 기록하고, 전체 테스트 결과를 정리하는 것도 큰 작업이 될 수 있다.

<br> 


## 2.2 UserDaoTest 개선
### 테스트 검증 자동화
> 기존의 테스트 코드에서는  ```get()```에서 가져온 결과를 사람이 눈으로 확인하도록 콘솔에 출력하기만 했다. 이제는 테스트 코드에서 결과를 직접 확인하고, 기대한 결과와 달라서 실패했을 경우에는 “테스트 실패”라는 메시지를 출력하도록 만들어보자. 

**기존 코드**
```java
System.out.println(user2.getName());
System.out.println(user2.getPassword());
System.out.println(user2.getId() + " 조회 성공");
```

**수정 코드**
```java
if (!user.getName().equals(user2.getName())) {
    System.out.println("테스트 실패 (name)");
} else if (!user.getPassword().equals(user2.getPassword())) {
    System.out.println("테스트 실패 (password)");
}
else {
    System.out.println("조회 테스트 성공");
}
```
처음 `add()`에 전달한 `User` 오브젝트와 `get()`을 통해 가져오는 `User` 오브젝트의 값을 비교해 일치하는지 확인한다. `name`, `password` 중 어느 것 때문에 실패했는 지를 알 수 있다.

기존 애플리케이션 코드 수정 시, 마음이 편안해지고 항상 자신감을 가지기 위해서는 빠르게 실행 가능하고 스스로 테스트 수행과 기대하는 결과에 대한 확인까지 해주는 코드로 된 자동화된 테스트를 만들어두는 것이 필수다!

### 테스트의 효율적인 수행 & 결과 관리
`main()` 메소드를 이용한 테스트는 애플리케이션 규모가 커지고 테스트 개수가 많아지면 부담이 된다. 따라서 **JUnit**을 사용하여 테스트 코드를 다시 작성할 것이다. 테스트가 `main()` 메소드로 되어있으면 제어권을 직접 갖는다는 의미이다. 하지만 우리는 **제어의 역전 IoC** 원리를 따를 필요가 있다.

1. `main()` 메소드를 일반 메소드로 옮기기!
    - 메소드가 public으로 선언되어야 한다.
    - 메소드에 `@Test` 어노테이션 붙여야 한다.

```java
import org.junit.Test;

public class UserDaoTest {
    
    @Test   // JUnit에게 테스트용 메소드임을 알려준다.
    public void addAndGet() throws SQLException {   // JUnit 테스트 메소드는 반드시 public으로 선언되어야 한다.
        ApplicationContext context = 
            new ClassPathXmlApplicationContext("applicationContext.xml");

        UserDao dao = context.getBean("userDao", UserDao.class);
    }
}
```

2. if/else 문장을 JUnit이 제공하는 방법을 이용해 전환!
    - JUnit이 제공해주는 `assertThat`이라는 스태틱 메소드를 이용한다. 첫 번째 파라미터의 값을 뒤에 나오는 **매처(matcher)**라고 불리는 조건으로 비교해서 일치하면 다음으로 넘어가고, 아니면 테스트가 실패하도록 만들어준다. `is()`는 매처의 일종으로, `equals()`로 비교해주는 기능을 가진다.

```java
assertThat(user2.getName(), is(user.getName()));
assertThat(user2.getPassword(), is(user.getPassword()));
```

다음은 최종 JUnit 프레임워크에서 실행할 수 있는 `UserDaoTest` 클래스다.

```java
import static org.hamcrest.CoreMatchers.is;
import static org.junit.Assert.assertThat;

...
public class UserDaoTest { 
    @Test
    public void addAndGet() throws SQLException {
        ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");

        UserDao dao = context.getBean("userDao", UserDao.class);
        User user = new User();

        user.setId("gyumee");
        user.setName("박성철");
        user.setPassword("springno1");

        dao.add(user);

        User user2 = dao.get(user.getId());

        assertThat(user2.getName(), is(user.getName()));
        assertThat(user2.getPassword(), is(user.getPassword()));
    }
}
```
<br>


## 2.3 테스팅 프레임워크 JUnit
### 테스트 결과의 일관성
테스트가 외부 상태에 따라 성공 여부가 달려 있으면 좋은 테스트라고 할 수 없다. 코드에 변경사항이 없다면 **테스트는 항상 동일한 결과**를 내야 한다. `UserDaoTest`의 문제는 이전 테스트로 인해 DB에 등록된 중복 데이터가 있을 수 있다. 이를 위해 `addAndGet()` 테스트 이후, **테스트가 등록한 값을 삭제해서,** 테스트 수행 이전 상태로 만들어야 여러 번 반복해서 실행해도 항상 동일한 결과를 얻을 수 있다.

일관성 있는 결과를 보장하기 위해 `UserDao`에 USER 테이블의 모든 레코드를 삭제해주는 `deleteAll()` 메소드를 추가한다. 파라미터 바인딩이 없어 단순하다.
```java
public void deleteAll() throws SQLException{
    Connection c = dataSource.getConnection();

    PreparedStatement ps = c.prepareStatement("delete from users");
    ps.executeUpdate();

    ps.close();
    c.close();
}
```

USER 테이블의 레코드 개수를 돌려주는 `getCount()` 메소드를 추가한다. `get()` 메소드와 비슷한 구조다.
```java
public int getCount() throws SQLException{
    Connection c = dataSource.getConnection();

    PreparedStatement ps = c.prepareStatement("select count(*) from users");

    ResultSet rs = ps.executeQuery();
    rs.next();
    int count = rs.getInt(1);

    rs.close();
    ps.close();
    c.close();

    return count;
}
```

아래는 `addAndGet()` 테스트에서 확장하여 `deleteAll()` 메소드와 `getCount()` 메소드를 테스트하기 위한 코드다. `deleteAll()`는 테스트가 시작될 때 실행하며 잘 동작한다면 `getCount()`로 레코드 개수를 가져올 경우 0이 나올 수 있다. 그러나 `getCount()`가 0을 돌려주는 버그를 가지고 있을 수도 있으니, `add()` 메소드 실행 후 `getCount()`의 레코드 개수 결과가 1이 나오는지도 확인해보자.
```java
@Test
public void addAndGet() throws SQLException {
    
    ...
    
    dao.deleteAll();    // 추가
    assertThat(dao.getCount(), is(0));  // 추가

    User user = new User();
    user.setId("gyumee");
    user.setName("박성철");
    user.setPassword("springno1");

    dao.add(user);
    assertThat(dao.getCount(), is(1));  // 추가

    User user2 = dao.get(user.getId());

    assertThat(user2.getName(), is(user.getName()));
    assertThat(user2.getPassword(), is(user.getPassword()));
}
```
### 포괄적인 테스트
테스트를 작성할 때 **네거티브 케이스**를 먼저 만들자. 대부분의 개발자는 "내 PC에서는 잘 되는데" 라는 변명을 늘여놓는데, 정상적인 케이스만 테스트 해봤기 때문이다.

#### getCount() 테스트
여러 개의 `User`를 등록하면서 `getCount()` 결과를 확인하자. `addAndGet()` 테스트가 아니라 `getCount()`를 위한 새로운 테스트 메소드를 만들 것이다.

테스트 시나리오:
- USER 테이블의 데이터를 모두 지우고 `getCount()`로 레코드 개수가 0임을 확인
- 3개의 사용자 정보를 하나씩 추가하며 매번 `getCount()`의 결과가 하나씩 증가하는지 확인

```java
@Test
public void count() throws SQLException {
    ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");

    UserDao dao = context.getBean("userDao", UserDao.class);
    User user1 = new User("gyumee", "박성철", "springno1");
    User user2 = new User("leegw700", "이길원", "springno2");
    User user3 = new User("bumjin", "박범진", "springno3");

    dao.deleteAll();
    assertThat(dao.getCount(), is(0));

    dao.add(user1);
    assertThat(dao.getCount(), is(1));

    dao.add(user2);
    assertThat(dao.getCount(), is(2));

    dao.add(user3);
    assertThat(dao.getCount(), is(3));
}
```

#### addAndGet() 테스트
`id`를 조건으로 사용자를 검색하는 기능의 `get()`에 대한 테스트가 부족하다. 따라서 `get()`이 파라미터로 주어진 `id`에 해당하는 사용자를 가져온 것인지 테스트해야 한다.

테스트 시나리오
- `User`를 하나 더 추가해서 두 개의 `User`를 `add()` 한다.
- 각 `User`의 `id`를 파라미터로 전달해서 `get()`을 실행하도록 만든다.

```java
@Test
public void addAndGet() throws SQLException {

    ...

    UserDao dao = context.getBean("userDao", UserDao.class);
    User user1 = new User("gyumee", "박성철", "springno1");
    User user2 = new User("leegw700", "이길원", "springno2");   // 중복되지 않는 두 개의 User 오브젝트 준비

    dao.deleteAll();
    assertThat(dao.getCount(), is(0));

    dao.add(user1);
    dao.add(user2);
    assertThat(dao.getCount(), is(2));

    User userget1 = dao.get(user1.getId());
    assertThat(userget1.getName(), is(user1.getName()));
    assertThat(userget1.getPassword(), is(user1.getPassword()));    // 첫 번째 User의 id로 get()을 실행하면 그 오브젝트 돌려주는지 확인

    User userget2 = dao.get(user2.getId());
    assertThat(userget2.getName(), is(user2.getName()));
    assertThat(userget2.getPassword(), is(user2.getPassword()));    // 두 번째 User의 id로 get()을 실행하면 그 오브젝트 돌려주는지 확인
}
```

#### get() 예외조건 테스트
`get()` 메소드에 전달된 `id` 값에 해당하는 사용자 정보가 없을 때, `id`에 해당하는 정보를 찾을 수 없다고 예외를 던지도록 만들자. 이때 스프링이 미리 정의해 놓은 예외를 사용한다. 다만 이 테스트는 `get()`에서 쿼리 결과의 첫 행을 가져오는 `rs.next()` 실행 시 가져올 로우가 없어 `SQLException`이 발생하여 당연히 실패된다. 

테스트 시나리오
- 모든 데이터를 지우고, 존재하지 않는 `id`로 `get()`을 호출한다.
- `EmptyResultDataAccessException`이 던져지면 성공이고, 아니면 실패다.

```java
@Test(expected=EmptyResultDataAccessException.class)
public void getUserFailure() throws SQLException {
    ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");

    UserDao dao = context.getBean("userDao", UserDao.class);

    dao.deleteAll();
    assertThat(dao.getCount(), is(0));

    dao.get("unknown_id");
}
```
> `@Test` 어노테이션의 `expected` 엘리먼트: 테스트 메소드 실행 중 발생을 기대하는 예외 클래스를 넣어주면 된다. 보통의 테스트와는 반대로, 정상적으로 테스트 메소드를 마치면 테스트가 실패하고, `expected`에서 지정한 예외가 던져지면 테스트 성공이다.

```java
public User get(String id) throws SQLException {
    ...
    ResultSet rs = ps.executeQuery();

    User user = null;   // User를 null 상태로 초기화
    if (rs.next()) {
        user = new User();
        user.setId(rs.getString("id"));
        user.setName(rs.getString("name"));
        user.setPassword(rs.getString("password"));
    }   // id를 조건으로 한 쿼리의 결과가 있으면 User 오브젝트를 만들고 값을 넣는다.

    rs.close();
    ps.close();
    c.close();

    if (user == null) throw new EmptyResultDataAccessException(1);  // 결과 없으면 User는 null, 확인해서 예외 던지기

    return user;
}
```
> 주어진 `id`에 해당하는 데이터가 없으면 `EmptyResultDataAccessException`을 던지는 `get()` 메소드로 수정

### 테스트 주도 개발
**TDD**
- 개발자가 테스트를 만들어가며 개발하는 방법의 장점을 극대화
- 실패한 테스트를 성공시키기 위한 목적이 아닌 코드는 만들지 않는다.
- 아예 테스트를 먼저 만들고, 이 테스트를 성공하도록 하는 코드만 만들어 테스트를 꼼꼼히 할 수 있다.
- 코드를 만들어 테스트를 실행하는 그 간격이 매우 짧다. 따라서 오류를 빨리 발견해 쉽게 대응할 수 있다.

### 테스트 코드 리팩토링
반복되는 부분을 별도의 메소드로 뽑거나 JUnit의 기능을 활용하자.

#### @Before
중복됐던 코드를 `setUp()` 메소드에 넣어준다. `dao` 변수를 테스트 메소드에서 접근할 수 있도록 인스턴스 변수로 변경한다.  `setUp()` 메서드에 `@Before`라는 어노테이션 추가한다.

```java
import org.junit.Before;
...
public class UserDaoTest {
    private UserDao dao;    // 인스턴스 변수로 선언
    
    @Before     // @Test 메소드가 실행되기 전, 먼저 실행돼야 하는 메소드 정의
    public void setup() {
        ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");
        this.dao = context.getBean("userDao", UserDao.class);
    }

    @Test
    public void addAndGet() throws SQLException {
        ...
    }

    @Test
    public void count() throws SQLException {
        ...
    }

    @Test(expected=EmptyResultDataAccessException.class)
    public void getUseFailure() throws SQLException {
        ...
    }
}
```

Junit이 하나의 테스트 클래스를 가져와 수행하는 방식
1. 테스트 클래스에서 @Test + public + void형 + 파라미터가 없는 테스트 메서드 모두 찾기
2. 테스트 클래스의 오브젝트 하나 만들기
3. `@Before`가 붙은 메서드가 있으면 실행
4. `@Test`가 붙은 메서드를 하나 호출 & 테스트 결과 저장
5. `@After`가 붙은 메서드가 있으면 실행
6. 나머지 테스트 메서드에 대해 2~5번을 반복
7. 모든 테스트의 결과를 종합해서 돌려주기
<br>


## 2.4 스프링 테스트 적용
현재 애플리케이션 컨텍스트 생성 방식에 의해 많은 시간이 소요될 수 있다. 생성에 많은 시간과 자원이 소모된다면 **테스트 전체가 공유하는 오브젝트**를 만들기도 한다. 다만 JUnit이 매번 테스트 클래스의 오브젝트를 새로 만들기 때문에 테스트들이 함께 참조할 애플리케이션 컨텍스트를 **스태틱 필드**에 저장하는 것이 좋다. 그러나 이를 위해 스프링이 제공하는 기능이 있다.

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "/applicationContext.xml")    // 테스트 컨텍스트가 자동으로 만들어줄 애플리케이션 컨텍스트의 위치 지정
public class UserDaoTest {
    @Autowired
    private ApplicationContext context;     // 테스트 오브젝트가 만들어지고 나면 스프링 테스트 컨텍스트에 의해 자동으로 값이 주입된다.
    ...

    @Before
    public void setup() {
        this.dao = this.context.getBean("userDao", UserDao.class);
        ...
    }
}
```

- `@Autowired`
    1. 스프링 DI에 사용되는 특별 어노테이션
    2. 테스트 컨텍스트 프레임워크가 이 어노테이션이 붙은 변수 타입과 일치하는 컨텍스트 내의 빈을 찾는다. 타입이 일치하는 빈이 있으면 인스턴스 변수에 주입한다. 

## 2.5 학습 테스트로 배우는 스프링
### 학습 테스트의 장점
1. 다양한 조건에 따른 기능 확인
    - 자동화된 테스트 코드로 만들어져 어떻게 동작하는지 빠르게 확인이 가능하다.
2. 학습 테스트 코드를 개발 중 참고 가능
    - 복잡한 기능 개발이라면 테스트에 관련된 설정파일이 만들어지고, 초기화 방법, API 호출 방법, 결과 가져오는 방법 등을 테스트 안에서 참고할 수 있다.
3. 프레임워크나 제품 업그레이드 시 호환성 검증 도움
    - 학습 테스트에 애플리케이션에서 자주 사용하는 기능에 대한 테스트가 있다면 새로운 버전의 프레임워크를 학습 테스트에만 적용해 볼 수 있다.
    - 버그가 있거나 API 사용 방법에 변화가 발생하면 이에 맞춰 애플리케이션 코드를 수정할 수 있다.
4. 테스트 작성에 좋은 훈련
    - 실제 프레임워크 사용 애플리케이션 코드의 테스트와 비슷하게 만들어진다.
    - 한두 가지 간단한 기능에만 초점이 맞춰져 단순하다.
    - 새로운 테스트 방법 연구에도 도움이 된다.
5. 새로운 기술을 공부하는 과정에서 오는 즐거움
    - 실제로 동작하는 모습을 볼 수 있다.

#### JUnit 테스트 오브젝트 생성 - 학습 테스트 예제

```java
import static org.junit.matchers.JUnitMatchers.hasItem;
...
public class JUnitTest {
    static Set<JUnitTest> testObjects = new HashSet<JUnitTest>();

    @Test
    public void test1() {
        assertThat(testObjects, not(hasItem(this)));
        testObjects.add(this);
    }

        @Test
        public void test2() {
            assertThat(testObjects, not(hasItem(this)));
            testObjects.add(this);
    }

        @Test
        public void test3() {
            assertThat(testObjects, not(hasItem(this)));
            testObjects.add(this);
    }
}
```
#### 스프링 테스트 컨텍스트 - 학습 테스트 예제

```java
import static org.hamcrest.CoreMatchers.nullValue;
import static org.junit.Assert.assertTrue;
import static org.junit.matchers.JUnitMatchers.either;
...

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration
public class JUnitTest {
    @Autowired
    ApplicationContext context; // 테스트 컨텍스트가 매번 주입해주는 애플리케이션 컨텍스트는 항상 같은 오브젝트인지 테스트로 확인해본다.

    static Set<JUnitTest> testObjects = new HashSet<JUnitTest>();
    static ApplicationContext contextObject = null;

    @Test
    public void test1() {
        assertThat(testObjects, not(hasItem(this)));
        testObjects.add(this);

        assertThat(contextObject == null || contextObject == this.context, is(true));
        contextObject = this.context;
    }

    @Test
    public void test2() {
        assertThat(testObjects, not(hasItem(this)));
        testObjects.add(this);

        assertTrue(contextObject == null || contextObject == this.context);
        contextObject = this.context;
    }

    @Test
    public void test3() {
        assertThat(testObjects, not(hasItem(this)));
        testObjects.add(this);

        assertThat(contextObject, either(isNullValue()).or(is(this.context)));
        contextObject = this.context;
    }
}
```