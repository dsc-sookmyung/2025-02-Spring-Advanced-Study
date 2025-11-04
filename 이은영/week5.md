# 5장. 서비스 추상화

## 5.1 사용자 레벨 관리 기능 추가

UserDao에 간단한 비즈니스 로직을 추가해보자

다수의 회원이 가입할 수 있는 인터넷 서비스의 사용자 관리 기능에서 구현해야 할 비즈니스 로직

▪️ 사용자의 레벨은 BASIC, SILVER, GOLD 세 가지 중 하나다.

▪️ 사용자가 처음 가입하면 BASIC 레벨이며, 이후 활동에 따라 한 단계씩 업그레이드 될 수 있다.

▪️ 가입 후 50회 이상 로그인을 하면 BASIC에서 SILVER 레벨이 된다.

▪️ SILVER 레벨이면서 30번 이상 추천을 받으면 GOLD 레벨이 된다.

▪️ 사용자 레벨의 변경 작업은 일정한 주기를 가지고 일괄적으로 진행된다. 변경 작업 전에는 조건을 충족하더라도 레벨의 변경이 일어나지 않는다.

**5.1.1 필드 추가**

숫자 타입을 직접사용하기 보다 Level Enum을 사용하자

```java
public enum Level {
    BASIC(1), SILVER(2), GOLD(3);

    private final int value;

    Level(int value){
        this.value = value;
    }

    public int intValue(){ // 값을 가져오는 메소드
        return value;
    }

    public static Level valueOf(int value){ // 값으로부터 Level 타입 오브젝트를 가져오도록 만든 스태틱 메소드
        switch (value){
            case 1: return BASIC;
            case 2: return SILVER;
            case 3: return GOLD;
            default: throw new AssertionError("Unknown value : " + value);
        }
    }
}
```

**5.1.3 UserService.upgradeLevels()**

레벨 관리 기능을 구현해보자

UserService 클래스를 생성하고 UserDao 오브젝트를 저장해둘 인스턴스 변수 선언하고, UserDao 오브젝트의 DI가 가능하도록 수정자 메소드를 추가한다.

```java
public class UserService {

	private UserDao userDao;

	public void setUserDao(UserDao userDao) {
		this.userDao = userDao;
	}
}
```

스프링 설정파일에 userService 아이디로 빈을 추가한다.

```html
<beans ...>
  <bean id="userService" class="springbook.user.service.UserService">
    <property name="userDao" ref="userDao" />
  </bean>
  <bean id="userDao" class="springbook.dao.UserDaoJdbc">
    <property name="dataSource" ref="dataSource" />
  </bean>
  <beans></beans
></beans>
```

```java
public void upgradeLevels() {
        List<User> users = userDao.getAll();
        for (User user : users) {
            Boolean changed = null; // 레벨의 변화가 있는지 확인하는 플래그
            if (user.getLevel() == Level.BASIC && user.getLogin() >= 50) {
                user.setLevel(Level.SILVER); // Basic 레벨 업그레이드
                changed = true;
            } else if (user.getLevel() == Level.SILVER && user.getRecommend() >= 30) {
                user.setLevel(Level.GOLD); // Silver 레벨 업그레이드
                changed = true; // 레벨 변경 플래그 설정
            } else if (user.getLevel() == Level.GOLD) {
                changed = false; // Gold 레벨은 변경X
            } else { // 일치하는 조건이 없으면 변경 X
                changed = false;
            }
            // 레벨 변경이 있는 경우에만 update() 호출
            if (changed) userDao.update(user);
        }
    }
}
```

**5.1.4 UserService.add()**

사용자 비즈니스 로직 중 처음 가입하는 사용자의 기본 레벨을 BASIC으로 설정해야 하는 기능을 아직 구현하지 않았다. 이 로직을 Service에서 구현해보자.

```java
public void add(User user) {
		if (user.getLevel() == null) user.setLevel(Level.BASIC);
		userDao.add(user);
}
```

```java
@Test
	public void add() {
		userDao.deleteAll();

		User userWithLevel = users.get(4);
		User userWithoutLevel = users.get(0);
		userWithoutLevel.setLevel(null);

		userService.add(userWithLevel);
		userService.add(userWithoutLevel);

		User userWithLevelRead = userDao.get(userWithLevel.getId());
		User userWithoutLevelRead = userDao.get(userWithoutLevel.getId());

		assertThat(userWithLevelRead.getLevel(), is(userWithLevel.getLevel()));
		assertThat(userWithoutLevelRead.getLevel(), is(Level.BASIC));
	}
```

테스트를 통해 UserService의 add()를 호출하면 레벨이 BASIC으로 설정되는 것을 검증해야한다.

- 두 가지의 테스트 케이스 → 두 가지 경우 모두 add() 메소드를 호출하고 결과를 확인

  ☑️ 레벨이 미리 정해진 경우

  ☑️ 레벨이 비어 있는 경우

