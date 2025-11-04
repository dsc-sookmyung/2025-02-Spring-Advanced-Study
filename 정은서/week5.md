# [정은서] 2025 GDG Spring Advanced Study - 5주차
# 토비의 스프링 3.1(vol1) - 5장 서비스 추상화

## 서비스 추상화
- 지금까지 만든 DAO 에 트랜잭션을 적용해보면서 스프링이 어떻게 성격이 비슷한 여러 종류의 기술을 추상화하고 이를 일관된 방법으로 사용할 수 있도록 지원하는지 알아보자.

### 5.1 사용자 레벨 관리 기능 추가
- 지금까지 만들었던 UserDao 는 사용자 정보를 등록, 조회, 수정, 삭제하는 CRUD 작업만 가능하다. 이제 여기에 간단한 비즈니스 로직을 추가해보자.

    - 사용자의 레벨은 BASIC, SILVER, GOLD 중 하나이다.
    사용자가 처음 가입하면 BASIC 레벨에 되며, 이후 활동에 따라서 한 단계의 업그레이드될 수 있다.
    - 가입 후 50회 이상 로그인을 하면 BASIC 에서 SILVER 레벨이 된다.
    - SILVER 레벨이면서 30번 이상 추천을 받으면 GOLD 레벨이 된다.
    - 사용자 레벨의 변경 작업은 일정한 주기를 가지고 일괄적으로 진행한다. 반경 작업 전에는 조건을 충족하더라도 레벨의 변경이 일어나지 않는다.

5.1.1 필드 추가

Level 이늄 (enum)

- User 클래스에 사용자의 레벨을 저장할 필드를 추가하자.
    - varchar 타입으로 문자열로 넣는 방법, 상수값을 정해놓고 int 타입으로 사용하는 방법 → 적절하지 않음
    - 자바 5이상에서 제공하는 enum 을 이용하자.

```java
package springbook.user.domain;
...
public enum Level {
	BASIC(1), SILVER(2), GOLD(3); // enum 오브젝트 정의
	
	private final int value;
	
	Level(int value) { // DB 에 저장할 값을 넣어줄 생성자 만들기
		this.value = value;
	}
	
	public int intValue() { // 값 가져오기
		return value;
	}
	
	// 값으로부터 Level 타입 오브젝트를 가져오도록 만든 스태틱 메소드
	public static Level valueOf(int value) {
		switch(value) {
			case 1: return BASIC;
			case 2: return SILVER;
			case 3: return GOLD;
			default: throw new AssertionError("Unknown value: " + value);
		}
	}
}
```
Level Enum을 추가함에 따라  User 필드 추가, UserDaoTest 테스트, UserDaoJdbc 수정해야 한다. 

5.1.2 사용자 수정 기능 추가

사용자 정보는 여러 번 수정될 수 있다. 수정할 정보가 담긴 User 오브젝트를 전달하면 id 를 참고해서 사용자를 찾아 필드 정보를 UPDATE 문을 이용해 모두 변경해주는 메소드를 만들자.

```java
public void update(User user) {
	this.jdbcTemplate.update(
		"update users set name=?, password=?, level=?, login=?, " +
		"recommend=?, where id=?", user.getName(), user.getPassword(), 
		user.getLevel().intValue(), user.getLogin(), user.getRecommend(),
		user.getId());
}
```

→ 현재 update 테스트는 수정할 로우의 내용이 바뀐 것만 확인할 뿐이지 수정하지 않아야 할 로우의 내용이 그대로 남아 있는지는 확인해주지 못한다는 문제가 있다.

- 해결 방법

    1. JdbcTemplate 의 update() 가 돌려주는 리턴 값을 확인하는 것
    2. 테스트를 보강해서 원하는 사용자 외의 정보는 변경되지 않았음을 직접 확인하는 것이다.
        - 사용자를 두 명 등록해놓고 그 중 하나만 수정한 뒤에 수정된 사용자와 수정하지 않은 사용자의 정보를 모두 확인하면 된다.

