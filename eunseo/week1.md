# [정은서] 2025 GDG Spring Advanced Study - 1주차
# 토비의 스프링 3.1(vol1) - 1장 오브젝트와 의존관계 

## 1.1 초난감 DAO

### DAO
- **DAO(Data Access Object)**  
  DB를 사용해 데이터를 조회하거나 조작하는 기능만 전담하는 오브젝트.  

---

### 1.1.1 User
- **사용자 정보를 담는 객체**
- **JavaBean 규약**을 따름 → DTO 역할
  - 디폴트 생성자 필요 (파라미터 없는 생성자)
  - 프로퍼티(속성)는 getter/setter 메서드를 통해 접근

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

- **UserDao 클래스**
사용자 정보를 DB에 넣고 관리하는 클래스  
- 사용자 생성(`add`), 정보 읽어오기(`get`) 두 개의 메서드 작성

```java
public class UserDao {
    public void add(User user) throws ClassNotFoundException, SQLException {
        Class.forName("com.mysql.Driver");
        Connection c = DriverManager.getConnection(~); // DB 연결을 위한 Connection 객체
        PreparedStatement ps = c.prepareStatement( // SQL 실행을 위한 객체
            "insert into users(id, name, password) values(?,?,?)"
        );
        ps.setString(1, user.getId()); // SQL의 첫 번째 ?에 user.getId() 값을 넣음
        ps.setString(2, user.getName());
        ps.setString(3, user.getPassword());

        ps.executeUpdate(); // insert 문 실행
    }

    public User get(String id) throws ClassNotFoundException, SQLException {
        Class.forName("com.mysql.Driver");
        Connection c = DriverManager.getConnection(~);
        PreparedStatement ps = c.prepareStatement(
            "select * from users where id = ?"
        );
        ps.setString(1, id);

        ResultSet rs = ps.executeQuery(); // SELECT 실행, 결과 집합(ResultSet) 반환
        rs.next();
        User user = new User(); // DB에서 꺼낸 값을 담을 User 객체 생성
        user.setId(rs.getString("id"));
        user.setName(rs.getString("name"));
        user.setPassword(rs.getString("password"));

        return user; // 객체 반환
    }
}
```

### 1.1.3 main()을 이용한 DAO 테스트 코드

```java
public static void(String[] args) throws ClassNotFoundException, SQLException {
userDao dao = new Userdao();

User user = new User();
user.setId("eunseo");
user.setName("정은서");
user.setPassword("030903");

dao.add(user);
System.out.println(user.getId() + "등록 성공");

User user2 = dao.get(user.getId());
System.out.println(user2.getName());
System.out.println(user.getPassword());
}
```

## 1.2 DAO의 분리

### 1.2.1 관심사의 분리

- 분리와 확장을 고려한 설계 → 관심이 같은 것끼리는 하나의 객체 안으로 다른 것은 떨어지도록 분리

### 1.2.2 커넥션 만들기의 추출

- DB 와 연결을 위한 커넥션을 어떻게 가져올까라는 관심
- 사용자 등록을 위해 DB 에 보낼 SQL 문장을 담은 statement 를 만들고 실행하는 관심
- 문제
    - add() 메서드에 있는 db 커넥션 코드와 동일한 코드가 get 메서드에도 중복되어 있음
        - 커넥션을 가져오는 중복된 코드를 분리 → getConnection() 이라는 이름의 독립적인 메서드로 만들기
        - Connection c = getConnection();
        - 리팩토링(기존의 코드를 외부의 동작방식에는 변화 없이 내부 구조를 변경해서 재구성), 메서드 추출

### 1.2.3 DB 커넥션 만들기의 독립

- 상속을 통한 확장
    - userDao 에서 getconnection 을 추상 메서드로 만든다
    - userDao 를 구입한 포탈사는 userdao 를 상속해서 각각 서브클래스를 만든다. (메소드의 구현은 서브 클래스가 담당)

```java
public abstract class UserDao {
}
public abstract Connection getConnection() {}
```
```java
public class NUserDao extends UserDao {}
public class DUserDao extends UserDao {}
```

- 기존에는 같은 클래스에 다른 메소드로 분리됐던 DB 커넥션 연결이라는 관심→ 상속을 통한 서브클래스로 분리 

