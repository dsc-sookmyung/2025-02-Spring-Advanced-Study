# [정은서] 2025 GDG Spring Advanced Study - 3주차
# 토비의 스프링 3.1(vol1) - 3장 템플릿

## 템플릿

- 개방 폐쇄 원칙
    - 확장에는 자유롭게 열려있고 변경에는 굳게 닫혀 있음
- 템플릿
    - 바뀌는 성질이 다른 코드 중에서 변경이 거의 일어나지 않으며 일정한 패턴으로 유지되는 특성을 가진 부분을 자유롭게 변경되는 성질을 가진 부분으로부터 독립시켜서 효과적으로 활용할 수 있도록 하는 방법

### 3.1 다시 보는 초난감 DAO
- UserDao 에는 예외상황에 대한 처리가 없음

3.1.1 예외 처리 기능을 갖춘 DAO

- 어떠한 상황에서도 가져온 리소스를 반환하도록 try/catch/finally 구문 사용을 권장
- finally : try 블록을 수행한 후에 예외가 발생하든 정상적으로 처리되든 상관없이 반드시 실행해야 하는 코드를 넣을 때 사용

```java
public void deleteAll() throws SQLException {
	Connection C = null;
	PreparedStatement ps = null;
	
	try {
	// 예외가 발생할 가능성이 있는 코드를 모두 try 블록으로
		c = dataSource.getConnection();
		ps = c.prepareStatemnet("delete from users");
		ps.executeUpdate();
	}catch (SQLException e) {
		throw e; // 예외가 발생했을 때 부가적인 작업을 해줄 수 있도록 
	}finally {
		if (ps!=null) {
			try {
				ps.close();
			}catch (SQLException e){}
		}
		if (c != null) {
			try {
				c.close();
			} catch(SQLException e) {} 
		}
	}
}
```
```java
public int getCount() throws SQLException {
	Connection C = null;
	PreparedStatement ps = null;
	ResultSet rs = null;
	
	try {
	// 예외가 발생할 가능성이 있는 코드를 모두 try 블록으로
		c = dataSource.getConnection();
		ps = c.prepareStatemnet("select count(*) from users");
		
		rs = ps.execureQuery();
		rs.next();
		return re.getInt(1);
	}catch (SQLException e) {
		throw e; 
	}finally {
		if (rs != null) {
			try {
				rs.close();
			}catch (SQLException e) {}
		}
		if (ps!=null) {
			try {
				ps.close();
			}catch (SQLException e){}
		}
		if (c != null) {
			try {
				c.close();
			} catch(SQLException e) {} 
		}
	}
}
```

### 3.2 변하는 것과 변하지 않는 것

3.2.1 JDBC try/catch/finally 코드의 문제점

- 반복되는 코드가 너무 많다
- 변하지 않는, 그러나 많은 곳에서 중복되는 코드와 로직에 따라 확장되고 변하는 코드를 잘 분리해내는 작업이 필요

3.2.2 분리와 재사용을 위한 디자인 패턴 적용

- 변하는 성격이 다른 것을 찾아내기
- 메소드 추출
    - 변하는 부분을 메소드로 추출하고 변하지 않는 부분에서 호출하도록
    - 메소드 추출 리팩토링을 적용하는 경우에는 분리시킨 메소드를 다른 곳에서 재사용할 수 있어야 하는데 이건 분리시키고 남은 메소드가 재사용이  필요한 부분이고 분리된 메소드는 DAO 로직마다 새롭게 만들어서 확장돼야 하는 부분이 되어버림
- 템플릿 메소드 패턴의 적용
    - 상속을 통해 기능을 확장해서 사용하는 부분
    - 변하지 않는 부분은 슈퍼 클래스에 두고 변하는 부분은 추상 메소드로 정의
    - DAO 로직마다 상속을 통해 새로운 클래스를 만들어야 한다는 점이 불편
    - 관계에 대한 유연성이 떨어짐
- 전략 패턴의 적용
    - 오브젝트를 아예 둘로 분리하고 클래스 레벨에서는 인터페이스를 통해서만 의존하도록 만드는 전략 패턴
        - 개방폐쇄원칙을 잘 지키면서도 템플릿 메소드 패턴보다 유연하고 확장성이 좋음

