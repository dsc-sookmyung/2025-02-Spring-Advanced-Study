# 1장 오브젝트와 의존관계

### 📖 배운 내용

1장에서 가장 중요한 것은 **오브젝트의 설계 방법, 어떤 관계를 맺고 사용되는지** 이다. 오브젝트를 어떻게 설계하고, 분리하고, 관심사에 따라 리팩토링하는 방법 등은 개발자에게 달려있다. 스프링을 사용할 때 깔끔한 코드를 위해 **객체지향 설계와 프로그래밍** 을 열심히 공부해야 한다.
<br>

#### 1.2 DAO의 분리
1.1장에서 알 수 있는 초난감 DAO 코드로 부터 관심사를 기반으로 DAO를 분리하는 방법을 배운다. 처음 만들어진 코드는 서비스의 종료까지 계속해서 변경된다. **변경에 유연한 코드**를 위해 DAO를 바꿔보자. 
<br>
> 중복 코드의 메소드 추출
> :초난감 DAO 코드에서는 중복된 DB 연결 코드가 존재했다. DB에 대한 변경이 있을 경우 한 메소드에서만 변경을 하기 위해 리팩토링을 해보자.

```java
    public void add(User user) throws ClassNotFoundException, SQLException {
		Connection c = getConnection();
		...
	}
	
	public User get(String id) throws ClassNotFoundException, SQLException {
		Connection c = getConnection();
		...
	}
	
	private Connection getConnection() throws ClassNotFoundException, SQLException {
		Class.forName("com.mysql.jdbc.Driver");
		Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook", "spring", "book");
		return c;
	}
```
userDao 클래스에 메소드가 계속 추가 되어도 DB 연결과 관련된 부분은 getConnection() 이기 때문에 이 메소드의 코드만 수정하면 된다.
-> 관심사에 따라 코드를 구분해놓았기 때문에 한 가지 관심에 대한 변경이 일어날 경우 그 부분의 코드만 수정하면 되는 편리함이 생겼다!
<br>

#### 1.3 DAO의 확장

##### 관계설정 책임의 분리

DAO 코드에는 아직 관심사의 분리가 완벽히 이루어지지 않았다. 스프링 개발을 하다 보면 익숙한 아래의 코드로 DAO 코드를 바꿔야 분리할 수 있다.
```java
public UserDao(ConnectionMaker simpleConnectionMaker) {
		this.connectionMaker = simpleConnectionMaker;
	}
```
위 코드 이전의 코드에서는 ```UserDao```의 관심사 밖이지만 클라이언트 오브젝트와 ```UserDao``` 사이의 관계를 ```UserDao``` 내에서 설정했다. 어떤 클래스의 오브젝트를 사용할지를 결정하는 책임을 DAO가 가지고 있었던 것이다.

-> 객체지향 프로그램의 다형성 특징을 이용하여 외부에서 만든 오브젝트를 인터페이스 타입으로 받아서 사용하자. 쉽게 말하자면, **책임을 내가 아닌 외부로 전가**하는 것이다. 따라서 위 코드의 생성자는 클라이언트가 미리 만든 ```ConnectionMaker```의 오브젝트를 전달받을 수 있도록 파라미터가 추가된 것이다.

아래 코드는 클라이언트 측 코드이다. 클라이언트는 ```UserDaoTest```라는 이름이며, 관계설정 책임이 추가된 모습을 볼 수 있다.

```java
public static void main(String[] args) throws ClassNotFoundException, SQLException {
		ConnectionMaker connectionMaker = new DConnectionMaker();
		UserDao dao = new UserDao(connectionMaker);
```
<br>

#### 1.4 제어의 역전(IoC)

- 자신이 사용할 오브젝트를 **스스로 선택, 생성하지 않는다.**
 - 모든 제어 권한은 다른 대상에게 위임되어 있다.
 - 서블릿
	 - 서블릿 제어 권한을 가진 컨테이너가 서블릿 클래스의 오브젝트 만들고 그 안의 메소드를 호출한다.
	 - 서버에 배포 가능하지만 개발자가 직접 제어할 수 있는 방법이 없다.
 - 프레임워크
	 - **라이브러리**를 사용하는 코드는 애플리케이션 흐름을 직접 제어하며, 필요한 기능 있을 때 능동적으로 라이브러리를 사용한다.
	 - 애플리케이션 코드가 **프레임워크**에 의해 사용된다. 따라서 프레임워크 위에 개발한 클래스를 등록해두고, 프레임워크가 흐름을 주도하는 중에 애플리케이션 코드를 사용한다.
