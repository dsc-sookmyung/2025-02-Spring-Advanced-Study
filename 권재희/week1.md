# 1. 오브젝트와 의존관계

- 스프링 ← 오브젝트에 대한 이해
- 오브젝트의 설계와 구현, 동작원리

## 1.1 초난감 DAO

- DAO(Data Access Object) : DB를 사용해 데이터를 조회하거나 조작하는 기능을 전담하도록 만든 오브젝트

### 1.1.1 User

- 사용자 정보 저장 시 자바빈 규약을 따르는 오브젝트 이용
    - 자바빈 : 디폴트 생성자와 프로퍼티 관례를 따라 만들어진 오브젝트, =빈
- User 클래스 : id, name, password를 가짐

### 1.1.2 UserDao

- UserDAO 클래스
    - 새로운 사용자 생성(add), 아이디를 통해 사용자 가져오는(get) 메소드
- JDBC를 이용하는 순서
    - DB 커넥션 → SQL Statement(or PreparedStatement) 생성 → Statement 실행 → 조회의 경우, 실행 결과를 ResultSet으로 받아서 오브젝트에 옮김 → 작업 중에 생성된 Connection, Statement, ResultSe 같은 리소스는 닫기 → 예외 처리
        - 예외의 경우 밖으로 던지는 것이 편리

```java
// Exception thorw
public User get(String id) throws ClassNotFoundException, SQLException{
  // DB connection
  Class.forName("com.mysql.jdbc.Driver");
  Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook","spring","book");

  // Statement 생성
  PreparedStatement ps = c.prepareStatement("select * from users where id=?");
  ps.setString(1, id);

  // 실행 결과 ReusultSet으로 받기
  ResultSet rs = ps.executeQuery();
  rs.next();

  // 오브젝트에 옮기기
  User user = new User();
  user.setId(rs.getString("id"));
  user.setName(rs.getString("name"));
  user.setPassword(rs.getString("password"));

  // 닫기
  rs.close();
  ps.close();
  c.close();

  return user;
}

```

- add의 경우 결과값x, ps.executeUpdate()를 통해 업데이트

### 1.1.3 main()을 이용한 DAO 테스트 코드

- 오브젝트 스스로 검증하는 방법
- add() 후 get() 을 통해 검증
- UserDao 클래스 코드의 문제점

## 1.2 DAO의 분리

### 1.2.1 관심사의 분리

- 객체지향 → 추상세계를 효과적으로 구성, 이를 자유롭고 편리하게 변경, 발전, 확장 가능
- **분리와 확장**을 고려한 설계 필요
- 관심사의 분리 : 관심이 같은 것끼리 하나의 객체 안으로, 관심이 다른 것은 서로 영향x도록 분리

### 1.2.2 커넥션 만들기의 추출

**UserDao의 관심사항**

- UserDao의 관심사항
    - DB와 연결을 위한 커넥션을 어떻게 가져올까 : 어떤 DB 사용, 어떤 드라이버 사용, 어떤 로그인 정보 사용, 커넥션 생성 방법
    - 사용자 등록을 위한 SQL 문장을 담을 Statement를 만들고 실행
    - 작업이 끝난 리소스 닫기
- add() 메소드와 get() 메소드에 DB 커넥션을 위한 코드 중복

**중복 코드의 메소드 추출**

- DB 연결 코드 → getConnection() 으로 분리

```java
public void add(User user) throws ClassNotFounException, SQLException{
	Connection c = getConnection();
	...
}

private Connection getConnection() throws ClassNotFoundException, SQLException{
	Class.forName("~~~");
	Connection c = DriverManager.getConnection("~~~","~","~");
	return c
}
```

- 만약 DB 연결과 관련된 부분에 변경 → getConnection만 수정하면 됨

**변경사항에 대한 검증 : 리팩토링과 테스트**

- 리팩토링과 메소드 추출을 통해 기능 변경은 없지만 변화에 유연한 코드

### 1.2.3 DB 커넥션 만들기의 독립

- DB 커넥션을 변경하고 싶지만 소스코드를 공개하고 싶지는 않음

**상속을 통한 확장**

- UserDao 코드 한 단계 더 분리
- UserDao에서 getConnection()을 추상 메소드로

```java
public abstract class UserDao{
    public void add(User user) throws ClassNotFoundException, SQLException{
        Connection c = getConnection();
        ...
    }

    public abstract Connection getConnection() throws ClassNotFoundException, SQLException;
}

public class NUserDao extends UserDao{
    public Connection getConnection() throws ClassNotFoundException, SQLException{
        // connection 생성 코드
    }
}
```

