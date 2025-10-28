# 4장 예외
`JdbcTemplate`이 가진 스프링의 데이터 액세스 기능에 담긴 **예외 처리** 와 관련된 접근 방법을 공부해보자.
<br>

## 사라진 SQLException
`JdbcTemplate`을 적용하기 전과 후의 `deleteAll()` 메소드
```java
public void deltetAll() throws SQLException{    // JDBC API 메소드가 던지기 때문에 필요한 예외
    this.jdbcContext.executeSql("delete from users");

}
/*JdbcTemplate 적용 전*/

public void deleteAll(){
    this.jdbcTemplate.update("delete from users");

}
/*JdbcTemplate 적용 후*/
```
<br>

### 초난감 예외처리
예외를 처리할 때 반드시 지켜야 하는 원칙은 **모든 예외는 적절하게 복구 or 작업 중단 후 운영자나 개발자에게 분명하게 통보** 해야 한다는 것이다.

#### 무의미하고 무책임한 throws
예외 처리가 귀찮은 개발자들의 `throws Exception` 을 기계적으로 붙이는 코드, 모든 예외를 무조건 던져버리는 선언을 모든 메소드에 기계적으로 넣는 모습을 아래 코드에서 확인할 수 있다.
```java
public void method1() throws Exception {
    method2();
    ...
}

public void method2() throws Exception {
    method3();
    ...
}
...
```
<br>

### 예외의 종류와 특징
`throw` 를 통해 발생시킬 수 있는 예외

1. Error
    - `java.lang.Error` 클래스의 서브클래스들
    - 시스템에 뭔가 비정상적인 상황이 발생했을 경우에 사용
    - 애플리케이션 코드에서 잡지 않는다.

2. Exception과 체크 예외
    - 애플리케이션 코드의 작업 중 예외상황이 발생했을 경우에 사용
    - 체크 예외
        1. Exception 클래스의 서브클래스
        2. RuntimeException 클래스를 상속하지 않음
        3. 해당 예외가 발생할 수 잇는 메소드 사용할 경우 반드시 예외 처리 코드 함께 작성해야 한다.
            - 코드를 작성하지 않을 시 컴파일 에러 발생
            - 예외 처리 방법
                1. catch 문으로 잡는 방법
                2. 다시 throws를 정의해서 메소드 밖으로 던지는 방법
    - 언체크 예외
        1. RuntimeException을 상속한 클래스

3. RuntimeException과 언체크/런타임 예외
    - `java.lang.RuntimeException` 클래스를 상속한 예외
        1. 명시적인 예외처리를 강제하지 않아 위에서 언급한 **언체크 예외** 라고 부른다.
    - 프로그램의 오류가 있을 때 발생하도록 의도된 것
        1. 예시: 오브젝트를 할당하지 않은 레퍼런스 변수 사용 시도 ➡️ `NullPointerException`
        2. 예시: 허용되지 않는 값을 사용해서 메소드 호출 ➡️ `IllegalArgumentException`
    - 코드에서 미리 조건을 체크하도록 주의 깊게 만든다면 피할 수 있다.

<br>

### 예외처리 방법
1. 예외상황을 파악하고 문제를 해결해서 정상 상태로 돌려놓기
    - 예외로 인해 기본 작업 흐름이 불가능 ➡️ 다른 파일을 이용하도록 안내해서 예외상황 해결!
    - 사용자에게 예외상황으로 비쳐도 애플리케이션에서는 **정상적으로 설계된 흐름을 따라 진행** 돼야 한다.

재시도를 통해 예외를 복구하는 코드
```java
int maxretry = MAX_RETRY;
while(maxretry --> 0) {
    try {
        ...         // 예외가 발생할 가능성이 있는 시도
        return;     // 작업 성공
    }
    catch(SomeException e) {
        // 로그 출력. 정해진 시간만큼 대기

    }
    finally {
        // 리소스 반납. 정리 작업
    }
}
throw new RetryFailedException();   // 최대 재시도 횟수를 넘기면 직접 예외 발생
```

2. 예외처리를 자신이 담당하지 않고 자신을 호출한 쪽으로 던지기
    - `throws` 문으로 선언해서 예외가 발생하면 알아서 던져지게 하기
    - or `catch` 문으로 일단 예외를 잡고, 로그를 남기고, 다시 예외를 던지기

예외처리 회피 코드 1, 2
```java
// 1번 회피 코드
public void add() throws SQLException {
    // JDBC API
}

// 2번 회피 코드
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

3. 예외 전환

## 예외 전환

예외 전환의 목적
1. 런타임 예외로 포장해서 굳이 필요치 않은 `catch/throws`를 줄여주는 것
2. 로우레벨의 예외를 의미 있고 추상화된 예외로 바꿔서 던져주는 것

### JDBC의 한계

#### 1. 비표준 SQL
대부분의 DB는 표준을 따르지 않는 비표준 문법과 기능을 제공한다. 최적화 기법을 SQL에 적용하거나, 쿼리에 조건을 포함하는 등 다양한 곳에서 비표준 SQL 문장이 들어가며 특정 DB에 종속적인 코드가 되버리고 만다.
➡️ DB의 변경 가능성을 고려해서 유연하게 만들어야 한다면? 걸림돌이 되어버린다.

#### 2. 호환성 없는 SQLException의 DB 에러 정보
DB마다 SQL만 다른 것뿐만 아니라, 에러의 종류와 원인도 제각각이다. 호환성 없는 에러 코드와 표준을 따르지 않는 상태 코드를 가진 `SQLException` 만으로 DB에 독립적이고 유연한 코드를 작성하는 것은 불가능하다.