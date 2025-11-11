# [정은서] 2025 GDG Spring Advanced Study - 6주차
# 토비의 스프링 3.1(vol1) - 6장 AOP

## AOP

AOP는 IoC/DI, 서비스 추상화와 더불어 스프링의 3대 기반 기술의 하나이다. 

스프링에 적용된 가장 인기 있는 AOP의 적용 대상은 바로 선언적 트랜잭션 기능이다. 서비스 추상화를 통해 많은 근본적인 문제를 해결했던 트랜잭션 경계설정 기능을 AOP를 이용해 더욱 세련되고 깔끔한 방식으로 바꿔보자. 그리고 그 과정에서 스프링이 AOP를 도입해야 했던 이유도 알아보자.

### 6.1 트랜잭션 코드의 분리

UserService 에 트랜잭션 코드가 더 많은 자리를 차지한다.

6.1.1 메소드 분리

- 트랜잭션 경계설정과 비즈니스 로직이 공존하는 메소드

```java
public void upgradeLevels() throws Exception {
	TransactionStatus status = this.transactionManager
		.getTransaction(new DefaultTransactionDefinition());
	try {
		// 비즈니스 로직
		List<User> users = userDao.getAll();
		for (User user : users) {
			if (canUpgradeLevel(user)) {
				upgradedLevel(user);
			}
		}
		this.transactionManager.commit(status);
	}catch (Exception e) {
		this.transactionManager.rollback(status);
		throw e;
	}
}
```

비즈니스 로직 코드를 사이에 두고 트랜잭션 시작과 종료를 담당하는 코드가 앞뒤에 위치하고 있다. 또한 트랜잭션 경계설정의 코드와 비즈니스 로직 코드 간에 서로 주고 받는 정보가 없다.

- 비즈니스 로직과 트랜잭션 경계설정의 분리

```java
public void upgradeLevels() throws throws Exception {
	TransactionStatus status = this.transactionManager
		.getTransaction(new DefaultTransactionDefinition());
	try {
		upgradeLevelsInternal();
		this.transactionManager.commit(status);
	}catch (Exception e) {
		this.transactionManager.rollback(status);
		throw e;
	}
}

private void upgradeLevelsInternal() {
	List<User> users = userDao.getAll();
		for (User user : users) {
			if (canUpgradeLevel(user)) {
				upgradedLevel(user);
			}
		}
	}
```

6.1.2 DI를 이용한 클래스의 분리

트랜잭션을 담당하는 기술적인 코드가 UserService 안에 자리 잡고 있다.  

DI 적용을 이용한 트랜잭션 분리

DI의 기본 아이디어는 실제 사용할 오브젝트의 클래스 정체를 감춘 채 인터페이스를 통해 간접으로 접근하는 것이다. 그 덕분에 구현 클래스는 얼마든지 외부에서 변경할 수 있다. 

UserService를 인터페이스로 만들고 기존 코드는 UserService 인터페이스의 구현 클래스를 만들어넣도록 한다. 그러면 클라이언트와 결합이 약해지고, 직접 구현 클래스에 의존하고 있지 않기 때문에 유연한 확장이 가능해진다. 

UserService를 구현한 또 다른 구현 클래스를 만든다. 이 클래스는 트랜잭션의  경계설정이라는 책임을 맡고 있다. 그리고 스스로는 비즈니스 로직을 담고 있지 않기 때문에 또 다른 비즈니스 로직을 담고 있는 UserService의 구현 클래스에 실제적인 로직 처리 작업은 위임하는 것이다. 그 위임을 위한 호출 작업 이전과 이후에 적절한 트랜잭션 경계를 설정해주면, 클라이언트 입장에서 결국 트랜잭션이 적용된 비즈니스 로직의 구현이라는 기대하는 동작이 일어날 것이다.  

- UserService 인터페이스 도입 - 기존의 UserService 클래스를 UserServiceImpl로 이름을 변경한다.
- UserService 인터페이스

```java
public interface UserService {
	void add(User user);
	void upgradeLevels();
}
```

- 트랜잭션 코드를 제거한 UserService 구현 클래스

```java
public class UserServceImpl implements UserServce {
	UserDao userDao;
	MailSender mailSender;
	
	public void upgradeLevel() {
		List<User> users = userDao.getAll();
		for (User user : users) {
			if (canUpgradeLevel(user)) {
				upgradedLevel(user);
			}
		}
	}
	...
```

