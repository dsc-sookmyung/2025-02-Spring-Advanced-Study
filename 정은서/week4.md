# 4주차

## 예외

JdbcTemplete 를 대표로 하는 스프링의 데이터 엑세스 기능에 담겨 있는 예외처리와 관련된 접근 방법에 대해 알아보자 

### 4.1 사라진 SQLException

```java
public void deleteAll() throws SQLException {
	this.jdbcContext.executeSql("delete from users");
}
-------------------------------------------------------------------
public void deleteAll() {
	this.jdbcTemplete.update("delete from users");
}
```

4.1.1 초난감 예외처리

- 예외 블랙홀
    - try-catch 블록을 써서 예외를 잡는 것까지는 좋으나 아무것도 하지 않고 넘어가 버리는 것이 문제가 됨
    - 예외를 무시하고 정상적으로 동작하고 있는 것처럼 다음 코드로 실행을 이어가면 안됨

```java
try {}
catch(SQLException e) { // 예외를 잡고는 아무것도 하지 않는다. 
	System.out.println(e);
	e.printStackTrace();
} // 출력을 한다고 해도 다른 로그나 메시지에 금방 묻혀버림 
```

- 예외 처리할 때 반드시 지켜야 할 핵심 원칙 - 모든 예외는 적절하게 복구되든지 아니면 작업을 중단시키고 운영자 또는 개발자에게 분명하게 통보돼야 한다.
- 무의미하고 무책임한 throws
    - API 등에서 발생하는 예외를 일일이 catch 하기 귀찮으니 아예 throws Exception 이라는 모든 예외를 무조건 던져버리는 선언을 모든 메소드에 기계적으로 넣음
    - 그런 메소드 선언에서는 의미 있는 정보를 얻을 수 없음
- 두 가지 나쁜 습관(예외 블랙홀, 무책임한 throws) 는 용납 x

4.1.2 예외의 종류와 특징

- Error : 시스템에 뭔가 비정상적인 상황이 발생했을 경우에 사용
    - java.lang.Error
- Exception 과 체크 예외
    - java.lang.Exception 클래스와 그 서브클래스로 정의되는 예외들은 에러와 달리 개발자가 만든 애플리케이션 코드의 작업 중에 예외상황이 발생했을 경우에 사용됨
    - Exception Class
        - 체크 예외 : Exception 클래스의 서브 클래스이면서 RuntimeException 클래스를 상속하지 않은 것들 (일반적인 예외)
            - 체크 예외가 발생할 수 있는 메소드를 사용할 경우 반드시 예외 처리를 해야함
        - 언체크 예외 : RuntimeException 을 상속한 클래스들
            - 런타임 예외는 주로 프로그램의 오류가 있을 때 발생하도록 의도된 것들이다.
            - NullPointerException, IllegalArgumentException 등
            - 예상하지 못했던 예외 상황에서 발생하는 것이 아니기 때문에 굳이 catch 나 throws 를 사용하지 않아도 되도록 만든 것임

4.1.3 예외 처리 방법

1. 예외 복구 : 예외 상황을 파악하고 문제를 해결해서 정상 상태로 돌려놓는 것
- IOException 이 발생했을 때 사용자에게 상황을 알려주고 다른 파일을 이용하도록 안내해서 예외 상황을 해결하는 경우
- 재시도를 통해 예외를 복구하는 코드

```java
int maxretry = MAX_RETRY;
while(maxtry-- > 0) {
	try{ return; } // 예외가 발생할 수 있는 시도
	catch(SomeException e) {} // 로그 출력, 정해진 시간만큼 대기
	finally {} // 리소스 반납, 정리 작업
}
throw new RetryFailedException(); // 최대 재시도 횟수를 넘기면 직접 예외 발생 
```

2. 예외 처리 회피 : 예외 처리를 자신이 담당하지 않고 자신을 호출한 쪽으로 던져버리는 것
- 예외를 회피하려면 반드시 다른 오브젝트나 메소드가 예외를 대신 처리할 수 있도록 던져줘야 함
- 콜백 오브젝트는 작업하다 발생하는 SQLException 을 자신이 처리하지 않고 템플릿으로 던져버림
- 무책임한 회피하면 안됨, 의도가 분명해야 함