```java
@Test
public void update() {
	dao.deleteAll();
	
	dao.add(user1); // 수정할 사용자
	dao.add(user2); // 수정하지 않을 사용자
	
	user1.setName("user4");
	user1.setPassword("springno6");
	user1.setLevel(Level.GOLD);
	user1.setLogin(1000);
	user1.setRecommend(999);
	
	dao.update(user1);
	
	User user1update = dao.get(user1.getId());
	checkSameUser(user1, user1update);
	User user2same = dao.get(user2.getId());
	checkSameUser(user2, user2same);
}
```

5.1.3 UserService.upgradeLevels()

사용자 정보를 수정하는 기능을 추가했으니 이제 본격적인 사용자 관리 비즈니스 로직을 구현할 차례이다. UserDao 의 getAll() 메소드로 사용자를 다 가져와서 사용자별로 레벨 업그레이드 작업을 진행하면서 UserDao의 update() 를 호출해 DB에 결과를 넣어주면 된다.

비즈니스 로직 서비스를 제공한다는 의미에서 클래스 이름은 UserService 로 한다. UserService는 UserDao 인터페이스 타입으로 userDao 빈을 DI 받아 사용하게 된다. 

```java
// upgradeLevels() 메소드
public void upgradeLevels() {
	List<User> users = userDao.getAll();
	for (User user : users) {
		Boolean changed = null;
		if (user.getLevel() == Level.BASIC && user.getLogin() >= 50) {
			user.setLevel(Level.SILVER);
			changed = true;
		}
		else if (user.getLevel() == Level.SILVER && user.getRecommend() >= 30) {
			user.setLevel(Level.GOLD);
			changed = true;
		}
		else if (user.getLevel() == Level.GOLD) {changed = false;}
		else {changed = false;}
		if (changed) {userDao.update(user);} // 레벨의 변경이 있는 경우에만 호출 
	}
}
```
(테스트 시에는 데이터를 경계가 되는 값의 전후로 선택하는 것이 좋다.)

5.1.4 UserService.add()

처음 가입하는 사용자는 기본적으로 BASIC 레벨이어야 한다는 부분을 추가하자.

UserDao의 add() 메소드는 사용자 정보를 담은 User 오브젝트를 받아서 DB에 넣어주는 데 충실한 역할을 한다면, UserServce 에도 add()를 만들어두고 사용자가 등록될 때 적용할 만한 비즈니스 로직을 담당하게 하면 된다.

5.1.5 코드 개선

upgradeLevels() 메소드 코드의 문제점

- 성격이 다른 여러 로직이 한데 섞여 있어 if/elseif/else 블록들이 읽기 불편하다.
- 현재 레벨과 업그레이드 조건을 동시에 비교하는 부분도 문제가 될 수 있다.
- 첫 단계에서는 레벨을 확인하고 각 레벨별로 다시 조건을 판단하는 조건식을 넣어야 한다.

```java
public void upgradeLevels() {
	List<User> users = userDao.getAll();
	for(User user : users) {
		if (canUpgradeLevel(user)) {
			upgradeLevel(user);
			}
		}
	}
    
    private boolean canUpgradeLevel(User user) {
	Level currentLevel = user.getLevel();
	switch(currentLevel) {
		case BASIC: return (user.getLogin() >= 50);
		case SILVER: return (user.getRecommend() >= 30);
		case GOLD: return false;
		default: throw new IllegalArgumaentException("Unknown Level: " + currentLevel);
	}
}
```
레벨의 순서와 다음 단계 레벨이 무엇인지를 결정하는 일은 Level 에게 맡기자. 레벨의 순서를 굳이 UserService 에 담아둘 이유가 없다.

```java
public enum Level {
	// enum 선언에 다음 단계의 레벨 정보도 추가
	GOLD(3, null), SILVER(2, GOLD), BASIC(1, SILVER);
	private final int value;
	private final Level next; // 다음 단계의 레벨 정보를 스스로 갖고 있도록 Level 타입의 next 변수를 추가
	
	Level(int value, Level next) {
		this.value = value;
		this.next = next;
	}
	public Level nextLevel() {
		return this.next;
	}
	...
```
사용자 정보가 바뀌는 부분을 UserService 메소드에서 User로 옮겨보자. User 의 내부 정보가 변경되는 것은 UserService 보다는 User가 스스로 다루는 것이 적절하다.

Level 의 순서와 다음 단계 정보는 모두 Level enum 에서 관리하기 때문에 User는 Level 타입의 level 필드에서 다음 레벨이 무엇인지 알려달라고 요청해서 현재 레벨을 변경해주면 된다. 

