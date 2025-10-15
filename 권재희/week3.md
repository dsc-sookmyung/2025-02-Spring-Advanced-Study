# 3. 템플릿

- 템플릿 : 바뀌는 성질이 다른 코드 중에서 변경이 거의 일어나지 않으며 일정한 패턴으로 유지되는 특성을 가진 부분을 자유롭게 변경되는 성질을 가진 부분으로부터 독립시켜서 효과적으로 활용할 수 있도록 하는 방법

# 3.1 다시 보는 초난감 DAO

## 3.1.1 예외 처리 기능을 갖춘 DAO

### JDBC 수정 기능의 예외처리 코드

- 이전 PreparedStatement에서 예외가 발생할 경우 바로 메소드 중단 발생 → 리소스 반환 불가 → 문제 발생 가능성 → 예외 처리 필요
    - Connection과 PreparedStatement는 풀 방식으로 운영, 미리 리소스를 생성하고 필요할 때 할당, 반환하면 다시 풀에 넣음
- 예외 시점에 따라 어떤 것의 close() 메소드를 실행할 지 달라짐
    - Connection 문제 → ps를 close하면 NullPointerException 발생 가능
    - PreparedStatement 문제 → c와 ps 둘 다 close()
- close()도 SQLException 발생 가능

### JDBC 조회 기능의 예외처리

- ResultSet 추가로 더 복잡해짐
- try/catch/finally 블록 적용

# 3.2 변하는 것과 변하지 않는 것

## 3.2.1 JDBC try/catch/finally 코드의 문제점

- 복잡한 이중 중첩, 모든 메소드마다 반복 → DAP와 DB 연결 기능 분리

## 3.2.2 분리와 재사용을 위한 디자인 패턴 적용

- 변하는 것과 변하지 않는 것 분리, 변하지 않는 부분 재사용

### 템플릿 메소드 패턴의 적용

- 템플릿 메소드 패턴 : 상속을 통해 기능을 확장해서 사용하는 부분 , 변하지 않는 부분 슈퍼클래스로, 변하는 부분 추상 메소드로 정의
    
    ```java
    abstract protected PreparedStatement makeStatement(Connection c) throws SQLException
    ```
    
    ```java
    protected PreparedStatement makeStatement(Connection c) throws SQLException{
    	PreparedStatement ps = c.prepareStatement("");
    	return ps;
    }
    ```
    
- DAO 로직마다 상속을 통해 새로운 클래스 생성해야 함
- 확장구조가 설계하는 시점에 고정

### 전략 패턴의 적용

- 오브젝트를 아예 둘로 분리하고 클래스 레벨에서는 인터페이스를 통해서만 의존
- 컨텍스트 안에서 구체적인 전략 클래스 사용 → OCP 위반

### DI 적용을 위한 클라이언트/컨텍스트 분리

- 어떤 전략을 사용하게 할 것인가는 Client가 결정하도록
- 선정한 전략 클래스의 오브젝트 생성 → 컨텍스트 호출 및 전략 오브젝트 전달하여 메소드 호출

# 3.3 JDBC 전략 패턴의 최적화

- DAO 메소드는 전략 패턴의 클라이언트로서 컨텍스트에 해당하는 메소드에 적절한 전략, 즉 바뀌는 로직 제공

## 3.3.1 전략 클래스의 추가 정보

- add() 메소드 클래스 분리 후 생성자를 통해서 user 제공

## 3.3.2 전략과 클라이언트의 동거

- DAO 메소드마다 새로운 StatementStrategy 구현 클래스 만들어야 함 → 클래스 파일의 개수 많이 늘어남
- DAO 메소드에서 부가적인 정보를 전달하기 위해 생성자와 인스턴스 변수를 만들어야 함

### 로컬 클래스

- StatementStrategy 전략 클래스를 내부 클래스로 정의
- AddStatement 클래스를 add() 메소드 안에
- 로컬 클래스는 자신이 선언된 곳의 정보에 접근 가능

### 익명 내부 클래스

- 익명 내부 클래스 → 선언과 동시에 오브젝트 생성

# 3.4 컨텍스트와 DI

## 3.4.1 JdbcContext의 분리

- jdbcContextWithStatementStrategy()를 UserDao 클래스 밖으로 독립시켜 다른 DAO에서도 사용 가능하도록

### 클래스 분리

- 클래스 분리 : JdbcContext
- 컨텍스트 메소드 : workWithStatementStrategy

## 3.4.2 JdbcContext의 특별한 DI

- 인터페이스 사용 → 스프링 빈으로 DI/코드로 수동 DI 가능

# 3.5 템플릿과 콜백

- 템플릿/콜백 패턴
    - 전략 패턴의 컨텍스트 = 템플릿
    - 익명 내부 클래스로 만들어지는 오브젝트 = 콜백

## 3.5.1 템플릿/콜백의 동작원리

### 템플릿/콜백 특징

![img1.daumcdn.png](attachment:38dc0e40-4772-4dbd-89cf-b2f222736fd1:img1.daumcdn.png)

- 클라이언트 - 템플릿 안에서 실행될 로직을 담은 콜백 오브젝트 생성, 콜백이 참조할 정보 제공
- 만들어진 콜백은 클라이언트가 메소드를 호출할 때 파라미터로 제공
- 템플릿은 정해진 작업 흐름을 따라 진행하다 내부에서 생성한 참조정보를 통해 콜백 오브젝트 메소드 호출
- 콜백은 클라이언트 메소드에 있는 정보와 템플릿의 참조정보를 이용해서 작업 수행
- 콜백의 정보를 사용하여 작업 마저 수행

## 3.5.2 편리한 콜백의 재활용

- 가독성 떨어짐

### 콜백의 분리와 재활용

- 익명 내부 클래스 사용 최소화
- query 분리 → 재활용 가능하도록 수정

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

### 콜백과 템플릿의 결합

- 템플릿 클래스 내부로 → 재사용 가능

```java
public void deleteAll() throws SQLException {
    this.jdbcContext.workWithStatementStrategy(
        new StatementStrategy() {
            public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
                return c.preparedStatement("delete from users");
            }
        }
    );
}
```

# 3.6 스프링의 JdbcTemplate

- 스프링이 제공하는 JDBC 코드용 기본 템플릿 : JdbcTemplate

## 3.6.1 update()

- deleteAll() 메소드 수정
    
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
    

## 3.6.2 queryForInt()

- 두 번의 콜백(Connection, ResultSet)
- integer 타입의 결과를 가져올 수 있는 SQL 문장 전달

## 3.6.3 queryForObject()

- RowMapper 콜백 사용
    
    ```java
    public User get(String id) {
        return this.jdbcTemplate.queryForObject("select * from users where id = ?",
            new Object[] {id},
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
        );
    }
    ```
    

## 3.6.4 query()

- getAll() 구현
    - RowMapper 사용, queryForObject와 비슷

- 최종 완성 DAO
    
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
    		};
    
    	
    	public void add(final User user) {
    		this.jdbcTemplate.update("insert into users(id, name, password) values(?,?,?)",
    						user.getId(), user.getName(), user.getPassword());
    	}
    
    	public User get(String id) {
    		return this.jdbcTemplate.queryForObject("select * from users where id = ?",
    				new Object[] {id}, this.userMapper);
    	} 
    
    	public void deleteAll() {
    		this.jdbcTemplate.update("delete from users");
    	}
    
    	public int getCount() {
    		return this.jdbcTemplate.queryForInt("select count(*) from users");
    	}
    
    	public List<User> getAll() {
    		return this.jdbcTemplate.query("select * from users order by id",this.userMapper);
    	}
    
    }
    ```