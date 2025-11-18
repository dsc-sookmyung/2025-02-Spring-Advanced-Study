# 6. AOP

## 6.4 스프링의 프록시 팩토리 빈

### 6.4.1 ProxyFactoryBean

- 스프링의 프록시 추상화
    - 프록시 기술(JDK 다이내믹 프록시, CGLIB 등)에 대한 서비스 추상화
    - 프록시 생성 책임을 ProxyFactoryBean(팩토리 빈) 으로 위임
    - 프록시는 스프링 컨테이너에 일반 빈처럼 등록 및 DI 대상이 됨
- ProxyFactoryBean의 역할
    - 타깃 오브젝트 설정 (`setTarget()`)
    - 부가기능(어드바이스) 설정 (`addAdvice()`, `addAdvisor()`)
    - 프록시 생성 및 반환 (`getObject()`)

```java
public class DynamicProxyTest {
    @Test
    public void proxyFactoryBean() {
        ProxyFactoryBean pfBean = new ProxyFactoryBean();
        pfBean.setTarget(new HelloTarget());          // 타깃 설정
        pfBean.addAdvice(new UppercaseAdvice());      // 부가기능(어드바이스) 추가

        Hello proxiedHello = (Hello) pfBean.getObject(); // 프록시 획득

        assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
        assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
        assertThat(proxiedHello.sayThankYou("Toby"), is("THANK YOU TOBY"));
    }

    private static class UppercaseAdvice implements MethodInterceptor {
        @Override
        public Object invoke(MethodInvocation invocation) throws Throwable {
            String ret = (String) invocation.proceed();  // 타깃 메소드 호출
            return ret.toUpperCase();                    // 부가기능(대문자 변환)
        }
    }
}

```

- JDK 다이내믹 프록시 vs ProxyFactoryBean
    - JDK 다이내믹 프록시
        - `Proxy.newProxyInstance()` 직접 호출
        - `InvocationHandler`가 타깃을 직접 필드로 가짐
        - 타깃 인터페이스 배열을 직접 명시해야 함
    - ProxyFactoryBean
        - 프록시 생성 로직 캡슐화
        - `MethodInterceptor`는 타깃을 알지 않고, `MethodInvocation.proceed()`만 호출
        - 타깃이 구현한 인터페이스를 스프링이 **자동 탐색**
- 개인적 정리
    - 다이내믹 프록시 = “코드로 프록시를 직접 만드는 방식”
    - ProxyFactoryBean = “프록시 생성 자체를 설정으로 빼낸 DI 친화적 추상화”
        
        → 프록시를 “프로그래밍”이 아니라 “설정”의 문제로 바꿔버림
        

---

### 6.4.2 ProxyFactoryBean 적용

- 어드바이스(Advice)
    - `MethodInterceptor`를 구현해서 만드는 순수 부가기능 모듈
    - 타깃에 의존하지 않음 → 여러 프록시에서 재사용 가능
    - 예: 트랜잭션, 로깅, 보안, 성능 측정 등

```java
public class TransactionAdvice implements MethodInterceptor {
    private PlatformTransactionManager transactionManager;

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        TransactionStatus status =
                this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            Object ret = invocation.proceed();   // 타깃 메소드 호출
            this.transactionManager.commit(status);
            return ret;
        } catch (RuntimeException e) {
            this.transactionManager.rollback(status);
            throw e;
        }
    }
}

```

- ProxyFactoryBean을 이용한 트랜잭션 적용
    - 트랜잭션 부가기능을 하나의 어드바이스 빈으로 만들어 여러 서비스에 공유
    - 타깃 교체 시에도 트랜잭션 로직 수정 없이 설정만 변경
- 테스트에서 ProxyFactoryBean 재사용

```java
ProxyFactoryBean txProxyFactoryBean =
        context.getBean("&userService", ProxyFactoryBean.class); // FactoryBean 자체 조회

txProxyFactoryBean.setTarget(testUserService);   // 테스트용 타깃 교체
UserService txUserService = (UserService) txProxyFactoryBean.getObject();

```