UserServceTest

```java
@Test
public void upgradeLevels() {
	userDao.deleteAll();
	for (User user : users) userDao.add(user);
	
	userService.upgrageLevels();
	
	checkLevelUpgraded(users.get(0), false);
	checkLevelUpgraded(users.get(1), true);
	checkLevelUpgraded(users.get(2), false);
	checkLevelUpgraded(users.get(3), true);
	checkLevelUpgraded(users.get(4), false);
}

// 어떤 레벨로 바뀔 것인가가 아니라 다음 레벨로 업그레이드될 것인가 아닌가를 지정한다.
private void checkLevelUpgraded(User user, boolean upgraded) {
	User userUpdate = userDao.get(user.getId());
	if (upgraded) {
	// 다음 레벨이 무엇인지는 Level 에게 물어보면 된다. 
		assertThat(userUpdate.getLevel(), is(user.getLevel().nextLevel()));
	}
	else {
		assertThat(userUIpdat.getLevel(), is(user.getLevel)));
	}
}
```

```java
// 상수의 도입 
public static final int MIN_LOGCOUNT_FOR_SILVER = 50;
public static final int MIN_RECCOMEND_FOR_GOLD = 30;

private boolean canUpgradeLevel(User user) {
	Level currentLevel = user.getLevel();
	switch(currentLevel) {
		case BASIC: return (user.getLogin() >= MIN_LOGCOUNT_FOR_SILVER);
		case SILVER: return (user.getRecommend() >= MIN_RECCOMEND_FOR_GOLD);
		case GOLD: return false;
		default: throw new IllegalArgumaentException("Unknown Level: " + currentLevel);
	}
}
```

```java
// 상수를 사용하도록 만든 테스트 
public static final int MIN_LOGCOUNT_FOR_SILVER = 50;
public static final int MIN_RECCOMEND_FOR_GOLD = 30;
	...
	@Before
	public void setUp() {
		users = Arrays.asList(
			new User("user1", "사용자1", "springno1", Level.BASIC, MIN_LOGCOUNT_FOR_SILVER-1, 0);
			new User("user2", "사용자2", "springno2", Level.BASIC, MIN_LOGCOUNT_FOR_SILVER, 10);
			new User("user3", "사용자3", "springno3", Level.SILVER, 60, MIN_RECCOMEND_FOR_GOLD-1);
			new User("user4", "사용자4", "springno4", Level.SILVER, 60, MIN_RECCOMEND_FOR_GOLD);
		);
	}
```
상수를 도입하면 비즈니스 로직을 상세히 코멘트로 달아놓거나 설계문서를 참조하기 전에는 이해하기 힘들었던 부분이 이제는 무슨 의도로 어떤 값을 넣었는지 이해하기 쉬워진다.

### 5.2 트랜잭션 서비스 추상화

정기 사용자 레벨 관리 작업을 수행하는 도중에 네트워크가 갑자기 끊기거나 서버에 장애가 생겨서 작업을 완료할 수 없다면, 그때까지 변경된 사용자의 레벨은 그대로 둘까? 아니면 모두 초기 상태로 되돌려 놓아야 할까?

중간에 문제가 발생해서 작업이 중단된다면 그때까지 진행된 변경 작업도 모두 취소시키도록 결정했다.

테스트용 UserService 대역

예외를 강제로 발생시키도록 애플리케이션 코드를 수정하자. 네 번째 사용자를 처리하는 도중에 예외를 발생시키고, 그 전에 처리한 두 번째 사용자의 정보가 취소됐는지, 아니면 그대로 남았는지를 확인하자. 먼저 UserService 의 upgradeLevel() 메소드 접근 권한을 protected로 수정해서 상속을 통해 오버라이딩이 가능하도록 하자.

```java
protected void upgradeLevel(User user) {…}
```

```java
static class TestUserService extends UserService {
	private String id;
	
	private TestUserService(String id) {
		this.id = id; // 미리 지정된 id를 가진 사용자가 발견되면 강제로 예외를 던지도록 
	}
	
	protected void upgradeLevel(User user) {
		if (user.getId().equals(this.id)) throw new TestUserServiceException();
    }
}
```

