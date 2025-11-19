# 6.4~6.5 AOC

## 6.4 스프링의 프록시 팩토리 빈

1. 기존 방식의 문제점

- 기존의 `TransactionHandler`와 같은 방식은 핸들러 자체가 타깃 오브젝트(Target Object)를 들고 있어야 했음

- 타깃이 달라질 때마다 새로운 핸들러 오브젝트를 생성해야 했고, 핸들러를 싱글톤 빈(Singleton Bean)으로 등록하여 공유할 수 없었음

- 설정 파일에 타깃의 개수만큼 중복된 설정이 발생

2. 해결책: 스프링 `ProxyFactoryBean`

스프링은 프록시 생성 로직을 추상화한 `ProxyFactoryBean`을 제공한다. 이 팩토리 빈은 프록시를 생성하는 작업만 담당하고, 실제 부가기능(Advice)은 별도로 관리한다.

3. `MethodInterceptor`의 도입

기존의 `InvocationHandler` 대신 `MethodInterceptor`를 사용한다. `InvocationHandler`의 `invoke()` 메소드는 타깃 정보를 파라미터로 받지 않아 핸들러가 타깃을 가지고 있어야 했지만, `MethodInterceptor`의 `invoke()`는 타깃 오브젝트에 대한 정보가 담긴 `MethodInvocation` 오브젝트를 파라미터로 받는다.

- 장점: `MethodInterceptor`는 타깃 오브젝트에 의존하지 않기 때문에 상태를 가지지 않는(Stateless) 싱글톤 빈으로 등록하여 여러 프록시에서 공유할 수 있다.

4. 자동화된 프록시 기술 선택

`ProxyFactoryBean`은 인터페이스 유무 등을 판단하여 JDK 다이내믹 프록시를 사용할지, CGLIB 등을 사용할지 자동으로 결정해준다.

⭐ 스프링 ProxyFactoryBean을 이용한 다이내믹 프록시 테스트 코드
```java
package springbook.learningtest.jdk.proxy;

import org.junit.Test;
import org.springframework.aop.framework.ProxyFactoryBean;
import static org.hamcrest.CoreMatchers.is;
import static org.junit.Assert.assertThat;

public class DynamicProxyTest {

    @Test
    public void simpleProxy() {
        // JDK 다이내믹 프록시 생성 (비교용 기존 코드)
        Hello proxiedHello = (Hello)java.lang.reflect.Proxy.newProxyInstance(
            getClass().getClassLoader(),
            new Class[] { Hello.class },
            new UppercaseHandler(new HelloTarget()));
        
        // (이후 검증 로직 생략됨)
    }

    @Test
    public void proxyFactoryBean() {
        ProxyFactoryBean pfBean = new ProxyFactoryBean();
        
        // 타깃 설정
        pfBean.setTarget(new HelloTarget()); 
        
        // 부가기능을 담은 어드바이스를 추가
        // (여러 개를 추가할 수도 있다)
        pfBean.addAdvice(new UppercaseAdvice()); 

        // FactoryBean이므로 getObject()로 생성된 프록시를 가져옴
        Hello proxiedHello = (Hello) pfBean.getObject(); 
        
        // 아래는 책의 다음 페이지로 이어질 내용(예측되는 검증 코드)
        assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
        assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
        assertThat(proxiedHello.sayThankYou("Toby"), is("THANK YOU TOBY"));
        
    }

    // MethodInterceptor 구현 클래스 (Advice)
    static class UppercaseAdvice implements MethodInterceptor {
        public Object invoke(MethodInvocation invocation) throws Throwable {
            // 리플렉션의 Method와 달리 메소드 실행 시 타깃 오브젝트를 전달할 필요가 없다.
            // MethodInvocation은 메소드 정보와 함께 타깃 오브젝트를 알고 있기 때문이다.
            String ret = (String)invocation.proceed(); // 타깃 메소드 실행
            return ret.toUpperCase(); // 부가기능 적용 (대문자 변환)
        }
    }

    // 타깃과 프록시가 구현할 인터페이스
    static interface Hello {
        String sayHello(String name);
        String sayHi(String name);
        String sayThankYou(String name);
    }

    // 타깃 클래스
    static class HelloTarget implements Hello {
        public String sayHello(String name) { return "Hello " + name; }
        public String sayHi(String name) { return "Hi " + name; }
        public String sayThankYou(String name) { return "Thank You " + name; }
    }
}
```

- `MethodInterceptor` vs `InvocationHandler`

기존의 방법인 `InvocationHandler`: 타깃 오브젝트를 내부 변수로 가지고 있어야 했다. 타깃이 다르면 매번 새로운 핸들러 객체를 만들어야 했다.

