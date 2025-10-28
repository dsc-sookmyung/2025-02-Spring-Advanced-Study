# 4장. 예외

JdbcTemplate을 대표로 하는 스프링의 데이터 엑세스 기능에 담겨있는 예외처리와 관련된 접근방법에 대해 알아보자.

## 4.1 사라진 SQLException

### **4.1.1 초난감 예외처리**

개발자 코드에서 종종 발견되는 초난감 예외처리의 대표선수들을 살펴보자.

- catch블록으로 예외를 잡고서 아무것도 하지 않지 않는 경우

```java
try {
	...
	} catch(SQLException e) {
	}
```

→ 예외 발생을 무시해버리고 계속 진행해버리기 때문에, 발생한 예외로 인해 어떤 기능이 비정상적으로 동작하거나, 메모리나 리소스가 소진되거나, 예상치 못한 다른 문제를 일으킨다.

- 예외가 발생 시 화면에 출력만 하는 경우

```java
} catch (SQLException e) {
	System.out.println(e);
}

} catch (SQLException e) {
	e.printStackTrace();
}
```

→ 화면에 메시지를 출력하는 것은 예외를 **처리**한 것이 아니다.

위의 두 코드는 예외를 흔적도 없이 먹어치우는 예외 블랙홀이다

🌟 예외 처리를 할때, 지켜야 할 핵심은 **모든 예외는 적절하게 복구되든지, 작업을 중단시키고 운영자 또는 개발자에게 분명하게 통보돼야 하는 것**이다.

**무의미하고 무책임한 throws Exception**

예외를 일일이 catch하기도 귀찮고, 별 필요도 없으며 매번 정확하게 예외 이름을 적어서 선언하기도 귀찮으니 아예 throws Exception이라는 모든 예외를 무조건 던져버리는 선언을 모든 메소드에 기계적으로 넣는 짓도 하지 말자. 적절한 처리를 통해 복구될 수 있는 예외상황을 제대로 다룰 수 있는 기회를 박탈하게 된다.

### **4.1.2 예외의 종류와 특징**

자바에서 throw를 통해 발생시킬 수 있는 예외는 크게 세 가지가 있다.

**👁️‍🗨️ Error**

: java.lang.Error 클래스의 서브클래스들. 에러는 시스템에 비정상적인 상황이 발생할 경우에 사용된다.
주로 자바의 VM에서 발생시키는 것이기 때문에 시스템 레벨에서 특별한 작업을 하는 게 아니라면 애플리케이션에서는 이런 에러에 대한 처리는 신경 쓰지 않아도 된다.

**👁️‍🗨️ Exception과 체크예외**

: java.lang.Exception 클래스와 그 서브클래스로 정의되는 예외. 개발자들이 만든 애플리케이션 코드의 작업 중에 예외상황이 발생했을 경우에 사용된다.
Exception 클래스는 체크예외(Exception 클래스의 서브클래스이면서 RuntimeException 클래스를 상속하지 않은 것들)와 언체크 예외(RuntimeException을 상속한 클래스)로 구분된다.
일반적으로 예외는 체크예외를 의미하며, 체크예외(**IOException, SQLException**)를 잡거나 던지지 않으면 컴파일 에러가 발생한다.

**👁️‍🗨️ RuntimeException과 언체크(런타임 예외)**

: java.lang.RuntimeException 클래스를 상속한 예외. 명시적인 예외처리를 강제하지 않는다.
**NullPointerException, IndexOutOfBoundsException** 같이 피할 수 있지만 개발자가 부주의해서 발생할 수 있는 경우에 발생하도록 만든 것이 런타임 예외이기 때문에 굳이 Catch나 thows를 사용하지 않아도 되도록 만든 것이다.

### **4.1.3 예외처리 방법**

**▪️ 예외복구**

: 예외상황을 파악하고 문제를 해결해서 정상 상태로 돌려놓는 것