강제 예외 발생을 통한 테스트

사용자 레벨 업그레이드를 시도하다가 중간에 예외가 발생한 경우, 그 전에 업그레이드 했던 사용자도 다시 원래 상태로 돌아갔는지를 확인한다.

```java
@Test
public void upgradeAllorNothing() {
	// 예외를 발생시킬 네 번째 사용자의 id를 넣어서 테스트용 UserService 대역 오브젝트 생성 
	userService testUserService = new TestUserService(users.get(3).getId());
	testUserService.setUserDao(this.userDao); // 수동DI
	
	userDao.deleteAll();
	for(User user : users) userDao.add(user);
	
	try {
		// 업그레이드 작업 중에 예외가 발생해야 함 
		testUserService.upgradeLevels();
		fail("TestUserServiceException expected");
	} catch(TestUserServiceException e) {}
	
	checkLevelUpgraded(users.get(1), false); // 예외가 발생하기 전에 레벨 변경이 있었던 사용자의 레벨이 처음 상태로 바뀌었나 확인
}
```
테스트 결과는 두 번째 사용자의 레벨이 BASIC에서 SILVER로 바뀐 것이 네 번째 사용자 처리 중 예외가 발생했지만 예외가 그대로 유지된다.

→ 모든 사용자의 레벨을 업그레이드 하는 작업인 upgradeLevels() 메소드가 하나의 트랜잭션 안에서 동작하지 않았기 때문이다.

5.2.2 트랜잭션 경계설정

- 트랜잭션 롤백
    - 이때 두 가지 작업이 하나의 트랜잭션이 되려면 두 번째 SQL이 성공적으로 DB에서 수행되기 전에 문제가 발생할 경우 앞에서 처리한 SQL 작업도 취소시켜야 한다.
- 트랜잭션 커밋
    - 여러 개의 SQL을 하나의 트랜잭션으로 처리하는 경우에 모든 SQL 수행 작업이 다 성공적으로 마무리 됐다고 DB에 알려줘서 작업을 확정시켜야 한다.

트랜잭션이 한 번 시작되면 commit() 또는 rollback() 메소드가 호출될 때까지의 작업이 하나의 트랜잭션으로 묶인다.

setAutoCommit(false) 로 트랜잭션의 시작을 선언하고 commit() 또는 rollback()으로 트랜잭션을 종료하는 작업을 트랜잭션의 경계설정이라고 한다. 트랜잭션의 경계는 하나의 Connection이 만들어지고 닫히는 범위 안에 존재한다는 점도 기억해두자. 이렇게 하나의 DB 커넥션 안에서 만들어지는 트랜잭션을 로컬 트랜잭션이라고도 한다.

UserService 와 UserDao의 트랜잭션 문제

지금까지 만든 코드 어디에도 트랜잭션을 시작하고, 커밋하고, 롤백하는 트랜잭션 경계설정 코드가 존재하지 않는다.

템플릿 메소드가 호출될 때마다 트랜잭션이 새로 만들어지고 메소드를 빠져나오기 전에 종료된다. 결국 JdbcTemplate 메소드를 사용하는 UserDao는 각 메소드마다 하나씩의 독립적인 트랜잭션으로 실행되기 때문에 오류가 발생해서 작업이 중단된다고 해도 첫 번째 커밋한 트랜잭션의 결과는 DB에 그대로 남는다. 어떤 일련의 작업이 하나의 트랜잭션으로 묶이려면 그 작업이 진행되는 동안 DB 커넥션도 하나만 사용해야하고 이렇게 하려면 upgradeLevels() 메소드 안에 트랜잭션 경계를 두어야 한다.

```java
class UserService {
	public void upgradeLevels() throws Exception {
		Connection c = ...;
							
		...        
		try {        
			...         
			upgradeLevel(c, user);
			...            
		}           
		...
	}
	
	protected void upgradeLevel(Connection c, User user) {
		user.upgradeLevel();
		userDao.update(c, user);
	}
}
=====================================================================

interface UserDao {
	public update(Conneceion c, User user);
	...
}
```
UserService 트랜잭션 경계설정의 문제점

