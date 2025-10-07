## 1.1 초난감 DAO

### 1.1.1 User

사용자 정보를 담는 객체

```java
public class User {
    String id;
    String name;
    String password;

    public String getId() { return id; }
    public void setId(String id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
}
```

### 1.1.2 UserDao

사용자 정보를 DB에 넣고 관리하는 클래스

```java
public class UserDao {
    public void add(User user) throws ClassNotFoundException, SQLException {
        Class.forName("com.mysql.Driver");
        Connection c = DriverManager.getConnection(~); // DB 연결을 위한 Connection 객체
        PreparedStatement ps = c.prepareStatement( // SQL 실행을 위한 객체
            "insert into users(id, name, password) values(?,?,?)"
        );
        ps.setString(1, user.getId()); 
        ps.setString(2, user.getName());
        ps.setString(3, user.getPassword());

        ps.executeUpdate();
    }

    public User get(String id) throws ClassNotFoundException, SQLException {
        Class.forName("com.mysql.Driver");
        Connection c = DriverManager.getConnection(~);
        PreparedStatement ps = c.prepareStatement(
            "select * from users where id = ?"
        );
        ps.setString(1, id);

        ResultSet rs = ps.executeQuery(); 
        rs.next();
        User user = new User(); 
        user.setId(rs.getString("id"));
        user.setName(rs.getString("name"));
        user.setPassword(rs.getString("password"));

        return user; 
    }
}
```

- JDBC를 이용하는 순서

DB 커넥션 → SQL Statement(or PreparedStatement) 생성 → State 실행
→ 조회의 경우, 실행 결과를 ResultSet으로 받아서 오브젝트에 옮김 
→ 작업 중에 생성된 Connection, Statement, ResultSe 같은 리소스는 닫기 → 예외 처리

### 1.1.3 main()을 이용한 DAO 테스트 코드

UserDao 클래스에 있는 add(), get() 메소드가 제대로 작동하는지 확인하기 위해

main() 메소드에서 함수를 호출하고 출력되는 값을 본다

## 1.2 DAO의 분리

### 1.2.1 관심사의 분리

관심이 같은 것끼리 하나의 객체 안으로, 관심이 다른 것은 서로 영향을 주지 않도록 분리한다

→ “분리와 확장”을 고려한 설계 필요!

### 1.2.2 커넥션 만들기의 추출

UserDao의 문제점

: add() 메서드에 있는 db 커넥션 코드와 동일한 코드가 get 메서드에도 중복되어 있음

```java
//  커넥션을 가져오는 중복된 코드를 분리
Connection c = getConnection();
```

### 1.2.3 DB 커넥션 만들기의 독립

```java
public abstract class UserDao {
}
public abstract Connection getConnection() {}
```

상속을 통해서 UserDao클래스를 확장하여 구한다 ex)NUserDao, DUserDao

문제점

- 이미 UserDao가 다른 목적을 위해 상속을 하고 있다면 다중 상속하여 오류가 난다
- 밀접한 관계

디자인 패턴

- **템플릿 메소드 패턴**: 상위 클래스에서 알고리즘의 뼈대를 정의하고, 일부 구체적인 단계는 하위 클래스에서 구현하게 하는 패턴
- **팩토리 메소드 패턴**: 객체 생성을 하위 클래스에 위임하여, 상위 클래스는 구체적인 클래스 이름에 의존하지 않고 객체를 생성할 수 있게 하는 패턴

## 1.3 DAO의 확장

### 1.3.1 클래스의 분리

```java
public abstract class UserDao{
    private SimpleConnectionMaker simpleConnectionMaker;

    public UserDao(){
        simpleConnectionMaker = new simpleConnectionMaker()
    }

    public void add(User user) throws ClassNotFoundException, SQLException{
        Connection c = simpleConnectionMaker.makeNewConnection();
    } 
    
    public void get(User user) throws ClassNotFoundException, SQLException{
        Connection c = simpleConnectionMaker.makeNewConnection();
    } 
}
```

DB 커넥션 관련된 코드를 별도의 클래스로 만들어서 new로 객체로 만든 후, 생성자에 주입한다

add(), get() 메소드에서 사용한다

→ 클래스로 분리했다

❗But, 문제점

: DB 커넥션 관련 코드를 변형하고 싶을 경우 각각의 add()와 get()에 있는 DB 커넥션 관련 코드를 모두 변형해야 한다 

### 1.3.2 인터페이스의 도입

추상적인 **“느슨한 연결고리”**를 만들어준다