```java
public interface StatementStrategy {
    PreparedStatement makePreparedStatement(Connection c) throws SQLException;
}
```
```java
public class DeleteAllStatement implements StatementStrategy{
    @Override
    public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
        return c.prepareStatement("delete from users");
    }
}
```

```java
public void deleteAll() throws SQLException {
        Connection c = null;
        PreparedStatement ps = null;

        try {
            c = dataSource.getConnection();

            StatementStrategy strategy = new DeleteAllStatement();
            ps = strategy.makePreparedStatement(c);

            ps.executeUpdate();
        } catch (SQLException e) {
            throw e;
        } finally {
            if(ps != null) { try { ps.close(); } catch (SQLException e) { } }
            if(c != null) { try { c.close(); } catch (SQLException e) { } }
        }
    }
```

- 전략패턴은 필요에 따라 컨텍스트는 그대로 유지되면서 전략을 바꿔 쓸 수 있다는 것인데 이렇게 컨텍스트 안에서 이미 구체적인 전략 클래스인 DeleteAllStatement 를 사용하도록 고정되어 있는것이 이상함
- DI 적용을 위한 클라이언트/컨텍스트 분리
    - 전략 패턴에 따르면 Context가 어떤 전략을 사용하게 할 것인가는 Context를 사용하는 앞단의 Client 가 결정하는 것이 일반적
    - Client 가 구체적인 전략의 하나를 선택하고 오브젝트로 만들어서 Context 에 전달하는 것
    - Context 는 전달받은 Object 사용
    - 결국 이 구조에서 전략 오브젝트 생성과 컨텍스트로의 전달을 담당하는 책임을 분리 시킨 것이 ObjectFactory이며, 이를 일반화한 것이 앞에서 살펴봤던 의존관계 주입(DI)
    - 결국 DI란 이러한 전략 패턴의 장점을 일반적으로 활용할 수 있도록 만든 구조라고 볼 수 있음
    - 컨텍스트에 해당하는 부분은 별도의 메소드로 독립시켜보자
        - 클라이언트는 DeleteAllStatement 오브젝트 같은 전략 클래스의 오브젝트를 컨텍스트 메소드로 전달
        - 전략 인터페이스인 StatementStrategy를 컨텍스트 메소드 파라미터로 지정하자

```java
public void jdbcContextWithStatementStrategy(StatementStrategy statementStrategy) throws SQLException {
    Connection c = null;
    PreparedStatement ps = null;

    try {
        c = dataSource.getConnection();
        ps = statementStrategy.makePreparedStatement(c);

        ps.executeUpdate();
    } catch (SQLException e) {
        throw e;
    } finally {
        if(ps != null) { try { ps.close(); } catch (SQLException e) { } }
        if(c != null) { try { c.close(); } catch (SQLException e) { } }
    }
}
```

## 3.3 JDBC 전략 패턴의 최적화

3.3.2 전략과 클라이언트의 동거

- 로컬 클래스
    - 전략 클래스를 매번 독립된 파일로 만들지 말고 UserDao 클래스 안에 내부 클래스로 정의해버리면 클래스 파일이 많아지는 문제 해결 가능
    - 특정 메소드에서만 사용되는 것이라면 로컬 클래스로 만들자
    - 로컬 클래스는 클래스가 내부 클래스이기 때문에 자신이 선언된 곳의 정보에 접근할 수 있다는 점
    - AddStatement가 사용될 곳이 add() 메소드뿐이라면, 사용하기 전에 바로 정의해서 쓰는것도 나쁘지 않다. 덕분에 클래스 파일이 하나 줄었고, add() 메소드 안에서 생성로직을 함께 볼 수 있으니 코드를 이해하기도 좋음
    - 다만 내부 클래스에서 외부의 변수를 사용할 때 외부 변수는 반드시 final로 선언

```java
public void add(final User user) throws SQLException {
        class AddStatement implements StatementStrategy {
                ...
        }
        StatementStrategy st = new AddStatement();
        jdbcContextWithStatementStrategy(st);
}
```

- 익명 내부 클래스
    - 이름을 갖지 않는 클래스
    - 선언과 동시에 오브젝트 생성

## 3.4 컨텍스트와 DI

3.4.1 jdbcContext의 분리