3. 예외 전환 : 예외를 복구해서 정산적인 상태로는 만들 수 없기 때문에 예외를 메소드 밖으로 던지는 것 
- 방법 1 : 예외 회피와 달리 발생한 예외를 그대로 넘기는 게 아니라 적절한 예외로 전환해서 던진다
    - 목적
        - 내부에서 발생한 예외를 그대로 던지는 것이 그 예외 상황에 대한 적절한 의미를 부여해주지 못하는 경우에 의미를 분명하게 해줄 수 있는 예외로 변경하는 것
    - 새로운 사용자를 등록할 때 아이디가 같은 사용자가 있어 SQLException 이 발생하는데 이 경우 DAO 에서 SQLExceptio 의 정보를 해석해서 DuplicateUserIdException 같은 예외로 바꿔서 던져주는 것이 좋다.
    
    ```java
    public void add(User user) throws DuplicateUserIdException, SQLException {
    	try {}
    	catch(SQLException e) {
    		if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY) 
    			throw DuplicateUserIdException(); // 중첩 예외, 처음 발생한 예외가 무엇인지 확인
    		else throws e;
         }
     } // 그 외의 경우 SQLException
    ```
    
    - 의미가 분명한 예외가 던져지면 적절한 복구 작업을 시도할 수 있다.
    
- 방법 2 : 포장
    - 주로 예외처리를 강제하는 체크 예외를 언체크 예외인 런타임 예외로 바꾸는 경우에 사용함
    - EJBException
    
    ```java
    try {
    	OderHome orderHome = EJBHomeFactory.getInstance().getOrderHome();
    	Order order = orderHome.findByPrimaryKey(Integer id);
    } catch (NamingException ne) {
    	throw new EJBException(ne);
    }
    ```
    
4.1.4 예외처리 전략 

- 일관된 예외처리 전략을 정리해보자
- 런타임 예외의 보편화
    - 애플리케이션 차원에서 예외 상황을 미리 파악하고 예외가 발생하지 않도록 차단하는게 좋다.
    - 대응이 불가능한 체크 예외라면 빨리 런타임 예외로 전환해서 던지는 게 낫다.
- add() 메소드의 예외처리
    - 앞서의 add() 메소드는 DuplicateUserIdException, SQLException 두 가지의 체크 예외를 던지게 되어 있다.
    - DuplicateUserIdException를 굳이 체크 예외로 둬야 하는 것은 아님 → 어디에서는 DuplicateUserIdException을 잡아서 처리할 수 있다면 굳이 체크 예외로 만들지 않고 런타임 예외로 만드는 것이 낫다.
    - add() 메소드를 사용하는 쪽에서 아이디 중복 예외를 처리하고 싶은 경우 활용할 수 있음을 알려주도록 DuplicateUserIdException 을 메소드의 throws 선언에 포함
    
    ```java
    public class DuplicateUserIdException extends RuntimeException {
    	public DuplicateUserIdException(Throwable cause) {
    		super(case);
    	}
    }
    
    public void add() throws DuplicateUserIdException {
    	try {}
    	catch (SQLException e) {
    		if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
    			throw new DuplicateUserIdException(e);
    		}
    		else
    			throw new RuntimeException(e); // 포장
    }
    ```
    
- 애플리케이션 예외 : 시스템 또는 외부의 예외 상황이 원인이 아니라 애플리케이션 자체의 로직에 의해 의도적으로 발생시키고 반드시 catch 해서 무엇인가 조치를 취하도록 요구하는 예외
    - 예 ) 출금하는 경우
        - 정상적인 출금처리를 했을 경우와 잔고 부족이 발생했을 경우에 각각 다른 종류의 리턴 값을 돌려주는 방법
            - 예외 상황에 대한 리턴 값을 잘 관리해야 함
            - 조건문이 다수 생겨 코드가 지저분해질 수 있음
        - 정상적인 흐름을 따르는 코드는 그대로 두고, 잔고 부족과 같은 예외 상황에서는 비즈니스적인 의미를 띤 예외를 던지도록 만드는 방법
            - 잔고 부족인 경우 InsufficientBalanceException 등을 던짐
            - 예외가 발생할 수 있는 코드를 try 블록에 깔끔하게 정리해두고 예외 상황에 대한 처리는 catch 블록에 모아 둘 수 있기 때문에 코드가 깔끔
 