<br>

#### 1.5 스프링의 IoC

```@Bean```과 ```@Configuration``` 어노테이션으로 스프링의 빈 팩토리 or 애플리케이션 컨텍스트가 IoC 방식 기능 제공 시 사용할 완벽한 설정정보를 제공할 수 있다. 아래는 두 어노테이션을 적용한 ```DaoFactory```클래스다.

```java
@Configuration
public class DaoFactory{
    @Bean
    public UserDao userDao(){
        return new UserDao(connectionMaker());
    }

    @Bean
    public ConnectionMaker connectionMaker(){
        return new DConnectionMaker();
    }
}
```

> 스프링 IoC의 용어 정리

- 빈(Bean)
 - 스프링이 IoC방식으로 관리하는 오브젝트
- 빈 팩토리(Bean Factory)
 - 스프링의 IoC를 담당하는 핵심 컨테이너, 빈을 등록/생성/조회/관리 기능 담당
	 - 서버에 배포 가능하지만 개발자가 직접 제어할 수 있는 방법이 없다.
 - 프레임워크
	 - **라이브러리**를 사용하는 코드는 애플리케이션 흐름을 직접 제어하며, 필요한 기능 있을 때 능동적으로 라이브러리를 사용한다.
	 - 애플리케이션 코드가 **프레임워크**에 의해 사용된다. 따라서 프레임워크 위에 개발한 클래스를 등록해두고, 프레임워크가 흐름을 주도하는 중에 애플리케이션 코드를 사용한다.

<br>

#### 1.6 싱글톤 레지스트리와 오브젝트 스코프

> 싱글톤과 싱글톤 레지스트리

스프링은 엔터프라이즈 시스템을 위해 만들어진 기술이기 때문에 오브젝트들이 **싱글톤 패턴**을 따른다. 애플리케이션 안에 보통 한 개의 오브젝트만 만들어서 사용하는 것이다. 다만 싱글톤 패턴은 아래와 같은 한계점을 가지고 있다.

- private 생성자를 가지고 있기 때문에 상속 불가
	- 싱글톤 패턴은 생성자를 private으로 제한, private이 아닌 다른 생성자가 없다면 상속이 불가능해 다형성을 적용할 수 없다.
	- 싱글톤에서 스태틱 필드와 메소드를 사용하는데, 이도 마찬가지.
- 어려운 테스트
	- 초기화 과정에서 생성자 등을 통해 사용할 오브젝트를 동적으로 주입하기 힘들다.
	- 싱글톤에서는 테스트 시 필요한 오브젝트를 만들어 사용해야 한다.
- 서버환경에서 싱글톤이 하나만 생성, 보장 X
- 전역 상태를 만들 가능성
	- 사용하는 클라이언트가 정해져 있지 않아 스태틱 메소드를 이용해 쉽게 접근 가능 -> 객체지향 프로그래밍에서 권장 X

여러 한계점이 있는 싱글톤 패턴 구현 방식 대신 **싱글톤 레지스트리**를 사용한다. 싱글톤 레지스트리는 스프링이 직접 싱글톤 형태의 오브젝트를 만들고 관리하며, 이 기능을 사용자에게 제공한다.
<br>

> 스프링 빈의 스코프

스프링 빈의 기본 스코프는 싱글톤이다. **싱글톤 스코프**는 컨테이너 내에 한 개의 오브젝트만 만들어져 계속 유지된다. 싱글톤 외의 스코프로는 프로토타입 스코프가 있다. 프로토타입은 컨테이너에 빈을 요청할 때마다 매번 새로운 오브젝트를 만들어준다. 이외에도 요청 스코프, 세션 스코프 등이 있다.
<br>

#### 1.7 의존관계 주입(DI)

##### 의존관계 주입의 핵심

설계할 때는 알지 못 했던 오브젝트들의 관계를 맺도록 도와주는 또 다른 존재가 있다!
또 다른 존재는 관계설정 책임을 가진 코드를 분리해서 만들어진 오브젝트이다. (UserDaoTest에게 책임을 전가했던 1.3장을 생각해보라.)