- 템플릿 메소드 패턴 : 기능의 일부를 추상 메소드나  오버라이딩이 가능한 protected 메소드 등으로 만든 뒤 서브클래스에서 메소드에 맞게 구현해서 사용하는 방법, 스프링에서 애용되는 디자인 패턴
- 팩토리 메소드 패턴 : 서브클래스에서 구체적인 오브젝트 생성 방법을 결정하게 하는 디자인 패턴
    - Connection 오브젝트의 종류가 달라질 수 있게 하는 것을 목적, 같은 종류를 리턴할 수도 있지만 생성 방식이 다르다면 팩토리 메소드 패턴

**주요 디자인 패턴**

- 디자인 패턴 : 재사용 가능한 솔루션, 주로 객체지향 설계에 관한 것, 클래스 상속 or 오브젝트 합성 주로, 상황+문제+솔루션 구조+요소의 역할+핵심의도
- 템플릿 메소드 패턴 : 상속을 통해 기능 확장
    - 변하지 않는 기능 → 슈퍼클래스 / 자주 변경 → 서브클래스
    - 서브클래스에서 선택적으로(반드시x) 오버라이드 할 수 있도록 만들어둔 메소드 = 훅 메소드
    - 추상 메소드 구현 or 훅 메소드 오버라이드 → 기능 확장
- 팩토리 메소드 패턴 : 상속을 통해 기능 확장
    - 서브클래스에서 오브젝트 생성 방법과 클래스 결정 가능하도록 미리 정의해둔 메소드 = 팩토리 메소드

**문제점**

- 상속을 통한 확장 → 다른 상속 적용 불가, 관계 밀접

## 1.3 DAO의 확장

### 1.3.1 클래스의 분리

- 독립적인 클래스 → 만든 클래스를 UserDao가 이용

```java
public abstract class UserDao{
    private SimpleConnectionMaker simpleConnectionMaker;

    public UserDao(){
        // 한 번 만들어 인스턴스 변수에 저장, 메소드에서 사용
        simpleConnectionMaker = new simpleConnectionMaker()
    }

    public void add(User user) throws ClassNotFoundException, SQLException{
        Connection c = simpleConnectionMaker.makeNewConnection();
        // ~~~
    } 
}
```

```java
public class SimpleConnectionMaker{
    public Conection makeNewConnection() throws ClassNotFoundException, SQLException{
        Class.forName("~~");
        Connection c = DriverManager.getConnection("~~~","~","~");
        return c;
    }
}
```

- DB 커넥션만 변경 불가능해짐, makeNewConnection() 메소드 이름이 달라질 경우 변경 어려움, DB 커넥션을 제공하는 클래스가 어떤 것인지를 UserDao가 구체적으로 알고있어야 함 → 종속

### 1.3.2 인터페이스 도입

- 추상화로 해결
    - 추상화 : 공통적인 성격을 뽑아내 따로 분리 → 인터페이스

```java
public interface ConnectionMaker{
    public Connection makeConnection() throws ClassNotFoundException, SQLException;
}
```

```java
public class DConnectionMaker implements ConnectionMaker{
    public Connection makeConnection() throws ClassNotFoundException, SQLException{
        // 생성 코드 
    }
}
```

```java
public abstract class UserDao{
    private ConnectionMaker connectionMaker;

    public UserDao(){
        connectionMaker = new DConnectionMaker();
    }

    public void add(User user) throws ClassNotFoundException, SQLException{
        Connection c = connectionMaker.makeConnection();
        // ~~~
    } 
}
```

- UserDao에서 ConnectionMaker를 생성 시 DConnectionMaker 생성자 호출 필요 → 종속적

### 1.3.3 관계설정 책임의 분리

- UserDao 안에 분리되지 않은 관심사항 존재 : UserDao가 어떤 ConnectionMaker 구현 클래스의 오브젝트를 사용할지 결정
- 클라이언트 오브젝트에 관심사항 분리 → 파라미터를 통해서 외부에서 전달 가능
- 기존 생성자의에게 연결 책임 → 클라이언트에게 책임

```java
public UserDao(ConnnectionMaker connectionMaker){
    this.connectionMaker = connectionMaker;
}
```

```java
ConnectionMaker connectionMaker = new DConnectionMaker();
UserDao dao = new UserDao(connectionMaker);
```

### 1.3.4 원칙과 패턴

