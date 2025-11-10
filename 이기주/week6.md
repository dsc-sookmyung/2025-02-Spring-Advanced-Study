# 6장 AOP
**선언적 트랜잭션 기능**에 AOP가 적용 된다. 트랜잭션 경계설정 기능을 AOP를 이용해 깔끔하게 바꿀 것이다.

## 6.1 트랜잭션 코드의 분리
스프링이 제공하는 트랜잭션 인터페이스를 쓴 `UserService`에는 여전히 비즈니스 로직이 잘 보이지 않고 트랜잭션 코드가 더 많이 보이는 상황이다. 사실은 트랜잭션 경계설정 코드와 비즈니스 로직 코드는 잘 구분되어 있다. 다만, 이 두 코드 간에 서로 주고받는 정보는 없다. 다시 말해, 비즈니스 로직 코드에서는 직접 DB를 사용하지 않기 때문에 **트랜잭션 정보**는 트랜잭션 동기화 방법을 통해 **DAO가 알아서 활용**한다.

따라서 이 두 코드를 메소드로 분리하는 것이 좋을 듯하다. 아래 코드는 사용자 레벨 업그레이드를 담당하는 비즈니스 로직 코드만 독립적인 메소드에 담겨 있어 이해하기 쉽고 수정에도 부담이 없으며 트랜잭션 코드를 건드릴 염려도 없다.

⭐ 비즈니스 로직과 트랜잭션 경계설정의 분리
```java
public void upgradeLevels() throws Exception {
    TransactionStatus status = this.transactionManager
        .getTransaction(new DefaultTransactionDefinition());
    try {
        upgradeLevelsInternal();
        this.transactionManager.commit(status);
    } catch (Exception e) {
        this.transactionManager.rollback(status);
        throw e;
    }
}

private void upgradeLevelsInternal() {      // 분리된 비즈니스 로직 코드로,
    List<User> users = userDao.getAll();    // 트랜잭션 적용 전과 동일
    for (User user : users) {
        if (canUpgradeLevel(user)) {
            upgradeLevel(user);
        }
    }
}
```

그러나 위 코드에는 아직 트랜잭션 담당을 위한 기술적 코드가 `UserService` 안에 있다. 비즈니스 로직 코드와 트랜잭션 관련 코드는 어차피 정보를 주고 받지 않으니, `UserService` **클래스 밖으로** 뽑아내보자.

`UserService`를 인터페이스로 만들고 기존 코드는 `UserService` 인터페이스의 구현 클래스를 만들어 넣도록 하자. 이렇게 되면 클라이언트와 결합이 약해지면서 구현 클래스에 의존하고 있지 않기 때문에 유연하게 확장할 수 있다. 이때, **구현 클래스를 하나 더** 만들 것이다. 따라서 사용자 관리 로직을 담고 있는 구현 클래스인 `UserServiceImpl`과 트랜잭션 경게설정을 위한 구현 클래스인 `UserServiceTx` 두 개로 구성되는 것이다.

- `UserService` 클래스를 `UserServiceImpl`로 이름 변경
- 클라이언트가 사용할 로직을 담은 핵심 메소드만 `UserService` 인터페이스로 만들기, 구현은 `UserServiceImpl`이 담당

⭐ `UserService` 인터페이스
```java
public interface UserService {
    void add(User user);
    void upgradeLevels();
}
```

⭐ 트랜잭션 코드를 제거한 `UserService` 구현 클래스
```java
package springbook.user.service;

public class UserServiceImpl implements UserService {
    UserDao userDao;
    MailSender mailSender;

    public void upgradeLevels() {
        List<User> users = userDao.getAll();
        for (User user : users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(user);
            }
        }
    }
    ... 
}
```

다음 코드는 **비즈니스 트랜잭션 처리**를 담은 `UserServiceTx`이다. `UserServiceTx`는 `UserService`를 기본적으로 구현하며 같은 인터페이스를 구현한 다른 오브젝트에게 고스란히 작업을 위임한다. 클라이언트에 대해 `UserService` 타입 오브젝트의 하나로서 행세하며 `UserService` 오브젝트를 DI 받는다.

`transactionManager` 이름의 빈으로 등록된 트랜잭션 매니저를 DI로 받아뒀다가 트랜잭션 안에서 동장하도록 만들어줘야 하는 메소드 호출의 전후에 필요한 트랜잭션 경계설정 API를 사용한다.

⭐ 트랜잭션이 적용된 `UserServiceTx`
```java
/* 리스트 6-6 트랜잭션이 적용된 UserServiceTx */
public class UserServiceTx implements UserService {
    UserService userService;
    PlatformTransactionManager transactionManager;

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public void setUserService(UserService userService) {
        this.userService = userService;
    }

    public void add(User user) {
        this.userService.add(user);
    }

    public void upgradeLevels() {
        TransactionStatus status = this.transactionManager
            .getTransaction(new DefaultTransactionDefinition());

        try {
            userService.upgradeLevels();

            this.transactionManager.commit(status);
        } catch (RuntimeException e) {
            this.transactionManager.rollback(status);
            throw e;
        }
    }
}
```