예를 들어, 사용자가 요청한 파일을 읽으려고 시도했는데 해당 파일이 없다거나 다른 문제가 있어서 읽히지가 않아서 IOException이 발생한 경우, 사용자에게 상황을 알려주고 다른 파일을 이용하도록 안내해서 예외상황을 해결할 수 있다.

네트워크 접속이 원활하지 않아, 원격 DB 서버에 접속하다 실패해서 SQLException이 발생하는 경우에 재시도를 해볼 수 있다.

```java
int maxretry = MAX_RETRY;
while(maxretry - > 0) {
	try {
	  ...
		return;
	}
	catch(SomeException e) {
	}
	finally {
	}
}
throw new RetryFailedException():
```

예외처리 코드를 강제하는 체크 예외들은 예외를 어떤 식으로든 복구할 가능성이 있는 경우에 사용한다.
API를 사용하는 개발자로 하여금 예외상황이 발생할 수 있 음을 인식하도록 도와주고 이에 대한 적절한 처리를 시도해보도록 요구하는 것이다.

**▪️ 예외처리 회피**

: 예외처리를 자신이 담당하지 않고 자신을 호출한 쪽으로 던져버리는 것

```java
public void add() throws SQLException {
	// JDBC API
}
```

```java
public void add() throws SQLException {
	try {
		// JDBC API
	}
	catch(SQLException e) {
		// 로그 출력
		throw e;
	}
}
```

콜백/템플릿처럼 긴밀하게 역할을 분담하고 있는 관계가 아니라면, 자신의 코드에서 발생하는 예외를 그냥 던져버리는 건 무책임한 책임회피일 수 있다.

**▪️ 예외 전환**

: 예외 회피와 비슷하게 예외를 복구해서 정상적인 상태로는 만들 수 없기 때문에 예외를 메소드 밖으로 던지는 것이다. 그러나 예외회피와 달리, 적절한 예외로 전환해서 던진다.

**1️⃣ 내부에서 발생한 예외를 그대로 던지는 것이 그 예외상황에 대한 적절한 의미를 부여해주지 못하는 경우**

예를 들어, 새로운 사용자를 등록하려고 시도했을 때 아이디가 같은 사용자가 있어 SQLException이 발생한 경우에 DAO에서 SQLException의 정보를 해석해서 DuplicateUserIdException 같은 예외로 바꾸어 던져주게 하는 것이다. 의미가 분명한 예외가 던져지면 서비스 계층 오브젝트에는 적절한 복구 작업을 시도할 수가 있다.

비즈니스 로직을 담은 서비스 계층 오브젝트에서 SQLException의 원인을 해석해서 대응하게 하는 것이 아니라, DAO 메소드에서 기술 독립적이며 의미가 분명한 예외로 전환해서 던져주자.

```java
public void add(User user) throws DuplicateUserIdException, SQLException {
	try {
					//  JDBC를 이용해 user 정보를 DB에 추가하는 코드
	catch(SQLException e) {
					// ErrorCode가 MySQL의 "Duplicate Entry(1062)"이면 예외 전환
		if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
			throw DuplicateUserIdException();
		else
			throw e;   // 그 외의 경우는 SQLException 그대로
	}
}
```

→ 전환하는 예외에 i**nitCause() 메소드**나 **새로운 예외의 생성자**를 통해 원래 발생한 예외를 담아 중첩 예외로 만드는 것이 좋다.

2️⃣ **예외처리를 강제하는 체크 예외를 언체크 예외인 런타임 예외로 바꾸는 경우**

**비즈니스 로직으로 볼 때 의미 없는 예외**이거나 **복구가 불가능한 예외**라면 런타임 예외로 포장해서 던져서, 다른 계층의 메소드를 작성할 때 불필요한 throws 선언이 들어가지 않도록 해줘야 한다.

런타임 예외로 포장해서 던져버리고, 예외처리 서비스 등을 이용해 자세한 로그를 남기고, 관리자에게는 메일 등으로 통보해주고, 사용자에게는 친절한 안내 메시지를 보여주는 식으로 처리하는 게 바람직하다.