- 개인적 정리
    - 트랜잭션 코드를 “UserServiceTx 클래스”로 빼내는 것에서 한 단계 더 나아가,
    - **트랜잭션 자체를 어드바이스 빈 하나로 만들고, 어떤 서비스든 설정만으로 붙였다 뗐다 할 수 있게 만든 단계**라고 보면 이해가 쉽다.

---

## 6.5 스프링 AOP

### 6.5.1 자동 프록시 생성

- 문제 상황
    - 프록시가 필요한 빈이 늘어나면, ProxyFactoryBean 설정이 **빈마다 반복**
    - 트랜잭션 대상 서비스가 많을수록 XML, 자바 설정이 복잡해짐
- 해결: 빈 후처리기(BeanPostProcessor) 기반 자동 프록시 생성기
    - 스프링은 빈이 생성된 후에 한 번 더 가공할 수 있는 확장 포인트 제공
    - `DefaultAdvisorAutoProxyCreator`가 그 중 하나
        - Advisor 목록 수집
        - 생성되는 빈마다 “이 빈에 프록시를 씌울 것인가” 판단
        - 프록시가 필요하면 타깃 대신 프록시를 컨테이너에 등록
- 작동 과정 정리
    1. 컨테이너가 일반 빈을 생성
    2. `DefaultAdvisorAutoProxyCreator`가 해당 빈과 등록된 Advisor들의 포인트컷을 비교
    3. 포인트컷에 걸리는 빈이면 프록시 생성
    4. 프록시에 어드바이저(=포인트컷+어드바이스) 연결
    5. 최종적으로 컨테이너에는 프록시가 빈으로 등록
    6. 다른 빈들은 타깃이 아니라 프록시를 DI 받음
- ProxyFactoryBean이 “프록시를 직접 등록하는 방법”이라면,
- DefaultAdvisorAutoProxyCreator는 “프록시를 자동으로 씌우는 필터” 느낌
    
    → “ServiceImpl 이면서 upgrade* 메서드를 가진 빈은 알아서 트랜잭션 프록시 적용” 같은 규칙 정의 가능
    

---

### 6.5.2 확장된 포인트컷 (ClassFilter + MethodMatcher)

- 기존 포인트컷
    - 메서드 이름만 비교 (`NameMatchMethodPointcut.setMappedName("sayH*")`)
    - 자동 프록시 생성 단계에서는 **“어떤 클래스에 프록시를 적용할지”**도 필요
- 포인트컷의 두 가지 구성 요소
    - `ClassFilter` : 프록시 적용 대상 클래스 선정
    - `MethodMatcher` : 어드바이스를 적용할 메서드 선정
- 클래스 이름까지 고려하는 포인트컷 구현

```java
public class NameMatchClassMethodPointcut extends NameMatchMethodPointcut {
    public void setMappedClassName(String mappedClassName) {
        this.setClassFilter(new SimpleClassFilter(mappedClassName));
    }

    private static class SimpleClassFilter implements ClassFilter {
        private String mappedName;

        public SimpleClassFilter(String mappedName) {
            this.mappedName = mappedName;
        }

        @Override
        public boolean matches(Class<?> clazz) {
            return PatternMatchUtils.simpleMatch(mappedName, clazz.getSimpleName());
        }
    }
}

```

- 예시 설정
    - 클래스 이름: `ServiceImpl`
    - 메서드 이름: `upgrade*`

```xml
<bean id="transactionPointcut"
      class="springbook.service.NameMatchClassMethodPointcut">
    <property name="mappedClassName" value="*ServiceImpl"/>
    <property name="mappedName" value="upgrade*"/>
</bean>

```

---

### 6.5.3 포인트컷 표현식(AspectJExpressionPointcut)

- NameMatch 방식의 한계
    - 클래스/메서드 패턴을 프로퍼티로 따로따로 지정
    - 패키지, 파라미터 타입, 예외 타입 등 정교한 조건 지정 어려움
