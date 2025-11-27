# 6.6~6.8 AOC 마무리

### 트랜잭션 개념 및 관리

한 단위로 묶인 작업들이 전부 성공하거나, 전부 실패해야 하는 논리적인 작업 단위이다. 트랜잭션은 데이터의 무결성을 보장하기 위해 도입되었다. 

ACID 원칙: 트랜잭션은 원자성(Atomicity), 일관성(Consistency), 격리성(Isolation), **지속성(Durability)**의 4가지 원칙(ACID)을 만족해야 한다.

**스프링의 트랜잭션 관리:**
- `TransactionInterceptor`를 사용하여 메서드 호출 전후에 트랜잭션 처리를 수행한.
- 선언적 트랜잭션(Declarative Transaction) 방식을 주로 사용하며, 개발자는 비즈니스 로직에 집중하고 트랜잭션 처리는 AOP(Aspect-Oriented Programming)를 통해 위임할 수 있다.

### TransactionAttribute를 이용한 트랜잭션 속성 지정

TransactionAttribute: 스프링 트랜잭션 관리에서 사용되는 트랜잭션의 동작 방식을 정의하는 속성들의 집합

**주요 속성:**
- 전파(Propagation)
- 격리 수준(Isolation Level)
- 읽기 전용(Read-Only) 여부
- 타임아웃(Timeout)
- 롤백 규칙(Rollback Rules)

트랜잭션 전파(Propagation): 이미 진행 중인 트랜잭션이 있을 때, 새로운 메서드가 호출될 때 트랜잭션을 어떻게 처리할지 결정하는 규칙

**주요 전파 속성:**
- PROPAGATION_REQUIRED: 가장 흔하게 사용. 트랜잭션이 없으면 새로 생성하고, 있으면 기존 트랜잭션에 참여.
- PROPAGATION_SUPPORTS: 트랜잭션이 있으면 참여하고, 없으면 트랜잭션 없이 실행.
- PROPAGATION_MANDATORY: 반드시 트랜잭션이 있어야 함. 없으면 예외 발생.
- PROPAGATION_REQUIRES_NEW: 항상 새로운 트랜잭션을 시작. 기존 트랜잭션이 있으면 잠시 일시 중지(suspend)
- PROPAGATION_NOT_SUPPORTED: 트랜잭션이 있으면 잠시 일시 중지하고, 트랜잭션 없이 실행.
- PROPAGATION_NEVER: 트랜잭션이 없어야 함. 있으면 예외 발생.
- PROPAGATION_NESTED: 기존 트랜잭션 내에 중첩된 트랜잭션을 생성. (JDBC Savepoint 사용)

⭐ 트랜잭션 인터셉터 예시
```java
// Spring IoC Container에서 관리하는 객체 설정 예시 (XML 설정을 통해 주입)
public class TransactionInterceptorExample {
  private TransactionManager transactionManager;
  // (생략: Constructor or Setter Injection)

  public Object invoke(MethodInvocation invocation) throws Throwable {
    TransactionStatus status = this.transactionManager.getTransaction(this); // this 대신 TransactionDefinition 사용
    try {
      Object ret = invocation.proceed();
      this.transactionManager.commit(status);
      return ret;
    } catch (RuntimeException ex) {
      this.transactionManager.rollback(status);
      throw ex;
    }
  }
}
```

위 코드는 `TransactionInterceptor`를 사용하여 트랜잭션 관리자를 설정하고 AOP를 통해 특정 메서드에 적용하는 예시이다.

### 트랜잭션 전파와 트랜잭션 격리 수준

트랜잭션 격리 수준(Isolation Level): 여러 트랜잭션이 동시에 실행될 때, 한 트랜잭션이 다른 트랜잭션의 변경 내용을 어디까지 볼 수 있도록 허용할지 정의하는 수준

**주요 격리 수준:**
- ISOLATION_DEFAULT (DB 드라이버 기본값 사용)
- ISOLATION_READ_UNCOMMITTED (가장 낮은 수준, Dirty Read 발생 가능)
- ISOLATION_READ_COMMITTED (가장 일반적으로 사용)
- ISOLATION_REPEATABLE_READ
- ISOLATION_SERIALIZABLE (가장 높은 수준, 성능 저하 가능)