### 4.1.4 예외처리 전략

자바가 엔터프라이즈 서버환경으로 가면서, 차라리 애플리케이션 차원에서 예외상황을 미리 파악하고 예외가 발생하지 않도록 차단하거나 프로그램의 오류나 외부 환경으로 인해 예외가 발생하는 경우라면 빨리 해당 요청의 작업을 취소하고 서버 관리자나 개발자에게 통보해주는 편이 낫다.

→ 지금은 항상 복구할 수 있는 예외가 아니라면 일단 언체크 예외로 만드는 경향이 있다.

**add() 메소드의 예외처리**

☑️ DuplicatedUserIdException처럼 의미 있는 예외는 add() 메소드를 바로 호출한 오브젝트 대신 더 앞단의 오브젝트에서 다룰 수도 있다
→ 어디에서든 잡아서 처리할 수 있다면 굳이 체크 예외로 만들지 않고 런타임 예외로 만드는 것이 낫다.
☑️ \*\*\*\*SQLException은 대부분 복구 불가능한 예외이므로 잡아봤자 처리할 것도 없고, 결국 throws를 타고 계속 앞으로 전달되다가 애플리케이션 밖으로 던져질 것이다
→ 런타임 예외로 포장해 던져버려서 그 밖의 메소드들이 신경 쓰지 않게 해주는 편이 낫다.

```java
public class DuplicateUserIdException extends RuntimeException {
	public DuplicateUserIdException(Throwable cause) {
		super (cause);
	}
}
```

```java
public void add(User user) throws DuplicateUserIdException {
	try {
					//  JDBC를 이용해 user 정보를 DB에 추가하는 코드
	catch(SQLException e) {
		if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
			throw new DuplicateUserIdException(e);  // 예외 전환
		else
			throw new RuntimeException(e);   // 예외 포장
	}
}
```

**애플리케이션 예외**

시스템 또는 외부의 예외상황이 원인이 아니라 애플리케이션 자체의 로직에 의해 의도적으로 발생시키고, 반드시 catch 해서 무엇인가 조치를 취하도록 요구하는 예외

예를 들어, 잔고 부족과 같은 예외 상황에서는 비즈니스적인 의미를 띤 예외를 던지도록 만드는 것이다. 이때 사용하는 예외는 의도적으로 체크 예외로 만든다. 그래서 개발자가 잊지 않고 잔고 부족처럼 자주 발생 가능한 예외상황에 대한 로직을 구현하도록 강제해주는 게 좋다.

## 4.2 예외 전환

### 4.2.1 JDBC의 한계

JDBC는 자바를 이용해 DB에 접근하는 방법을 추상화된 API 형태로 정의해놓고, 각 DB 업체가 JDBC 표준을 따라 만들어진 드라이버를 제공하게 해주기 때문에 개발자는 DB의 종류에 상관없이 일관된 방법으로 프로그램을 개발할 수 있다.

하지만, DB를 자유롭게 변경해서 사용할 수 있는 유연한 코드를 보장해주지는 못한다.

현실적으로 DB를 자유롭게 바꾸어 사용할 수 있는 DB 프로그램을 작성하는 데는 두 가지 걸림돌이 있다.

**▪️ 비표준 SQL**

JDBC 코드에서 사용하는 SQL은 어느 정도 표준화된 언어이고 몇 가지 표준 규약이 있긴 하지만, 대부분의 DB는 표준을 따르지 않는 비표준 문법과 기능도 제공한다. 해당 DB의 특별한 기능을 사용하거나 최적화된 SQL을 만들 때 유용하기 때문이다.

대용량 데이터 처리의 성능향상을 위한 최적화 기법을 SQL에 적용하거나, 페이징처리를 위한 로우의 개수와 위치 등 특별한 기능을 제공하는 함수를 SQL에 사용하려면 비표준 SQL문장이 만들어지게 된다. 결국, 해당 DAO는 특정 DB에 대해 종속적인 코드가 되고 만다.