- 개방 폐쇄 원칙(OCP) : 확장에서는 열려 있어야 하고 변경에는 닫혀 있어야 함
- 높은 응집도와 낮은 결합도
    - 높은 응집도 : 변화가 일어날 때 해당 모듈에서 변하는 부분이 크다
    - 낮은 결합도 : 다른 오브젝트 또는 모듈과는 느슨하게 연결 → 변화에 대응 속도 높아지고 구성 깔끔, 확장에 용이
        - 결합도 : 변경이 일어날 때 관계를 맺고 있는 다른 오브젝트에 변화를 요구하는 정도
- 전략 패턴 : 기능 맥락에서, 필요에 따라 변경이 필요한 알고리즘을 인터페이스를 통해 통째로 외부로 분리시키고, 이를 구현한 구체적인 알고리즘 클래스를 필요에 따라 바꿔서 사용

## 1.4 제어의 역전(IoC)

### 1.4.1 오브젝트 팩토리

- UserDaoTest : 기능 동작 테스트 + 구현 클래스 사용 결정 두 가지 책임

**팩토리**

- 팩토리 : 객체 생성 방법을 결정하고 만들어진 오브젝트를 돌려줌
    - ≠ 추상 팩토리 패턴, 팩토리 메소드 패턴
    - 오브젝트 생성, 생성된 오브젝트 사용 분리

```java
public class DaoFactory{
    public UserDao userDao(){
        ConnectionMaker connectionMaker = new DConnectionMaker();
        UserDao userDao = new UserDao(connectionMaker);

        return userDao;
    }
}
```

```java
UserDao dao = new DaoFactory().userDao();
```

**설계도로서의 팩토리**

- DaoFactory → 컴포넌트 구조와 관계 정의 설계도 역할

### 1.4.2 오브젝트 팩토리의 활용

- UserDao에 다른 생성 기능 추가 → ConnectionMaker 생성 코드 중복 → ConnectionMaker 메소드로 분리

### 1.4.3 제어권의 이전을 통한 제어관계 역전

- 제어의 역전(IoC) : 프로그램 제어 흐름 구조가 뒤바뀌는 것, 자신이 사용할 오브젝트를 스스로 선택하지 않고 제어 권한을 다른 대상에게 위임
- 프레임워크, 많은 디자인패턴에서 사용
- 컴포넌트의 생성, 관계설정, 사용, 생명주기 관리 등을 제어하는 존재 필요

## 1.5 스프링의 IoC

- 빈 팩토리 또는 애플리케이션 컨텍스트

### 1.5.1 오브젝트 팩토리를 통한 이용한 IoC

- 빈 : 스프링이 제어권을 가지고 직접 만들고 관계를 부여하는 오브젝트
- 빈 팩토리 : 빈의 생성과  관계설정 같은 제어 담당 IoC 오브젝트, =애플리케이션 컨텍스트

**DaoFactory를 사용하는 애플리케이션 컨텍스트**

