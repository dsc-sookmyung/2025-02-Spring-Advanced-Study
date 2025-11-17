# 6장 AOP(6.1-6.3)

### 6.1 트랜잭션 코드의 분리

현재 코드에서는 비즈니스 로직이 주가 되어야 하는 반면 트랜잭션 코드가 많은 자리를 차지하고 있다.

성격이 다른 코드를 두개의 메소드로 분리하자.

```java
public void upgradeLevels() {
    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    try{
        upgradeLevelsInternal();
        this.transactionManager.commit(status);
    } catch (Exception e) {
        this.transactionManager.rollback(status);
        throw e;
    }
}

private void upgradeLevelsInternal() {
    List<User> users = userDao.getAll();
    for(User user : users) {
        if(canChangedLevel(user)) {
            upgradeLevel(user);
        }
    }
}
```

인터페이스를 이용하여, 비즈니스 로직에만 충실한 깔끔한 코드가 되도록 하자.

```java
public interface UserService {
    void add(User user);
    void upgradeLevels();
}
```

```java
public class UserServiceImpl implements UserService {
    UserDao userDao;

    ...

    public void upgradeLevels() {
        List<User> users = userDao.getAll();
        for(User user : users) {
            if(canChangedLevel(user)) {
                upgradeLevel(user);
            }
        }
    }

    ...
}
```

비즈니스 트랜잭션 처리를 담은 UserServiceTx를 만들어서 사용자 관리라는 비즈니스 로직을 전혀 갖지 않고, 다른 UserService 구현 오브젝트에 기능을 위임하도록 하자.

```java
public class UserServiceTx implements UserService {
    UserService userService;
    private PlatformTransactionManager transactionManager;

    public void setUserService(UserService userService) {
        this.userService = userService;
    }

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    @Override
    public void add(User user) {
        userService.add(user);
    }

    @Override
    public void upgradeLevels() {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try{
            userService.upgradeLevels();
            this.transactionManager.commit(status);
        } catch (Exception e) {
            this.transactionManager.rollback(status);
            throw e;
        }
    }
}
```

### 6.2 고립된 단위 테스트

작은 단위로 쪼개서 테스트를 하는 것이 가장 편하고 좋은 테스트 방법 !

그러나, 테스트 대상이 다른 오브젝트와 환경에 의존하고 있다면 작은 단위의 테스트가 주는 장점을 얻기 힘들다.

현재 UserService는 `UserDao`, `TransactionManager`, `MailSender` 세 가지 의존 관계를 갖고 있다.

UserService를 테스트 하는데 오브젝트, 환경, 서비스, 서버까지 함께 테스트하는 셈인 것이다.

→ 테스트의 대상이 다른 것에 종속되고 영향을 받지 않도록 **! 고립 !** 시킬 필요가 있다.

**단위테스트**
테스트 대상 클래스를 테스트 대역을 이용해 의존 오브젝트나 외부의 리소스를 사용하지 않도록 고립시켜서 테스트하는 것

**통합테스트**
두 개 이상의, 성격이나 계층이 다른 오브젝트가 연동하도록 만들어 테스트하거나 외부의 리소스가 참여하는 테스트

**단위 테스트와 통합 테스트를 선택하는 가이드라인**

▪️ 항상 단위 테스트를 먼저 고려한다.

▪️ 하나의 클래스나 긴밀한 클래스 몇 개를 모아서 외부와의 의존관계를 모두 차단하고 필요에 따라 테스트 대역을 이용하도록 테스트를 만든다.

▪️ 외부 리소스를 사용해야만 가능한 테스트는 통합 테스트로 만든다.

▪️ 애초에 단위 테스트로 만들기 어려운 코드도 있다.

▪️ DAO 테스트는 DB라는 외부 리소스를 사용하기 때문에 통합 테스트로 분류되지만, 코드에서 보자면 하나의 기능 단위를 테스트하는 것이기도 하다.

▪️ 여러 개의 단위가 의존관계를 가지고 동작할 때를 위한 통합 테스트는 필요하다.

▪️ 단위 테스트를 만들기가 너무 복잡하다고 판단되는 코드는 처음부터 통합 테스트를 고려해 본다.

▪️ 스프링 테스트 컨텍스트 프레임워크를 이용하는 테스트는 통합 테스트다.

### **6.3 다이내믹 프록시와 팩토리 빈**

**🪻 데코레이터 패턴 : 타깃에 부가적인 기능을 런타임 시 다이내믹하게 부여해주기 위해 프록시를 사용하는 패턴**

▪️ 프록시로서 동작하는 각 데코레이터는 위임하는 대상에도 인터페이스로 접근하기 때문에 자신이 최종 타깃으로 위임하는지, 다음 단계의 데코레이터 프록시로 위임하는지 알지 못한다.

▪️ 데코레이터의 다음 위임 대상은 인터페이스로 선언하고 생성자나 수정자 메소드를 통해 위임 대상을 외부에서 런타임 시에 주입받을 수 있도록 만들어야 한다.

🪻**프록시 패턴 : 대상 원본 객체를 대리하여 대신 처리하게 함으로써 로직의 흐름을 제어하는 행동 패턴**

(타깃에 대한 접근 방법을 제어)
프록시 패턴은 타깃의 기능 자체에는 관여하지 않으면서 접근하는 방법을 제어해주는 프록시를 이용하는 것이다.

**사용 용도**

▪️ 타깃 오브젝트에 대한 레퍼런스가 미리 필요한 경우

▪️ 원격 오브젝트를 이용하는 경우

▪️ 특별한 상황에서 타깃에 대한 접근 권한을 제어해야 하는 경우

프록시 팩토리 빈을 이용하면 데코레이터 패턴 적용의 문제점을 해결해주기 때문에,( 타깃 인터페이스를 구현하는 클래스를 일일이 만드는 번거로움 제거 및 부가 기능 코드의 중복 문제 해결 ) 프록시 기법을 빠르고 효과적으로 적용할 수 있다.
