# 3장 템플릿
계속 바뀌는 코드 내에서 **거의 변경이 없고 일정한 패턴으로 유지되는 특성**을 가진 부분을 자유롭게 변경될 수 있도록 독립시켜 활용하는 방법

## 다시 보는 초난감 DAO
### 예외처리 기능을 갖춘 DAO
예외 상황에 대한 처리가 아직 부족한 `UserDao`. JDBC 코드에서는 예외가 발생했을 경우에도 **사용한 리소스를 반드시 반환**해야 한다. 예전 코드에서는 예외가 발생하면 메서드 실행을 끝내지 못하고 바로 메서드를 빠져나가는데, `close()` 메서드가 실행되지 않아서 제대로 리소스가 반환되지 않을 수 있다.

보통 재사용 가능한 풀로 DB 커넥션을 관리한다. 그래서 가져간 커넥션을 `close()`해서 돌려줘야 다시 풀에 넣었다가 다음 커넥션 요청에서 재사용할 수 있다. 그런데 오류 발생 시, 반환되지 못한 Connection이 쌓여 리소스가 모자라는 심각한 오류가 발생할 수 있다. 이를 방지하기 위해 아래와 같은 코드로 수정한다.

🕴️JDBC 수정 기능의 예외처리
```java
public void deleteAll() throws SQLException {
    Connection c = null;
    PreparedStatement ps = null;

    try {   // 예외가 발생할 수 있는 코드를 모두 try 블록으로 감싼다
        c = dataSource.getConnection();
        ps = c.prepareStatement("delete from users");
        ps.executeUpdate();
    } catch (SQLException e) {
        throw e;
    } finally {     // 예외가 발생했을 때나 안 했을 때나 모두 실행
        if (ps != null) {
            try {
                ps.close();
            } catch (SQLException e) {  // SQLException이 발생할 수 있기 때문에 처리
            }
        }
        if (c != null) {
            try {
                c.close();
            } catch (SQLException e) {
            }
        }
    }
}
```

등록된 `User`의 수를 가져오는 `getCount()` 메소드에 예외처리 블록을 적용해보자.

🕴️JDBC 조회 기능의 예외처리
```java
public int getCount() throws SQLException {
    Connection c = null;
    PreparedStatement ps = null;
    ResultSet rs = null;

    try {
        c = dataSource.getConnection();

        ps = c.prepareStatement("select count(*) from users");

        rs = ps.executeQuery();
        rs.next();
        return rs.getInt(1);    // ResultSet도 SQLException이 발생할 수 있는 코드
    } catch (SQLException e) {
        throw e;
    } finally {
        if (rs != null) {
            try {
                rs.close();
            } catch (SQLException e) {      // close()는 만들어진 순서의 반대로
            }
        }

        if (ps != null) {
            try {
                ps.close();
            } catch (SQLException e) {
            }
        }

        if (c != null) {
            try {
                c.close();
            } catch (SQLException e) {
            }
        }
    }
}

```
<br>


## 변하는 것과 변하지 않는 것
앞서 예외처리 기능을 추가한 `DAO` 코드는 복잡한 try/catch/finally 블록이 2중으로 중첩되어 나오는데, 모든 메소드마다 반복되어 복잡하다. 복잡한 코드는 리팩토링과 복사 및 붙여넣기 등을 통해 필요한 부분만 바꾸다 보면 실수가 나올 수 있고 이는 리소스가 꽉 찼다는 에러가 나오는 중대한 상황이 펼쳐질 수 있다.

그래서 **변하지 않지만 많은 곳에서 중복되는 코드**와 로직에 따라 **자꾸 확장되고 자주 변하는 코드**를 잘 분리해내는 작업을 적용할 필요가 있다.

`UserDao`의 메소드를 개선해보자. 변하는 성격이 다른 것을 찾아 수정하자.

🕴️`deleteAll()` 메소드
```java
Connection c = null;
PreparedStatement ps = null;
try {
	c = dataSource.getConnection();     // 변하지 않음
	
	ps = c.prepareStatement("delete from users");   // 변함
	
	ps.executeUpdate();
} catch (SQLException e) {
	throw e;
} finally {
	if (ps != null) { try { ps.close(); } catch (SQLException e) {} }
	if (c != null) { try { c.close(); } catch (SQLException e) {} }     // 변하지 않음
}
```