- 위임 기능을 가진 UserServiceTx 클래스

```java
public class UserServiceTx implements UserService {
		// UserService를 구현한 다른 오브젝트를 DI 받는다. 
		UserService userService;
		PlatformTransactionManger transactionManager;
		
		public void setTransactionManager(
				PlatformTransactionManger transactionManager) {
			this.transactionManager = transactionManager;
		}
	
		public void setUserService(UserService userService) {
			this.userService = userService;
		}
		
		// DI 받은 UserService 오브젝트에 모든 기능을 위임
		public void add(User user) {
			userService.add(user);
		}
		
		public void upgradeLevels() {
		TransactionStatus status = this.transactionManager
		.getTransaction(new DefaultTransactionDefinition());
		try {
			userService.upgradeLevels();
			
			this.transactionManager.commit(status);
		}catch (RuntimeException e) {
			this.transactionManager.rollback(status);
			throw e;
		}
	}
}
```

- 트랜잭션 적용을 위한 DI 설정
    - 기존에 userService 빈이 의존하고 있던 transactionManager는 UserServiceTx의 빈이, userDao와 mailSender는 UserServiceImpl 빈이 각각 의존하도록 프로퍼티 정보를 분리한다.
- 트랜잭션 경계설정 코드 분리의 장점
    - 비즈니스 로직을 담당하고 있는 UserServiveImpl의 코드를 작성할 때는 트랜잭션과 같은 기술적인 내용에는 전혀 신경 쓰지 않아도 된다. 따라서 언제든지 트랜잭션을 도입할 수 있다.
    - 비즈니스 로직에 대한 테스트를 손쉽게 만들어낼 수 있다.

### 6.2 고립된 단위 테스트

가장 편하고 좋은 테스트 방법은 가능한 한 작은 단위로 쪼개서 테스트하는 것이다. 

6.2.1 복잡한 의존관계 속의 테스트

UserService의 구현 클래스들이 동작하려면 UserDao 타입의 오브젝트를 이용해 DB와 데이터를 주고받아야 하고, MailSender를 구현한 오브젝트를 이용해 메일을 발송해야 한다. 마지막으로 트랜잭션 처리를 위해 PlatformTransactionManger와 커뮤니케이션이 필요하다. 

테스트 시에 UserService를 테스트하는 것처럼 보이지만 사실은 그 뒤에 존재하는 훨씬 더 많은 오브젝트와 환경, 서비스, 서버, 네트워크까지 함께 테스트 하는 셈이 된다. 

6.2.2 테스트 대상 오브젝트 고립시키기 

테스트의 대상이 환경이나 외부 서버, 다른 클래스의 코드에 종속되고 영향을 받지 않도록 고립시킬 필요가 있다. 

- 테스트 대역 사용
- 목 오브젝트 사용
- 테스트를 위한 UserServiceImpl 고립
    - 테스트 대상인 UserServiceImpl과 그 협력 오브젝트인 UserDao에게 어떤 요청을 했는지를 확인하는 작업이 필요하다. UserDao 와 같은 역할을 하면서 UserServiceImpl과의 사이에서 주고받은 정보를 저장해뒀다가, 테스트의 검증에 사용할 수 있게 하는 목 오브젝트를 만들 필요가 있다.
- 고립된 단위 테스트 활용

```java
@Test
public void upgradeLevels() throws Exception {
	// 1. DB 테스트 데이터 준비
	userDao.deleteAll();
	for (User user : users) userDao.add(user);
	
	// 2. 메일 발송 여부 확인을 위해 목 오브젝트 DI
	MockMailSender moclMailSender = new MockMailSender();
	userServiceImpl.setMailSender(mockMailSender);
	
	userServce.upgradeLevels(); // 테스트 대상 실행
	
	// 3. DB에 저장된 결과 확인
	checkLevelUpgraded(users.get(0), false);
	checkLevelUpgraded(users.get(1), true);
	checkLevelUpgraded(users.get(2), false);
	checkLevelUpgraded(users.get(3), true);
	checkLevelUpgraded(users.get(4), false);
	
	// 4. 목 오브젝트를 이용한 결과 확인
	List<String> request = mockMailSender.getRequests();
	assertThat(request.size, is(2));
	assertThat(request.get(0), is(users.get(1).getEamil()));
	assertThat(request.get(1), is(users.get(3).getEmail()));
}

private void checkLevelUpgraded(User user, boolean upgraded) {
	User userUpdate = userDao.get(user.getId());
	...
}
```