- 추상화: 어떤 것들의 공통적인 성격을 뽑아내어 이를 따로 분리해내는 작업

→ 인터페이스 사용

인터페이스: 자신을 구현한 클래스에 대한 구체적인 정보는 모두 감춰버린다

```java
public interface ConnectionMaker {
	public Connection makeConnection() throws ClassNotFoundException, SQLException;
}
```

```java
public abstract class UserDao{
		// DB 커넥션 관련 인터페이스로 다형성 확장
    private ConnectionMaker connectionMaker;

    public UserDao(){
        connectionMaker = new DConnectionMaker();
    }

    public void add(User user) throws ClassNotFoundException, SQLException{
        Connection c = connectionMaker.makeConnection();
    }
}
```

### 1.3.3 관계설정 책임의 분리

```java
// 생성자 정의
public UserDao(ConnnectionMaker connectionMaker){
    this.connectionMaker = connectionMaker;
}
```

UserDao와 ConnectionMaker 구현 클래스의 관계를 결정해주는 기능 분리

→ 클래스가 아니라 오브젝트와 오브젝트 사이의 관계를 설정(기존 생성자의에게 연결 책임)

### 1.3.4 원칙과 패턴

- 개방 폐쇄 원칙(OCP, Open-Closed Principle)

클래스나 모듈은 확장에는 열려 있어야 하고 변경에는 닫혀 있어야 한다

- **높은 응집도**: 하나의 모듈, 클래스가 하나의 책임 또는 관심사에만 집중되어 있다는 뜻
- **낮은 결합도**: 관계를 유지하는 데 꼭 필요한 최소한의 방법만 제공하고, 나머지는 서로 독립적이어야 한다
- 전략 패턴(Strategy Pattern)

자신의 기능 맥락에서 필요에 따라 변경이 필요한 알고리즘을 인터페이스를 통해 통째로 외부로 분리시키고, 이를 구현한 구체적인 알고리즘 클래스를 필요에 따라 바꿔서 사용할 수 있게 하는 디자인 패턴

## 1.4 제어의 역전(IoC)

### 1.4.1 오프젝트 팩토리

- 팩토리

객체의 생성 방법을 결정하고 그렇게 만들어진 오브젝트를 돌려주는 것

목적: 오브젝트를 생성하는 쪽과 생성된 오브젝트를 사용하는 쪽의 역할과 책임을 깔끔하게 분리

UserDaoTest에서는 DaoFactory에 요청해서 미리 만들어진 UserDao 오브젝트를 가져와 사용하게 만든다

팩토리의 메소드는 UserDao 타입의 오브젝트를 어떻게 만들고, 어떻게 준비시킬지를 결정한다

```java
public class DaoFactory{
// 두 개 이상의 Dao 포함
	public UserDao userDao(){
		ConnectionMaker connectionMaker = new DConnectionMaker();
		UserDao userDao = new UserDao(connectionMaker);
		return userDao;
	}
}
```

- 설계도로서의 팩토리

애플리케이션의 컴포넌트 역할을 하는 오브젝트와 애플리케이션의 구조를 결정하는 오브젝트를 분리

### 1.4.2 오브젝트 팩토리의 활용

DaoFactory에 UserDao가 아니 다른 Dao 생성 기능을 넣을 경우?

어떤 ConnectionMaker 구현 클래스를 사용할지를 결정하는 기능이 중복돼서 나타난다

→ 분리하도록 한다

ConnectionMaker의 구현 클래스를 결정하고 오브젝트를 만드는 코드를 별도의 메소드로 뽑아낸다

### 1.4.3 제어권의 이전을 통한 제어관계 역전

- 제어의 역전

프로그램의 제어 흐름 구조가 뒤바뀌는 것, 오브젝트가 자신이 사용할 오브젝트를 스스로 선택하지 않는다(모든 제어 권한을 자신이 아닌 다른 대상에게 위임)

즉, 제어권을 상위 템플릿 메소드에 넘기고 자신은 필요할 때 호출되어 사용되도록 한다

ex) 프레임워크

- 라이브러리 VS 프레임워크
    - 라이브러리
    : 사용하는 애플리케이션 코드는 애플리케이션 흐름을 직접 제어
    - 프레임워크
        
        : 애플리케이션 코드가 프레임워크에 의해 사용된다
        
        프레임워크가 흐름을 주도하는 중에 개발자가 만든 애플리케이션 코드를 사용하도록 만드는 방식이다
        

## 1.5 스프링의 IoC

### 1.5.1 오브젝트 팩토리를 이용한 스프링 IoC