분리하기 위해 변하는 부분을 메소드로 빼자. 변하지 않는 부분이 변하는 부분을 감싸고 있어서 변하지 않는 부분을 추출하기가 어렵기 때문이다. 사실 이것은 반대로 진행하는 것이기 때문에 분리시키고 남은 메소드가 재사용이 필요하지만 아래 코드로는 불가능하다.

🕴️변하는 부분을 메소드로 추출한 `deleteAll()` 메소드
```java
public void deleteAll() throws SQLException {
    ...
	try {
		c = dataSource.getConnection();
		
		ps = makeStatement(c); // 변하는 부분을 메소드로 추출하고 변하지 않는 부분에서 호출
		
		ps.executeUpdate();
	} catch (SQLException e) {}
    ...
}

private PreparedStatement makeStatement(Connection c) throws SQLException {
	PreparedStatement ps;	
	ps = c.prepareStatement("delete from users");	
	return ps;
}
```

**템플릿 메소드 패턴**을 이용해서 분리하는 방법도 존재한다. 변하지 않는 부분은 슈퍼클래스에, 변하는 부분은 추상 메소드로 정의해 서브클래스에서 오버라이드해 새롭게 정의하는 것이다. 이 방법은 상속을 통해 자유롭게 확장할 수 있고, 불필요한 변화가 생기지 않는다. 다만, DAO 로직마다 상속을 통해 새로운 클래스를 만들어야 한다. 그리고 확장 구조가 클래스 설계를 고정시켜 버린다. 이는 관계에 대한 유연성이 떨어지게 만든다.

### 전략 패턴의 적용
템플릿 메소드 패턴보다 **유연하고 확장성**이 뛰어나며, 오브젝트를 아예 둘로 분리하여 클래스 레벨에서는 **인터페이스를 통해서만 의존**하도록 만드는 전략 패턴을 사용해보자. 전략 패턴의 구조는 **Context(컨텍스트)**에서 일정한 구조를 가지고 동작하다가, 특정 확장 기능은 **Strategy(전략) 인터페이스**를 통해 외부의 독립된 전략 클래스에 위임하는 것으로 이루어져 있다. `deleteAll()`은 JDBC를 이용해 DB를 업데이트하는 작업이기 때문에 변하지 않는 맥락을 가진다.

`deleteAll()`의 컨텍스트
- DB 커넥션 가져오기
- PreparedStatement를 만들어줄 외부 기능 호출하기 **-> 전략**
- 전달받은 `PreparedStatement` 실행하기
- 예외가 발생하면 이를 다시 메소드 밖으로 던지기
- 모든 경우에 만들어진 PreparedStatement와 Connection을 적절히 닫아주기

중요한 것은 커넥션이 없으면 `PreparedStatement`도 만들 수 없어 `PreparedStatement` 생성 전략을 호출할 때는 이 컨텍스트 내에서 만들어둔 DB 커넥션을 전달해야 한다는 점이다.

🕴️`PreparedStatement`를 만드는 전략의 `StatementStrategy` 인터페이스 (Connection을 전달받아 `PreparedStatement`를 만들고 만들어진 `PreparedStatement` 객체를 돌려주기)
```java
package springbook.user.dao;
...
public interface StatementStrategy {
    PreparedStatement makePreparedStatement(Connection c) throws SQLException;
}
```

🕴️`deleteAll()` 메소드의 기능을 구현한 `StatementStrategy` 전략 클래스
```java
package springbook.user.dao;

public class DeleteAllStatement implements StatementStrategy {      // 인터페이스 상속
    public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
        PreparedStatement ps = c.prepareStatement("delete from users");     // 바뀌는 부분인 PreparedStatement를 생성
        return ps;
    }
}
```

### DI 적용을 위한 클라이언트/컨텍스트 분리
컨텍스트 안에서 이미 구체적인 전략 클래스를 사용하도록 고정되어 있다면 전략 패턴에도, OCP에도 잘 맞지 않는다. 전략 패턴에 따르면 Client가 *Context가 어떤 전략을 사용할지* 결정하는 것이 일반적이다. 컨텍스트에 해당하는 JDBC try/catch/finally 코드를 **클라이언트 코드**인 `StatementStrategy`를 만드는 부분에서 독립시켜야 한다.