**트랜잭션 경계설정 코드 분리의 장점**
1. 비즈니스 로직을 담당하고 있는 `UserServiceImpl`의 코드 작성 시 트랜잭션과 같은 기술적인 내용에 전혀 신경쓰지 않아도 된다.
    - 트랜잭션은 DI를 이용하여 `UserServiceTx`와 같은 *트랜잭션 기능을 가진 오브젝트*가 먼저 실행되도록 만들기만 하면 된다.
    - 비즈니스 로직을 잘 이해하고 자바 언어 기초에 충실하면 복잡한 비즈니스 로직을 담은 `UserService` 클래스 개발 가능하다.
2. 비즈니스 로직에 대한 테스트를 손쉽게 만들어낼 수 잇다.

<br>

## 6.2 고립된 단위 테스트
작은 단위의 테스트는 테스트 실패 시 그 **원인을 찾기 쉽다.** 테스트 단위가 작아야 **테스트의 의도와 내용이 분명하고, 만들기 쉽다.**

현재는 `UserService` 테스트에는 그 뒤에 존재하는 훨씬 더 많은 오브젝트와 환경, 서비스, 서버, 네트워크까지 함께 테스트해야 할만큼 모두가 의존하고 있는 상황이다. 따라서 테스트의 대상이 환경이나, 외부 서버, 다른 클래스의 코드에 종속되고 영향을 받지 않도록 **고립**시킬 필요가 있다. 이를 위해 테스트를 위한 대역을 사용할 것이다.

`UserServiceImpl`의 `upgradeLevels()` 메소드는 void 타입으로 테스트가 불가능하다. 이럴 때 테스트 대상인 `UserServiceImpl`과 그 협력 오브젝트인 `UserDao`에게 어떤 요청을 했는지 확인하는 작업이 필요하다. 결과가 반영되었는지는 알 수 없지만 `UserDao`의 `update()` 메소드가 호출된 것을 확인할 수 있다면, DB에 결과가 반영되었다고 결론을 내릴 수 있기 때문이다.

아래 코드는 고립된 단위 테스트 방법을 적용한 사례이며, 다섯 단계의 작업으로 구성된다.

1. 테스트 실행 중 `UserDao`를 통해 가져올 테스트용 정보를 DB에 저장
2. 메일 발송 여부 확인 위해 `MailSender` 목 오브젝트를 DI
3. 실제 테스트 대상인 `userService`의 메소드 실행
4. 결과가 DB에 반영됐는지 확인 위해 `UserDao`를 이용해 DB에서 데이터 가져와 결과 확인
5. 목 오브젝트 통해 `UserService`에 의한 메일 방송이 있었는지 확인

⭐ `upgradeLevels()` 테스트
```java
/* 리스트 6-10 upgradeLevels() 테스트 */
@Test
public void upgradeLevels() throws Exception {
    userDao.deleteAll();
    for(User user : users) userDao.add(user); // DB 테스트 데이터 준비

    MockMailSender mockMailSender = new MockMailSender(); // 메일 발송 여부 확인을 ~
    userServiceImpl.setMailSender(mockMailSender);        // ~ 위해 목 오브젝트 DI

    userService.upgradeLevels(); // -> 테스트 대상 실행

    checkLevelUpgraded(users.get(0), false);
    checkLevelUpgraded(users.get(1), false);
    checkLevelUpgraded(users.get(2), true); 
    checkLevelUpgraded(users.get(3), true);
    checkLevelUpgraded(users.get(4), false);    // DB에 저장된 결과 확인

    List<String> request = mockMailSender.getRequests(); 
    assertThat(request.size(), is(2));                  
    assertThat(request.get(0), is(users.get(1).getEmail()));    // 목 오브젝트를 이용한 ~
    assertThat(request.get(1), is(users.get(3).getEmail()));    // ~ 결과 확인
}

private void checkLevelUpgraded(User user, boolean upgraded) {
    User userUpdate = userDao.get(user.getId());
    ...
}
```

<br>

## 6.3 다이내믹 프록시와 팩토리 빈
분리된 부가기능을 담은 클래스는 부가기능 외의 *나머지 모든 기능은 원래 핵심기능을 가진 클래스로 위임*해줘야 한다. 핵심기능은 부가기능을 가진 클래스의 존재 자체를 모르기 때문에 부가기능이 핵심기능을 사용하는 구조가 되어야 한다.

