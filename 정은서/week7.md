# [정은서] 2025 GDG Spring Advanced Study - 7주차
# 토비의 스프링 3.1(vol1) - 6장 AOP

## AOP(2)

### 6.4 스프링의 프록시 팩토리 빈

6.4.1 ProxyFactoryBean

스프링은 일관된 방법으로 프록시를 만들 수 있게 도와주는 추상 레이어를 제공한다.생성된 프록시는 스프링의 빈으로 등록돼야 한다. 스프링은 프록시 오브젝트를 생성해주는 기술을 추상화한 팩토리 빈을 제공해준다. (ProxyFactoryBean)

MethodInterceptor 인터페이스는 ProxyFactoryBean이 생성하는 프록시에서 사용할 부가기능을 담당한다. → 타킷 오브젝트에 상관없이 **독립적으로 만들어질 수 있다.**

```python
package springbook.learningtest.jdk.proxy;

import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
import org.junit.Test;
import org.springframework.aop.framework.ProxyFactoryBean;

import java.lang.reflect.Proxy;

import static org.hamcrest.CoreMatchers.is;
import static org.junit.Assert.assertThat;

public class DynamicProxyTest {
    @Test
    public void simpleProxy() {
        Hello proxiedHello = (Hello) Proxy.newProxyInstance(
                getClass().getClassLoader(),
                new Class[] { Hello.class },
                new UppercaseHandler(new HelloTarget())); // JDK 다이내믹 프록시 생성
        
        ... 
    }

    @Test
    public void proxyFactoryBean() {
        ProxyFactoryBean pfBean = new ProxyFactoryBean();
        pfBean.setTarget(new HelloTarget()); // -> 타깃 설정
        pfBean.addAdvice(new UppercaseAdvice()); // -> 부가기능을 담은 어드바이스를 추가한다. 여러 개를 추가할 수도 있다.

        Hello proxiedHello = (Hello) pfBean.getObject(); // -> FactoryBean이므로 getObject()로 생성된 프록시를 가져온다.
        
        assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
        assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
        assertThat(proxiedHello.sayThankYou("Toby"), is("THANK YOU TOBY"));
    }

    static class UppercaseAdvice implements MethodInterceptor {
        public Object invoke(MethodInvocation invocation) throws Throwable {
            String ret = (String)invocation.proceed(); // -> 리플렉션의 Method와 달리 메소드 실행 시 타깃 오브젝트를 전달할 필요가 없다. MethodInvocation은 메소드 정보와 함께 타깃 오브젝트를 알고 있기 때문이다.
            return ret.toUpperCase(); // -> 부가기능 적용
        }
    }

    static interface Hello { // -> 타깃과 프록시가 구현할 인터페이스
        String sayHello(String name);
        String sayHi(String name);
        String sayThankYou(String name);
    }

    static class HelloTarget implements Hello { // -> 타깃 클래스
        public String sayHello(String name) { return "Hello " + name; }
        public String sayHi(String name) { return "Hi " + name; }
        public String sayThankYou(String name) { return "Thank You " + name; }
    }
}
```

- 어드바이스: 타깃이 필요없는 순수한 부가기능

MethodInvocation 구현 클래스는 일종의 공유 가능한 템플릿처럼 동작한다.

JDK의 다이내믹 프록시를 직접 사용하는 코드와 스프링이 제공해주는 프록시 추상화 기능인 ProxyFactoryBean을 사용하는 코드의 가장 큰 차이점이자 ProxyFactoryBean의 장점이다. 

ProxyFactoryBean 하나만으로 여러 개의 부가 기능을 제공해주는 프록시를 만들 수 있다. 

MethodInterceptor처럼 타깃 오브젝트에 적용하는 부가 기능을 담은 오브젝트를 스프링에서는 **어드바이스(advice)**라고 부른다. 

- 포인트컷: 부가기능 적용 대상 메소드 선정 방법