**컨텍스트**에 해당하는 부분은 **별도의 메소드**로 분리한다. 클라이언트는 전략 클래스의 오브젝트를 컨텍스트 메소드로 전달해야 한다. 이를 위해 전략 인터페이스를 컨텍스트 메소드 파라미터로 지정해야 한다. 컨텍스트를 별도의 메소드로 분리했기 때문에 `deleteAll()` 메소드가 클라이언트가 된다.

🕴️메소드로 분리한 try/catch/finally 컨텍스트 코드
```java
public void jdbcContextWithStatementStrategy(StatementStrategy strategy) throws SQLException { // 클라이언트가 컨텍스트를 호출할 때 넘겨줄 전략 파라미터 strategy
    Connection c = null;
    PreparedStatement ps = null;

    try {       // JDBC try/catch/finally 구조로 만들어진 컨텍스트 내에서 작업 수행
        c = dataSource.getConnection();
        
        ps = strategy.makePreparedStatement(c);
        
        ps.executeUpdate();
    } catch (SQLException e) {
        throw e;
    } finally {
        if (ps != null) { try { ps.close(); } catch (SQLException e) {}}
        if (c != null) { try { c.close(); } catch (SQLException e) {}}
    }
}
```

<br>


## JDBC 전략 패턴의 최적화
위 코드는 모든 `DAO` 메소드마다 새로운 `Strategy` 구현 클래스를 만들어야 한다. 그리고 부가정보를 전달하기 위해 Strategy 구현체에 생성자나 인스턴스 변수 등을 번거롭게 만들어야 한다.

이 문제들을 해결하기 위해 **로컬 클래스**를 사용할 수 있다. 매번 독립된 파일이 아니라 `UserDao` 클래스 안에 내부 클래스로 정의하는 것이 로컬 클래스다. 자신이 정의된 메소드의 파라미터 로컬 변수에 접근할 수 있기 때문에 로컬 클래스에 `user`를 받는 생성자가 필요 없다. 대신 **메소드의 파라미터 로컬 변수는 반드시 `final`로 정의**해줘야 한다.

🕴️`add()` 메소드의 로컬 변수를 직접 사용하도록 수정한 `AddStatement`
```java
public void add(final User user) throws SQLException {
    class AddStatement implements StatementStrategy {
        public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
            PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
            ps.setString(1, user.getId());
            ps.setString(2, user.getName());
            ps.setString(3, user.getPassword()); // 로컬(내부) 클래스의 코드에서 외부의 메소드 로컬 변수에 직접 접근할 수 있다.
            return ps;
        }
    }
    StatementStrategy st = new AddStatement(); // 생성자 파라미터로 user를 전달하지 않아도 된다.
    jdbcContextWithStatementStrategy(st);
}
```

또 다른 방법으로는 **익명 내부 클래스**가 있다. 로컬 클래스 `AddStatement`가 `add()`에서만 사용된다면 클래스 선언과 오브젝트 생성이 결합된 형태로 만들어지는, 이름조차 필요 없는 익명 내부 클래스로 만들 수 있다.

🕴️`add()` 메소드 파라미터로 넘긴 익명 내부 클래스
```java
public void add(final User user) throws SQLException {
    jdbcContextWithStatementStrategy(
        new StatementStrategy() {
            public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
                PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
    
                ps.setString(1, user.getId());
                ps.setString(2, user.getName());
                ps.setString(3, user.getPassword());
    
                return ps;
            }
        }
    );
}
```

<br>


## 컨텍스트와 DI
`jdbcContextWithStateStrategy()`는 JDBC의 기본적 흐름을 담고 있는 컨텍스트로써 다른 DAO에서도 사용이 가능하다. 그래서 `UserDao`에서 따로 **클래스 밖으로 독립**시키는 작업을 통해 모든 DAO가 사용할 수 있게 만들어보자.

`JdbcContext` 클래스에 `workWithStatementStrategy()` 라는 이름으로 옮긴다. 그리고 `JdbcContext`가 `DataSource`에 의존하고 있기 때문에 `DataSource` 타입 빈을 DI 받을 수 있도록 만든다.