- UserDao 목 오브젝트

upgradeLevels() 메소드가 실행되는 중에 UserDao와 어떤 정보를 주고받는지 입출력 내역을 먼저 확인할 필요가 있다.

```java
static class MockUserDao implements UserDao {
	private List<User> users;
	private List<User> updated = new ArrayList();
	
	private MockUserDao(List<User> users) {
		this.users = users;
	}
	
	publc List<User> getUpdated() {
		return this.updated;
	}
	
	public List<User> getAll() {
		return this.users; // 스텁 기능 제공
	}
	
	public void update(User user) {
		updated.add(user); // 목 오브젝트 기능 제공
	}
	
	// 테스트에 사용되지 않는 메소드
	public void add(User user) {throw new UnsupportedOperationException();}
	public void deleteAll() {throw new UnsupportedOperationException();}
	public User get(String id) {throw new UnsupportedOperationException();}
	public int getCount() {throw new UnsupportedOperationException();}
```

- MockUserDao를 사용해서 만든 고립된 테스트

```java
@Test
public void upgradeLevels() throws Exception {
	UserServiceImpl userServiceImpl = new UserServiceImpl();
	
	// 목 오브젝트로 만든 UserDao를 직접 DI
	MockUserDao mockUserDao = new MockUserDao(this.users);
	userServiceImpl.setUserDao(mockUserDao);
	
	// 메일 발송 여부 확인을 위해 목 오브젝트 DI
	MockMailSender moclMailSender = new MockMailSender();
	userServiceImpl.setMailSender(mockMailSender);
	
	userServce.upgradeLevels(); // 테스트 대상 실행
	
	List<User> updated = mockUserDao.getUpdated();
	checkUserAndLevel(updated.get(0), "joytouch", Level.SILVER);
	checkUserAndLevel(updated.get(1), "madnite1", Level.GOLD);
	
	// 목 오브젝트를 이용한 결과 확인
	List<String> request = mockMailSender.getRequests();
	assertThat(request.size, is(2));
	assertThat(request.get(0), is(users.get(1).getEamil()));
	assertThat(request.get(1), is(users.get(3).getEmail()));
}
```

테스트 수행 성능 시간이 빨라진다.

6.2.3 단위 테트스와 통합 테스트

- 단위 테스트 - 테스트 대상 클래스를 목 오브젝트 등의 테스트 대역을 이용해 의존 오브젝트나 외부 리소르르 사용하지 않도록 고립시켜서 테스트하는 것
- 통합 테스트 - 두 개 이상의, 성격이나 계층이 다른 오브젝트가 연동하도록 만들어 테스트하거나, 또는 외부의 DB나 파일, 서비스 등의 리소스가 참여하는 테스트하는 것
- 어떤 방법을 쓸지에 대한 가이드 라인
    - 항상 단위 테스트를 먼저 고려
    - 외부 리소스를 사용해야만 가능한 테스트는 통합 테스트
    - 단위 테스트를 만들기가 너무 복잡하다고 판단되는 코드는 처음부터 통합 테스트를 고려

6.2.4 목 프레임워크

단위 테스트를 만들기 위해서는 스텁이나 목 오브젝트의 사용이 필수적이다. 하지만 목 오브젝트를 만드는 일이 가장 큰 짐이다.

이런 번거로운 목 오브젝트를 편리하게 작성하도록 도와주는 다양한 목 오브젝트 지원 프레임워크가 있다.

Mockito 프레임워크 - 간단한 메소드 호출만으로 다이내믹하게 특정 인터페이스를 구현한 테스트용 목 오브젝트를 만들 수 있다. 

