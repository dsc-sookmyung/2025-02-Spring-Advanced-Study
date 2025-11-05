# 5장 서비스 추상화

스프링이 **어떻게 성격이 비슷한 여러 종류의 기술을 추상화** 하고 이를 **일관된 방법으로 사용**할 수 있도록 지원하는지 살펴보자.

## 5.1 사용자 레벨 관리 기능 추가

지금까지 만들어왔던 `UserDao`는 `User` 오브젝트에 담겨 있는 사용자 정보를 **등록, 조회, 수정, 삭제**하는 일명 **CRUD**라고 불리는 가장 기초적인 작업만 할 수 있다. 이제 정기적으로 사용자의 **활동 내역**을 참고해서 레벨을 조정하는 기능을 추가해보자.

아래는 사용자 관리 기능에서 구현할 비즈니스 로직이다.
- 사용자의 레벨은 BASIC, SILVER, GOLD 세 가지 중 하나
- 처음 가입 시 BASIC 레벨, 이후 활동에 따라 한 단계씩 업그레이드 가능
- 가입 후 50회 이상 로그인 시 BASIC에서 SILVER로 업그레이드
- SILVER 레벨이자 30번 이상 추천 받을 시 GOLD로 업그레이드
- 레벨 변경 작업은 일정한 주기를 가지고 일괄적으로 진행 & 변경 작업 전 조건 충족 시 레벨 변경 없음

### 필드 추가

첫 요구사항을 충족하기 위해 Level을 만들어야 한다고 가정하자. Level을 저장할 때, DB에는 varchar 타입으로 선언하고, "BASIC", "SILVER", "GOLD"로 저장할 수도 있겠지만, **약간의 메모리라도 중요한 케이스**라고 가정하고, **각 레벨을 코드화**해서 숫자로 넣는다고 가정하자.
숫자로 넣기로 했다고 가정하면, `User` 객체에 추가할 프로퍼티도 Integer 타입의 level 프로퍼티를 만드는 것이 좋을까? 상수적이며 범위가 한정적인 데이터를 코드화해서 사용할 때는 ENUM을 이용해 구성하는 편이 좋다. 왜냐하면 단순히 1, 2, 3과 같은 코드 값을 넣으면 작성자 외에는 1이 어떤 Level을 가리키는 것인지 알 방법이 없다.

⭐ Enum을 사용하지 않은 사용자 레벨 코드
```java
public class User {
    private static final int BASIC = 1;
    private static final int SILVER = 2;
    private static final int GOLD = 3;

    int level;

    public setLevel(int level) {
        this.level = level;
    }
    ...
```

Enum을 사용하지 않은 코드에서는 누군가 그냥 0, 4, 5 등 우리가 정의한 **Level의 코드 범위에 속하지 않는 값**을 넣으면 속수무책으로 당하고 만다. 정확하게 코드를 운영하기 위해 **Level의 도메인 자체를 ENUM 클래스로 분리해서 관리**하는 편이 훨씬 깔끔하다. ENUM 클래스로 분리하면 자연적으로 **허가되지 않은 단순한 int 값은 못 들어오며**, 추후에 Level에 대한 요구사항이 확장되었을 때도 **해당 도메인에 대한 코드 확장이 용이**해진다.

⭐ 사용자 레벨용 Enum
```java
public enum Level {
    BASIC(1), SILVER(2), GOLD(3);   // 세 개의 Enum 오브젝트 정의

    private final int value;

    Level(int value) {      // DB에 저장할 값을 넣어줄 생성자
        this.value = value;
    }

    public int intValue() {     // 값을 가져오는 메소드
        return value;
    }

    public static Level valueOf(int value) {    // 값으로부터 Level 타입 오브젝트를 가져오도록 만든 스태틱 메소드 
        return switch (value) {
            case 1 -> BASIC;
            case 2 -> SILVER;
            case 3 -> GOLD;
            default -> throw new AssertionError("Unknown value: " + value);
        };
    }
}
```

Level 도메인에 대한 책임을 맡을 훌륭한 ENUM 클래스가 생성되었다. 이제 컴파일 타임에 잘못된 int 값이 `setLevel()`로 들어올 위험성은 줄였다. ENUM 클래스로 생성한 Level과 함께 로그인 회수를 카운트할 `loginCount`과 추천 회수를 카운트할 `recommendCount`도 추가했다.