### 트랜잭션 속성의 적용과 테스트

#### 트랜잭션 속성 적용의 일반 원칙

가장 구체적인 메서드명에 적용된 속성이 우선한다. 예를 들어, `get*`과 `*` 속성이 동시에 정의된 경우, `getName()` 메서드에는 `get*`에 정의된 `read-only="true"`가 적용된다. 서비스 계층(Service Layer)에서 트랜잭션을 적용하는 것이 일반적이다. DAO(Data Access Object) 계층에서 직접 적용하는 것은 적절하지 않다. DAO는 순수한 데이터 접근 로직만 처리하고, 트랜잭션 경계 설정은 비즈니스 로직을 담당하는 서비스 계층에서 이루어져야 한다.

#### 읽기 전용(Read-Only) 트랜잭션 테스트

읽기 전용 트랜잭션의 특징은 데이터 변경 작업(INSERT, UPDATE, DELETE)을 허용하지 않는 것이다. 데이터베이스 커넥션 설정에 따라 변경 작업 시 예외(`TransientDataAccessResourceException` 또는 SQL 예외)가 발생할 수 있다.

아래 테스트 시나리오에서는 `get*` 메서드에 `read-only="true"` 트랜잭션을 설정하고, 해당 메서드 내에서 데이터 변경(예: `userDao.update()`) 작업을 시도하면 예외가 발생하는지 확인한다.

⭐ 읽기 전용(Read-Only) 트랜잭션 테스트 코드
```java
public class TestService extends UserServiceImpl {
  
  // getList() 메서드가 read-only 트랜잭션으로 실행될 때,
  // 내부에서 update() 호출 시 예외가 발생하는지 확인하기 위한 오버라이드
  @Override
  public List<User> getList() {
    User user = new User("tester", 0);
    super.update(user); // read-only 트랜잭션 내에서 쓰기 작업 시도
    return super.getList();
  }
}
```

### 프록시 AOP와 타 객체 메서드 호출 시 주의점

스프링의 선언적 트랜잭션은 AOP 프록시를 통해 구현된다. 프록시가 동작하기 위해서는 **외부(클라이언트)**에서 해당 빈(Bean)의 메서드를 호출해야 한다.

**타 객체(Self-Invocation) 호출 시 트랜잭션 적용 문제**
- 특정 빈의 메서드 A가 같은 빈의 다른 메서드 B를 호출할 때 (이를 Self-Invocation이라고 함), 메서드 B에 정의된 트랜잭션 속성은 적용되지 않는다.
    - 메서드 B는 프록시를 통하지 않고 대상 객체(Target Object)를 통해 직접 호출되기 때문에, 프록시에 의해 AOP 로직(트랜잭션 시작/종료)이 적용되지 않는다.
    - 해결책
        - 메서드 분리: 트랜잭션이 필요한 메서드 B를 다른 서비스 객체로 분리하고, 메서드 A가 해당 서비스 객체를 호출하도록 구조를 변경한다.
        - AOP 설정 변경: Spring 2.0 이상에서는 Self-Invocation 시에도 트랜잭션을 적용할 수 있도록 설정하는 방법이 있으나, 일반적인 해결책은 아니다.

⭐ `UserServiceImpl` 예시
```java
public class UserServiceImpl implements UserService {
    // ...
    public void delete(String id) {
        userDao.delete(id); // [1] 클라이언트가 delete를 호출하면 트랜잭션 적용 O
    }

    public void update(User user) {
        userDao.update(user); // [2] 클라이언트가 update를 호출하면 트랜잭션 적용 O
    }
    
    // deleteAndUpdate() 메서드가 외부에서 호출되면 트랜잭션이 적용됨
    public void deleteAndUpdate(String id, User user) {
        this.delete(id); // 같은 객체 내부 호출: delete에 정의된 트랜잭션 속성 적용 X
        this.update(user); // 같은 객체 내부 호출: update에 정의된 트랜잭션 속성 적용 X
        // 전체 deleteAndUpdate() 메서드의 트랜잭션만 적용됨.
    }
}
```

### 트랜잭션 어노테이션 (@Transactional)