ProxyFactoryBean과 MethodInterceptor를 사용하는 방식에서도 메소드 선정 기능을 넣을 수 없다. MethodInterceptor 오브젝트는 여러 프록시가 공유해서 사용할 수 있다. 그러기 위해서 MethodInterceptor 오브젝트는 타깃 정보를 갖고 있지 않도록 만들었다. 여기에 트랜잭션 적용 대상 메소드 이름 패턴을 넣어주는 것은 곤란하다. 

메소드 선정 알고리즘을 담은 오브젝트. 

- 해결

스프링의 ProxyFactoryBean 방식은 두 가지 확장 기능인 부가기능(Advice)과 선정 알고리즘(Pointcut)을 활용하는 유연한 구조를 제공한다. 어드바이스와 포인트컷은 모두 프록시에 DI로 주입돼서 사용된다. 

프록시로부터 어드바이스와 포인트컷을 독립시키고 DI를 사용하게 한 것은 전형적인 전략 패턴 구조다. 덕분에 여러 프록시가 공유해서 사용할 수도 있고, 또 구체적인 부가기능 방식이나 메소드 선정 알고리즘이 바뀌면 구현 클래스만 바꿔서 설정에 넣어주면 된다. 

```python
// 포인트컷까지 적용한 ProxyFactoryBean
@Test
public void pointcutAdvisor() {
    ProxyFactoryBean pfBean = new ProxyFactoryBean();
    pfBean.setTarget(new HelloTarget());

    // 메소드 이름을 비교해서 대상을 선정하는 알고리즘을 제공하는 포인트컷 생성
    NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
    pointcut.setMappedName("sayH*"); // 이름 비교조건 설정. sayH로 시작하는 모든 메소드를 선택하게 한다.

    // 포인트컷과 어드바이스를 Advisor로 묶어서 한 번에 추가
    pfBean.addAdvisor(new DefaultPointcutAdvisor(pointcut, new UppercaseAdvice()));

    Hello proxiedHello = (Hello) pfBean.getObject();

    assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
    assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
    
    // 메소드 이름이 포인트컷의 선정조건에 맞지 않으므로, 부가기능(대문자변환)이 적용되지 않는다.
    assertThat(proxiedHello.sayThankYou("Toby"), is("Thank You Toby"));
}
```

ProxyFactoryBean에는 여러 어드바이스와 포인트컷이 추가될 수 있기 때문에 포인트컷과 어드바이스를 따로 등록하면 어떤 어드바이스(부가기능)에 대해 어떤 포인트컷(메소드 선정)을 적용할지 애매해진다. 그래서 이 둘을 Advisor 타입의 오브젝트에 담아서 조합을 만들어 등록하는 것이다. 

→ 어드바이스와 포인트컷을 묶은 오브젝트를 어드바이저라고 부른다. (어드바이저 = 포인트컷 + 어드바이스)

6.4.2 ProxyFactoryBean 적용

- 부가 기능을 담당하는 어드바이스는 테스트에서 만들어본 것처럼 MethodInterceptor라는 Advice 서브 인터페이스를 구현해서 만든다.

```python
package springbook.learningtest.jdk.proxy;

import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.TransactionStatus;
import org.springframework.transaction.support.DefaultTransactionDefinition;

public class TransactionAdvice implements MethodInterceptor { // 스프링의 어드바이스 인터페이스 구현
    PlatformTransactionManager transactionManager;

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    // 타깃을 호출하는 기능을 가진 콜백 오브젝트를 프록시로부터 받는다.
    // 덕분에 어드바이스는 특정 타깃에 의존하지 않고 재사용 가능하다.
    public Object invoke(MethodInvocation invocation) throws Throwable {
        TransactionStatus status =
                this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            // 콜백을 호출해서 타깃의 메소드를 실행한다. 타깃 메소드 호출 전후로 필요한 부가기능을 넣을 수 있다.
            // 경우에 따라서 타깃이 아예 호출되지 않게 하거나 재시도를 위한 반복적인 호출도 가능하다.
            Object ret = invocation.proceed();
            this.transactionManager.commit(status);
            return ret;
        } catch (RuntimeException e) {
            // JDK 다이내믹 프록시가 제공하는 Method와는 달리 스프링의 MethodInvocation을 통한 타깃 호출은
            // 예외가 포장되지 않고 타깃에서 보낸 그대로 전달된다.
            this.transactionManager.rollback(status);
            throw e;
        }
    }
}
```

