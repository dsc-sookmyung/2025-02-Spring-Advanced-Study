# 4. 예외

## 4.1 사라진 SQLException

### 4.1.1. 초난감 예외처리

- 예외 블랙홀 : try-catch 문으로 Exception을 잡은 후 아무 처리도 하지 않은 경우 → 예외로 인해 기능이 비정상적 동작 or 예상치 못한 다른 문제들
    - 예외를 표시하는 것 ≠ 예외 처리
- 무의미, 무책임

### 4.1.2 예외의 종류와 특징

- Error / Exception과 체크예외 / RuntimeException과 언체크(런타임 예외)
- Error : java.lang.Error 클래스의 서브 클래스들, 시스템에 비정상적인 상황(OutOfMemory, ThreadDeath)이 발생할 경우 사용, 애플리케이션에서는 이런 에러에 대한 처리 x
- Exception과 체크예외 : java.lang.Exception 클래스와 그 서브클래스, 애플리케이션 코드 작업 중 예외 상황 발생 시 사용,
    - Checked Exception : RuntimeException을 상속받지 않은 클래스, CheckedException을 발생시키는 메서드를 사용할 경우 반드시 예외 처리 코드 함께 작성
    - UnChecked Exception : RuntimeException을 상속한 클래스,  예외처리 강제x

### 4.1.3 예외처리 방법

- 예외 복구 / 예외처리 회피 / 예외 전환
- 예외 복구 : 예외 상황 파악, 문제 해결 후 정상 상태로 돌려놓는 것, 다른 작업 흐름으로 유도하는 것도 예외 복구 중 하나
    - Checked Exception들은 예외를 어떤 식으로든 복구할 가능성이 있는 경우에 사용
- 예외처리 회피 : 자신이 담당x, 호출한 쪽으로 던지는 방법, 메소드에 throws 문으로 선언하거나, catch문으로 잡은 후 로그 남기지 않고 예외 던짐
    
    ```cpp
        public static void exceptionEvasion() throws SQLException{
    
            try{
                /// JDBC API
                throw new SQLException();
            }catch(SQLException e){
                throw e;
            }
        }
    ```
    
    - 콜백과 템플릿처럼 긴밀한 관계가 아니라면 자신의 코드에서 발생하는 예외를 던지는 건 무책임한 책임회피일 수 있음. 긴밀한 관계가 아니라면 자신을 사용하는 쪽에서 예외 처리
- 예외 전환 : 다른 예외로 전환
    - 발생한 예외가 그 예외 상황에 대해 적절한 의미를 부여받지 못할 경우
        
        ```java
        public static void requestJoin(User user){
            try{
            	join(user);
            }catch(DuplicateUserIdException e){ // ID 중복이 발생할 수 있구나!
            	// 예외 처리
            }
        }
        
        ```
        
        ```java
        public void add(User user) throws DuplicateUserIdException, SQLException {
        	try {
        					//  JDBC를 이용해 user 정보를 DB에 추가하는 코드
        	catch(SQLException e) {
        		if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
        			throw DuplicateUserIdException();
        		else
        			throw e;   // 그 외의 경우는 SQLException 그대로
        	}
        }
        ```
        
    - 체크 예외를 언체크 예외(런타임 예외)로 바꾸는 경우
        
        ```java
        try{
            OrderHome orderHome = EJBHomeFactory.getInstance().getOrderHome();
            Order order = orderHome.findByPrimaryKey(Integer id);
        } catch(NamingException ne){
        	 throw new EJBException(ne);
        } catch(SQLException se){
        	 throw new EJBException(se);
        } catch(RemoteException re){
        	 throw new EJBException(re);
        }
        ```
        
        - 런타임 예외의 특징 중 하나는 시스템 오류로 판단하고 트랜잭션을 자동으로 롤백 : 코드는 단순히 예외 포장, but 롤백 처리 가능

### 4.1.4 예외처리 전략

- 당장 복구할 수 있는 예외가 아니라면 언체크 예외로
- 체크 예외는 런타임(언체크) 예외로 전환
- SQLException은 대부분 복구 불가능 → DuplicateUserIdException과 같은 런타임 예외로 전환/포장

- 애플리케이션 예외 : 애플리케이션 자체 로직에 의해 의도적으로 발생시키는 예외
    - ex) 잔고보다 많은 금액 출금 시 예외 발생 후 잔고가 부족하다는 메시지 전달

## 4.2 예외 전환

### 4.2.1 JDBC의 한계

- DB를 자유롭게 변경해서 사용 x : 비표준 SQL, 호환성 없는 SQLException의 DB 에러 정보

### 4.2.2 DB 에러 코드 매핑을 통한 전환

- DB 에러 코드 : DB에서 직접 제공 → 버전이 바뀌더라도 일관성 유지
- SQLException은 너무 광범위, 의미x → DB 별 에러 코드를 분류하여 스프링 정의 예외 클래스와 매핑한 에러 코드 매핑정보 테이블을 만들어두고 이를 사용
    - ex) 17003 = invalidResultSetAccessCodes
- SQLException → DacaAccessException(언체크 예외)

### 4.2.3 DAO 인터페이스와 dataAccessException 계층구조

- DataAccessException : 런타임 예외 중 하나, 예외들을 추상화한 것들을 계층구조 형태로 모아놓은 클래스
- DataAccessException 으로 SQLException 전환, 의미가 같은 예외라면 데이터 액세스 기술의 종류(JPA, Hibernate 등)와 상관없이 일관된 예외 발생
- DataAccessException 계층구조
    - 기술에 상관없이 InvalidDataAccessResourceUsageException 타입 예외로 던짐 → 시스템 예외처리, 개발자에게 빠르게 통보
    - 오브젝트/엔티티 단위로 정보 업데이트 시 발생할 수 있는 낙관적 락킹 예외 → 기술에 상관없이 ObjectOptimisticLockingFailureException로 통일
    - DataAccessException 예외 추상화 → 데이터 액세스 기술에 독립적인 DAO 가능

### 4.2.4 기술에 독립적인 UserDao 만들기

- 인터페이스, DataAccessException 활용
- UserDaoJdbc → UserDao 구현

```java
try {
    dao.add(user1);
    dao.add(user1);
}
catch(DuplicateKeyException ex) {
	SQLException sqlEx = (SQLException)ex.getCause();
	SQLExceptionTranslator set = new SQLErrorCodeSQLExceptionTranslator(this.dataSource);	// 코드를 이용한 SQLException의 전환

	DataAccessException transEx = set.translate(null, null, sqlEx);
	assertThat(transEx, is(DuplicateKeyException.class));
}
```