**▪️ 호환성 없는 SQLException의 DB 에러정보**

DB마다 SQL만 다른 것이 아니라 에러의 종류와 원인도 제각각이다. 따라서 JDBC는 데이터 처리 중에 발생하는 다양한 예외를 그냥 SQLException 하나에 모두 담아 한가지만 던지도록 설계되어 있다.

예외가 발생한 원인은 SQLEXception 안에 담긴 에러 코드와 SQL 상태정보를 참조해야한다.
DB 벤더가 정의한 고유한 에러 코드를 사용하기 때문에, SQLException의 getErrorCode()로 가져올 수 있는 DB 에러 코드는 DB별로 모두 다르다.

### 4.2.2 DB 에러 코드 매핑을 통한 전환

SQLException에 담긴 SQL 상태 코드는 신뢰할 만하지 못하다. 반면 DB 에러 코드는 DB에서 직접 제공해주는 것이라 버전이 올라가더라도 어느 정도 일관성이 유지되기 때문에, 스프링은 DB별 에러 코드를 분류해서 스프링이 정의한 예외 클래스와 매핑해놓은 에러 코드 매핑정보 테이블을 만들어두고 이를 이용한다.

```html
<bean id="Oracle" class="org.springframework.jdbc.support.SQLErrorCodes">
  <property name="badSqlGrammarCodes">
    <value>900,903,904,917,936,942, 17006</value>
  </property>
  <property name="invalidResultSetAccessCodes">
    <value>17003</value>
  </property>
  <property name="duplicateKeyCodes">
    <value>1</value>
  </property>
  ...</bean
>
```

스프링은 DataAccessException이라는 SQLException을 대체할 수 있는 런타임 예외를 정의하고 있을 뿐 아니라 DataAccessException의 서브클래스로 세분화된 예외클래스들을 정의하고 있다.

JdbcTemplate 안에서 DB별로 미리 준비된 에러 코드와 비교해서 적절한 예외를 던져주기 때문에, DB를 변경하더라도 동일한 예외가 던져지는 것이 보장된다.

### 4.2.3 DAO 인터페이스와 DataAccessException 계층구조

DataAccessException은 JDBC의 SQLException을 전환하는 용도뿐만 아니라, 의미가 같은 예외라면 데이터 액세스 기술의 종류와 상관없이 일관된 예외가 발생하도록 만들어준다.

**▪️ DAO 인터페이스와 구현의 분리**

DAO의 사용 기술과 구현 코드는 전략 패턴과 DI를 통해서 DAO를 사용하는 클라이언트에게 감출 수 있지만, 메소드 선언에 나타나는 예외정보가 문제가 될 수 있다.

만약 JDBC API를 사용하는 UserDao 구현 클래스의 add() 메소드라면 SQLException을 던질 것이다. 인터페이스의 메소드 선언에는 없는 예외를 구현 클래스 메소드의 thows에 넣을 수는 없다. 따라서 인터페이스 메소드도 다음과 같이 선언돼야 한다.

```java
public void add(User user) throws SQLException;
```

이렇게 정의한 인터페이스는 JDBC가 아닌 데이터 액세스 기술로 DAO 구현을 전환하면, 사용할 수 없게 된다.
데이터 액세스 기술의 API는 아래와 같이 자신만의 독자적인 예외를 던지기 때문이다.

```java
public void add(User user) throws PersistentException; // JPA
public void add(User user) throws HibernateException; // Hibernate
public void add(User user) throws JdoException;    // JDO
```

인터페이스로 메소드의 구현은 추상화했지만 구현 기술마다 던지는 예외가 다르기 때문에 메소드의 선언이 달라진다는 문제가 발생한다.