위의 설명처럼 구성했더라도 클라이언트가 핵심기능을 가진 클래스를 직접 사용해 버린다면 부가기능이 적용될 기회가 없다는 점이다. 그래서 부가기능은 마치 자신이 핵심기능을 가진 클래스인 것처럼 꾸며서, 클라이언트가 자신을 거쳐서 핵심기능을 사용하도록 만들어야 한다. 이를 위해 클라이언트는 **인터페이스를 통해서만 핵심기능을 사용**하게 하고, 부가기능 자신도 같은 인터페이스를 구현한 뒤에 **자신이 그 사이에 끼어들어야** 한다.

프록시: 자신이 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 것
타깃 or 실체: 프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트

프록시의 사용 목적
1. 클라이언트가 타깃에 접근하는 방법 제어
2. 타깃에 부가적인 기능을 부여

데코레이터 패턴: 타깃에 부가적인 기능을 런타임 시 다이내믹하게 부여해주기 위해 프록시를 사용하는 패턴, 프록시가 하나로 제한되지 않는다. 같은 인터페이스를 구현한 타켓과 여러 개의 프록시를 사용할 수 있다. 프록시가 여러 개인 만큼 순서를 정해서 단계적으로 위임하는 구조로 만들어야 한다.

>>> 다이내믹하게 기능 부여? 컴파일 시점, 즉 코드상에서는 어떤 방법과 순서로 프록시와 타깃이 연결되어 사용되는지 정해져 있지 않다는 의미!

---

`UserServiceTx`를 다이내믹 프록시 방식으로 변경. `UserServiceTx`는 서비스 인터페이스의 메소드를 모두 구현해야 하며 트랜잭션이 필요한 메소드마다 트랜잭션 처리코드가 중복돼서 나타나는 비효율적인 방법으로 구현되어 있다.

이를 해결하기 위해 트랜잭션 부가기능을 제공하는 다이내믹 프록시를 만들어 적용하자. 트랜잭션 기능을 부가해주는 `InvocationHandler`는 한 개만 있어도 충분하다. 아래 코드에서는 요청을 위임할 타깃을 DI로 제공받는다. 타깃을 저장할 변수는 `Object`로 선언했다.

⭐ 다이내믹 프록시를 위한 트랜잭션 부가기능
```java
public class TransactionHandler implements InvocationHandler {
    private Object target;          // 부가기능을 제공할 타깃 오브젝트, 어떤 타입의 오브젝트에도 적용 가능하다.
    private PlatformTransactionManager transactionManager;      // 트랜잭션 기능을 제공하는 데 필요한 트랜잭션 매니저
    private String pattern;         // 트랜잭션을 적용할 메서드 이름 패턴

    public void setTarget(Object target) {
        this.target = target;
    }

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public void setPattern(String pattern) {
        this.pattern = pattern;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getName().startsWith(pattern)) { // 트랜잭션 적용 대상 메소드를 선별해서
            return invokeInTransaction(method, args); // 트랜잭션 경계설정 기능을 부여한다.
        } else {
            return method.invoke(target, args);
        }
    }

    private Object invokeInTransaction(Method method, Object[] args) throws Throwable {
        TransactionStatus status = this.transactionManager
            .getTransaction(new DefaultTransactionDefinition());

        try {
            Object ret = method.invoke(target, args); // 트랜잭션을 시작하고 타깃 오브젝트의 메소드를 호출한다. 예외가 발생하지 않았다면 커밋한다.
            this.transactionManager.commit(status);
            return ret;
        } catch (InvocationTargetException e) {
            this.transactionManager.rollback(status); // 예외가 발생하면 트랜잭션을 롤백한다.
            throw e.getTargetException();
        }
    }

}
```

---

**프록시 팩토리 빈 방식 장점**
- 타깃 인터페이스를 구현하는 클래스를 일일이 만들지 않아도 된다.
- 하나의 핸들러 메소드를 구현하는 것만으로도 수많은 메소드에 부가기능 부여, 부가기능 코드의 중목 문제도 사라진다.

**프록시 팩토리 빈 한계점**
- 한 번에 여러 개의 클래스에 공통적인 부가기능을 제공하는 일은 불가능하다.
    - 트랜잭션과 같이 비즈니스 로직을 담은 많은 클래스의 메소드에 적용할 필요가 있다면 거의 비슷한 프록시 팩토리 빈의 설정이 중복되는 것을 막을 수 없다.
- 하나의 타깃에 여러 개의 부가기능을 적용하기 어렵다.
    - XML 설정을 사람이 할 수 없을 정도로 길어지며 복잡해진다.
- `TransactionHandler` 오브젝트가 프록시 팩토리 빈 개수만큼 만들어진다.
    - 트랜잭션 부가기능을 제공하는 동일한 코드임에도 타깃 오브젝트가 달라지면 새로운 `TransactionHandler` 오브젝트를 만들어야 한다.