1. DB 커넥션을 비롯한 리소스의 깔끔한 처리를 가능하게 했던 JdbcTemplete를 더 이상 활용할 수 없다.
2. DAO의 메소드와 비즈니스 로직을 담고 있는 UserService 의 메소드에 Connection 파라미터가 추가돼야 한다.
3. Connection 파라미터가 UserDao 인터페이스 메소드에 추가되면서 UserDao는 더 이상 데이터 액세스 기술에 독립적일 수 없다는 것이다.
4. DAO 메소드에 Connection 파라미터를 받게 하면 테스트 코드에도 영향을 미친다.

5.2.3 트랜잭션 동기화

UserService에서 트랜잭션을 시작하기 위해 만든 Connection 오브젝트를 특별한 저장소에 보관해두고, 이후에 호출되는 DAO의 메소드에서는 저장된 Connection 을 가져다가 사용하게 하는 것이다.

1. UserService 는 Connection을 생성하고
2. 이를 트랜잭션 동기화 저장소에 저장해두고 Connection의 setAutoCommit(false) 를 호출해 트랜잭션을 시작시킨 후에 본격적으로 DAO의 기능을 이용하기 시작한다.
3. 첫 번째 update() 메소드가 호출되고, update() 메소드 내부에서 이용하는 4.JdbcTemplate 메소드에서는 가장 먼저
4. 트랜잭션 동기화 저장소에 현재 시작된 트랜잭션을 가진 Connection 오브젝트가 존재하는지 확인한다. 2번의 upgradeLevels() 메소드 시작 부분에서 저장해둔 Connection 을 발견하고 이를 가져온다.
5. 가져온 Connection 을 이용해 PreparedStatement 를 만들어 수정 SQL을 실행한다. 트랜잭션 동기화 저장소에서 DB 커넥션을 가져왔을 때는 JdbcTemplate 은 Connection을 닫지 않은 채로 작업을 마친다.
6. 두 번째도 같은 트랜잭션을 가진 Connection을 가져와 사용한다.
7. 트랜잭션 내의 모든 작업이 정상적으로 끝났으면 UserService는 이제 Connection의 commit() 을 호출해서 트랜잭션을 완료시킨다.
8. 트랜잭션 저장소가 더 이상 Connection 오브젝트를 저장해두지 않도록 이를 제거한다.

이렇게 트랜잭션 동기화를 사용하면 파라미터를 통해 일일이 Connection 오브젝트를 전달할 필요가 없어진다.

트랜잭션 동기화 적용

스프링은 JdbcTemplate 와 더불어 트랜잭션 동기화 기능을 지원하는 간단한 유틸리티 메소드를 제공한다.

```java
// 트랜잭션 동기화 방식을 적용한 UserService
private DataSource dataSource;

public void setDateSource(DataSource dataSource) {
	this.dataSource = dataSource;
}

public void upgradeLevels() throws Exception {
	// 트랜잭션 동기화 관리자를 이용해 동기화 작업을 초기화
	TransactionSynchronizationManager.initSynchronization();
	// DB 커넥션을 생성하고 트랜잭션을 시작 
	// 이후의 DAO 작업은 모두 여기서 시작한 트랜잭션 안에서 진행
	// DB 커넥션 생성과 동기화를 함께 해주는 유틸리티 메소드
	Connection c = DataSourceUtils.getConnection(dataSource);
	c.setAutoCommit(false);
	
	try {
		List<User> users = userDao.getAll();
		for (User user : users) {
			if (canUpgradeLevel(user)) {
				upgradeLevel(user);
			}
		}
		c.commit();
	}catch (Exception e) {
		c.rollback();
		throw e;
	} finally {
		// 스프링 유틸리티 메소드를 이용해 DB 커넥션을 안전하게 닫음
		DataSourceUtils.releaseConnection(c, dataSource);
		TransactionSynchronizationManager.unbindResource(this.dataSource);
		TransactionSynchronizationManager.clearSynchronizaion();
	}
}
```
→ JDBC 의 트랜잭션 경계설정 메소드를 사용해 트랜잭션을 이용하는 전형적인 코드에 간단한 트랜잭션 동기화 작업만 붙여줌으로써, 지저분한 Connection 파라미터 문제를 해결할 수 있다.

5.2.4 트랜잭션 서비스 추상화