```java
@Test
public void upgradeLevels() throws Exception {
	UserServiceImpl userServiceImpl = new UserServiceImpl();
	
	UserDao mockUserDao = mock(UserDao.class);
	when(mockUserDao.getAll()).thenReturn(this.users);
	userServiceImpl.setUserDao(mockUserDao);
	
	// 메일 발송 여부 확인을 위해 목 오브젝트 DI
	MockMailSender moclMailSender = new MockMailSender();
	userServiceImpl.setMailSender(mockMailSender);
	
	userServce.upgradeLevels(); // 테스트 대상 실행
	
	// 목 오브젝트가 제공하는 검증 기능을 통해서 어떤 메소드가 몇 번 호출됐는지,
	// 파라미터는 무엇인지 확인할 수 있다.
	verify(mockUserDao, times(2)).update(any(User.class));
	verify(mockUserDao).update(users.get(1));
	assertThat(users.get(1).getLevel(), is(Level.SILVER));
	verify(mockUserDao).update(users.get(3));
	assertThat(users.get(3).getLevel(), is(Level.GOLD));
	
	
	// 파라미터를 정밀하게 검사하기 위해 캡쳐할 수도 있다.
	ArgumentCaptor<SimpleMailMessage> mailMessageArg =
	        ArgumentCaptor.forClass(SimpleMailMessage.class);
	verify(mockMailSender, times(2)).send(mailMessageArg.capture());
	
	List<SimpleMailMessage> mailMessages = mailMessageArg.getAllValues();
	assertThat(mailMessages.get(0).getTo()[0], is(users.get(1).getEmail()));
	assertThat(mailMessages.get(1).getTo()[0], is(users.get(3).getEmail()));
	}
```

### 6.3 다이내믹 프록시와 팩토리 빈

6.3.1 프록시와 프록시 패턴, 데코레이터 패턴

- 프록시 - 마치 자신이 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 것
    - 대리자, 대리인과 같은 역할
    - 프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트를 타깃 또는 실체라고 부른다.
    - 클라이언트 → 프록시 → 타깃
    - 클라이언트가 타깃에 접근하는 방법을 제어하기 위함
    - 타깃에 부가적인 기능을 부여해주기 위함
- 데코레이터 패턴 - 타깃에 부가적인 기능을 런타임 시 다이내믹하게 부여해주기 위해 프록시를 사용하는 패턴
    - 컴파일 시점에 어떤 방법과 순서로 프록시와 타깃이 연결되어 사용되는지 정해져 있지 않음
    - 프록시가 꼭 한 개로 제한되지 않음
    - 프록시가 직접 타깃을 사용하도록 고정시킬 필요가 없음
    - 데코레이터의 다음 위임 대상은 인터페이스로 선언하고 생성자나 수정자 메소드를 통해 위임 대상을 외부에서 런타임 시에 주입받을 수 있도록 만들어야 함
    - 코드에서 자신이 만들어나 접근할 타깃 클래스 정보를 알고 있는 경우가 많음
    - 자바 IO 패키지의 InputStream, OutputStream
- 프록시 패턴 - 프록시를 사용하는 방법 중에서 타깃에 대한 접근 방법을 제어하려는 목적을 가진 경우를 말함
    - 클라이언트가 타깃에 접근하는 방식을 변경해줌
    - 타깃 오브젝트에대한 레퍼런스가 미리 필요한 경우
    - 원격 오브젝트를 이용하는 경우
    - 타깃의 기능 자체에는 관여하지 않으면서 접근하는 방법을 제어해주는 프록시를 이용하는 것

6.3.1 다이내믹 프록시

프록시는 기존 코드에 영향을 주지 않으면서 타깃의 기능을 확장하거나 접근 방법을 제어할 수 있는 유용한 방법

- 타깃과 같은 메소드를 구현하고 있다가 메소드가  호출되면 타깃 오브젝트로 위임한다.
- 지정된 요청에 대해서는 부가기능을 수행한다.

```java
public class UserServiceTx implements UserService {
		UserService userService; // 타깃 오브젝트 

		// 메소드 구현과 위임
		public void add(User user) {
			userService.add(user);
		}
		
		public void upgradeLevels() { // 메소드 구현
		// 부가 기능 수행
		TransactionStatus status = this.transactionManager
		.getTransaction(new DefaultTransactionDefinition());
		try {
			userService.upgradeLevels(); // 위임
			
			// 부가 기능 수행
			this.transactionManager.commit(status);
		}catch (RuntimeException e) {
			this.transactionManager.rollback(status);
			throw e;
		}
	}
}
```

- 프록시를 만들기 번거로운 이유
    - 타깃의 인터페이스를 구현하고 위임하는 코드를 작성하기 번거로움
    - 부가기능 코드가 중복될 가능성이 많음
- 리플렉션

다이내믹 프록시는 리플렉션 기능을 이용해서 프록시를 만들어준다. 리플렉션은 자바 코드 자체를 추상화해서 접근하도록 만든 것이다. 

java.long.reflect.Method 인터페이스는 메소드에 대한 자세한 정보를 담고 있을 뿐 아니라, 이를 이용해 특정 오브젝트의 메소드를 실행시킬 수 있다.

