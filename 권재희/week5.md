# 5. 서비스 추상화

## 5.1 사용자 레벨 관리 기능 추가

- 사용자 레벨 관리 기능
    - 사용자 레벨 : BASIC, SILVER, GOLD
    - 처음 가입 시 BASIC, 이후 활동에 따라 승급
    - 가입 후 50회 이상 로그인 시 SILVER로 승급
    - SILVER 레벨이자 30번 이상 추천 시 GOLD로 승급
    - 레벨 변경은 주기적, 일괄적
    - 변경 작업 전 조건 충족 시 레벨 변경 없음

### 5.1.1 필드 추가

- `enum Level` 추가
    
    ```jsx
    public enum Level {
        BASIC(1), SILVER(2), GOLD(3);
    
        private final int value;
    
        Level(int value){
            this.value = value;
        }
    
        public int intValue(){ 
            return value;
        }
    
        public static Level valueOf(int value){ 
            switch (value){
                case 1: return BASIC;
                case 2: return SILVER;
                case 3: return GOLD;
                default: throw new AssertionError("Unknown value : " + value);
            }
        }
    }
    ```
    
- User 필드 추가
    - `Level` 필드와 Level 가져오는 `getLevel()`  메소드 추가
    - `Level` 을 파라미터로 포함하는 생성자 추가
- `userMapper`에 level 가져오는 것 추가

### 5.1.2 사용자 수정 기능 추가

- `update()`  메소드 추가
    - interface UserDao에 User를 업그레이드하는 void 메소드 추가
- 사용자 정보 수정용 `update()` 메소드
    
    ```jsx
    public void update(User user) {
            this.jdbcTemplate.update(
                    "update users set name = ?, password = ?, level = ?, login_count = ?, recommend_count = ? where id = ? "
                    , user.getName(), user.getPassword(), user.getLevel().intValue(), user.getLoginCount(), user.getRecommendCount(), user.getId()
            );
        }
    ```
    

### 5.1.3 UserService.upgradeLevels()

- 레벨 변경은 주기적, 일괄적 → 사용자를 모두 가져와 레벨 업그레이드
- UserService
    - UserDao 오브젝트 저장 인스턴스 변수 선언
    - UserDao 수정자 메소드 추가 : DI
- 스프링 설정파일에 userService 빈 추가
- `upgradeLevels()`
    
    ```jsx
    public void upgradeLevels() {
            List<User> users = userDao.getAll();
    
            for (User user : users) {
                Boolean changed = null;     // 레벨의 변화가 있는지 확인하는 플래그
    
                if (user.getLevel() == Level.BASIC && user.getLoginCount() >= 50) {     
                    user.setLevel(Level.SILVER);
                    changed = true;                                                     
                }                                   
                else if (user.getLevel() == Level.SILVER && user.getRecommendCount() >= 30) {
                    user.setLevel(Level.GOLD);
                    changed = true;
                }                                   
                else if (user.getLevel() == Level.GOLD) {
                    changed = false;
                }                                   
                else {
                    changed = false;
                }                                  
    
                if(changed) {
    		            // 플래그 true 일 경우 update() 호출
                    userDao.update(user);
                }                                   
        }
    ```
    

### 5.1.4 UserService.add()

- 처음 가입하는 사용자 레벨 설정
- add 메소드에서 user 레벨이 null 일 경우 BASIC으로 설정
    
    ```jsx
    public void add(User user) {
    		if (user.getLevel() == null) user.setLevel(Level.BASIC);
    		userDao.add(user);
    }
    ```
    

### 5.1.5 코드 개선

- `upgradeLevels()` : 모든 사용자 정보를 가져온 후 한 명씩 switch문을 통해 업그레이드 가능 유무 확인 후 업그레이드 진행
- `nextLevel` 을 통해 다음 레벨 얻기

## 5.2 트랜젝션 서비스 추상화

### 5.2.1 트랜잭션 경계설정

- 커밋 또는 롤백되는 트랜잭션의 경계(시작과 끝) 설정

```jsx
Connection c = dataSource.getConnection();

c.setAutoCommit(false); // 트랜잭션 시작

try {
  PreparedStatement st1 = c.prepareStatement("update users ...");
  st1.executeUpdate();

  PreparedStatement st2 = c.prepareStatement("delete users ...");
  st2.executeUpdate();

  c.commit(); // 커밋
} catch(Exception e) {
  c.rollback(); // 롤백 
}

c.close();
```

- Connection을 사용해 트랜잭션을 시작 / 종료
- `setAutoCommit(flase)`
    - 자동 커밋 : 각 DB 작업이 끝날 때마다 자동으로 커밋, 여러 DB 작업을 하나의 트랜잭션으로 묶을 수 있음
    - false : 트랜잭션 시작 선언

### 5.2.2 트랜잭션 동기화

- `UserService`
- 트랜잭션 동기화 : 트랜잭션 시작을 위한 Connection 오브젝트를 특별한 장소에 보관, 이후 가져다가 사용하는 방식
- JdbcTemplate → 트랜잭션 동기화 방식 이용

```jsx
private DataSource dataSource;

public void setDataSource(DataSource dataSource) {
    this.dataSource = dataSource;
}

public void upgradeLevels() throws Exception {
    TracnsactionSynchronizationManager.initSynchronization();

    Connection c = DataSourceUtils.getConnection(dataSource);
    c.setAutoCommit(false);

    try {
        List<User> users = userDao.getAll();
        for (User user : users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(user);
            }
        }
        c.commit();

    } catch (Exception e) {
        c.rollback();
        throw e;
    } finally {
        DataSourceUtils.releaseConnection(c, dataSource);
        TracnsactionSynchronizationManager.unbindResource(this.dataSource);
        TracnsactionSynchronizationManager.clearSynchronization();
    }
}
```

## 5.3 서비스 추상화와 단일 책임 원칙

### 5.3.1 수직, 수평 계층구조와 의존관계

- UserDao ↔ UserService : 같은 애플리케이션 로직을 내용에 따라 분리, 수평적으로 분리
- 상위 비지니스 로직 ↔ 하위 트랜잭션 기술 : 아예 다른 특성, 수직적 분리
- 단일 책임 원칙

### 5.3.2 단일 책임 원칙

- 트랜잭션 기술 분리 → 단일 책임 원칙 준수