지금까지 만들어온 코드들은 별 문제가 없어 보인다. 하지만 이 사용자 관리 모듈을 구매해서 사용하기로 한 G사에서 들어온 새로운 요구 때문에 문제가 생겼다.

G사는 여러 개의 DB를 사용하고 있다. 그래서 하나의 트랜잭션 안에서 여러 개의 DB에 데이터를 넣는 작업을 해야하는데 로컬 트랜잭션으로는 불가능하다. 로컬 트랜잭션은 하나의 DB Connection에 종속되기 때문이다.

따라서 별도의 트랜잭션 관리자를 통해 관리하는 글로벌 트랜잭션을 사용해야 한다. 자바는 JDBC 외에 이런 글로벌 트랜잭션을 지원하는 트랜잭션 매니저를 지원하기 위한 API인 JTA를 제공하고 있다. 이를 사용하면 여러 개의 DB나 메시징 서버에 대한 작업을 하나의 트랜잭션으로 통합하는 글로벌 트랜잭션이 가능해진다.

문제는 JDBC 로컬 트랜잭션을 JTA를 이용하는 글로벌 트랜잭션으로 바꾸려면 UserService 코드를 수정해야 한다는 점이다. UserService 에서 트랜잭션의 경계 설정을 해야 할 필요가 생기면서 다시 특정 데이터 액세스 기술에 종속되는 구조가 되고 말았다.

추상화란 하위 시스템의 공통점을 뽑아내서 분리시키는 것을 말한다. 이 개념을 트랜잭션 처리 코드에도 도입해보자. 스프링은 트랜잭션 기술의 공통점을 담은 트랜잭션 추상화 기술을 제공하고 있다. 이를 이용하면 애플리케이션에서 직접 각 기술의 트랜잭션 API를 이용하지 않고도, 일관된 방식으로 트랜잭션을 제어하는 트랜잭션 경계설정 작업이 가능해진다. 

```java
// 스프링의 트랜잭션 추상화 API를 적용한 upgradeLevels() 
public void upgradeLevels() {
	// 스프링이 제공하는 트랜잭션 경계설정을 위한 추상 인터페이스는 PlateformTransactionManager
	PlateformTransactionManager transactionManager = 
		new DataSourceTransactionManager(dateSource);
	
	TransactionStatus status = 
		transactionManager.getTransaction(new DefaultTransactionDefinition()));
		
	try {
		List<User> users = userDao.getAll();
		for (User user :users) {
			if (canUpgradeLevel(user)) {
				upgradeLevel(user);
			}
		}
		transactionManager.commit(status);
	}catch (RuntimeException e) {
		transactionManager.rollback(status);
		throw 3;
	}
}
```
어떤 트랜잭션 매니저 구현 클래스를 사용할지 UserService 코드가 알고있는 것은 DI 원칙에 위배됨
```java
// 트랜잭션 메니저를 빈으로 분리시킨 UserService
public class UserService {
	...
	private PlateformTransactionManager transactionManager;
	
	public void setTransactionManager(PlateformTransactionManager transactionManager) {
		this.transactionManager = transactionManager;
	}
}
// 이후 DI 받은 트랜잭션 매니저를 공유해서 사용
```

### 5.3 서비스 추상화와 단일 책임 원칙

이제 스프링의 트랜잭션 서비스 추상화 기법을 이용해 다양한 트랜잭션 기술을 일관된 방식으로 제어할 수 있게 됐다. 이렇게 기술과 서비스에 대한 추상화 기법을 이용하면 특정 기술환경에 종속되지 않는 포터블한 코드를 만들 수 있다.

적절하게 책임과 관심이 다른 코드를 분리하고, 서로 영향을 주지 않도록 다양한 추상화 기법을 도입하고, 애플리케이션 로직과 기술/환경을 분리하는 등의 작업은 할수있는 데는 스프링의 DI가 중요한 역할을 하고 있다. 

- 단일 책임 원칙
하나의 모듈은 한 가지 책임을 가져야 한다는 의미이다. (ex. UserService) 어떤 변경이 필요할 때 수정 대상이 명확해진다. 기술이 바뀌면 기술 계층과의 연동을 담당하는 기술 추상화 계층의 설정만 바꿔주면 된다.

### 5.4 메일 서비스 추상화