- UserDao의 메소드는 클라이언트
- 익명 내부 클래스로 만들어지는 것은 개별적 전략
- jdbcContextWithStatementStrategy()를 UserDao 클래스 밖으로 독립시켜서 모든 DAO가 사용할 수 있게 한다.
- 클래스 분리
    - 빈의 의존관계 변경
        - 스프링의 DI 는 기본적으로 인터페이스를 사이에 두고 의존 클래스를 바꾸어 사용하도록 하는것이 목적
        - 스프링의 빈 설정은 런타임 시에 만들어지는 오브젝트 레벨의 의존관계에 따라 정의됨

3.4.2 JdbcContext 의 특별한 DI

- 스프링 빈으로 DI
    - 스프링의 DI 는 넓게 보면 객체의 생성과 관계 설정에 대한 제어 권한을 오브젝트에서 제거하고 외부로 위임했다는 IoC 개념을 포괄함
    - JdbcContext를 UserDao와 DI 구조로 만들어야 하는 이유
        - JdbcContext가 스프링 컨테이너의 싱글톤 레지스트리에서 관리되는 싱글톤 빈이 되기 때문이다.
        - JdbcContext가 DI를 통해 다른 빈에 의존하고 있기 때문이다.
- 수동 DI
    - UserDao 내부에서 직접 DI를 적용하는 방법
    - JdbcContext에 대한 제어권을 갖고 생성과 관리를 담당하는 UserDao에게 DI까지 맡긴다.
    - 한 오브젝트의 수정자 메소드에서 다른 오브젝트를 초기화하고 코드를 이용해 DI를 하는 것이다.
    - 인터페이스를 사용하지 않고 DAO와 밀접한 관계를 갖는 클래스를 DI에 적용하는 방법
        - 빈으로 등록하는 방법 → 오브젝트 간 실제 의존관계가 설정파일에 명확하게 드러난다, 구체적인 클래스와의 관계가 설정에 직접 노출된다.
        - 수동으로 DI하는 방법 → 관계가 외부에 드러나지 않는다. 싱글톤으로 만들 수 없고 부가적인 코드가 필요하다.

### 3.5 템플릿과 콜백

- 템플릿 : 어떤 목적을 위해 미리 만들어둔 모양이 있는 틀
    - 고정된 틀 안에 바꿀 수 있는 부분을 넣어서 사용하는 경우 템플릿이라고 부른다.
- 콜백 : 콜백은 실행되는 것을 목적으로 다른 오브젝트의 메소드에 전달되는 오브젝트
    - 파라미터로 전달되지만 값을 참조하기 위한 것이 아니라 특정 로직을 담은 메소드를 실행시키기 위해 사용
    - 자바에선 메소드 자체를 파라미터로 전달할 방법은 없기 때문에 메소드가 담긴 오브젝트를 전달

3.5.1 템플릿/콜백의 동작원리

- 템플릿/콜백의 특징
    - 보통 단일 메소드 인터페이스를 사용
    - 콜백은 일반적으로 하나의 메소들르 가진 인터페이스를 구현한 익명 내부 클래스로 만들어짐
    - 클라이언트의 역할은 템플릿 안에서 실행될 로직을 담은 콜백 오브젝트를 만들고, 콜백이 참조할 정보를 제공하는 것이다. 만들어진 콜백은 클라이언트가 템플릿의 메소드를 호출할 때 파라미터로 전달된다.
    - 템플릿은 정해진 작업 흐름을 따라 작업을 진행하다가 내부에서 생성한 참조정보를 가지고 콜백 오브젝트의 메소드를 호출한다. 콜백은 클라이언트 메소드에 있는 정보와 템플릿이 제공한 참조정보를 이용해서 작업을 수행하고 그 결과를 다시 템플릿에 돌려준다.
    - 템플릿은 콜백이 돌려준 정보를 사용해서 작업을 마저 수행한다. 경우에 따라 최종 결과를 클라이언트에 다시 돌려주기도 한다.

3.5.2 편리한 콜백의 재활용

- DAO 메소드에서 매번 익명 내부 클래스를 사용하기 때문에 상대적으로 코드를 작성하고 읽기가 조금 불편
- 콜백의 분리와 재활용

```java
public void deleteAll() throws SQLException {
    executeSql("delete from users");
}
--- 분리
private void executeSql(final String query) throws SQLException {
    this.jdbcContext.workWithStatementStrategy(
        new StatementStrategy() {
            public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
                return c.preparedStatement(query);
            }
        }
    );
}
```

3.5.3 템플릿/콜백의 응용