JDO, Hibernate, JPA 등의 기술은 SQLException 같은 체크 예외 대신 런타임 예외를 사용해 throws 선언을 하지않아도 된다.
JDBC를 이용한 DAO에서 모든 SQLException을 런타임 예외로 포장해준다면 단순히
public void add(User user); 으로 선언이 가능하다.

하지만, 중복 키 에러처럼 **비즈니스 로직에서 의미 있게 처리할 수 있는 예외, 애플리케이션 말고 시스템 레벨에서 데이터 액세스 예외**를 의미있게 분류할 필요도 있다.

단지 인터페이스로 추상화하고, 일부 기술에서 발생하는 체크 예외를 런타임 예외로 전환하는 것만으론 불충분하는 것 이다.

**▪️ 데이터 액세스 예외 추상화와 DataAccessException 계층구조**

그래서 스프링은 자바의 다양한 데이터 액세스 기술을 사용할 때 발생하는 예외들을 추상화해 DataAccessException 계층구조 안에 정리해놓았다.

☑️ 스프링이 기술의 종류에(JDBC, JDO, JPA, 하이버네이트 등) 상관없이 데이터 엑세스 기술을 잘못 작성해서 발생하는 예외를 InvalidDataAccessResourceUsageException 타입의 예외로 던져주므로, 시스템 레벨의 예외처리 작업을 통해 개발자한테 빠르게 통보하도록 만들 수 있다.

☑️ JDO, JPA, 하이버네이트처럼 오브젝트/엔티티 단위로 정보를 업데이트하는 경우에 발생할 수 있는 낙관적인 락킹예외에 대해서는 기술에 상관없이 ObjectOptimisticLockingFailureException로 통일시킬 수 있다.

JdbcTemplate과 같이 스프링의 데이터 액세스 지원 기술을 이용해 DAO를 만들면 사용 기술에 독립적인 일관성 있는 예외를 던질 수 있다.
결국 인터페이스 사용, 런타임 예외 전환과 함께 DataAccessException 예외 추상화를 적용하면 데이터 액세스 기술과 구현 방법에 독립적인 이상적인 DAO를 만들 수가 있다.

### 4.2.4 기술에 독립적인 UserDao 만들기

**▪️ 인터페이스 적용**

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
public class UserDaoJdbc implements UserDao {

}
```

**▪️ DataAccessException 활용 시 주의사항**

이렇게 스프링을 활용하면 DB 종류나 데이터 액세스 기술에 상관없이 키 값이 중복 이 되는 상황에서는 동일한 예외가 발생하리라고 기대할 것이다. 하지만 안타깝게도 DuplicateKeyException은 아직까지는 JDBC를 이용하는 경우에만 발생한다.

SQLException에 담긴 DB의 에러 코드를 바로 해석하는 JDBC의 경우와 달리 JPA나 하이버네이트, JDO 등에서는 각 기술이 재정의한 예외를 가져와 스프링이 최종적으로 DataAccessException으로 변환하는데,
DB의 에러 코드와 달리 이런 예외들은 세분화되어 있지 않기 때문이다.

만약 DAO에서 사용하는 기술의 종류와 상관없이 동일한 예외를 얻고 싶다면 DuplicatedUserIdException처럼 직접 예의를 정의해두고, 각 DAO의 add() 메소드에 서 좀 더 상세한 예외 전환을 해줄 필요가 있다.

```java
@Test
	public void sqlExceptionTranslate() {
		dao.deleteAll();

		try {
			dao.add(user1);
			dao.add(user1);
		}
		catch(DuplicateKeyException ex) {
			SQLException sqlEx = (SQLException)ex.getCause();
			SQLExceptionTranslator set = new SQLErrorCodeSQLExceptionTranslator(this.dataSource);	// 코드를 이용한 SQLException의 전환

			DataAccessException transEx = set.translate(null, null, sqlEx); // 에러 메세지 만들 때 사용하는 정보이므로 null로 넣어도 됨
			assertThat(transEx, is(DuplicateKeyException.class));
	}
}
```