### 4.2 예외 전환

4.2.1 JDBC 의 한계 

- JDBC는 자바를 이용해 DB에 접근하는 방법을 추상화된 API로 정의해놓고 각 DB 업체가 JDBC 표준을 따라 만들어진 드라이버를 제공하게 해줌
- 현실적으로 DB를 자유롭게 바꾸어 사용할 수 있는 DB 프로그램을 작성하는 데는 두 가지 걸림돌이 있다.
    1. JDBC 에서 사용하는 SQL  은 비표준 SQL 임. 호환 가능한 SQL 만 사용하는 방법, DB 별로 별도의 DAO 를 만들거나 SQL 을 외부에 독립
    2. SQLException → DB 마다 SQL 만 다른 것이 아니라 에러의 종류와 원인도 제각각이라는 점,  JDBC 드라이버에서 SQLException 을 담을 상태 코드를 정확하게 만들어주지 않는다는 점 

4.2.2 DB 에러 코드 매핑을 통한 전환

- DB 별 에러 코드를 참고해서 발생한 예외의 원인이 무엇인지 해석해주는 기능을 만드는 것 → DB 종류에 상관없이 동일한 상황에서 일관된 예외를 전달받을 수 있어 효과적인 대응이 가능하다

4.2.3 DAO 인터페이스와 DataAccessException 계층 구조

- DataAccessException 은 의미가 같은 예외라면 데이터 엑세스 기술의 종류와 상관없이 일관된 예외가 발생하도록 만들어줌
- DAO 인터페이스와 구현의 분리
    - DAO 의 사용 기술과 구현 코드는 전략 패턴과 DI를 통해 DAO 를 사용하는 클라이언트에게 감출 수 있지만 메소드 선언에 나타나는 예외 정보가 문제가 될 수 있다.
- 데이터 엑세스 예외 추상화와 DataAccessException 계층 구조
    - JBDC Templete 과 같이 스프링의 데이터 액세스 지원 기술을 이용해 DAO 를 만들면 사용 기술에 독립적인 일관성 있는 예외를 던질 수 있음
    - 인터페이스 사용, 런타임 예외 전환과 함께 DataAccessException 추상화를 적용하면 데이터 액세스 기술과 구현 방법에 독립적인 이상적인 DAO 를 만들 수 있음

4.2.4 기술에 독립적인 UserDao 만들기

- 인터페이스 적용

```java
public interface UserDao {
	void add(User user);
	User get(String id);
	List<User> getAll();
	void deleteAll();
	int getCount();
}
```

```java
public class UserDaoJdbc implements UserDao {}
```

- 테스트 보완

```java
public class UserDaoTest {
	@Autowired
	private UserDao dao;
}
```

- UserDao 의 인터페이스와 구현을 분리함으로써 데이터 액세스의 구체적인 기술과 UserDao 의 클라이언트 사이에 DI 가 적용된 모습을 보여줌
- DataAccessException 활용 시 주의사항
    - 스프링을 활용하면 DB 종류나 데이터 액세스 기술에 상관없이 키 값이 중복 이 되는 상황에서는 동일한 예외가 발생하리라고 기대
    - 그러나 DuplicateKeyException은 아직까지는 JDBC를 이용하는 경우에만 발생
    - 만약 DAO에서 사용하는 기술의 종류와 상관없이 동일한 예외를 얻고 싶다면 DuplicatedUserIdException처럼 직접 예의를 정의해두고, 각 DAO의 add() 메소드에 서 좀 더 상세한 예외 전환을 해줄 필요가 있음