🕴️`JdbcContext` 클래스
```java
public class JdbcContext {
    private DataSource dataSource;

    public void setDataSource(DataSource dataSource) {
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
            if (ps != null) { try { ps.close(); } catch (SQLException e) {} }
            if (c != null) { try { c.close(); } catch (SQLException e) {} }
        }
    }
}
```

위 코드에 맞게 `UserDao`도 수정해주면 `UserDao`는 `JdbcContext`에 의존하게 된다. 그런데 `JdbcContext`는 인터페이스인 `DataSource`와 달리 구체 클래스이다. `JbcContext`는 독립적인 JDBC 컨텍스트를 제공하는 서비스 오브젝트로서 의미가 있고 구현 방법이 바뀔 가능성이 없으니 `UserDao`와 `JdbcContext`는 인터페이스를 사이에 두지 않고 DI를 적용하는 특별한 구조를 가지게 되는 것이다.

### JdbcContext의 특별한 DI
`JdbcContext`를 `UserDao`와 DI 구조로 만드는 이유는 아래와 같다.

첫째는 `JdbcContext`가 스프링 컨테이너의 싱글톤 레지스트리에서 관리되는 **싱글톤 빈**이 되기 때문이다. `JdbcContext`는 그 자체로 변경되는 상태정보를 갖고 있지 않는다. 내부에서 사용할 `dataSource`라는 인스턴스 변수가 있지만, `dataSource`는 읽기전용이라 `JdbcContext`가 싱글톤이 되는데 문제가 없다. `JdbcContext`는 JDBC 컨텍스트 메소드를 제공해주는 일종의 서비스 오브젝트로서 의미가 있고, 그래서 싱글톤으로 등록돼서 여러 오브젝트에서 공유해 사용되는 것이 이상적이다.

그리고 둘째는 `JdbcContext`가 DI를 통해 **다른 빈에 의존**하고 있기 때문이다. `JdbcContext`는 `dataSource` 프로퍼티를 통해 `DataSource` 오브젝트를 주입받도록 되어 있고, DI를 위해선 주입되는 오브젝트와 주입받는 오브젝트 양쪽 모두가 스프링 빈으로 등록돼야 한다. 스프링이 생성하고 관리하는 IoC 대상이어야 DI에 참여할 수 있기 때문에 `JdbcContext`는 다른 빈을 DI 받기 위해서라도 스프링 빈으로 등록돼야 한다.

#### 코드를 이용하는 수동 DI
위 방법과 다르게 스프링 빈이 아니라 `UserDao`에서 **직접 DI를 적용**할 수도 있다. `UserDao` 내부에서 직접 DI를 할 수 있도록 맡기면 된다. `DataSource`를 `UserDao`가 대신 DI받고, `UserDao`가 사용할 목적이 아닌 `JdbcContext`에 전달해줄 목적으로 DI 받는다.

🕴️`JdbcContext` 생성 & DI 작업 수행하는 `setDataSource()` 메소드
```java
public class UserDao {
    ...
    JdbcContext jdbcContext;

    public void setDataSource(DataSource dataSource) {
        this.jdbcContext = new JdbcContext();   // 수정자 메소드이면서 JdbcContext에 대한 생성, DI 작업을 동시에 수행
        this.jdbcContext.setDataSource(dataSource);     // 의존 오브젝트 주입
        this.dataSource = dataSource;   // JdbcContext를 적용하지 않은 메소드를 위해 저장
    }
    
    // ...
    
}
```

<br>


## 템플릿과 콜백
템플릿과 콜백 패턴은 **'전략패턴 + 익명 내부 클래스'**라고 말할 수 있다. 그래서 *컨텍스트(JdbcContext)를 템플릿*, *익명 내부 클래스를 콜백*으로 본다.

전략패턴과 달리 콜백은 단일 메소드 인터페이스로 사용한다. 템플릿 작업 흐름 중 보통 한 번만 호출되기 때문이다. 즉 콜백은 하나의 메소드를 가진 인터페이스를 구현한 익명 내부 클래스라고 볼 수 있다. 또한 콜 백 인터페이스 메소드는 보통 파라미터가 있는데, 템플릿 작업 흐름 중에 만들어지는 컨텍스트 정보를 전달 받을 때 사용된다. `JdbcContext`의 `workWithStatementStrategy()` 메소드 내부에서 생성된 `Connection` 오브젝트가 콜백 메소드 `makePreparedStatement()` 파라미터로 넘어간다.