```
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

- `@Configuration` : 빈 팩토리를 위한 클래스
- `@Bean`: 오브젝트를 만들어주는 메소드

```java
Application context = new AnnotationConfigApplicationContext(DaoFactory.class);
UserDao dao = context.getBean("userDao", UserDao.class);
```

- 애플리케이션 컨텍스트 : ApplicationContext 타입
    - `@Configuration` 자바 코드를 설정정보로 사용 → AnnotationConfigApplicationContext 이용
- getBean() : ApplicationContext가 관리하는 오브젝트를 요청하는 메소드, 메소드의 호출 결과를 가져온다고 생각
    - 제네릭 메소드 방식 사용 → 두 번째 파라미터에 리턴 타입을 주어 캐스팅 코드 생략

### 1.5.2 애플리케이션 컨텍스트의 동작 방식

- 애플리케이션 컨텍스트 → DaoFactory 클래스를 설정정보로 등록, `@Bean` 이 붙은 메소드의 이름 가져옴
- getBean() 호출 → 애플리케이션 컨텍스트의 목록에 요청 이름이 있는지 확인 → 있다면 빈 생성 메소드 호출 → 오브젝트 생성 후 리턴

**장점**

- 클라이언트는 구체적인 팩토리 클래스를 알 필요 없음
- 애플리케이션 컨텍스트는 종합 IoC 서비스 제공
- 빈을 검색하는 다양한 방법 제공

### 1.5.3 스프링 IoC의 용어 정리

- 빈
- 빈 팩토리
- 애플리케이션 컨텍스트
- 설정정보/설정 메타정보
- 컨테이너 / IoC 컨테이너 = 애플리케이션 컨텍스트 /  빈팩토리
- 스프링 프레임워크

## 1.6 싱글톤 레지스트리와 오브젝트 스코프

- (스프링x) DaoFactory에서 userDao를 호출 → 매번 새로운 오브젝트
- (스프링o) DaoFactory에서 userDao() 메소드 호출 → 같은 오브젝트

### 1.6.1 싱글톤 레지스트리로서의 애플리케이션 컨텍스트

- 싱글톤 레지스트리 : 싱글톤을 저장하고 관리
- 싱글톤 패턴 : 클래스 내에서 인스턴스가 제한된 개수, 주로 하나만 존재하도록 강제하는 패턴
    - 생성자 private으로 → 스태틱 필드 정의 → 최초로 호출되는 시점에 한번만 오브젝트가 만들어지게 → 한 번 만들어지고 난 후에는 이미 만들어진 오브젝트 넘겨줌
- 싱글톤 패턴 구현 방식의 문제점
    - private 생성자 → 상속 제한
    - 테스트 어려움
    - 서버 환경에서는 하나만 만들어지는 것 보장x
    - 전역 상태 만들 수 있음
    - 싱글톤 패턴 단점 → 스프링은 직접 싱글톤 형태의 오브젝트를 만들고 관리하는 기능 제공 : 싱글톤 레지스트리

### 1.6.2 싱글톤과 오브젝트의 상태

- 무상태(stateless) 방식으로 만들어져야 함 → 상태유지(stateful, 필드의 값을 변경하고 유지) 방식x
- 메소드 파라미터, 로컬 변수, 리턴 값 등을 통해 정보 다룸

### 1.6.3 스프링 빈의 스코프

- 스코프 : 빈이 생성, 존재, 적용되는 범위
- 기본 스코프는 싱글톤 : 컨테이너 내에 한 개의 오브젝트 생성, 스프링 컨테이너가 존재하는 동안 계속 유지
- 프로토타입 스코프(매번 새로운 오브젝트 생성) / 요청 스코프 / 세션 스코프 등

## 1.7 의존관계 주입(DI)

### 1.7.2 런타임 의존관계 설정

- 의존관계 : A가 B에 의존 = B가 변경되면 A에 영향

**UserDao의 의존관계**

- UserDao는 ConnectionMaker 인터페이스에 의존
- 런타임/오브젝트 의존관계 : 설계 시점에서가 아닌 런타임 시에 의존관계를 맺는 것
- 의존 오브젝트 : 런타임 시에 의존관계를 맺는 대상
- 의존관계 주입 : 의존 오브젝트와 주체를 연결

**UserDao의 의존관계 주입**

- DaoFactory를 통해 의존 관계 설정 및 주입 → DI 컨테이너
- 생성자 파라미터를 통해 DI 컨테이너가 UserDao에게 주입 = 의존관계 주입

### 1.7.3 의존관계 검색과 주입

- 의존관계 검색 : 자신이 필요로 하는 의존 오브젝트를 능동적으로 찾음, 생성 작업은 IoC로 맡기고 가져올 때는 메소드나 생성자를 통한 주입이 아닌 컨테이너에 요청

```java
AnnotationConfigAPplicationContext context = new AnnotationConfigAPplicationContext(DaoFactory.class);
this.connectionMaker = context.getBean("connectionMaker", ConnectionMaker.class)
```

- 대개 의존관계 주입 방식을 사용하는 편이 낫지만, getBean처럼 검색 방식을 통해 가져와야 할 경우가 있음
- 의존관계 검색 → 검색하는 오브젝트는 자신이 스프링의 빈일 필요 없음
- 의존관계 주입 → 컨테이너가 만드는 빈 프로젝트여야 함

### 1.7.4 의존관계 주입의 응용

**기능 구현의 교환**  

- 로컬 DB → 서버 배포 시 DaoFactory를 수정하기만 하면 됨

**부가기능 추가**

- DB 연결횟수 카운팅 → ConnectionMaker 인터페이스 구현하여 연결횟수 카운터가 포함된 클래스 의존하도록 DaoFactory 수정

### 1.7.5 메소드를 이용한 의존관계 주입

- 수정자(setter) 메소드를 이용한 주입
    - 파라미터로 전달된 값 내부에 저장, 입력 값에 대한 검증이나 다른 작업 수행 가능
- 일반 메소드를 이용한 주입
    - 한 번에 여러 개의 파라미터 가능
