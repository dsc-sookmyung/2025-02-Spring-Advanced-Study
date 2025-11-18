# 6장 AOP(6.4-6.5)

### **6.4 스프링의 프록시 팩토리 빈**

스프링은 서비스 추상화를 프록시 기술에도 동일하게 적용하고 있다.

스프링은 일관된 방법으로 프록시를 만들 수 있게 도와주는 **추상 레이어**를 제공한다.

🪻 ProxyFactoryBean

: 프록시를 생성해서 빈 오브젝트로 등록하게 해주는 팩토리 빈, 순수하게 프록시를 생성하는 작업만을 담당하고 프록시를 통해 제공해줄 부가기능은 별도의 빈에 둘 수 있다.

```jsx
public class DynamicProxyTest {
    @Test
    public void simpleProxy(){
        // JDK 다이내믹 프록시 생성
        Hello proxiedHello = (Hello) Proxy.newProxyInstance(
                getClass().getClassLoader(),
                new Class[] {Hello.class},
                new UppercaseHandler(new HelloTarget())
        );
    }

    @Test
    public void proxyFactoryBean(){
        ProxyFactoryBean pfBean = new ProxyFactoryBean();
        pfBean.setTarget(new HelloTarget()); // 타깃 설정
        pfBean.addAdvice(new UppercaseAdvice()); // 부가기능. 여러개 가능

        //FactoryBean이므로 getObject()로 생성된 프록시를 가져옴
        Hello proxiedHello = (Hello) pfBean.getObject();

        assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
        assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
        assertThat(proxiedHello.sayThankYou("Toby"), is("THANK YOU TOBY"));
    }

    private static class UppercaseAdvice implements MethodInterceptor {
        @Override
        public Object invoke(MethodInvocation invocation) throws Throwable {
            //InvocationHandler와 달리 target이 필요 없음
            String ret = (String)invocation.proceed();
            return ret.toUpperCase();
        }
    }
}
```

🪻 어드바이스 : 타깃이 필요 없는 순수한 부가기능

ProxyFactoryBean을 적용한 코드를 기존의 JDK 다이내믹 프록시를 사용했던 코드와 비교해보자.

▪️ JDK 다이내믹 프록시에서는 타깃 오브젝트가 반드시 필요하지만, ProxyFactoryBean에서는 MethodInterceptor를 통해 타깃과 독립적으로 부가기능을 구현할 수 있다.
MethodInterceptor는 타깃을 직접 참조하지 않고, MethodInvocation 객체를 통해 타깃의 메소드를 호출한다. ( → MethodInterceptor는 부가기능을 제공하는 데만 집중할 수 있게 된다. )

▪️ 기존 JDK 다이내믹 프록시는 반드시 타깃 오브젝트가 구현하는 인터페이스를 명시적으로 제공해야 했다. ProxyFactoryBean은 타깃 오브젝트가 구현하고 있는 **인터페이스를 자동으로 검출**하여 프록시를 생성한다.

▪️ ProxyFactoryBean은 `addAdvice()` 메소드를 통해 여러 개의 Advice (MethodInterceptor)를 추가할 수 있다 ( → 새로운 부가기능을 추가할 때마다 별도의 프록시나 팩토리 빈을 만들 필요가 없다 )

**포인트컷: 부가기능 적용 대상 메소드 선정 방식**

스프링의 ProxyFactoryBean을 이용한 방식

1. 프록시는 클라이언트로부터 요청을 받으면, 먼저 포인트컷에게 부가기능을 부여할 메서드인지를 확인해달라고 요청한다.
2. 포인트컷으로부터 부가기능을 적용할 대상 메서드인지 확인받으면 MethodInterceptor 타입의 어드바이스를 호출한다.
3. 어드바이스에서 target 메서드의 호출이 필요하면 MethodInvocation 타입 콜백 오브젝트의proceed()메서드를 호출한다.

여기에서 부가기능을 제공하는 오브젝트를 어드바이스라고 하고, 메서드 선정 알고리즘을 담은 오브젝트는 포인트컷이라고 한다.

🪻**어드바이저**

어드바이스와 포인트컷을 묶은 오브젝트를 인터페이스이며,

어드바이저 = 포인트컷(메소드 선정 알고리즘) + 어드바이스(부가기능)

### **6.5 스프링의 AOP**

🪻 **빈 후처리기를 이용한 자동 프록시 생성기**

: 스프링 빈 오브젝트로 만들어지고 난 후에, 빈 오브젝트를 다시 가공할 수 있게 해준다

- DefaultAdvisorAutoProxyCreator 빈 후처리기가 등록되어 있으면 스프링은 빈 오브젝트를 만들 때마다 후처리기에게 빈을 보낸다.
- DefaultAdvisorAutoProxyCreator는 빈으로 등록된 모든 어드바이저 내의 포인트컷을 이용해 전달받은 빈이 프록시 적용 대상인지 확인
- 프록시 적용 대상인 경우 내장된 프록시 생성기에 현재 빈에 대한 프록시를 생성하게 한다.
- 만들어진 프록시에 어드바이저를 연결해준다.
- 프록시가 생성되면 빈 후처리기는 원래 컨테이너가 전달해준 빈 오브젝트 대신 프록시 오브젝트를 컨테이너에 돌려준다.
- 컨테이너는 최종적으로 빈 후처리기가 돌려준 오브젝트를 빈으로 등록하고 사용