- 프록시 클래스

```java
// Hello 인터페이스
interface Hello {
    String sayHello(String name);
    String sayHi(String name);
    String sayThankYou(String name);
}
```

```java
// 타깃 클래스
public class HelloTarget implements Hello {
    public String sayHello(String name) {
        return "Hello " + name;
    }

    public String sayHi(String name) {
        return "Hi " + name;
    }

    public String sayThankYou(String name) {
        return "Thank You " + name;
    }
}
```

```java
// 프록시 클래스
public class HelloUppercase implements Hello {
    Hello hello; // 위임할 타깃 오브젝트. 여기서는 타깃 클래스의 오브젝트인 것은 알지만
                 // 다른 프록시를 추가할 수도 있으므로 인터페이스로 접근한다.

    public HelloUppercase(Hello hello) {
        this.hello = hello;
    }

    public String sayHello(String name) {
        return hello.sayHello(name).toUpperCase(); // 위임과 부가기능 적용
    }

    public String sayHi(String name) {
        return hello.sayHi(name).toUpperCase();
    }

    public String sayThankYou(String name) {
        return hello.sayThankYou(name).toUpperCase();
    }
}
```

- 다이내믹 프록시 적용

프록시 팩토리에 의해 런타임 시 다이내믹하게 만들어지는 오브젝트다. 다이내믹 프록시 오브젝트는 타깃의 인터페이스와 같은 타입으로 만들어진다. 클라이언트는 다이내믹 프록시 오브젝트를 타깃 인터페이스를 통해 사용할 수 있다. 

다이내믹 프록시가 인터페이스 구현 클래스의 오브젝트는 만들어주지만, 프록시로서 필요한 부가기능 제공 코드는 직접 작성해야 한다. (InvocationHandler)

```java
public class UppercaseHandler implements InvocationHandler {
    Hello target;

    public UppercaseHandler(Hello target) { // 다이내믹 프록시로부터 전달받은 요청을 다시 타깃 오브
                                          // 젝트에 위임해야 하기 때문에 타깃 오브젝트를 주입받아 둔다.
        this.target = target;
    }

    public Object invoke(Object proxy, Method method, Object[] args)
            throws Throwable {
        String ret = (String)method.invoke(target, args); // 타깃으로 위임. 인터페이스의 메소드 호출에 모두 적용된다.
        return ret.toUpperCase(); // 부가기능 제공
    }
}
```

```java
// 생성된 다이내믹 프록시 오브젝트는 Hello 인터페이스를 구현하고 있으므로
// Hello 타입으로 캐스팅해도 안전하다.
Hello proxiedHello = (Hello)Proxy.newProxyInstance(
    getClass().getClassLoader(), // 동적으로 생성되는 다이내믹 프록시 클래스의 로딩에
                                 // 사용할 클래스 로더
    new Class[] { Hello.class }, // 구현할 인터페이스
    new UppercaseHandler(new HelloTarget()) // 부가기능과 위임 코드를 담은 InvocationHandler
);
```

- 다이내믹 프록시의 확장

어떤 종류의 인터페이스를 구현한 타깃이든 상관없이 재사용할 수 있고, 메소드의 리턴타입이 스트링인 경우만 대문자로 결과를 바꿔주도록 한다.

```java
public class UppercaseHandler implements InvocationHandler {
    Object target;
    
    // 어떤 종류의 인터페이스를 구현한 타깃에도 적용 가능하도록 Object 타입으로 수정
    private UppercaseHandler(Object target) {
        this.target = target;
    }

    public Object invoke(Object proxy, Method method, Object[] args)
            throws Throwable {
        Object ret = method.invoke(target, args);
        
        // 호출한 메소드의 리턴 타입이 String인 경우만 대문자 변경 기능을 적용하도록 수정
        if (ret instanceof String) {
            return ((String)ret).toUpperCase();
        }
        else {
            return ret;
        }
    }
}
```

6.3.3 다이내믹 프록시를 이용한 트랜잭션 부가기능

UserServiceTx를 다이내믹 프록시 방식으로 변경해보자. 

