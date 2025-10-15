# 3장. 템플릿

## 3.1 다시보는 초난감 DAO

DB 커넥션이라는 제한적인 리소스를 공유해 사용하는 서버에서 동작하는 JDBC 코드에는 반드시 **예외처리**를 해줘야 한다 !
→ 예외가 발생한 경우에도 사용한 리소스를 반드시 반환시켜야 하기 때문

Connetion, PreparedStatement는 보통 **pool방식**으로 운영하여, 제한된 수의 리소스를 미리 생성해두고 이를 할당 및 반환하여 사용한다.
JDBC 코드에서는 어떤 상황에서도 가져온 **리소스를 반환**하도록 try/ Catch/finally 구문 사용을 권장

```java
public void deleteAll() throws SQLException {
    Connection c = null;
    PreparedStatement ps = null;

    try {
        c = dataSource.getConnection();
        ps = c.preparedStatement("delete from users");
        ps.executeUpdate();
    } catch (SQLException e) {
        throw e;     // 예외 로깅 등의 부가작업
    } finally {
        if (ps != null) {
            try {
                ps.close();
            } catch (SQLException e) { // close()메소드에서도 SQLException 발생할 수 있다
            }
        }
        if (c != null) {
            try {
                c.close();
            } catch (SQLExeption e) {
            }
        }
    }
}
```

## 3.2 변하는 것과 변하지 않는 것

위의 코드는 try/catch/finally 블록이 2중으로 중첩까지 되어나오는데다, 모든 메소드마다 반복된다 🤧🤧
→ 🌟 분리와 재사용을 위한 디자인 패턴을 적용 하자 !

먼저, **메소드 추출**을 해보자
→ 분리시킨 메소드를 다른 곳에서 재사용 가능 해야 한다.

**템플릿 메소드 패턴**을 이용해 분리(추상메서드 상속해서 구현)해보자
→ DAO로직마다 상속을 통해 새로운 클래스를 만들어야 하며, 서브클래스들이 이미 클래스레벨에서 컴파일시점에 이미 그 관계가 결정된다 .

**🪻 전략패턴의 적용**

: 오브젝트를 아예 둘로 분리하고 클래스 레벨에서는 인터페이스를 통해서만 의존하도록 만드는 전략 패턴