- **템플릿 메소드 패턴** : 슈퍼 클래스에 기본적인 로직의 흐름을 만들고 그 기능의 일부를 추상 메서드나 오버라이딩이 가능한 protected 메소드 등으로 만든 뒤 서브 클래스에서 이런 메서드를 필요에 맞게 구현해서 사용하도록 하는 방법
- **팩토리 메소드 패턴** : 서브 클래스에서 구체적인 오브젝트 생성 방법을 결정하게 하는 것
    - UserDao 의 getConnection() 은 connection 타입 오브젝트를 생성한다는 기능을 정의해놓은 추상메서드, 서브 클래스의 getConnection 은 오브젝트를 어떻게 생성할 것인지를 결정하는 방법

## 1.3 DAO 의 확장

- 추상 클래스를 만들고 이를 상속한 서브 클래스에서 변화를 주도록 한 것은 각각 필요한 시점에 독립적으로 변경할 수 있게 함 → 상속을 사용했다는 단점 

### 1.3.1 클래스의 분리

- 상속이 아닌 완전히 독립적인 클래스로 만들어보자
    - db 커넥션과 관련된 부분을 서브 클래스가 아니라 아예 별도의 클래스로 담는다
    - 이 클래스를 userdao 가 이용하면 됨

### 1.3.2 인터페이스의 도입

- 두 개의 클래스가 서로 긴밀하게 연결되어 있지 않도록 중간에 추상적인 느슨한 연결고리를 만들어주는 것
- 추상화 : 어떤 것들의 공통적인 성격을 뽑아내어 이를 따로 관리하는 것
- 자바가 추상화를 위해 제공하는 가장 유용한 도구 → 인터페이스 (어떤 일을 하겠다는 기능만 정의)
- userdao → connectionMaker → Nconnectionmaker, Dconnectionmaker
- ConnectionMaker 인터페이스 정의
    - DB 커넥션을 가져오는 메서드를 makeconnection()
    - userdao 입장에서는 connectionmaker 인터페이스 타입의 오브젝트라면 어떤 클래스로 만들어졌던지 makeconnection 메소드를 호출하기만 하면 connection 타입의 오브젝트를 만들어서 돌려줄 것이라고 기대할 수 있음

```java
public interface ConnectionMaker {
	public Connection makeConnection() throws ClassNotFoundException, SQLException;
}
```
```java
public class DconnectionMaker implements ConnectionMaker {
	public Connection makeConnection() throws ClassNotFounException, SQLException {
	}
}
```
```java
public class UserDao {
	private ConnectionMaker connectionMaker; // 인터페이스를 통해 오브젝트에 접근하므로 구체적인 클래스 정보를 알 필요가 없다.
	
	public UserDao() {
	connectionMaker = new DConnectionMaker();
	}
	
	public void add(User user) throws ClassNotFoundException, SQLException {
		Connection c = connectionMaker.makeConnection(); // 인터페이스에 정의된 메서드를 사용하므로 클래스가 바뀐다고 해도 메소드 이름이 변경될 걱정은 없다.
	}
	
	public User get(String id) throws ClassNotFoundException, SQLException {
		Connection c = connectionMaker.makeConnection();
	}
}
```
### 1.3.3 관계 설정 책임의 분리

- 클라이언트 : 사용하는 오브젝트
- 서비스 : 사용되는 오브젝트
- UserDao 의 모든 코드는 ConnectionMaker 인터페이스 외에는 어떤 클래스와도 관계를 가지면 안됨
- 코드에서는 특정 클래스를 전혀 알지 못하더라도 해당 클래스가 구현한 인터페이스를 사용했다면 그 클래스의 오브젝트를 인터페이스 타입으로 받아서 사용 가능 → “다형성”
- 클라이언트와 같은 제 3의 오브젝트가 UserDao 오브젝트가 사용할 ConnectionMaker 오브젝트를 전달해주도록

```java
public UserDao(ConnectionMaker connectionMaker) {
this.connectionMaker = connectionMaker;
} //로 수정 가능 
// DConnectionMaker 는 UserDao 와 특정 connectionMaker 구현 클래스의 오브젝트 간 관계를 맺는 책임을 담당
//이제는 UserDao 의 클라이언트에게 넘김
```
- UserDaoTest는 UserDao 가 실제로 사용할 DConnectionMaker 오브젝트를 생성

```java
public class UserDaoTest {
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		// UserDao 가 사용할 ConnectionMaker 구현 클래스를 결정하고 오브젝트 생성
		ConnectionMaker connectionMaker = new ConnectionMaker();
		// UserDao 와 ConnectionMaker 간의 오브젝트 의존관계 생성
		UserDao dao = new UserDao(connectionMaker);
		}
}
```
- 서로 영향을 주지 않으면서도 필요에 따라 자유롭게 확장할 수 있음 