```java
public class TransactionHandler implements InvocationHandler {
    // 부가기능을 제공할 타깃 오브젝트. 어떤 타입의 오브젝트에도 적용 가능하다.
    private Object target;
    // 트랜잭션 기능을 제공하는 데 필요한 트랜잭션 매니저
    private PlatformTransactionManager transactionManager;
    // 트랜잭션을 적용할 메소드 이름 패턴
    private String pattern;

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
        if (method.getName().startsWith(pattern)) {
            return invokeInTransaction(method, args);
        } else {
            // 트랜잭션 적용 대상 메소드를 선별해서 트랜잭션 경계설정 기능을 부여해준다.
            return method.invoke(target, args);
        }
    }

    private Object invokeInTransaction(Method method, Object[] args) throws Throwable {
        TransactionStatus status =
                this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            Object ret = method.invoke(target, args);
            // 트랜잭션을 시작하고 타깃 오브젝트의 메소드를 호출한다. 예외가 발생하지 않았다면 커밋한다.
            this.transactionManager.commit(status);
            return ret;
        } catch (InvocationTargetException e) {
            // 예외가 발생하면 트랜잭션을 롤백한다.
            this.transactionManager.rollback(status);
            throw e.getTargetException();
        }
    }
}
```

6.3.4 다이내믹 프록시를 위한 팩토리 빈

TransactionHandler 와 다이내믹 프록시를 스프링의 DI를 통해 사용할 수 있도록 만들자. 

- 팩토리 빈 - 스프링을 대신해서 오브젝트의 생성로직을 담당하도록 만들어진 특별한 빈
- 팩토리 빈 설정

```java
<bean id="message"
    class="springbook.learningtest.spring.factorybean.MessageFactoryBean">
    <property name="text" value="Factory Bean" />
</bean>
```

- 다이내믹 프록시를 만들어주는 팩토리 빈

 팩토리 빈의 getObject() 메소드에 다이내믹 프록시 오브젝트를 만들어주는 코드를 넣으면 된다. 

- 트랜잭션 프록시 팩토리 빈

```java
package springbook.user.service;

// 생성할 오브젝트 타입을 지정할 수도 있지만 범용적으로 사용하기 위해 Object로 했다.
public class TxProxyFactoryBean implements FactoryBean<Object> {
    // TransactionHandler를 생성할 때 필요
    Object target;
    PlatformTransactionManager transactionManager;
    String pattern;
    // 다이내믹 프록시를 생성할 때 필요하다. UserService 이외의 인터페이스를
    // 가진 타깃에도 적용할 수 있다.
    Class<?> serviceInterface;

    public void setTarget(Object target) {
        this.target = target;
    }

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public void setPattern(String pattern) {
        this.pattern = pattern;
    }

    public void setServiceInterface(Class<?> serviceInterface) {
        this.serviceInterface = serviceInterface;
    }

    // FactoryBean 인터페이스 구현 메소드
    public Object getObject() throws Exception { // 이 받은 정보를 이용해서 TransactionHandler를
                                                 // 사용하는 다이내믹 프록시를 생성한다.
        TransactionHandler txHandler = new TransactionHandler();
        txHandler.setTarget(target);
        txHandler.setTransactionManager(transactionManager);
        txHandler.setPattern(pattern);
        return Proxy.newProxyInstance(
                getClass().getClassLoader(), new Class[] { serviceInterface },
                txHandler);
    }

    public Class<?> getObjectType() {
        // 팩토리 빈이 생성하는 오브젝트의 타입은 DI 받은 인터페이스 타입에
        // 따라 달라진다. 따라서 다양한 타입의 프록시 오브젝트 생성에 재사용
        // 할 수 있다.
        return serviceInterface;
    }

    public boolean isSingleton() {
        // 싱글톤 빈이 아니라는 뜻이 아니라 getObject()가 매번
        // 같은 오브젝트를 리턴하지 않는다는 의미다.
        return false;
    }
}
```

6.3.5 프록시 팩토리 빈 방식의 장점과 한계

- 프록시 팩토리 빈의 재사용
- 프록시 기법을 아주 빠르고 효과적으로 적용할 수 있다.
- 프록시를 적용할 대상이 구현하고 있는 인터페이스를 구현하는 프록시 클래스를 일일이 만들 필요 없음
- 부가적인 기능이 여러 메소드에 반복적으로 나타나게 될 일도 없음
- 비즈니스 로직을 담은 많은 클래스의 메소드에 적용하려면 비슷한 프록시 팩토리 빈의 설정이 중복되는 것을 막을 수 없음
- TransactionHandler 오브젝트가 프록시 팩토리 빈 개수만큼 만들어짐