⭐ `User` 필드 추가
```java
public class User {
    ...
    Level level;
    int loginCount;
    int recommendCount;

    public Level getLevel() {
        return level;
    }
    ...
    // login, recommend getter/setter 생략
```

⭐ 추가된 필드를 파라미터로 포함하는 생성자
```java
public User(String id, String name, String password, Level level, int loginCount, int recommendCount) {
        this.id = id;
        this.name = name;
        this.password = password;
        this.level = level;
        this.loginCount = loginCount;
        this.recommendCount = recommendCount;
    }
```

등록을 위한 INSERT 문장이 들어 있는 `add()` 메소드의 SQL과 각종 조회 작업에 사용되는 `User` 오브젝트 매핑용 콜백인 `userMapper`에 추가된 필드를 넣는다.

⭐ UserDaoJdbc 수정
```java
public UserDaoJdbc() {
        this.userRowMapper = (rs, rowNum) -> {
            User user = new User();
            user.setId(rs.getString("id"));
            user.setName(rs.getString("name"));
            user.setPassword(rs.getString("password"));
            user.setLevel(Level.valueOf(rs.getInt("level")));
            user.setLoginCount(rs.getInt("login_count"));
            user.setRecommendCount(rs.getInt("recommend_count"));
            return user;
        };
    }
```

**JDBC가 사용하는 SQL**은 컴파일 과정에서는 **자동으로 검증이 되지 않는 단순 문자열**에 불과하다. 그러나, 우리는 꼼꼼하게 `UserDao`에서 생성한 모든 메소드에 대한 테스트를 작성해두었기 때문에 실제 서비스로 올라가기 전에 테스트만 돌려봤어도 해당 에러를 잡을 수 있었을 것이다. **테스트를 작성하지 않았다면, 실 서비스 실행 중에 예외가 날아다녔을 것**이고, 한참 후에 수동 테스트를 통해 메세지를 보고 디버깅을 해야 그제서야 겨우 오타를 확인할 수 있었을 것이다. 그때까지 진행한 빌드와 서버 배치, 서버 재시작, 수동 테스트 등에 소모한 시간은 낭비에 가깝다. 빠르게 실행 가능한 포괄적인 테스트를 만들어두면 이렇게 기능의 추가나 수정이 일어날 때 그 위력을 발휘한다.

### 사용자 수정 기능 추가

사용자 관리 비즈니스 로직에 따르면 사용자 정보는 여러번 수정될 수 있다. 때때론 성능 최적화를 위해 수정되는 필드의 종류에 따라 여러 개의 수정용 DAO 메소드를 만들어야 할 때도 있지만, 아직 사용자 정보가 단순하고 필드도 몇개 되지 않고 수정이 자주 일어나지 않으므로 간단히 접근해보자.

⭐ `update()` 메소드 추가
```java
public interface UserDao {
    void add(User user);
    User get(String id);
    User getByName(String name);
    List<User> getAll();
    void deleteAll();
    int getCount();
    void update(User user1);
}
```

⭐ 사용자 정보 수정용 `update()` 메소드
```java
public void update(User user) {
        this.jdbcTemplate.update(
                "update users set name = ?, password = ?, level = ?, login_count = ?, recommend_count = ? where id = ? "
                , user.getName(), user.getPassword(), user.getLevel().intValue(), user.getLoginCount(), user.getRecommendCount(), user.getId()
        );
    }
```

### UserService.upgradeLevels()

`UserDao`의 `getAll()` 메소드로 **사용자를 모두 가져와 사용자별로 레벨 업그레이드** 작업을 진행하며 `UserDao`의 `update()`를 호출해 DB에 결과를 넣어주자. 사용자 관리 로직을 담을 클래스를 하나 추가하여 이름은 `UserService`로 한다. 이 클래스는 `UserDao`의 인터페이스 타입으로 `userDao` 빈을 DI 받아 사용하게 만든다. DAO의 인터페이스를 사용하고 DI를 적용하기 위해 `UserService`도 스프링의 빈으로 등록한다. 