```java
// 빈 팩토리를 위한 오브젝트 설정을 담당
@Csonfiguration
public class DaoFactory {
	// 오브젝트를 만들어주는 메소드
	@Bean
	public UserDao userDao() {
		return new UserDao(connectionMaker());
	}
	@Bean
	public ConnectionMaker connectionMaker() {
		return new DConnectionMaker();
	}
}
```

### 1.5.2 애플리케이션 컨텍스트의 동작방식

### 1.5.3 스프링 IoC의 용어 정리

- 빈
- 빈 팩토리
- 애플리케이션 컨텍스트
- 설정정보/설정 메타정보
- 컨테이너 / IoC 컨테이너 = 애플리케이션 컨텍스트 / 빈팩토리
- 스프링 프레임워크

## 1.6 싱글톤 레지스트리와 오브젝트 스코프

### 1.6.1 싱글톤 레지스트리로서의 애플리케이션 컨텍스트

- 싱글톤 패턴

어떤 클래스를 애플리케이션 내에서 제한된 인스턴스 개수, 이름처럼 주로 하나만 존재하도록 강제하는 패턴

하나만 만들어지는 클래스의 오브젝트는 애플리케이션 내에서 “전역으로 접근”이 가능

### 1.6.2 싱글톤과 오브젝트의 상태

무상태(Stateless) 방식으로 만들어져야 한다

→ 상태 유지(Stateful, 필드의 값을 변경하고 유지하는 방식 X)

### 1.6.3 스프링 빈의 스코프

- 스프링 빈의 기본 스코프는 싱글톤 컨테이너 내에 한 개의 오브젝트만 만들어져서, 강제로 제거하지 않는 한 스프링 컨테이너가 존재하는 동안 계속 유지된다.
- 프로토타입 스코프: 컨테이너에 빈을 요청할 때마다 매번 새로운 오브젝트를 생성함.
- 요청 스코프 : HTTP 요청이 있을 때마다 생성됨.
- 세션 스코프 : 웹의 세션과 스코프가 유사.

## 1.7 의존관계 주입

### 1.7.1 제어의 역전(IoC)과 의존관계 주입

스프링이 제공하는 IoC 방식의 핵심 = **"의존 관계 주입 (Dependency Injection)"**

### 1.7.2 런타임 의존관계 설정

- 의존관계 주입?
    - 구체적인 오브젝트와 클라이언트를 런타임 시에 연결해주는 작업
- 의존관계 주입의 핵심
    - 설계 시점에는 알지 못했던 두 오브젝트의 관계를 맺도록 도와주는 제 3의 존재가 있다는 것
    - 스프링의 애플리케이션 컨텍스트, 빈 팩토리, IoC 컨테이너 등이 모두 외부에서 오브젝트 사이의 런타임 관계를 맺어주는 책임을 지닌 제 3의 존재라고 볼 수 있다!

### 1.7.3 의존관계 검색과 주입

의존관계 검색은 런타임 시 의존관계를 맺을 오브젝트를 결정하는 것과 오브젝트의 생성 작업은 외부 컨테이너에게 IoC로 맡기지만, 이를 가져올 때는 메소드나 생성자를 통한 주입 대신 스스로 컨테이너에게 요청하는 방법을 사용한다.

```java
public UserDao() {

    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(DaoFactory.class);

    this.connectionMaker = context.getBean("connectionMaker",  ConnectionMaker.class);

}
```

### 1.7.4 의존관계 주입의 응용

- 기능 구현의 교환
    - 개발환경과 운영환경에서 DI 설정정보에 해당하는 DaoFactory만 다르게 만들어두면 나머지 코드에는 전혀 손대지 않고 개발 시와 운영 시에 각각 다른 런타임 오브젝트에 의존관계를 갖게 해준다.
- 부가 기능 추가
    - DI의 장점은 관심사의 분리(SOC)를 통해 얻어지는 높은 응집도에서 나온다.

### 1.7.5 메소드를 이용한 의존관계 주입

- **수정자 메소드를 이용한 주입:**
    - 수정자(Setter) 메소드는 외부에서 오브젝트 내부의 애트리뷰트 값을 변경하려는 용도로 주로 사용한다.
- **일반 메소드를 이용한 주입:**
    - 수정자 메소드처럼 set으로 시작해야 하고 한 번에 한 개의 파라미터만 가질 수 있다는 제약이 싫다면 여러 개의 파라미터를 갖는 일반 메소드를 DI용으로 사용할 수 있다