```python
// ProxyFactoryBean을 이용한 트랜잭션 테스트
@Test
@DirtiesContext // -> 컨텍스트 설정을 변경하기 때문에 여전히 필요하다.
public void upgradeAllOrNothing() {
    TestUserService testUserService = new TestUserService(users.get(3).getId());
    testUserService.setUserDao(userDao);
    testUserService.setMailSender(mailSender);

    // userService 빈은 이제 스프링의 ProxyFactoryBean이다.
    ProxyFactoryBean txProxyFactoryBean = 
            context.getBean("&userService", ProxyFactoryBean.class);
    txProxyFactoryBean.setTarget(testUserService);
    
    // FactoryBean 타입이므로 동일하게 getObject()로 프록시를 가져온다.
    UserService txUserService = (UserService) txProxyFactoryBean.getObject();
    
    // ...
}
```

- 트랜잭션 부가기능을 담은 TransactionAdvice는 하나만 만들어서 싱글톤 빈으로 등록해주면, DI 설정을 통해 모든 서비스에 적용과 재사용이 가능하다.

### 6.5 스프링 AOP

분리해낸 트랜잭션 코드는 투명한 부가기능 형태로 제공돼야 한다. 투명하기 때문에 언제든지 자유롭게 추가하거나 제거할 수 있고, 기존 코드는 항상 원래의 상태를 유지할 수 있다. 

6.5.1 자동 프록시 생성 

부가기능의 적용이 필요한 타깃 오브젝트마다 거의 비슷한 내용의 ProxyFactoryBean 빈 설정 정보를 추가해주는 부분에 대한 문제가 있다.  

- 빈 후처리기를 이용한 자동 프록시 생성기

스프링은 OCP의 가장 중요한 요소인 유연한 확장이라는 개념을 스프링 컨테이너 자신에게도 다양한 방법으로 적용하고 있다. 스프링은 컨테이너로서 제공하는 기능 중에서 변하지 않는 핵심적인 부분 외에는 대부분 확장할 수 있도록 확장 포인트를 제공해준다. 그 중 관심을 가질 만한 확장 포인트는 BeanPostProcessor 인터페이스를 구현해서 만드는 빈 후처리기(스프링 빈 오브젝트로 만들어지고 난 후에, 빈 오브젝트를 다시 가공할 수 있게 해준다.) 

여기서는 DafaultAdvisorAutoProxyCreator를 살펴보자. → 어드바이저를 이용한 자동 프록시 생성기 

- 작동 원리:
1. 스프링 컨테이너가 빈을 생성
2. 빈 후처리기가 등록된 모든 어드바이저(Advisor)의 포인트컷을 확인하여, 현재 빈이 프록시 적용 대상인지 판단
3. 대상이면 내장된 프록시 생성기가 프록시를 생성하고 어드바이저를 연결
4. 컨테이너는 원래 빈 대신 생성된 프록시를 빈으로 등록

번거롭게 매번 ProxyFactoryBean을 빈으로 등록하지 않아도, 타깃 오브젝트에 자동으로 프록시를 적용가능하다.

- 확장된 포인트컷

기존에는 포인트컷을 메소드 선정 용도로만 생각했지만, 자동 프록시 생성을 위해서는 어떤 빈에 프록시를 적용할지 먼저 판단해야 한다. 

- ClassFilter (getClassFilter()): 프록시를 적용할 클래스인지 확
- MethodMatcher (getMethodMatcher()): 어드바이스를 적용할 메소드인지 확인

6.5.2 DafaultAdvisorAutoProxyCreator의 적용

- 클래스 필터를 적용한 포인트컷 작성

NameMatchMethodPointcut을 상송해서 프로퍼티로 주어진 이름 패턴을 가지고 클래스 이름을 비교하는 ClassFilter를 추가하도록 만들 것이다. 