XML <tx:method> 방식 대신 `@Transactional` 어노테이션을 사용하여 트랜잭션 속성을 선언하는 것이 더 일반적이다. 이 어노테이션은 메서드나 클래스에 적용할 수 있으며, 스프링 AOP를 통해 런타임에 트랜잭션 경계를 설정한다.

⭐ `@Transactional` 어노테이션의 정의
```java
public @interface Transactional {
    // ...
    Propagation propagation() default Propagation.REQUIRED;
    Isolation isolation() default Isolation.DEFAULT;
    int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;
    boolean readOnly() default false;
    Class<? extends Throwable>[] rollbackFor() default {};
    String[] rollbackForClassName() default {};
    Class<? extends Throwable>[] noRollbackFor() default {};
    String[] noRollbackForClassName() default {};
}
```

- `propagation` (전파): 트랜잭션 경계 설정 시 가장 중요하며, 기본값은 `PROPAGATION_REQUIRED`
- `isolation` (격리 수준): 기본값은 데이터베이스 드라이버의 기본값인 `ISOLATION_DEFAULT`
- `readOnly` (읽기 전용): 기본값은 `false`이며, 데이터 변경 작업이 없는 메서드(`get*`, `find*` 등)에는 `true`로 설정하여 성능을 최적화
- `rollbackFor`/`noRollbackFor` (롤백 규칙): 특정 예외 발생 시 롤백하거나 롤백하지 않도록 규칙 지정 가능 (기본적으로 Checked Exception은 롤백하지 않고, Unchecked Exception은 롤백)

#### @Transactional 적용 위치 및 우선순위

1. 메서드 레벨에 적용된 속성: 가장 높은 우선순위
2. 클래스 레벨에 적용된 속성: 메서드에 어노테이션이 없을 경우 적용
3. 인터페이스 레벨에 적용된 속성: 클래스나 메서드에 어노테이션이 없을 경우 적용

⭐ 클래스와 인터페이스에 트랜잭션 적용 예시
```java
public interface Service { 
  void method1();
  void method2();
}

public class ServiceImpl implements Service { 
  public void method1() { }
  public void method2() { }
}
```

- J2SE 다이내믹 프록시 (기본값): 스프링의 기본 AOP 프록시는 인터페이스 기반이기 때문에 트랜잭션이 정상적으로 적용되려면 인터페이스나 인터페이스의 메서드에 `@Transactional`을 붙이는 것이 가장 안전하다.
- 클래스 기반 프록시 (CGLIB): CGLIB을 사용하여 클래스 기반 프록시를 생성하면, 클래스에 직접 붙은 `@Transactional`도 적용될 수 있다.

### AOP 트랜잭션의 내부 호출 (Self-Invocation) 문제

스프링의 AOP는 프록시 패턴을 기반으로 트랜잭션을 관리한다.

**같은 객체 내부 메서드 호출 시 트랜잭션 미적용**
- Self-Invocation: 트랜잭션이 적용된 Service 객체 내부의 메서드 A가 같은 Service 객체 내부의 다른 메서드 B를 호출하는 경우이다.
- 메서드 A가 B를 호출할 때, 호출이 AOP 프록시를 거치지 않고 *대상 객체(Target Object)*를 통해 직접 이루어진다. 이 경우, 메서드 B에 `@Transactional` 속성이 정의되어 있어도 트랜잭션이 적용되지 않는다.
- 메서드 B에 `PROPAGATION_REQUIRES_NEW`와 같이 독립적인 트랜잭션을 설정했더라도, 이는 무시되고 메서드 A의 트랜잭션에 종속되거나 트랜잭션 없이 실행될 수 있다.

위 문제를 해결하는 가장 일반적이고 권장되는 방법은 *메서드 호출이 반드시 AOP 프록시*를 거치도록 하는 것이다.

내부에서 트랜잭션 속성이 달라져야 하는 메서드(예: `REQUIRES_NEW`가 필요한 메서드)를 별도의 새로운 Service 클래스로 분리하여 컴포넌트 간의 호출을 통해 프록시를 거치도록 한다. 혹은 프록시 직접 주입하는 방법으로, 해당 객체의 프록시를 스스로 주입받아(`@Autowired`) 내부 호출 시 프록시를 통해 호출하는 방법도 있지만, 이는 코드를 복잡하게 만든다.