### 1.3.4 원칙과 패턴

- **개방 폐쇄 원칙**
    - 클래스나 모듈은 확장에는 열려 있어야 하고 변경에는 닫혀 있어야 한다.
- 높은 응집도와 낮은 결합도
    - **높은 응집도**
        - 하나의 모듈, 클래스가 하나의 책임 또는 관심사에만 집중되어 있다는 뜻
        - 변경이 일어날 때 해당 모듈에서 변하는 부분이 큼
    - **낮은 결합도**
        - 책임과 관심사가 다른 오브젝트 또는 모듈과는 낮은 결합도, 즉 느슨하게 연결된 형태를 유지
        - 관계를 유지하는 데 꼭 필요한 최소한의 방법만 간접적인 형태로 제공하고, 나머지는 서로 독립적이고 알 필요도 없게 만들어주는 것
- 전략 패턴
    - 재신의 기능 맥락에서 필요에 따라 변경이 필요한 알고리즘을 인터페이스를 통해 외부로 분리시키고 이를 구현한 구체적인 알고리즘 클래스를 필요에 따라 바꿔서 사용할 수 있게 하는 디자인 패턴
        - 알고리즘 : 독립적인 체임으로 분리가 가능한 기능

## 1.4 제어의 역전 (IoC)

### 1.4.1 오브젝트 팩토리

- UserDaoTest 는 UserDao 가 ConnectionMaker 인터페이스를 구현한 특정 클래스로부터 완벽하게 독립할 수 있도록 클라이언트가 되어 그 수고를 담당
- 또 분리
    1. UserDao 와 ConnectionMaker 의 오브젝트를 만드는 것
    2. 두 개의 오브젝트가 연결돼서 사용될 수 있도록 관계를 맺어주는 것
- 팩토리
    - 분리시킬 기능을 담당할 클래스, 객체의 생성 방법을 결정하고 그렇게 만들어진 오브젝트를 돌려주는 것
    - UserDaoTest 에 담겨있는 UserDao, ConnectionMaker 관련 생성 작업은 DaoFatory 에 옮기고 UserDaoTest 에서는 DaoFactory 에 요청에서 만들어진 UserDao 오브젝트를 가져와 사용
- 설계도로서의 팩토리
    - 애플리케이션의 컴포넌트 역할을 하는 오브젝트와 애플리케이션의 구조를 결정하는 오브젝트를 분리했다는 데 가장 의미가 있음

### 1.4.2 오브젝트 팩토리의 활용

- DaoFactory 안에 UserDao 가 아닌 다른 DAO도 만들면 ConnectionMaker 구현 클래스의 오브젝트를 생성하는 코드가 메서드마다 반복 → 이또한 분리 

### 1.4.3 제어권의 이전을 통한 제어관계 역전

- 제어의 역전 : 프로그램의 제어 흐름 구조가 뒤바뀌는 것
- 오브젝트가 자신이 사용한 오브젝트를 스스로 생성 및 선택하지 않음
- 제어권을 상위 템플릿 메소드에 넘기고 자신을 필요할 때 호출되어 사용되도록 함
    - 템플릿 메소드는 제어의 역전이라는 개념을 활용해 문제를 해결하는 디자인 패턴
- 원래는  UserDao 에게 ConnectionMaker 의 구현 클래스를 결정하고 오브젝트를 만드는 제어권이 있었음
- 지금은 DaoFactory에 있음 → 제어의 역전이 일어난 상황
- IoC 적용하면
    - 설계가 깔끔해지고 유연성이 증가하며 확장성이 좋아짐
- 스프링은 IoC를 모든 기능의 기초가 되는 기반 기술로 삼고 있음

## 1.5 스프링의 IoC

- 스프링의 핵심 : 빈 백토리(애플리케이션 컨텍스트)

### 1.5.1 오브젝트 팩토리를 이용한 스프링 IoC

- bean : 스프링이 제어권을 가지고 직접 만들고 관계를 부여하는 오브젝트
- 스프링 빈 : 스프링 컨테이너가 생성과 관계설정, 사용 등을 제어해주는 제어의 역전이 적용된 오브젝트
- 빈 팩토리 : 스프링에서 빈의 생성과 관계설정 같은 제어를 담당하는 IoC 오브젝트
- 애플리케이션 컨텍스트 : IoC 방식을 따라 만들어진 일종의 빈 팩토리
    - 애플리케이션 전반에 걸쳐 모든 구성 요소의 제어 작업을 담당