템플릿/콜백 패턴의 일반적인 작업 흐름
1. 클라이언트의 역할
    - 클라이언트의 역할은 템플릿 안에서 실행될 로직을 담은 콜백 오브젝트를 만들고, 콜백이 참조할 정보를 제공하는 것이다. 만들어진 콜백은 클라이언트가 템플릿의 메서드를 호출할 때 파라미터로 전달된다.
2. 템플릿의 역할
    - 템플릿은 정해진 작업 흐름을 따라 작업을 진행하다가 내부에서 생성한 참조 정보를 가지고 콜백 오브젝트의 메서드를 호출한다. 콜백은 클라이언트 메서드에 있는 정보와 템플릿이 제공한 참조 정보를 이용해서 작업을 수행하고 그 결과를 다시 템플릿에 돌려준다.
3. 최종 결과 처리
    - 템플릿은 콜백이 돌려준 정보를 사용해서 작업을 마저 수행한다. 경우에 따라 최종 결과를 클라이언트에 다시 돌려주기도 한다.

### 콜백의 분리와 재활용
콜백으로 전달하는 익명 내부 클래스의 코드는 고정된 SQL 문장을 제외하고는 비슷한 코드가 반복된다. 콜백의 중복코드를 메소드 추출 방식으로 따로 빼낸 후 SQL문장(바뀔 수 있는 건 오직 "delete from users" 라는 단순 SQL 문장이므로)만 인자로 넘겨주도록 수정한다.

🕴️변하지 않는 부분을 분리시킨 `deleteAll()` 메소드
```java
public void deleteAll() throws SQLException {
    executeSql("delete from users"); // 변하는 SQL 문장
}
// --- 분리 ---
private void executeSql(final String query) throws SQLException {
    this.jdbcContext.workWithStatementStrategy(
        new StatementStrategy() {       // 변하지 않는 콜백 클래스 정의와 오브젝트 생성
            public PreparedStatement makePreparedStatement(Connection c)
                throws SQLException {
                return c.prepareStatement(query);
            }
        }
    );
}
```

### 콜백과 템플릿의 결합
`executeSql()`은 `UserDao`만 사용하기 아깝기 때문에 재사용 가능한 콜백 기능이라면 DAO가 공유할 수있는 템플릿 클래스 안으로 옮겨도 된다. 템플릿은 `JdbcContext` 클래스가 아니라 메소드이기 때문에 클래스로 콜백 생성과 템플릿 호출이 담긴 `executeSql()` 메소드를 옮겨도 괜찮다.

🕴️`JdbcContext`로 옮긴 `executeSql()` 메소드
```java
public class JdbcContext {
    public void executeSql(final String query) throws SQLException {
        workWithStatementStrategy(
            new StatementStrategy() {
                public PreparedStatement makePreparedStatement(Connection c)
                    throws SQLException {
                    return c.prepareStatement(query);
                }
            }
        );
    }
}
```

<br>


## 스프링의 JdbcTemplate

### update()
`deleteAll()`에 적용된 콜백은 `StatementStrategy` 인터페이스의 `makePreapredStatement()` 메소드인데 `JdbcTemplate`의 콜백은 `PreparedStatementCreator` 인터페이스의 `createPreparedStatement()` 메소드이다. 흐름, 구조가 동일하며 `PreparedStatementCreator` 타입의 콜백을 받아서 사용하는 `JdbcTemplate` 템플릿 메소드는 `update()`다.

🕴️콜백과 템플릿 메소드를 사용하도록 수정한 `deleteAll()` 메소드
```java
public void deleteAll() {
    this.jdbcTemplate.update(
        new PreparedStatementCreator() {
            public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
                return con.prepareStatement("delete from users");
                }
            }
    );
}
```