- 템플릿/콜백 패턴은 DI 와 객체지향 설계를 적극적으로 응용한 결과
- 가장 전형적인 템플릿/콜백 패턴의 후보는 try/catch/finally
- BufferedReader를 전달받는 콜백 인터페이스

```java
public interface BufferedReaderCallback{
    Integer doSomethingWithReader(BufferedReader br) throws IOException;
}
```
BufferedReaderCallback을 사용하는 템플릿 메소드
```java
public integer fileReadTemplate(String filepath, BufferedReaderCallback callback) throws IOException {
    BufferedReader br = null;
    try {
        br = new BufferedReader(new FileReader(filepath));
        int ret = callback.doSomethingWithReader(br); //콜백오브젝트 호출
        return ret;
    }
    catch(IOException e) {
        System.out.println(e.getMessage());
        throw e;
    }
    finally {
        if (br != null) {
            try {br.close();}
            catch(IOException e) {
                System.out.println(e.getMessage());
            }
        }
    }
}
```
- 템플릿/콜백을 적용한 calcSum() 메소드
```java
public Integer calcSum(String filepath) throws IOException {
    BufferedReaderCallback sumCallback = new BufferedReaderCallback() {
        public Integer doSomethingWithReader(BufferedReader br) throws IOException {
            Integer sum = 0;
            String line = null;
            while ((line = br.readLine()) != null){
                sum += Integer.valueOf(line);
            }
            return sum;
        }
    };
    return fileReadTempalte(filepath, sumCallback);
}
```

### 3.6 스프링의 JdbcTemplate
- 스프링은 JDBC 를 이용하는 DAO 에서 사용할 수 있도록 준비된 다양한 템플릿과 콜백을 제공 

3.6.1 update()
```java
this.jdbcTemplate.update("insert into users(id, name, passowrd) values(?,?,?)",
user.getId(), user.getName(), user.getPassword());
```

3.6.2 queryForInt()
```java
public int getCount() {
    return this.jdbcTemplate.queryForInt("select count(*) from users");
}
```

3.6.3 queryForObject()
```java
public User get(String id) {
    return this.jdbcTemplate.queryForObject("select * from users where id = ?",
    new Object[] {id},
    // ResultSet 한 결과를 오브젝트에 매핑해주는 RowMapper 콜백
    new RowMapper<User>(){
        public User mapRow(ResultSet rs, int rowNum) throws SQLException {
            User user = new User();
            user.setId(rs.getString("id"));
            user.setName(rs.getString("name"));
            user.setPassword(rs.getString("password"));
            return user;
        }
    });
}
```

3.6.4 query() 
- query() 템플릿을 이용하는 getAll() 구현
```java
public List<User> getAll() {
    return this.jdbcTemplate.query(
        "select * from users order by id",
        new RowMapper<User>(){
        public User mapRow(ResultSet rs, int rowNum) throws SQLException {
            User user = new User();
            user.setId(rs.getString("id"));
            user.setName(rs.getString("name"));
            user.setPassword(rs.getString("password"));
            return user;
        }
    });
}
```
3.6.5 재사용 가능한 콜백의 분리
- jdbctemplate를 적용한 UserDao 클래스
```java
public class UserDao {
    public void setDataSource(DataSource dataSource){
        this.jdbcTemplate = new JdbcTemplate(dataSource);
    }

    private JdbcTemplate jdbcTemplate;

    private RowMapper<User> userMapper =
    new RowMapper<User>(){
        public User mapRow(ResultSet rs, int rowNum) throws SQLException {
            User user = new User();
            user.setId(rs.getString("id"));
            user.setName(rs.getString("name"));
            user.setPassword(rs.getString("password"));
            return user;
        }
    };

    public void add(final User user){
        this.jdbcTemplate.update("insert into users(id, name, passowrd) values(?,?,?)",
        user.getId(), user.getName(), user.getPassword());
    }

    public User get(String id){
        return this.jdbcTemplate.queryForObject("select * from users where id = ?",
        new Object[] {id}, this.userMapper);
    }

    public void deleteAll(){
        this.jdbcTemplate.update("delete from users");
    }

    public int getCount() {
    return this.jdbcTemplate.queryForInt("select count(*) from users");
    }

    public List<User> getAll(){
        return this.jdbcTemplate.query("select * from users order by id",
        this.userMapper);
    }
}
```