```python
package springbook.learningtest.jdk.proxy;

import org.springframework.aop.ClassFilter;
import org.springframework.aop.support.NameMatchMethodPointcut;
import org.springframework.util.PatternMatchUtils;

public class NameMatchClassMethodPointcut extends NameMatchMethodPointcut {
    public void setMappedClassName(String mappedClassName) {
        // 모든 클래스를 다 허용하던 디폴트 클래스 필터를 프로퍼티로 
        // 받은 클래스 이름을 이용해서 필터를 만들어 덮어씌운다.
        this.setClassFilter(new SimpleClassFilter(mappedClassName));
    }

    static class SimpleClassFilter implements ClassFilter {
        String mappedName;

        private SimpleClassFilter(String mappedName) {
            this.mappedName = mappedName;
        }

        public boolean matches(Class<?> clazz) {
            // 와일드카드(*)가 들어간 문자열 비교를 지원하는 스프링의 유틸리티 메소드다. 
            // *name, name*, *name* 세 가지 방식을 모두 지원한다.
            return PatternMatchUtils.simpleMatch(mappedName, clazz.getSimpleName());
        }
    }
}
```

적용할 자동 프록시 생성기인 DafaultAdvisorAutoProxyCreator는 등록된 빈 중에서 Advisor 인터페이스를 구현한 것을 모두 찾는다. 그리고 생성되는 모든 빈에 대해 어드바이저의 포인트컷을 적용해보면서 프록시 적용 대상을 선정한다. 빈 클래스가 프록시 선정 대상이라면 프록시를 만들어 원래 빈 오브젝트와 바꿔치기한다. 

- 자동 생성 프록시 확인

몇 가지 특별한 빈 등록과 포인트컷 작성만으로 프록시가 자동으로 만들어질 수 있다는 것을 확인해보자

1. 트랜잭션이 필요한 빈에 트랜잭션 부가기능이 적용됐는가
2. 아무 빈에나 트랜잭션이 부가기능이 적용된 것은 아닌가 
    1. 포인트컷 빈의 클래스 이름 패턴을 변경해서 testUserService 빈에 트랜잭션이 적용되지 않게 해보자.

6.5.3 포인트컷 표현식을 이용한 포인트컷

지금까지 사용했던 포인트컷은 메소드 이름과 클래스의 이름 패턴을 각각 클래스 필터와 메소드 매처 오브젝트로 비교해서 선정하는 것이었다. 포인트컷 표현식을 이용해서 간단하고 효과적인 방법으로 포인트컷의 클래스와 메소드를 선정하는 알고리즘을 작성해보자

- 포인트컷 표현식 문법

`execution([접근제한자 패턴] 타입패턴 [타입패턴.]이름패턴 (타입패턴 | “..”, …) [throws 예외 패턴])`

```python
// 메소드 시그니처를 이용한 포인트컷 표현식 테스트 
@Test
public void methodSignaturePointcut() throws SecurityException, NoSuchMethodException {
    AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
    pointcut.setExpression("execution(public int " +
            "springbook.learningtest.spring.pointcut.Target.minus(int,int) " +
            "throws java.lang.RuntimeException)"); // -> Target 클래스 minus() 메소드 시그니처

    // Target.minus()
    assertThat(pointcut.getClassFilter().matches(Target.class) && 
               pointcut.getMethodMatcher().matches(
                   Target.class.getMethod("minus", int.class, int.class), null), is(true)); 
                   // -> 포인트컷 조건 통과

    // Target.plus()
    assertThat(pointcut.getClassFilter().matches(Target.class) &&
               pointcut.getMethodMatcher().matches(
                   Target.class.getMethod("plus", int.class, int.class), null), is(false));
                   // -> 메소드 매처에서 실패 (메소드 이름이 다름)

    // Bean.method()
    assertThat(pointcut.getClassFilter().matches(Bean.class) &&
               pointcut.getMethodMatcher().matches(
                   Target.class.getMethod("method"), null), is(false));
                   // -> 클래스 필터에서 실패 (Target 클래스가 아님)
}
```

- 타입 패턴과 클래스 이름 패턴

TestUserServiceImpl이라고 변경했던 테스트용 클래스의 이름을 TestUserServce라고 바꿔도 테스트는 성공이다. 포인트컷 표현식의 클래스 이름에 적용되는 패턴은 클래스 이름 패턴이 아니라 타입 패턴이기 때문이다. 