🪻 **확장된 포인트컷**

사실 포인트 컷은 두 가지 기능을 모두 가지고 있다.

포인트컷의 기능을 간단한 학습 테스트로 확인해보자.

```jsx
public class DynamicProxyTest {
    ...
    @Test
    public void classNamePointcutAdvisor(){
		    // 포인트컷 준비
        NameMatchMethodPointcut classMethodPointcut = new NameMatchMethodPointcut(){
            @Override
            public ClassFilter getClassFilter() { // 익명 내부 클래스 방식으로 클래서 정의
                return new ClassFilter() {
                    @Override
                    public boolean matches(Class<?> clazz) {
                        //class 이름이 HelloT로 시작하는 것만 선정
                        return clazz.getSimpleName().startsWith("HelloT"); // 클래스 이름이 HelloT로 시작하는 것만 선정
                    }
                };
            }
        };
        classMethodPointcut.setMappedName("sayH*"); // sayH로 시작하는 메소드 이름을 가진 메소드만 선정됨

        //테스트
        checkAdviced(new HelloTarget(), classMethodPointcut, true);

        class HelloWorld extends HelloTarget{};
        checkAdviced(new HelloWorld(), classMethodPointcut, false);

        class HelloToby extends HelloTarget{};
        checkAdviced(new HelloToby(), classMethodPointcut, true);
    }

    private void checkAdviced(Object target, Pointcut pointcut,
                              boolean adviced) {
        ProxyFactoryBean pfBean = new ProxyFactoryBean();
        pfBean.setTarget(target);
        pfBean.addAdvisor(new DefaultPointcutAdvisor(pointcut, new UppercaseAdvice()));
        Hello proxiedHello = (Hello) pfBean.getObject();

        if(adviced){
            assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
            assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
            assertThat(proxiedHello.sayThankYou("Toby"), is("Thank You Toby"));
        }else{
            assertThat(proxiedHello.sayHello("Toby"), is("Hello Toby"));
            assertThat(proxiedHello.sayHi("Toby"), is("Hi Toby"));
            assertThat(proxiedHello.sayThankYou("Toby"), is("Thank You Toby"));
        }
    }
}
```

**DefaultAdvisorAutoProxyCreator의 적용**

🪻 **클래스 필터를 적용한 포인트컷 작성**

: 메소드 이름만 비교하던 포인트컷(NameMatchMethodPointcut)을 상속해서 프로퍼티로 주어진 이름 패천을 가지고 클래스 이름을 비교하는 ClassFilter를 추가

```jsx
public class NameMatchClassMethodPointcut extends NameMatchMethodPointcut {
    public void setMappedClassName(String mappedClassName){
        this.setClassFilter(new SimpleClassFilter(mappedClassName));
    }

    private class SimpleClassFilter implements ClassFilter {
        String mappedName;

        public SimpleClassFilter(String mappedClassName) {
            this.mappedName = mappedClassName;
        }

        @Override
        public boolean matches(Class<?> clazz) {
            return PatternMatchUtils.simpleMatch(mappedName,
                    clazz.getSimpleName());
        }
    }
}
```

🪻 **어드바이저를 이용하는 자동 프록시 생성기 등록**

▪️ DefaultAdvisorAutoProxyCreator는 등록된 빈 중에서 Advisor 인터페이스를 구현한 것을 모두 찾는다.

▪️ 적용 가능한 빈에 대해 어드바이저 포인트컷을 적용하면서 프록시 적용 대상을 선정한여 원래 빈 오브젝트와 바꿔치기 한다.

▪️ 타깃 빈에 의존한다고 정의한 다른 빈들은 프록시 오브젝트를 대신 DI 받게 된다.

🪻 **포인트컷 등록**

기존의 포인트컷 설정을 삭제하고 새로 만들 클래스 필터 지원 포인트컷을 빈으로 등록한다.

```jsx
<beans
    ...>
    <bean id="transactionPointcut"
          class="springbook.service.NameMatchClassMethodPointcut">
        <property name="mappedClassName" value="*ServiceImpl"/>
        <property name="mappedName" value="upgrade*"/>
    </bean>
<beans/>
```

→ 이제 DefaultAdvisorAutoProxyCreator에 의해 어드바이저가 자동 수집되고, 프록시 대상 선정 과정에 참여하여, 자동생성된 프록시에 다이내믹하게 DI 돼서 동작하는 어드바이저가 되었다.

🪻 **포인트컷 표현식을 이용한 포인트컷**

스프링은 간단하고 효과적인 방법으로 포인트컷의 클래스와 메소드를 선정하는 알고리즘을 작성하는 방법을 제공한다.

AspectJExpressionPointcut 클래스를 사용하면 클래스와 메소드의 선정 알고리즘을 포인트컷 표현식을 이용해 한 번에 지정할 수 있다.