⭐ UserService 클래스
```java
public class UserService {
    UserDao userDao;

    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }
}
```

먼저 모든 사용자 정보를 DAO에서 가져와 한 명씩 레벨 변경 작업을 수행한다. 현재 사용자 레벨이 변경 되었는지 확인하기 위해 **플래그**를 하나 선언할 것이다. 예를 들어, BASIC 레벨이면서 로그인 조건을 만족한다면, `User` 오브젝트의 레벨을 SILVER로 변경하고 레벨 변경 플래그인 `changed`를 `true`로 설정하는 것이다. `changed` 플래그로 레벨 변경이 있는 것을 확인한 경우에만 `UserDao`의 `update()`를 이용해 수정 내용을 DB에 반영한다.

⭐ upgradeLevels() 메소드
```java
public void upgradeLevels() {
        List<User> users = userDao.getAll();

        for (User user : users) {
            Boolean changed = null;     // 레벨의 변화가 있는지 확인하는 플래그

            if (user.getLevel() == Level.BASIC && user.getLoginCount() >= 50) {     
                user.setLevel(Level.SILVER);
                changed = true;                                                     
            }                                   // -- BASIC 레벨 업그레이드
            else if (user.getLevel() == Level.SILVER && user.getRecommendCount() >= 30) {
                user.setLevel(Level.GOLD);
                changed = true;
            }                                   // -- SILVER 레벨 업그레이드 
            else if (user.getLevel() == Level.GOLD) {
                changed = false;
            }                                   // -- GOLD 레벨은 변경 X
            else {
                changed = false;
            }                                   // 일치하는 조건 없으면 변경 X

            if(changed) {
                userDao.update(user);
            }                                   // 레벨 변경 있는 경우에만 update() 호출
        }
    }
```

### UserService.add()

`UserDao`의 `add()` 메소드는 사용자 정보를 담은 `User` 오브젝트를 받아서 DB에 넣어주는 데 충실한 역할을 한다면, `UserService`에도 `add()`를 만들어두고 사용자가 등록될 때 적용할만한 비즈니스 로직을 담당하게 하면 될 것이다. `UserDao`와 같이 리포지토리 역할을 하는 클래스를 컨트롤러에서 바로 쓰냐 마냐에 대한 논쟁이 있는데, 바로 쓰면 아무런 비즈니스 로직이 들어가지 않은 순수한 CRUD의 의미일 것이다.

⭐ UserService의 add 메소드
```java
public void add(User user) {

        // 간단히 level이 null이라면, Level.BASIC 삽입
        if(user.getLevel() == null) { user.setLevel(Level.BASIC); }

        userDao.add(user);
    }
```

<br>

## 5.2 트랜잭션 서비스 추상화

### 트랜잭션 경계설정

DB는 사실 그 자체로 완벽한 트랜잭션을 지원한다. 우리가 SQL 명령어로 다수의 ROW를 건드렸을 때, 하나의 ROW에만 반영되고 나머지 ROW에는 SQL 명령이 들어가지 않는 경우를 본 적이 없을 것이다. 하나의 SQL명령을 처리하는 경우에는 DB가 트랜잭션을 보장해준다고 믿을 수 있다.

트랜잭션은 시작 지점과 끝 지점이 있다. 시작 지점의 위치는 한 곳이며, 끝 지점의 위치는 두 곳이다. 끝날 때는 롤백되거나 커밋될 수 있다. 애플리케이션 내에서 트랜잭션이 시작되고 끝나는 위치를 트랜잭션의 경계라고 부른다. 복잡한 로직 흐름 사이에서 정확하게 트랜잭션 경계를 설정하는 일은 매우 중요하다.

⭐ 트랜잭션을 사용한 JDBC 코드
```java
Connection c = dataSource.getConnection();

c.setAutoCommit(false); // 트랜잭션 경계 시작
try {
  PreparedStatement st1 =
    c.prepareStatement("update users ...");
  st1.executeUpdate();

  PreparedStatement st2 =
    c.prepareStatement("delete users ...");
  st2.executeUpdate();

  c.commit(); // 트랜잭션 경계 끝지점 (커밋)
} catch(Exception e) {
  c.rollback(); // 트랜잭션 경계 끝지점 (롤백)
}

c.close();
```