- 변경된 레벨을 확인하는 두 가지 방법
  ☑️ 간단한 방법: add() 메소드를 호출할 때 파라미터로 넘긴 User 오브젝트에 level 필드를 확인
  ☑️ 다른 방법: get() 메소드를 이용해서 DB에 저장된 User 정보를 가져와 확인

**5.1.5 코드 개선**

비즈니스 로직의 구현을 모두 마쳤어도 아래의 기준을 가지고 코드를 다시 한번 검토 해보자.

▪️ 코드에 중복된 부분은 없는가?

▪️ 코드가 무엇을 하는 것인지 이해하기 불편하지 않은가?

▪️ 코드가 자신이 있어야 할 자리에 있는가?

▪️ 앞으로 변화가 일어난다면 그 변화에 쉽게 대응할 수 있게 작성되어 있는가?

**upgradeLevels() 리팩토링**

```java
public void upgradeLevels() {
     List<User> users = userDao.getAll();
     for (User user : users) {
         if(canUpgradeLevel(user)) {
             upgradeLevel(user);
         }
    }
}
```

: 모든 사용자 정보를 가져와 한 명씩 업그레이드가 가능한지 확인하고 가능하면 업그레이드를 진행한다.

```java
private boolean canUpgradeLevel(User user) {
    Level currentLevel = user.getLevel();

    switch(currentLevel) {
        case BASIC -> user.getLoginCount() >= 50;
        case SILVER -> user.getRecommendCount() >= 30;
        case GOLD -> return false;
        default -> throw new IllegalArgumentException("Unknown Level: " + currentLevel);
    }
}
```

: User 오브젝트에서 레벨을 가져와서, switch 문으로 구분하고, 업그레이드 조건을 만족하는지 확인하여 업그레이드가 가능하면 true, 그렇지 않으면 false를 리턴한다.

```java
private void upgradeLevel(User user) {
    if (user.getLevel() == Level.BASIC) user.setLevel(Level.SILVER);
    else if (user.getLevel() == Level.SILVER) user.setLevel(Level.GOLD);
    userDao.update(user);
}
```

: 사용자 오브젝트의 레벨정보를 다음 단계로 변경하고, 변경된 오브젝트를 DB에 업데이트한다.

```java
public enum Level {
    //  Enum 선언에 DB에 저장할 값과 함께 다음 단계의 레벨 정보도 추가한다
    GOLD(3, null), SILVER(2, GOLD), BASIC(1, SILVER);

    private final int value;
    private final Level next;

    Level(int value, Level next) {
        this.value = value;
        this.next = next;
    }

    public Level nextLevel() {
        return next;
    }

    public int intValue() {
        return value;
    }

    public static Level valueOf(int value) {
        return switch (value) {
            case 1 -> BASIC;
            case 2 -> SILVER;
            case 3 -> GOLD;
            default -> throw new AssertionError("Unknown value: " + value);
        };
    }
}
```

: Level Enum의 역할 확장을 통해 각 레벨의 다음 단계가 무엇인지 로직에서 반복적으로 조건문을 사용하지 않고 nextLevel()을 통해 얻을 수 있게 하자

```java
public void upgradeLevel() {
    Level nextLevel = this.level.nextLevel();

    if (nextLevel == null) {
       throw new IllegalStateException(this.level + "은 업그레이드가 불가능합니다.");
    } else {
        this.level = nextLevel;
    }
}
```

: 사용자 정보가 바뀌는 부분을 UserService 메소드에서 User로 옮겨 User가 레벨 업그레이드 작업을 스스로 처리하게 하자

## 5.2 트랜잭션 서비스 추상화

**모 아니면 도 :** 모든 사용자에 대한 업그레이드 작업은 전체 다 성공하든지 전체 다 실패해야 한다.

**JDBC 트랜잭션의 트랜잭션 경계설정**

- 트랜잭션 시작과 종료
  - 모든 트랜잭션은 시작과 종료 시점이 있다. 트랜잭션은 작업을 완료하고 확정하는 **커밋** 또는 작업을 무효화하는 **롤백** 중 하나로 종료된다.
  - 애플리케이션 내에서 트랜잭션이 시작되고 종료되는 위치를 **트랜잭션 경계**라고 하며, 올바르게 설정하는 것이 중요하다.
- JDBC를 이용한 간단한 트랜잭션 예제

```java
// DB 커넥션 시작
Connection c = dataSource.getConnection();

// 트랜잭션 시작
c.setAutoCommit(false);

try {
    PreparedStatement st1 = c.prepareStatement("update users ...");
    st1.executeUpdate();

    PreparedStatement st2 = c.prepareStatement("delete users ...");
    st2.executeUpdate();

    // 트랜잭션 커밋
    c.commit();
} catch (Exception e) {
    // 트랜잭션 롤백
    c.rollback();
}

// DB 커넥션 종료
c.close();

```