레벨이 업그레이드되는 사용자에게는 안내 메일을 발송해보자. 안내메일을 발송하기 위해서는 먼저 사용자의 이메일 정보를 관리해야 하고, upgradeLevel() 메소드에 메일 발송 기능을 추가해야 한다. 

5.4.1 JavaMail 을 이용한 메일 발송 기능

자바에서 메일을 발송할 때는 표준 기술인 JavaMail을 사용하면 된다.

```java
protected void upgradeLevel(User user) {
	user.upgradeLevel();
	userDao.update(user);
	sendUpgradeEMail(user);
}
private void sendUpgradeEmail(User user) {
    Properties props = new Properties();
    props.put("mail.smtp.host", "mail.ksug.org");
    Session s = Session.getInstance(props, null);

    MimeMessage message = new MimeMessage(s);
    try {
        message.setFrom(new InternetAddress("useradmin@ksug.org"));
        message.addRecipient(Message.RecipientType.TO,
                             new InternetAddress(user.getEmail()));
        message.setSubject("Upgrade 안내");
        message.setText("사용자님의 등급이 " + user.getLevel().name() +
                      "로 업그레이드되었습니다");

        Transport.send(message);
    } catch (AddressException e) {
        throw new RuntimeException(e);
    } catch (MessagingException e) {
        throw new RuntimeException(e);
    } catch (UnsupportedEncodingException e) {
        throw new RuntimeException(e);
    }
}
```

5.4.3 테스트를 위한 서비스 추상화

실제 메일 전송을 수행하는 JavaMail 대신에 테스트에서 사용할 JavaMail과 같은 인터페이스를 갖는 오브젝트를 만들어서 사용하는 방법은 JavaMail의 API가 적용할 수 없다.

스프링은 JavaMail을 사용해 만든 코드는 손쉽게 테스트하기 힘들다는 문제를 해결하기 위해서 JavaMail에 대한 추상화 기능을 제공하고 있다.

```java
package org.springframework.mail;

// ...

public interface MailSender {
    void send(SimpleMailMessage simpleMessage) throws MailException;
    void send(SimpleMailMessage[] simpleMessages) throws MailException;
}
```
```java
// 메일 전송 기능을 가진 오브젝트를 DI 받도록
public class UserService {
    ...
    private MailSender mailSender;

    public void setMailSender(MailSender mailSender) {
        this.mailSender = mailSender;
    }
}
private void sendUpgradeEmail(User user) {
    SimpleMailMessage mailMessage = new SimpleMailMessage();
    mailMessage.setTo(user.getEmail());
    mailMessage.setFrom("useradmin@ksug.org");
    mailMessage.setSubject("Upgrade 안내");
    mailMessage.setText("사용자님의 등급이 " + user.getLevel().name());

    this.mailSender.send(mailMessage);
}
```
-> sendUpgradeEmail() 메소드에는 MailSender 인터페이스만 남기고 구체적인 메일 전송 구현을 담은 클래스의 정보는 코드에서 모두 제거한다. 수정자 메소드를 추가해 DI가 가능하도록 만든다.

메일 발송 기능에도 트랜잭션 개념을 적용해야 한다.

1. 메일을 업그레이드할 사용자를 발견했을 때마다 발송하지 않고 발송 대상을 별도의 목록에 저장해두고 업그레이드 작업이 모두 성공적으로 끝났을 때 한 번에 메일을 전송한다.
2. MailSender를 확장해서 메일 전송에 트랜잭션 개념을 적용하는 것이다.

JavaMail 처럼 확장이 불가능하게 설계해놓은 API를 추상화 계층의 도입으로 테스트할 수 있게 되었다.

5.4.4 테스트 대역

- 테스트 대역 : 테스트 환경을 만들어주기 위해 테스트 대상이 되는 오브젝트의 기능에만 충실하게 수행하면서 빠르게, 자주 테스트를실행할 수 있도록 사용하는 오브젝트

- 테스트 스텁 : 대표적인 테스트 대역, 테스트 대상 오브젝트의 의존 객체로서 존재하면서 테스트 동안에 코드가 정상적으로 수행할 수 있도록 돕는 것을 말한다.

- 테스트 대역 중에서 테스트 대상으로부터 전달받은 정보를 검증할 수 있도록 설계된 것을 목 오브젝트라고 한다.