- 해결: AspectJ 포인트컷 표현식 지원
    - `AspectJExpressionPointcut` 사용
    - `execution(...)` 표현식으로 하나의 문자열에 규칙 정의

```java
AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
pointcut.setExpression(
    "execution(* *..*ServiceImpl.upgrade*(..))"
);

```

- execution 문법 (요약)
    - `execution( [접근제한자] 리턴타입 패턴 [패키지.타입 패턴].메서드이름패턴(파라미터 패턴..) [throws 예외패턴] )`
    - 예:
        - `execution(* com.example.service.*Service.*(..))`
        - `execution(public int com.example.Target.minus(int, int))`
- 개인적 정리
    - NameMatch는 if-문 수준 필터라면,
    - AspectJ 표현식은 **SQL WHERE 절 같은 필터** 느낌
        
        → “이런 시그니처를 가진 메서드 전체” 라고 선언적으로 필터링
        

---

### 6.5.4 AOP란 무엇인가?

- AOP(Aspect Oriented Programming, 관점 지향 프로그래밍)
    - 핵심 관심사(Core Concern)와 횡단 관심사(Cross-cutting Concern) 분리
    - 트랜잭션, 로깅, 보안, 캐싱 등 여러 곳에 흩어지는 부가기능을 한 곳(Aspect)에 모아서 관리
- 핵심 용어 정리
    - 타깃(Target)
        - 부가기능이 적용되는 원본 객체
    - 어드바이스(Advice)
        - 부가기능 자체를 담은 객체 (ex. `TransactionAdvice`, `UppercaseAdvice`)
    - 조인 포인트(Join Point)
        - 어드바이스가 끼어들 수 있는 지점 (메서드 호출, 예외 발생 등)
    - 포인트컷(Pointcut)
        - 조인 포인트 중 실제로 어드바이스를 적용할 대상 선정 규칙
    - 어드바이저(Advisor)
        - 포인트컷 + 어드바이스를 한 쌍으로 갖는 객체
    - 프록시(Proxy)
        - 클라이언트와 타깃 사이에 서서 부가기능을 제공하는 객체
    - 애스펙트(Aspect)
        - AOP의 기본 모듈 단위, 하나 이상의 어드바이스 + 포인트컷의 묶음

---

### 6.5.5 AOP 적용 기술

- 프록시 기반 AOP
    - 스프링이 사용하는 기본 방식
    - 프록시 객체를 만들어 클라이언트와 타깃 사이에 주입
    - DI 컨테이너와 결합 → 매우 자연스러운 적용 (주로 서비스 계층)
- 바이트코드 조작 기반 AOP
    - 컴파일/로딩 시점에 클래스 바이트코드를 직접 수정
    - 프록시 없이 타깃 코드에 부가기능을 삽입
    - 더 강력하고 유연하지만, 설정/환경이 복잡 (AspectJ compile-time / load-time weaving 등)
- 개인적 정리
    - 스프링 AOP = “프록시로 충분히 처리 가능한 대부분의 실무 케이스용”
    - 바이트코드 AOP = “프록시로 잡히지 않는 세밀한 지점(생성자, 필드 접근 등)까지 건드리고 싶을 때”

---

### 6.5.7 AOP 네임스페이스

- AOP 관련 빈 등록 요소
    - 자동 프록시 생성기 (`DefaultAdvisorAutoProxyCreator`)
    - 어드바이스 빈 (ex. `TransactionAdvice`)
    - 포인트컷 빈 (`AspectJExpressionPointcut` 등)
    - 어드바이저 빈 (`DefaultPointcutAdvisor`)
- 문제점
    - 위 요소들을 전부 `<bean>`으로 등록하면 설정이 장황해짐
- 해결: `<aop:config>` 네임스페이스
    - 포인트컷, 어드바이저, 자동 프록시 생성까지 한 번에 정의

```xml
<aop:config>
    <aop:pointcut id="txPointcut"
                  expression="execution(* *..*ServiceImpl.upgrade*(..))"/>
    <aop:advisor advice-ref="transactionAdvice"
                 pointcut-ref="txPointcut"/>
</aop:config>

```