- 트랜잭션의 경계 설정은 Connection을 사용해 트랜잭션을 시작하고 종료하는 작업을 의미한다.
- JDBC 트랜잭션은 하나의 Connection 객체에서 일어나며, `setAutoCommit(false)` 메서드로 트랜잭션을 시작한다.
- JDBC의 기본 설정은 자동 커밋 모드인데, 이는 각 DB 작업이 끝날 때마다 자동으로 커밋되므로, 여러 DB 작업을 하나의 트랜잭션으로 묶을 수 없다. 자동 커밋을 `false`로 설정하면 **트랜잭션 시작**을 선언한 것이 된다.
- 이후 `commit()` 메서드를 호출하면 트랜잭션이 완료되어 DB에 작업 결과가 반영되고, `rollback()` 메서드를 호출하면 작업 결과가 취소된다. 일반적으로 예외 발생 시 트랜잭션은 롤백된다.

**트랜잭션 동기화**

UserService에서 트랜잭션을 시작하기 위해 만든 Connection 오브젝트를 특별한 저장소에 보관해두고, 이후에 호출되는 DAO 의 메소드에서는 저장된 Connection을 가져다가 사용하게 하는 것

🪻 작업흐름

UserService가 **Connection 생성**, 트랜잭션 동기화 저장소에 Connection 저장

Connection.setAutoCommit()을 호출해 **트랜잭션 시작**시킨 후에 본격적으로 DAO의 기능을 이용

**DAO 기능 사용** : JdbcTemplate 메소드 호출

▪️ 현재 시작된 트랜잭션을 가진 Connection 오브젝트의 존재 확인

▪️ 저장해둔 Connection 받아서 작업 수행

▪️ 트랜잭션을 종료하기 전까지는, 계속 같은 Connection을 사용하고, Connection을 닫지 않는다.

**종료**

▪️ 작업이 정상적으로 끝난 경우, Connection.commit() 호출

▪️ 예외상황이 발생한 경우, Connection.rollback() 호출

▪️ Connection 오브젝트 제거

## **5.3 서비스 추상화와 단일 책임 원칙**

**수직, 수평 계층구조와 의존관계**

기술과 서비스에 대한 추상화 기법을 이용하면 특정 기술환경에 종속되지 않는 포터블한 코드를 만들 수 있었다.

수평적인 구분이든, 수직적인 구분이든 모두 결합도가 낮으며, 서로 영향을 주지 않고 자유롭게 확장될 수 있는 구조를 만들 수 있는 데는 스프링의 DI가 중요한 역할을 한다.

→ 🌟 이렇게 관심, 책임, 성격이 다른 코드를 깔끔하게 분리하는것이 DI의 가치 🌟

**단일 책임 원칙**

UserService에 JDBC 커넥션 메소드를 직접 사용하는 트랜잭션 코드가 들어 있었을 때에는 UserService가 사용자 레벨을 관리하는 것과 트랜잭션을 관리하는 것, 총 두 가지 책임을 가지고 있었다. 이렇게 두 가지 책임을 갖는다는 것은 수정되는 이유가 두 가지라는 뜻이다 ..

트랜잭션 서비스의 추상화 방식을 도입하고, DI를 통해 외부에서 제어하도록 만들고 나서는 바뀔 이유는 한가지 뿐이게 되었다. 이로써, 단일 책임 원칙을 충실하게 지키고 있다고 할 수 있는 것이다.

**단일 책임 원칙의 장점**

**👁️‍🗨️ 변경 시 수정 대상의 명확성**

- 단일 책임 원칙을 지키면 변경이 필요한 부분을 쉽게 찾아 수정할 수 있다
- 예를 들어, 기술이 바뀌면 기술 추상화 계층의 설정만 변경하면 되고, 데이터베이스 테이블 이름이 바뀌면 UserDao만 수정하면 된다.

**👁️‍🗨️ 복잡한 구조에서도 유지보수 용이성**

- 모듈이 많아질 경우 단일 책임 원칙을 지키지 않으면 의존 관계가 복잡해지고, 특정 DAO가 변경될 때 그에 의존하는 서비스 클래스들도 수정해야 하는 문제가 발생한다.

**👁️‍🗨️ 기술 변경 시 유연성 제공**

- 애플리케이션 코드가 특정 기술에 종속되지 않도록 설계하면, 기술이 바뀌어도 XML 설정만으로 트랜잭션 기술을 한 번에 전환할 수 있다.
- 코드 수정을 최소화해 실수를 줄이고, 치명적인 버그의 가능성을 낮춘다.

1. UserService가 **Connection 생성**, **트랜잭션 동기화 저장소에 Connection 저장**
2. **트랜잭션 시작 : Connection.setAutoCommit()**
3. DAO 기능 사용 : **JdbcTemplate 메소드 호출**
   1. 현재 시작된 트랜잭션을 가진 **Connection 오브젝트의 존재 확인**
   2. **저장해둔 Connection 받아서 작업 수행**
   3. 트랜잭션을 **종료하기 전까지는, 계속 같은 Connection을 받아와서 사용**하고, Connection을 닫지 않는다
4. 종료
   1. 작업이 정상적으로 끝난 경우, Connection.**commit()** 호출
   2. 예외상황이 발생한 경우, Connection.**rollback()** 호출
   3. **Connection 오브젝트 제거**