⭐ `UserService` 인터페이스에 트랜잭션 속성 적용
```java
public interface UserService { 
  @Transactional // 디폴트 속성(REQUIRED) 적용
  void addUser(User user);
  @Transactional(readOnly=true) // 읽기 전용 트랜잭션 적용
  List<User> getList();
  void update(User user);
  void delete(String id);
}
```

### 트랜잭션 동기화와 트랜잭션 매니저

#### 트랜잭션 동기화

트랜잭션 매니저와 트랜잭션 동기화 기능을 통해 트랜잭션을 일관되게 관리한다. `TransactionSynchronizationManager`는 `PlatformTransactionManager`가 실행하는 트랜잭션 정보를 스레드 로컬(ThreadLocal)에 보관하여 DAO를 포함한 모든 레이어에서 사용할 수 있도록 한다.선언적 트랜잭션의 역할은 `@Transactional`이 적용된 메서드가 호출되면 AOP를 통해 트랜잭션을 시작하고 동기화한다. 이 트랜잭션은 해당 스레드에서 실행되는 모든 DB 작업에 참여하게 되는 것이다.

#### 트랜잭션 매니저 주입 및 테스트 환경 설정

테스트 클래스에서 `@Autowired`를 사용하여 트랜잭션 매니저를 주입받아 사용할 수 있다.

⭐ 트랜잭션 매니저 주입
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "/test-applicationContext.xml")
public class UserServiceTest {
    @Autowired
    PlatformTransactionManager transactionManager;
    // ...
}
```

⭐ 트랜잭션 매니저를 이용해 트랜잭션을 관리하는 테스트
```java
@Test
public void transactionSync() {
    // 트랜잭션 정의
    DefaultTransactionDefinition txDefinition = new DefaultTransactionDefinition();
    
    // 트랜잭션 시작 (상태 객체 반환)
    TransactionStatus txStatus = transactionManager.getTransaction(txDefinition);

    try {
        userService.delete(1);
        userService.add(new User("user1"));
        // ... 모든 DB 작업이 하나의 트랜잭션에 묶임
        
        transactionManager.commit(txStatus); // 성공 시 커밋
    } catch (Exception e) {
        transactionManager.rollback(txStatus); // 예외 발생 시 롤백
        throw e;
    }
}
```

##### 롤백 테스트 (Rollback Test)

테스트의 독립성을 위해 테스트 메서드가 실행한 DB 작업은 테스트가 끝난 후 반드시 롤백되어야 한다. 스프링 테스트 프레임워크는 `@Transactional` 어노테이션을 테스트 메서드에 붙이면 기본적으로 자동 롤백을 수행한다.

- `@Rollback` 어노테이션: 테스트 메서드 레벨에서 롤백 여부를 명시적으로 설정
- `@Rollback`(`true`): (기본값) 테스트 후 롤백
- `@Rollback`(`false`): 테스트 후 커밋 (테스트 환경에서는 권장되지 않음)

⭐ 롤백 테스트 (기본적으로 @Transactional 테스트는 롤백을 수행)
```java
@Test
@Transactional // 이 어노테이션 덕분에 테스트가 끝난 후 DB 변경 사항은 자동으로 롤백됨.
public void transactionSync() {
    // ... DB 변경 작업 수행 ...
    // 테스트가 끝나면 모든 작업이 롤백되어 DB에 영향을 주지 않음
}
```

##### 트랜잭션 테스트의 중요성

1. 독립적인 테스트
    - 트랜잭션 롤백 기능을 활용하면 테스트 간의 DB 데이터 의존성을 완전히 제거할 수 있다. 
    - 각 테스트는 자신만의 데이터를 가지고 독립적으로 실행되며, 테스트 순서나 이전 테스트의 성공 여부에 영향을 받지 않아 자동화된 테스트 환경을 구축하는 데 필수적이다.
2. 테스트를 위한 데이터 관리 
    - 트랜잭션 테스트를 위해 사전에 테스트 전용 데이터를 DB에 등록하거나, 테스트 코드 내에서 트랜잭션 시작 후 데이터를 삽입하고 롤백되도록 처리하여 테스트의 독립성을 유지한다.