### 트랜잭션 동기화

독립적인 트랜잭션 동기화(transaction synchronization) 방식이다. 트랜잭션 동기화란 `UserService`에서 트랜잭션을 시작하기 위해 만든 Connection 오브젝트를 특별한 장소에 보관해두고, 이후에 호출되는 DAO의 메소드에서는 저장된 Connection을 가져다가 사용하게 하는 것이다. 정확히는 DAO가 사용하는 JdbcTemplate이 트랜잭션 동기화 방식을 이용하도록 하는 것이다. 그리고 트랜잭션이 모두 종료되면 그 때는 동기화를 마치면 된다.

⭐ 트랜잭션 동기화 방식을 적용한 UserService
```java
private DataSource dataSource;

public void setDataSource(DataSource dataSource) {
    this.dataSource = dataSource;
}

public void upgradeLevels() throws Exception {
    TracnsactionSynchronizationManager.initSynchronization();

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

    } catch (Exception e) {
        c.rollback();
        throw e;
    } finally {
        DataSourceUtils.releaseConnection(c, dataSource);
        TracnsactionSynchronizationManager.unbindResource(this.dataSource);
        TracnsactionSynchronizationManager.clearSynchronization();
    }
}
```

JdbcTemplate은 JDBC를 사용할 때 까다로울 수 있는
- try/catch/finally 작업 흐름 지원
- SQLException 예외 변환
- 트랜잭션 동기화 관리
와 같은 작업들에 대한 **템플릿**을 제공하여, 개발자가 비즈니스 로직에 집중할 수 있고 애플리케이션 레이어를 설계하기 좋은 환경을 만들어준다.

### 트랜잭션 서비스 추상화

스프링은 **트랜잭션 기술의 공통점**을 담은 **트랜잭션 추상화 기술**을 제공한다. 이를 이용하면 특정 기술에 종속되지 않고 트랜잭션 경계 설정 작업이 가능해진다.

⭐ 스프링의 트랜잭션 추상화 API를 적용한 upgradeLevels()
```java
public void upgradeLevels() {
        // 트랜잭션 시작
        TransactionStatus status =
                transactionManager.getTransaction(new DefaultTransactionDefinition());  

        try {
            List<User> users = userDao.getAll();
            for (User user : users) {
                if (canUpgradeLevel(user)) {
                    upgradeLevel(user);
                }
            }

            transactionManager.commit(status);
        }catch(Exception e) {
            transactionManager.rollback(status);
            throw e;
        }
    }
```

<br>

## 5.3 서비스 추상화와 단일 책임 원칙

### 수직, 수평 계층구조와 의존관계

UserDao와 UserService는 각각 담당하는 코드의 기능적인 관심에 따라 분리되었다. 사실 둘은 같은 애플리케이션 로직을 담은 코드이지만 내용에 따라 분리하여, 수평적으로 분리했다고 볼 수 있다. 트랜잭션의 추상화는 이와는 좀 다르다. 애플리케이션의 비즈니스 로직과 그 하위에서 동작하는 로우레벨의 트랜잭션 기술이라는 아예 다른 계층의 특성을 갖는 코드를 분리한 것이다.

### 단일 책임 원칙

하나의 모듈은 한 가지의 책임을 가져야 한다는 의미다. 다른 말로 풀면 하나의 모듈이 바뀌는 이유는 한 가지여야 한다고 설명할 수도 있다.

트랜잭션을 구현하기 위해 UserService에 JDBC 코드가 들어가있을 때는 UserService의 책임은 두 가지였다. "어떻게 사용자 레벨을 관리할 것인가 & 어떻게 트랜잭션을 관리할 것인가". 책임이 두 가지라는 것은 코드가 수정되는 이유도 두 가지라는 뜻이다.

레벨 관리 로직이 바뀌면 UserService를 수정해야 하고 트랜잭션 기술이 바뀌면 UserService를 수정해야 한다. 결국 이렇게 두 가지 이상의 책임을 가지는 순간 단일 책임 원칙은 깨지는 것이다.

<br>