# 6. AOP

## 6.1 트랜잭션 코드의 분리

- 트랜잭션 코드의 분리 필요

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

- `upgradeLevels()`  → 트랜잭션 코드
- 여기서 비지니스 로직 부분(`upgradeLevelsInternal()`) 분리 : 인터페이스 사용
- UserService 인터페이스와 UserServiceImpl, UserServiceTx 구현 클래스로 나눔
    
    ```java
    public class UserServiceImpl implements UserService {
        UserDao userDao;
        // ...
        public void upgradeLevels() {
            List<User> users = userDao.getAll();
            for(User user : users) {
                if(canChangedLevel(user)) {
                    upgradeLevel(user);
                }
            }
        }
    }
    ```
    
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
    
    - UserServiceTx : transactionManager(PlatformTransactionManager)를 통해 upgradeLevels 전후에 필요한 트랜잭션 경계설정
- 트랜잭션 경계설정 코드 분리
    - 비지니스 로직 작성 시 트랜잭션과 같은 기술적인 내용에 신경x
    - 비지니스 로직에 대한 테스트 더 쉽게 생성 가능

## 6.2 고립된 단위 테스트

- 작은 단위의 테스트 → 원인 파악 쉬움, 내용 분명
- 작은 단위 테스트를 위해 다른 클래스에 종속되지 않도록 고립시켜야 함 → **테스트 대역** 사용 가능
    - 협력 오브젝트(UserDao)의 결과는 알 수 없지만, 협력 오브젝트의 메소드(update)가 호출되었다면 DB에 반영되었다고 함
    - Mock 오브젝트 활용
- 단위 테스트 : 의존 오브젝트나 외부 리소스 사용하지 않도록 **고립**시켜 테스트
- 통합 테스트 : 두 개 이상의 다른 오브젝트가 연동하도록 만들어 테스트, 외부의 리소스 참여하는 테스트
    - 스프링 테스트 컨텍스트 프레임워크 → 통합 테스트

```java
@Test
public void upgradeLevels() throws Exception {
    userDao.deleteAll();
    for(User user : users) userDao.add(user);

    MockMailSender mockMailSender = new MockMailSender();
    userServiceImpl.setMailSender(mockMailSender);
    userService.upgradeLevels(); 

    checkLevelUpgraded(users.get(0), false);
    checkLevelUpgraded(users.get(1), false);
    checkLevelUpgraded(users.get(2), true); 
    checkLevelUpgraded(users.get(3), true);
    checkLevelUpgraded(users.get(4), false); 

    List<String> request = mockMailSender.getRequests(); 
    assertThat(request.size(), is(2));                  
    assertThat(request.get(0), is(users.get(1).getEmail()));
    assertThat(request.get(1), is(users.get(3).getEmail()));
}

```

## 6.3 다이내믹 프록시와 팩토리 빈

- 프록시 : 간접 호출, 클라이언트의 요청을 받아주는 것
    - 클라이언트가 타깃에 접근하는 방법 제어
    - 타깃(프록시가 호출하는 대상)에 부가적인 기능 부여
- 클라이언트는 인터페이스를 통해서만 핵심 기능 사용, 부가기능은 그 사이에 끼어들어야 함
- 데코레이터 패턴 : 타깃에 부가적인 기능을 런타임 시 다이내믹하게 부여해주기 위해 프록시 사용, 프록시가 하나로 제한되지 않음, 같은 인터페이스를 구현한 타겟과 여러 개의 프록시 사용 가능, 프록시 순서를 정해서 단계적으로 위임하는 구조로 만들어야 함
    - 새로운 기능 추가가 목적
- 프록시 패턴 : 대상 원본 객체를 대리하여 처리 → 로직의 흐름 제어하는 행동 패턴, 타깃의 기능 자체에는 관여x
    - 접근 제어가 목적
- UserServiceTx : 현재 트랜잭션이 필요한 메소드마다 트랜잭션 코드 중복 → 프록시 방식으로 변경
    
    ```java
    public class TransactionHandler implements InvocationHandler {
        private Object target;          // 부가기능 제공할 타깃 오브젝트
        private PlatformTransactionManager transactionManager;
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
            if (method.getName().startsWith(pattern)) { // 트랜잭션 적용 대상 메소드
                return invokeInTransaction(method, args); // 트랜잭션 경계설정 기능
            } else {
                return method.invoke(target, args);
            }
        }
    
        private Object invokeInTransaction(Method method, Object[] args) throws Throwable {
            TransactionStatus status = this.transactionManager
                .getTransaction(new DefaultTransactionDefinition());
    
            try {
                Object ret = method.invoke(target, args); // 타깃 오브젝트 메소드를 호출
                this.transactionManager.commit(status);
                return ret;
            } catch (InvocationTargetException e) {
                this.transactionManager.rollback(status);
                throw e.getTargetException();
            }
        }
    
    }
    ```