![그림3-2.png](https://github.com/user-attachments/assets/18611caa-0cad-4f28-a74c-92ab41e5c5e5)

PreparedStatement를 만들어주는 외부 기능이 바로 전략 패턴에서 말하는 전략

deleteAll()의 context(변하지 않는 작업)

- DB 커넥션 가져오기
- PreparedStatement를 만들어줄 외부 기능 호출하기
- 전달받은 PreparedStatement 실행하기
- 예외가 발생하면 이를 다시 메소드 밖으로 던지기
- 모든 경우에 만들어진 PreparedStatement와 Connection을 적절히 닫아주기

```java
public interface StatementStrategy {
    PreparedStatement makePreparedStatement(Connection c)
        throws SQLException;
}
```

```java
public class DeleteAllStatement implements StatementStrategy {
    public PreapredStatement makePreparedStatement(Connection c)
    throws SQLException {
        PreparedStatement ps = c.prepareStatement("delete from users");

        return ps;
    }
}
```

```java
public void deleteAll() throws SQLException {
    ...
    try {
        c = dataSource.getConnection();

        StatementStrategy strategy = new DeleteAllStatement();
        ps = strategy.makePreparedStatement(c);

        ps.executeUpdate();
    } catch (SQLException e) {
      ...
    }
}
```

⁉️ 컨텍스트가 Statementstrategy 인터페이스뿐 아니라 특정 구현 클래스인 DeleteAllStatement를 직접 알고 있다

**🪻 DI 적용을 위한 클라이언트/컨텍스트 분리**

![그림3-3.png](https://github.com/user-attachments/assets/61650b3d-1db9-48cd-87b0-2dd65e3b8c45)

```java
public void jdbcContextWithStatementStrategy(StatementStrategy stmt)
    throws SQLException {

    Connection c = null;
    PreparedStatement ps = null;

    try {
        c = dataSource.getConnection();

        ps = stmt.makePreparedStatement(c);

        ps.executeUpdate();
    } catch (SQLException e) {
        throw e;
    } finally {
        if (ps != null) {
            try { ps.close(); } catch (SQLException e) {}
        }
        if (ps != null) {
            try { c.close(); } catch (SQLExeption e) {}
        }
    }
}
```

```java
public void deleteAll() throws SQLException {
    StatementStrategy st = new DeleteAllStatement();
    jdbcContextWithStatementStrategy(st);
}
```

## 3.3 JDBC 전략패턴의 최적화

**🪻 로컬 클래스**

StatementStrategy 전략 클래스를 매번 독립된 파일로 만들지 말고 UserDao 클래스 안에 내부 클래스로 정의
내부 클래스에서 외부의 변수를 사용할 때는 외부 변수는 반드시 final로 선언

장점

- 클래스 파일이 하나 줄어든다.
- 메소드 안에서 PreparedStatement 생성 로직을 함께 볼 수 있어 코드에 대한 이해도가 높아진다.
- 클래스가 선언된 곳의 정보에 접근할 수 있다.

```java
public void add(final User user) throws SQLException {
    class AddStatement implements StatementStrategy {
        public PreparedStatement makePreparedStatement(Connection c)
            throws SQLException {

            PreparedStatement ps = c.prepareStatement(
                "insert into users(id, name, password) values(?,?,?)");

            ps.setString(1, user.getId());
            ps.setString(2, user.getName());
            ps.setString(3, user.getPassword());
						// 로컬(내부) 클래스의 코드에서 외부의 메소드 로컬변수에 직접 접근할 수 있다
            return ps;
        }
    }

    StatementStrategy st = new AddStatement();
    jdbcContextWithStatementStrategy(st);
}
```

좀 더 간결하게 클래스 이름을 제거해보자

**🪻 익명 내부 클래스**

상속할 클래스나 구현할 인터페이스를 생성자 대신 사용하며, 클래스를 재사용할 필요가 없고 구현한 인터페이스 타입으로만 사용할 경우에 유용

```java
public void add(final User user) throws SQLException {

    jdbcContextWithStatementStrategy(
        new StatementStrategy() {
            public PreparedStatement makePreparedStatement(Connection c)
                throws SQLException {

                PreparedStatement ps = c.prepareStatement(
                    "insert into users(id, name, password) values(?,?,?)");

                ps.setString(1, user.getId());
                ps.setString(2, user.getName());
                ps.setString(3, user.getPassword());

                return ps;
            }
    });
}
```

## 3.4 컨텍스트와 DI

UserDao외에도 다른 DAO에서도 사용가능하게 jdbcContextWithStatementStrategy()를 UserDao 클래스 밖으로 독립시켜보자

```java
public class JdbcContext {
    private DataSource dataSource;

    public void setDataSource(DataSource datasource) {
        this.dataSource = dataSource;
    }

    public void workWithStatementStrategy(StatementStrategy stmt) throws SQLException {
        Connection c = null;
        PreparedStatement ps = null;

        try {
            c = this.dataSource.getConnection();

            ps = stmt.makePreparedStatement(c);
            ps.executeUpdate();
        } catch (SQLException e) {
            throw e;
        } finally {
            if (ps != null) { try { ps.close(); } catch (SQLException e){} }
            if (c != null) { try { c.close(); } catch (SQLException e){} }
        }
    }
}
```

```java
public class UserDao {
    ...
    private JdbcContext jdbcContext;

    public void setJdbcContext(JdbcContext jdbcContext) {
        this.jdbcContext = jdbcContext;
    }

    public void add(final User user) throws SQLException {
        this.jdbcContext.workWithStatementStrategy(
            new StatementStrategy() {...}
        );
    }

    public void deleteAll(final User user) throws SQLException {
        this.jdbcContext.workWithStatementStrategy(
            new StatementStrategy() {...}
        );
    }
}
```

**🪻 스프링 빈으로 DI**

UserDao는 인터페이스를 거치지 않고 코드에서 바로 JdbcContext라는 구체 클래스를 사용하고 있다.
스프링 DI의 기본 의도에 맞게 JdbcContext의 메소드를 인터페이스로 뽑아내어 정의해두고, 이를 UserDao에서 사용하게 해야 하지 않을까? → 꼭 그럴 필요는 없다 !

중요한 것은 인터페이스의 사용 여부이다. 왜 인터페이스를 사용하지 않았을까?
인터페이스를 사용하지 않았기 때문에 UserDao와 JdbcContext 클래스는 긴밀한 관계로 강하게 결합되어 있으며, 비록 클래스는 구분되어 있지만 이 둘은 강한 응집도를 갖고 있다.

- JDBC 방식이 아닌 JPA나 하이버네이트 같은 ORM을 사용해야 한다면 JdbcContext도 통째로 바뀌어야 한다.
- JdbcContext는 테스트에서도 다른 구현으로 대체해서 사용할 이유가 없다.

→ 이런 경우에는 굳이 인터페이스를 두지 말고 강력한 결합을 가진 관계를 허용하고, 스프링 빈으로 등록해서 UserDao에 DI 되도록 만들어도 좋다.

**🪻 코드를 이용하는 수동 DI**

JdbcContext를 UserDao에 DI 하는 대신 UserDao 내부에서 직접 DI를 적용하는 방법도 존재한다. 그러나, JdbcContext를 싱글톤으로 만들려는 것은 포기해야 한다.
DAO 개수만큼 JdbcContext 오브젝트가 만들어지겠지만, JdbcContext에는 내부에 두는 상태정보가 없기 때문에 메모리에 주는 부담이 거의 없다

→ 상황에 따라 적절하다고 판단되는 방법을 취사 선택해 사용하면 된다. 왜 그렇게 선택했는지에 대한 분명한 이유와 근거를 설명할 수 없다면 차라리 인터페이스를 만들어서 평범한 DI구조로 만드는 것이 나을 수도 있다.

## 3.5 템플릿과 콜백

템플릿/콜백 패턴이란, 전략 패턴의 기본 구조에 익명 내부 클래스를 활용한 방식

**🪻 템플릿/콜백의 특징**

- 전략 패턴의 전략과 달리 템플릿/콜백 패턴의 콜백은 보통 단일 메소드 인터페이스를 사용한다.
- 콜백은 일반적으로 하나의 메소드를 가진 인터페이스를 구현한 익명 내부 클래스로 만들어진다.
- 콜백 인터페이스의 메소드에는 보통 파라미터가 존재한다.

**🪻 템플릿/콜백의 작업 흐름**

클라이언트는 실행될 로직을 담은 콜백 오브젝트를 만들고, 콜백이 참조할 정보를 제공한다. 콜백은 클라이언트가 템플릿 메소드를 호출할 때 파라미터로 전달한다.
→ 템플릿은 정해진 작업 흐름을 따라 작업을 진행하다가 내부에서 생성한 참조정보를 가지고 골백 오브젝트의 메소드를 호출한다.
→ 템플릿은 콜백이 돌려준 정보를 사용해서 작업을 마저 수행한다.

고정된 작업 흐름을 가지고 있으면서 자주 반복되는 코드가 있다면 분리할 방법을 생각해보는 습관을 기르자.

중복된 코드는 먼저 **메소드로 분리**하는 간단한 시도를 해본다.
그중 일부 작업을 필요에 따라 바꾸어 사용해야 한다면 **인터페이스를 사이에 두고 분리해서 전략 패턴을 적용**하고 DI로 의존관계를 관리하도록 만든다.
바뀌는 부분이 한 애플리케이션 안에서 동시에 여러 종류가 만들어질 수 있다면 이번엔 **템플릿/콜백 패턴**을 적용하는 것을 고려해볼 수 있다.

**🪻 제네릭스를 이용한 콜백 인터페이스**

파일을 라인 단위로 처리해 서 만드는 결과의 타입을 다양하게 가져가고 싶다면, 자바 언어에 타입 파라미터라는 개념을 도입한 제네릭스Generies를 이용하면 된다