### queryForInt()
`query()` 메소드는 인자로 콜백함수 2개를 받도록 되어 있다. 첫 번째 `PreparedStatementCreator` 콜백의 실행 결과가 템플릿에 전달되고, 두 번째 `ResultSetExtractor` 콜백은 템플릿이 제공하는 `ResultSet`을 이용해 원하는 값을 템플릿에 전달하고 최종적으로 `query()`의 리턴값으로 돌려준다. 또한 `ResultSetExtractor`는 제네릭스 타입 파라미터를 가지게 되어 숫자뿐만 아니라 다양한 타입의 값을 추출할 수 있다.

🕴️`JdbcTemplate`을 이용해 만든 `getCount()`
```java
public int getCount() {
    return this.jdbcTemplate.query(new PreparedStatementCreator() {
        public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
            return con.prepareStatement("select count(*) from users");
        }

    }, new ResultSetExtractor<Integer>() {  // 두 번째 콜백, ResultSet으로부터 값 추출
        public Integer extractData(ResultSet rs) throws SQLException, DataAccessException {
            rs.next();
            return rs.getInt(1);
        }
    });
}
```

### queryForObject()
`getCount()`에 적용했던 `ResultSetExtractor` 콜백 대신 `ResultSet`의 로우 하나를 매핑하기 위해 사용하는 `RowMapper` 콜백을 사용하자. 기본키 값으로 조회하는 `get()`의 SQL 실행 결과는 로우가 하나인 `ResultSet`이다. 따라서 `ResultSet`의 첫 번째 행에 `RowMapper`를 적용하도록 만들 것이다.

🕴️`queryForObject`와 `RowMapper`를 적용한 `get()` 메소드
```java
public User get(String id) {
    return this.jdbcTemplate.queryForObject("select * from users where id = ?",
        new Object[] {id}, // SQL에 바인딩할 파라미터 값, 가변인자 대신 배열을 사용한다.
        new RowMapper<User>() {
            public User mapRow(ResultSet rs, int rowNum)
                throws SQLException {
                User user = new User();
                user.setId(rs.getString("id"));
                user.setName(rs.getString("name"));
                user.setPassword(rs.getString("password"));
                return user;
            }
        }
    ); // ResultSet 한 로우의 결과를 오브젝트에 매핑해주는 RowMapper 콜백
}
```

### query()

🕴️`query()` 템플릿을 이용하는 `getAll()` 메소드
```java
public List<User> getAll() {
    return this.jdbcTemplate.query("select * from users order by id",
        new RowMapper<User>() {
            public User mapRow(ResultSet rs, int rowNum)
                throws SQLException {
                User user = new User();
                user.setId(rs.getString("id"));
                user.setName(rs.getString("name"));
                user.setPassword(rs.getString("password"));
                return user;
            }
        });
}
```

### 재사용 가능한 콜백의 분리
`UserDao`의 모든 메소드가 `JdbcTemplate`을 이용하기 때문에 필요 없어진 `DataSource`는 삭제한다. 대부분은 `JdbcTemplate`을 이용해 중복된 코드를 제거했으나 `get()`이나 `getAll()`에 중복되어 있는 `RowMapper` 콜백을 메소드에서 분리해 중복을 없애고 하나만 만들어서 재사용되게 만들어야 한다.

🕴️`JdbcTemplate`을 이용한 `UserDao` 클래스
```java
public class UserDao {
    public void setDataSource(DataSource dataSource) {
        this.jdbcTemplate = new JdbcTemplate(dataSource);
    }

    private JdbcTemplate jdbcTemplate;

    private RowMapper<User> userMapper =
        new RowMapper<User>() {
            public User mapRow(ResultSet rs, int rowNum) throws SQLException {
                User user = new User();
                user.setId(rs.getString("id"));
                user.setName(rs.getString("name"));
                user.setPassword(rs.getString("password"));
                return user;
            }
        );

    public void add(final User user) {
        this.jdbcTemplate.update("insert into users(id, name, password) values(?,?,?)", user.getId(), user.getName(), user.getPassword());
    }

    public User get(String id) {
        return this.jdbcTemplate.queryForObject("select * from users where id = ?", new Object[] {id}, this.userMapper);
    }

    public void deleteAll() {
        this.jdbcTemplate.update("delete from users");
    }

    public int getCount() {
        return this.jdbcTemplate.queryForInt("select count(*) from users");
    }

    public List<User> getAll() {
        return this.jdbcTemplate.query("select * from users order by id", this.userMapper);
    }
}
    
```