- DaoFactory 를 사용하는 애플리케이션 컨텍스트
    - DaoFactory 를 스프링의 빈 팩토리가 사용할 수 있는 본격적인 설정 정보로 만들어보자
    - @Configuration → 빈 팩토리를 위한 오브젝트 설정을 담당하는 클래스라고 인식하도록
    - @Bean → 오브젝트를 만들어주는 메소드
- 스프링 빈 팩토리가 사용할 설정 정보를 담은 DaoFactory 클래스

```java
@configuration
public class DaoFactory {
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
### 1.5.2 애플리케이션 컨텍스트의 동작 방식

- 클라이언트는 구체적인 팩토리 클래스를 알 필요가 없다
- 애플리케이션 컨텍스트는 종합 IoC 서비스를 제공해준다
- 애플리케이션 컨텍스트는 빈을 검색하는 다양한 방법을 제공한다

### 1.5.3 스프링 IoC 의 용어 정리

- bean : 스프링이 IoC 방식으로 관리하는 오브젝트라는 뜻
- bean factory : 스프링의 IoC를 담당하는 핵심 컨테이너
    - 빈을 등록하고 생성하고 조회하고 돌려주고 그 외에 부가적인 빈을 관리
    - 보통은 bean factory 를 확장한 applicationcontext 를 씀
- applicationcontext : 빈 팩토리를 확장한 IoC 컨테이너
    - 스프링이제공하는 애플리케이션 지원 기능을 모두 포함
    - BeanFactory 를 상속
- configuration metadata
    - 애플리케이션 컨텍스트가 IoC를 적용하기 위해 사용하는 메타 정보
- IoC container : IoC 방식으로 빈을 관리
- spring framework : IoC 컨테이너, 애플리케이션 컨텍스트를 포함해서 스프링이 제공하는 모든 기능
    - = spring

## 1.6 싱글톤 레지스트리와 오브젝트 스코프

- 오브젝트의 동일성과 동등성
    - 자바에서 두 개의 오브젝트가 완전히 같은 동일한(identical) 오브젝트라고 말하는 것과, 동일한 정보를 담고 있는(equality) 오브젝트라고 말하는 것은 다름
    - 동일성은 == 연산자
    - 동등성은 equals() 메소드로 비교
- DaoFactory 와 @Configuration (applicationcontext)의 차이점
    - DaoFactory 의 userDao() 메소드를 두 번 호출했을 때 리턴되는 오브젝트를 비교해보자
        - 두 개는 각기 다른 값을 가진 동일하지 않은 오브젝트 → 오브젝트가 2개 생김
    - applicationcontext 에 DaoFactory 를 설정정보로 등록하고 getBean() 메소드를 이용해 userDao()를 가져와보자
        - 두 오브젝트의 출력값이 같다 → 매번 new 에 의해 새로운 userDao 가 만들어지지 않음

### 1.6.1 싱글톤 레지스트리로서의 애플리케이션 컨텍스트

- applicationcontext는 IoC 컨테이너인 동시에 singleton registry 이기도 하다
- 싱글톤 패턴
    - 어떤 클래스를 애플리케이션 내에서 제한된 인스턴스 개수, 이름처럼 주로 하나만 존재하도록 강제하는 패턴
    - 이렇게 하나만 만들어지는 클래스의 오브젝트는 애플리케이션 내에서 전역으로 접근이 가능
    - 단일 오브젝트에만 존재해야 하고 이를 애플리케이션의 여러 곳에서 공유하는 경우에 주로 사용
- 서버 애플리케이션과 싱글톤
    - 스프링은 별다른 설정을 하지 않으면 내부에서 생성하는 빈 오브젝트를 모두 싱글톤으로 만든다.
        - 스프링이 주로 적용되는 대상이 자바 엔터프라이즈 기술을 사용하는 서버 환경이기 때문
- 싱글톤 패턴의 한계
    - private 생성자를 갖고 있기 때문에 상속할 수없다.
    - 싱글톤은 테스트하기 힘들다.
    - 서버 환경에서는 싱글톤이 하나만 만들어지는 것을 보장하지 못한다.
    - 싱글톤의 사용은 전역 상태를 만들 수 있기 때문에 바람직하지 못하다.
        - 싱글톤의 static 메소드를 사용해 언제든지 접근 가능 → 자연스레 전역상태로 사용되기 쉬움
- 싱글톤 레지스토리
    - 직접 싱글톤 형태의 오브젝트를 만들고 관리하는 기능을 제공
    - 스태틱 메소드와 private 생성자를 사용하는 비정상클래스가 아닌 평범한 자바 클래스를 싱글톤으로 활용 가능

### 1.6.2 싱글톤과 오브젝트의 상태

- 멀티쓰레드 환경이라면 여러 쓰레드가 동시에 접근해서 사용하기 때문에 상태 관리에 주의를 요함

### 1.6.3 스프링 빈의 스코프

- 빈의 스코프 : 빈이 생성되고 존재하고 적용되는 범위
    - 기본 스코프는 싱글톤
    - 컨테이너 내에 한 개의 오브젝트만 만들어져 유지
    - 프로토타입 스코프 : 컨테이너에 빈을 요청할 때마다 매번 새로운 오브젝트 생성
    - 요청 스코프, 세션 스코프

## 1.7 의존관계 주입 (DI)

### 1.7.1 제어의 역전(IoC)과 의존관계 주입

- 스프링 IoC 기능의 대표적인 동작원리는 주로 의존관계 주입이라고 불림
- IoC container → DI container
- Dependency Injection
    - DI는 오브젝트 레퍼런스를 외부로부터 제공받고 이를 통해 여타 오브젝트와 다이내믹하게 의존 관계가 만들어짐

### 1.7.2 런타임 의존관계 설정

- 의존 관계
    - 두 개의 모듈이나 클래스가 의존 관계에 있다고 말할 때는 항상 **방향성**을 부여해줘야 함
    - A가 B에 의존하는 관계에 있다 (A→B), 반대는 성립 x
    - 의존한다. → 의존 대상(B)이 변하면 A에 영향을 미침
    - A에서 B에 정의된 메소드를 사용할 때 B에 새로운 메소드가 추가되거나 하면 A도 그에 따라 수정되어야 함

- 의존 관계 주입
    - 의존 오브젝트 : 런타임 시에 의존 관계를 맺는 대상
    - 구체적인 의존 오브젝트와 그것을 사용할 주체(클라이언트 오브젝트)를 런타임 시에 연결해주는 작업
    - 클래스 모델이나 코드에는 런타임 시점의 의존 관계가 드러나지 않는다 그러기 위해서는 인터페이스에만 의존해야함
    - 런타임 시점의 의존 관계는 컨테이너나 팩토리 같은 제 3의 존재가 결정
    - 의존 관계는 사용할 오브젝트에 대한 레퍼런스를 외부에서 제공해줌으로써 만들어짐
    - DI 컨테이너에 의해 런타임 시에 의존 오브젝트를 사용할 수 있도록 그 레퍼런스를 전달받는 과정이 마치 메소드를 통해 DI 컨테이너가 userDao 에게 주입해주는 것과 같음 → 의존 관계 주입

### 1.7.3 의존관계 검색과 주입

- 구체적인 클래스에 의존하지 않고 런타임 시에 의존관계를 결정한다는 점에서 의존관계 주입과 비슷하지만
- 의존관계를 맺는 방법이 외부로부터의 주입이 아니라 스스로 검색을 이용하기 때문에 의존관계 검색이라고도 불림
- 의존관계 검색
    - 런타임 시 의존관계를 맺을 오브젝트를 결정하는 것과 오브젝트의 생성 작업은 외부 컨테이너에게 맡기지만 이를 가져올 때는 메소드나 생성자를 통한 주입 대신 스스로 컨테이너에게 요청하는 방법을 사용
    - 외부로부터의 주입이 아니라 스스로 IoC 컨테이너인 DaoFactory 에게 요청하는 것
    - getBean() 이라는 메소드가 검색에 사용
- 의존관계 주입이 검색보다 바람직함
- 의존관계 주입 vs 의존관계 검색
    - 의존관계 검색에서는 검색하는 오브젝트는 자신이 스프링의 빈일 필요가 없음
    - 주입에서는 반드시 컨테이너가 만드는 빈 오브젝트여야 함
        - DI 를 원하는 오브젝트는 먼저 자기 자신이 컨테이너가 관리하는 빈이 돼야 한다

### 1.7.4 의존관계 주입의 응용

- DI 기술의 장점
    - 런타임 시에 사용 의존관계를 맺을 오브젝트를 주입해줌
    - 런타임 클래스에 대한 의존관계가 나타나지 않음
    - 인터페이스를 통해 결합도가 낮은 코드를 만듦
    - 의존관계에 있는 대상이 바뀌거나 변경되더라도 자신은 영향을 받지 않음
    - 변경을 통한 다양한 확장 방법이 자유롭다
- 기능 구현의 교환
    - 개발환경과 운영환경에서 DI 의 설정정보에 해당하는 DaoFactory만 다르게 만들어두면
    - 나머지 코드에는 전혀 손대지 않고 개발 시와 운영 시에 각각 다른 런타임 오브젝트를 갖게 해줘서 문제 해결

### 1.7.5 메소드를 이용한 의존관계 주입

- 의존관계 주입 시 반드시 생성자를 사용해야 하는 것은 아님
- 생성자가 아닌 일반 메소드를 사용할 수도 있음
- 생성자가 아닌 일반 메소드를 이용해 의존 오브젝트와의 관계를 주입해주는 방법
    - 수정자 메소드를 이용한 주입
        - 파라미터로 전달된 값을 보통 내부의 인스턴스 변수에 저장
        - 외부로부터 제공받은 오브젝트 레퍼런스를 저장해뒀다가 내부의 메소드에서 사용하게 하는 DI 방식에서 활용하기에 적당
        - 가장 많이 사용
        - DI를 받을 오브젝트의 타입 이름을 따름
            - ConnectionMaker 인터페이스 타입의 오브젝트를 DI 받는다면 setConnectionMaker() 라고 하는 것
    - 일반 메소드를 이용한 주입
        - 여러 개의 파라미터를 갖는 일반 메소드를 DI 용으로 사용 가능
        - 한 번에 여러 개의 파라미터를 받을 수 있음

## 1.8 XML 을 이용한 설정

- DI 를 위한 오브젝트 의존 관계 정보를 XML 을 이용해 만들어보자

### 1.8.1 XML 설정

- - @Configuration 을 <beans>, @Bean 을 <bean> 에 대응
- 하나의 @Bean 메소드를  통해 얻을 수 있는 빈의 DI 정보
    1. 빈의 이름 : @Bean 메소드 이름이 빈의 이름, getBean() 에서 사용
    2. 빈의 클래스 : 빈 오브젝트를 어떤 클래스를 이용해서 만들지를 정의
    3. 빈의 의존 오브젝트 : 빈의 생성자나 수정자 메소드를 통해 의존 오브젝트를 넣어준다 의존 오브젝트도 하나의 빈이므로 이름이 잇고 그 이름에 해당하는 메소드를 호출해서 의존 오브젝트를 가져온다.

```java
@Bean → <bean 