포인트컷 표현식에서 타입 패턴이라고 명시된 부분은 모두 동일한 원리가 적용된다. 

6.5.4 AOP란 무엇인가?

AOP: 애스팩트 지향 프로그래밍, 관점 지향 프로그래밍

aspect: 애플리케이션의 핵심 기능을 담고 있지는 않지만, 애플리케이션을 구성하는 중요한 한 가지 요소이고, 핵심 기능에 부가되어 의미를 갖는 특별한 모듈을 가리킨다.

어드바이스와 포인트컷을 함께 갖고 있다.  

독립된 측면에 존재하는 애스펙트로 분리한 덕에 핵심 기능은 순수하게 그 기능을 담은 코드로만 존재하고 독립적으로 살펴볼 수 있도록 구분된 면에 존재 가능하다. 

이렇게 애플리케이션의 핵심적인 기능에서 부가적인 기능을 분리해서 애스펙트라는 독특한 모듈로 만들어서 설계하고 개발하는 방법을 AOP라고 한다. 

6.5.5 AOP 적용 기술

1. 프록시를 이용한 AOP
    - 프록시로 만들어서 DI로 연결된 빈 사이에 적용해 타깃의 메소드 호출 과정에 참여해서 부가기능을 제공해주도록 만들었다.
    - 독립적으로 개발한 부가기능 모듈을 다양한 타깃 오브젝트의 메소드에 다이내믹하게 적용해주기 위해 가장 중요한 역할을 맡고 있는 게 바로 프록시다.
2. 바이트코드 생성과 조작을 통한 AOP
    - 타깃 오브젝트를 뜯어고쳐서 부가 기능을 직접 넣어주는 직접적인 방법을 사용한다.
    - 컴파일된 타깃의 클래스 파일 자체를 수정하거나 클래스가 JVM에 로딩되는 시점을 가로채서 바이트코드를 조작하는 복잡한 방법을 사용한다.
        - 바이트코드를 조작해서 타깃 오브젝트를 직접 수정해버리면 스프링과 같은 DI 컨테이너의 도움을 받아서 자동 프록시 생성 방식을 사용하지 않아도 AOP를 적용할 수 있기 때문이다.
        - 프록시 방식보다 훨씬 강력하고 유연한 AOP가 가능하기 때문이다.

6.5.6 AOP의 용어

- 타깃: 부가기능을 부여할 대상
- 어드바이스: 타깃에게 제공할 부가기능을 담은 모듈
- 조인 포인트: 어드바이스가 적용될 수 있는 위치
- 포인트컷: 어드바이스를 적용할 조인 포인트를 선별하는 작업 또는 그 기능을 정의한 모듈
- 프록시: 투명하게 존재하면서 부가기능을 제공하는 오브젝트
- 어드바이저: 포인트컷과 어드바이스를 하나씩 갖고 있는 오브젝트
- 애스펙트: AOP의 기본 모듈

6.5.7 AOP 네임스페이스

스프링의 프록시 방식 AOP를 적용하려면 최소한 네 가지 빈을 등록해야 한다.

- 자동 프록시 생성기: DafaultAdvisorAutoProxyCreator 클래스를 빈으로 등록한다.
- 어드바이스: 부가기능을 구현한 클래스를 빈으로 등록한다.
- 포인트컷: 스프링의 AspectJExpressPointcut 을 빈으로 등록하고 expression 프로퍼티에 포인트컷 표현식을 넣어주면 된다.
- 어드바이저: DafaultPointAdvisor 클래스를 빈으로 등록해서 사용한다.

스프링에서는 이렇게 AOP를 위해 기계적으로 적용하는 빈들을 간편한 방법으로 등록할 수 있다. AOP와 관련된 태그를 정의해둔 aop 스키마를 제공한다. 포인트컷이나 어드바이저, 자동 포인트컷 생성기 같은 특별한 기능을 가진 빈들은 별도의 스키마에 정의된 전용 태그를 사용해 정의해주면 편리하다.