개선 방법인 `MethodInterceptor`: `invoke()` 메소드의 파라미터로 **MethodInvocation**을 받는다. 이 객체 안에 타깃 오브젝트와 메소드 정보가 모두 담겨 있다.

* 장점: `MethodInterceptor`는 타깃 정보에 의존하지 않는 **순수한 부가기능(Advice)**만 담당하므로, 상태를 가지지 않는(Stateless) 싱글톤 빈으로 만들어 여러 프록시에서 공유할 수 있다.

- 템플릿/콜백 구조의 적용

`ProxyFactoryBean`은 프록시 생성이라는 공통 로직(템플릿)을 담당하고, `MethodInterceptor`는 개별적인 부가기능(콜백)을 담당하는 구조이다.`MethodInvocation`은 일종의 콜백 오브젝트로, `proceed()` 메소드를 통해 실제 타깃의 메소드를 실행한다.

- 다중 부가기능 허용 `addAdvice`

`ProxyFactoryBean`은 `setTarget` 대신 `addAdvice`라는 메소드를 사용한다. 하나의 프록시 팩토리에 **여러 개의 부가기능(Advice)**을 추가할 수 있음을 의미한다. 프록시를 여러 번 감쌀 필요 없이 한 번에 여러 기능을 적용할 수 있다.

- 인터페이스 자동 감지

JDK 다이내믹 프록시를 직접 만들 때는 구현할 인터페이스를 명시해야 했지만, `ProxyFactoryBean`은 타깃 오브젝트가 구현하고 있는 인터페이스를 자동으로 알아낸다.

- **포인트컷**의 필요성
    - 스프링 `ProxyFactoryBean`의 특징
        - 인터페이스 자동 감지: 굳이 `setInterfaces()`를 쓰지 않아도 타깃 오브젝트가 구현한 인터페이스를 자동으로 감지하여 프록시를 만든다.
        - 기술의 추상화: JDK 다이내믹 프록시 외에도 필요에 따라 CGLIB(오픈소스 바이트코드 생성 프레임워크) 등을 이용해 프록시를 만들 수 있도록 지원한다.
    - 핵심 딜레마: 공유 가능한 어드바이스 vs 메소드 선정 기능
        - 목표: `MethodInterceptor`(어드바이스)를 스프링의 싱글톤 빈으로 등록하여 여러 프록시에서 공유해서 쓰는 것
        - 문제: 트랜잭션처럼 "특정 메소드(예: save로 시작하는 메소드)에만 적용해라"라는 **선정 기능(패턴)**을 `MethodInterceptor` 안에 넣으면, 이 어드바이스는 특정 프록시 전용이 되어버려 공유할 수 없게 된다.
    - 기존 방식(JDK 다이내믹 프록시)의 한계
        - 기존 `InvocationHandler` 방식은 **'부가기능(Advice)'**과 **'메소드 선정 알고리즘(Pointcut)'**이 한 곳에 뭉쳐 있었다.
        - 타깃이 달라지거나 메소드 선정 방식이 달라질 때마다 핸들러를 새로 만들어야 했으므로, 빈으로 등록하여 재사용하기가 불가능했다.
        - 결론: 유연한 확장을 위해 **'메소드를 선정하는 기능'**과 **'부가기능을 수행하는 기능'**을 분리해야 한다.

- 구조의 변화: 유연한 확장 (OCP 준수)
기존 방식은 어드바이스 내부에 메소드 선정 로직이 섞여 있어, 선정 기준이 바뀌거나 타깃이 바뀌면 코드를 수정해야 했다.부가기능을 제공하는 **'어드바이스'**와 메소드 선정 알고리즘을 담은 **'포인트컷'**을 완전히 분리했다. 덕분에 두 객체 모두 싱글톤 빈으로 등록하여 여러 프록시에서 공유가 가능하다.

- 프록시의 작동 방식
프록시는 클라이언트로부터 요청을 받으면 먼저 포인트컷에게 "이 메소드에 부가기능을 적용해야 하니?"라고 물어본다. 포인트컷이 확인(Check)해주면, 그제서야 어드바이스를 호출한다. 어드바이스는 자신이 적용될 타깃을 직접 호출하지 않고, `MethodInvocation` (일종의 콜백)을 통해 실행한다.

- 어드바이저 (Advisor)
`ProxyFactoryBean`에는 포인트컷과 어드바이스를 따로 등록하지 않고, 이 둘을 묶은 객체를 등록하며 이를 **어드바이저(Advisor)**라고 부른다.
어드바이저 = 포인트컷(메소드 선정) + 어드바이스(부가기능)