public ConnectionMaker

connectionMaker() { → id = “connectionMaker”

return new DConnectionMaker(); → class= “springbook..DConnectionMaker”/>

}
```
## 1.9 정리

- 먼저 책임이 다른 코드를 분리해서 두 개의 클래스로 만들었다(관심사의 분리, 리팩토링)
- 이렇게 해서 인터페이스를 정의한 쪽의 구현 방법이 달라져 클래스가 바귀어도 그 기능을 사용하는 클래스의 코드는 같이 수정할 필요가 없음(전략 패턴)
- 자신이 사용하는 외부 오브젝트의 기능은 자유롭게 확장하거나 변경할 수 있게 만들었다(개방 폐쇄 원칙)
- 한쪽 기능의 변화가 다른 쪽의 변경을 요구하지 않아도 되게 했고(낮은 결합도)
- 자신의 책임과 관심사에만 순수하게 집중하는 코드를 만들었다(높은 응집도)
- 오브젝트 팩토리의 기능을 일반화한 IoC 컨테이너로 넘겨서 오브젝트가 자신이 사용할 대상의 생성이나 선택에 관한 책임으로부터 자유롭게 만들어줌 (제어의 역전, IoC)
- 전통적인 싱글톤 패턴 구현 방식의 단점을 살펴보고 서버에서 사용되는 서비스 프로젝트로서의 장점을 살릴 수 있는 싱글톤을 사용하면서도 싱글톤 패턴의 단점을 극복할 수 있도록 설계된 컨테이너를 활용하는 방법에 대해 알아봤다(싱글톤 레지스트리)
- 실제 시점과 코드에는 클래스와 인터페이스 사이의 느슨한 의존관계만 만들어놓고 런타임 시에 실제 사용할 구체적인 의존 오브젝트를 제3자의 도움으로 주입받아서 다이나믹한 의존관계를 만들어주는 의존관계 주입(DI)