- 템플릿/콜백 패턴의 완성
`MethodInvocation`이 일종의 콜백 역할을 하고, 어드바이스가 템플릿 역할을 하여 타깃 메소드를 실행하는 구조이다. 이 덕분에 어드바이스는 재사용 가능한 순수 로직으로 남을 수 있다.

⭐ 포인트컷까지 적용한 ProxyFactoryBean 학습 테스트
```java
package springbook.learningtest.jdk.proxy;

import org.junit.Test;
import org.springframework.aop.framework.ProxyFactoryBean;
import org.springframework.aop.support.DefaultPointcutAdvisor;
import org.springframework.aop.support.NameMatchMethodPointcut;
import static org.hamcrest.CoreMatchers.is;
import static org.junit.Assert.assertThat;

public class DynamicProxyTest {
    // ... (이전 코드 생략) ...

    @Test
    public void pointcutAdvisor() {
        ProxyFactoryBean pfBean = new ProxyFactoryBean();
        pfBean.setTarget(new HelloTarget());

        // 메소드 이름을 비교해서 대상을 선정하는 알고리즘을 제공하는 포인트컷 생성
        NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
        // 이름 비교조건 설정. 'sayH'로 시작하는 모든 메소드를 선택하게 한다.
        pointcut.setMappedName("sayH*");

        // 포인트컷과 어드바이스를 Advisor로 묶어서 한 번에 추가
        pfBean.addAdvisor(new DefaultPointcutAdvisor(pointcut, new UppercaseAdvice()));

        Hello proxiedHello = (Hello) pfBean.getObject();

        // "sayH"로 시작하므로 포인트컷 선정 O -> 부가기능 적용됨 (대문자 변환)
        assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
        assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));

        // "sayH"로 시작하지 않으므로 포인트컷 선정 X -> 부가기능 미적용 (원래 값 리턴)
        // (대문자 변환이 적용되지 않는다)
        assertThat(proxiedHello.sayThankYou("Toby"), is("Thank You Toby"));
    }
}
```

- 왜 어드바이저(Advisor)가 필요한가?
ProxyFactoryBean에는 여러 개의 어드바이스와 여러 개의 포인트컷이 등록될 수 있다. 만약 이들을 따로 등록하면, '어떤 어드바이스(기능)'를 '어떤 포인트컷(대상)'에 적용해야 할지 모호해진다. 따라서 이 둘을 짝지어 어드바이저(Advisor = Pointcut + Advice) 라는 단위로 묶어서 등록해야 한다.

- `TransactionAdvice` 구현 (기존 핸들러 대체)
기존에 만들었던 `TransactionHandler`(JDK 다이내믹 프록시용)를 스프링의 **MethodInterceptor**를 구현한 TransactionAdvice로 교체한다.
    - 차이점: 타깃 오브젝트를 내부에 저장할 필요가 없다. (메소드 실행 시 `MethodInvocation`을 통해 전달됨) 따라서 상태를 가지지 않는 싱글톤 빈으로 등록하여 여러 서비스에서 재사용할 수 있다. 예외 처리(Throwable)가 훨씬 깔끔해진다. (JDK 프록시처럼 InvocationTargetException으로 포장되지 않고 그대로 넘어온다.)

⭐ 트랜잭션 어드바이스
```java
package springbook.user.service;

import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.TransactionStatus;
import org.springframework.transaction.support.DefaultTransactionDefinition;

public class TransactionAdvice implements MethodInterceptor { // 스프링의 어드바이스 인터페이스 구현
    
    private PlatformTransactionManager transactionManager;

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    // 타깃을 호출하는 기능을 가진 콜백 오브젝트를 프록시로부터 받는다.
    // 덕분에 어드바이스는 특정 타깃에 의존하지 않고 재사용 가능하다.
    public Object invoke(MethodInvocation invocation) throws Throwable {
        TransactionStatus status = 
            this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        
        try {
            // 콜백을 호출해서 타깃의 메소드를 실행한다.
            // 타깃 메소드 호출 전후로 필요한 부가기능을 넣을 수 있다.
            // 경우에 따라서 타깃이 아예 호출되지 않게 하거나 
            // 재시도를 위한 반복적인 호출도 가능하다.
            Object ret = invocation.proceed(); 
            
            this.transactionManager.commit(status);
            return ret;
            
        // JDK 다이내믹 프록시가 제공하는 Method와는 달리
        // 스프링의 MethodInvocation을 통한 타깃 호출은 
        // 예외가 포장되지 않고 타깃에서 보낸 그대로 전달된다.
        } catch (RuntimeException e) { 
            this.transactionManager.rollback(status);
            throw e;
        }
    }
}
```