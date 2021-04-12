5장 서비스 추상화
================

5.1 사용자 레벨 관리 기능 추가
------------------------------

비즈니스 기능 추가
 + 레벨 : BASIC ,SIVER, GOLD
 + BASIC(DEFAULT), SIVLER(BASIC에서 50이상), GOLD(SILVER에서 30이상)
 + 주기적으로 레벨 변경이 일어나며 작업전 충족하더라도 레벨 변경 X

5.1.1 필드 추가
---------------
 > Level 이늄
   숫자를 DB에 넣고 해당 숫자에 해당하는 상수 선언
``` java
class User {
  private static final int BASIC = 1;
  private static final int SILVER = 2;
  private static final int GOLD = 3;
  
  int level;
  
  public void setLevel(int level) {
    this.level = level;
  }
}

```

but, 엉뚱한 숫자를 넣을수 있기에 그것을 방지하기 위해 이늄 사용
``` java
public enum Level {
	BASIC(1), SILVER(2), GOLD(3);

	private final int value;
		
	Level(int value) {
		this.value = value;
	}

	public int intValue() {
		return value;
	}
	
	public static Level valueOf(int value) {
		switch(value) {
		case 1: return BASIC;
		case 2: return SILVER;
		case 3: return GOLD;
		default: throw new AssertionError("Unknown value: " + value);
		}
	}
}
```

> user 필드 추가
``` java
public class User {
  ...
	Level level;
	int login;
	int recommend;
	...
	public Level getLevel() {
		return level;
	}

	public void setLevel(Level level) {
		this.level = level;
	}
	...
}
``` 

테스트를 수정하고 테스트를 수행   
But 오타에 의한 에러 발생
SQL의 경우 컴파일 과정에서 에러를 못찾음. 따라서 실행을 해보지 않으면 오류를 찾기 힘듬
그렇기에 빠르게 실행 가능한 포괄적인 테스트를 만들어두어야함

5.1.2 사용자 수정 기능 추가
------------------------------

사용자 정보는 여러번 수정 가능

> 수정 기능 테스트 추가   
> UserDao와 UserDaoJdbc 수정

수정 이후 테스트 수행   
테스트 결과 수정이 되었지만 다른 row도 그대로 남아 있는지 확인 필요   

이때 고려해야할 사항
 + JdbcTemplate의 update()가 돌려주는 리턴의 값을 확인   
   SQL에서는 UPDATE와 DELETE와 같이 테이블 내용의 영향이 있을 경우 변경되는 roew 개수를 반환함   
   이러한 개수를 이용하여 제대로 만들어졌는지 테스트 가능   
 + 테스트 케이스를 추가하여 변경 필요 X한 로우를 직접 확인
 
5.1.3 UserService.upgradeLevels()
------------------------------

UserDaoJdbc는 사용자 관리 로직를 넣기에 적합하지 않음   
 그 이유는 DAO는 데이터를 조작하는 곳이지 비즈니스 로직을 넣는 곳이 아니기에
 
사용자 관리 로직을 담을 클래스 추가 -> UserService   
태스트 클래스 정의 UserServiceTest   

> UserService 클래스와 빈 등록   
> UserServiceTest 테스트 클래스   
> upgradeLevles() 메소드
 
``` java
public void upgradeLevels() {
  List<User> users = userDao.getAll();
  for (User user : users) {
    Boolean changed = null;
    if (user.getLevel() == Level.BASIC && user.getLogin() >= 50) {
      user.setLevel(Level.SILVER);
      changed = true;
    }
    else if (user.getLevel() == Level.SILVER && user.getLogin() >= 30) {
      user.setLevel(Level.GOLD);
      changed = true;
    }
    else if (user.getLevel() == Lvel.GOLD) { changed = false; }
    else { changed = false; }
    if (changed) { userDao.update(user); }
  }
}
```

만들고 테스트 수행 

5.1.4 UserService.add()
-----------------------

로직 위치   
처음 가입 사용자는 BASIC 이여야하는데 해당 로직이 없음   
UserDaoJdbc에 넣기 X -> DB에 정보 넣고 읽는 것에만 관심을 가져야지 비즈니스를 넣기에는 애매함   
User 클래스에서 level 필드를 Level.BASIC으로 초기화 X -> 처음을 제외하면 무의미한 정보임. 따라서 직접 초기화는 애매함   
UserService O   

테스트    
레빌이 미리 있는 경우와 비어있는 경우에 대해 테스트 수행


``` java
public void add(User user) {
  if (user.getLevel() ==  null) user.setLevel(Level.BASIC);
  userDao.add(user);
}
```

5.1.5 코드 개선
-----------------------

 질문   
 + 코드에 중복된 부분은 없는가?   
 + 코드가 무엇을 하는 것인지 이해하기 불편하지 않은가?   
 + 코드가 자신이 있어야 할 자리에 있는가?   
 + 앞으로 변경이 일어안다면 어떤 것이 있을 수 있고, 그 변화에 쉽게 대응할 수 있게 작성되어 있는가?   
 
 > upgradeLevels() 메소드 코드의 문제점
  + for 안의 if/elseif/else 블록들을 읽기 불편  
  + 레벨의 변화 단계와 업그레이드 조건, 조건이 충족됐을 떄 해야할 작업이 섞여 있어 로직 이해 쉽지 않음   
  + 플래그를 두고 이를 마지막에 확인하여 업데이트 하는 방식도 깔끔해 보이지 않음   
  
 > upgradeLevel() 리팩토링
 
  기능을 분리
  
``` java
public void upgradeLevels() {
  List<User> users = userDao.getAll();
  for (User user : users) {
    if (canUpgradeLevel(user)) {
      upgradeLevel(user);
    }
  }
}

private boolean canUpgradeLevel(User user) {
	Level currentLevel = user.getLevel(); 
	switch(currentLevel) {                                   
  	case BASIC: return (user.getLogin() >= 50); 
  	case SILVER: return (user.getRecommend() >= 30);
  	case GOLD: return false;
  	default: throw new IllegalArgumentException("Unknown Level: " + currentLevel); 
	}
}

private void upgradeLevel(User user) {
	if (user.getLevel() == Level.BASIC) user.setLevel(Level.SILVER);
	else if (user.getLevel() == Level.SILVER) user.setLevel(Level.GOLD);
	userDao.update(user)
}
```

upgradeLevels 메소드의 문제점 확인   
 + 다음 단계가 무엇인가 하는 로직과 그때 사용자 오브젝트의 level 필드를 변경해준다는 로직이 함께 있으며 노골적임   
 + 예외 상황에 대한 처리 X   
 + GOLD 레벨 사용자의 경우 아무처리 없이 update만 수행 될 것임   
 + 레벨이 늘어나면 if문이 추가될 것임   
 
 먼저 다음 레벨이 무엇인지 Level에 적시

``` java
public enum Level {
	GOLD(3, null), SILVER(2, GOLD), BASIC(1, SILVER);  
	
	private final int value;
	private final Level next; 
	
	Level(int value, Level next) {  
		this.value = value;
		this.next = next; 
	}
	
	public int intValue() {
		return value;
	}
	
	public Level nextLevel() { 
		return this.next;
	}
	
	public static Level valueOf(int value) {
		switch(value) {
  		case 1: return BASIC;
  		case 2: return SILVER;
  		case 3: return GOLD;
  		default: throw new AssertionError("Unknown value: " + value);
		}
	}
}

```

다음으로 사용자 정보가 바뀌는 부분을 User로 옮김
이유: User 내부 정보가 변경되는 것이니 스스로 다루는게 적절함
``` java
public void upgradeLevel() {
	Level nextLevel = this.level.nextLevel();	
	if (nextLevel == null) { 								
		throw new IllegalStateException(this.level + "은 업그레이드가 불가능합니다");
	}
	else {
		this.level = nextLevel;
	}	
}
```

nextLevel을 통해 다음 레벨을 받아오고 없을때 예외 처리 수행   
이런식으로 독립 메소드를 만들 경우 추후에 기타 정보 변경이 필요시 해당 데이터를 이곳에 추가함으로서 해결 가능      

UserService의 upgradeLevel()이 아래와 같이 간결해짐   
```java
private void upgradeLevel(User user) {
	if (user.getLevel() == Level.BASIC) user.setLevel(Level.SILVER);
	else if (user.getLevel() == Level.SILVER) user.setLevel(Level.GOLD);
	userDao.update(user)
}
```
위와 같이 리팩토링을 함으로써 객체 지향의 기본 원리인 오브젝트에게 데이터를 요구하지 말고 작업을 요청함을 수행

> User 테스트   
> UserServiceTest 개선   
```java
@Test
public void upgradeLevels() {
	userDao.deleteAll();
	for(User user : users) userDao.add(user);
	
	userService.upgradeLevels();
	
	checkLevelUpgraded(users.get(0), false);
	checkLevelUpgraded(users.get(1), true);
	checkLevelUpgraded(users.get(2), false);
	checkLevelUpgraded(users.get(3), true);
	checkLevelUpgraded(users.get(4), false);
}

private void checkLevelUpgraded(User user, boolean upgraded) {
	User userUpdate = userDao.get(user.getId());
	if (upgraded) {
		assertThat(userUpdate.getLevel(), is(user.getLevel().nextLevel()));
	}
	else {
		assertThat(userUpdate.getLevel(), is(user.getLevel()));
	}
}
```
기존의 upgradeLevels() 테스트 코드의 경우 테스트 로직이 분형히 드러나지 않는 단점 존재   
기존 코드에 비해 각 사용자에 대해 업그레이드를 확인하려는 것인지 아닌지 좀 더 이해 할 수 있도록 개선함


코드의 중복 부분을 상수로 변경

```java

public static final int MIN_LOGCOUNT_FOR_SILVER = 50;
public static final int MIN_RECCOMEND_FOR_GOLD = 30;

...
	
private boolean canUpgradeLevel(User user) {
	Level currentLevel = user.getLevel(); 
	switch(currentLevel) {                                   
	case BASIC: return (user.getLogin() >= MIN_LOGCOUNT_FOR_SILVER); 
	case SILVER: return (user.getRecommend() >= MIN_RECCOMEND_FOR_GOLD);
	case GOLD: return false;
	default: throw new IllegalArgumentException("Unknown Level: " + currentLevel); 
	}
}
```

상수를 씀으로써 무슨 의도로 넣었는지 쉽게 파악 가능   

레벨 업그레이드 정책을 유연하게 변경 개선 가능   
예를 들어 연말 이벤트나 평소와 다르게 변경 할 수 있도록 가능   
이런 경우 사용자 업그레이드 정책을 UserService에서 분리하는 방법을 고려해야함   
스프링 설정을 통해 평상시에는 정책을 구현한 클래스를 UserService에서 사용하게 하다가 이벤트 때는 새로 업그레이드 한 정책을 구현한 클래스를 따로 만들어 DI 해주면 됨


5.2 트랜잭션 서비스 추상화
--------------------------

요건 추가: 네크워크 문제나 서버 장애시 모두 변경하지 않고 공지후 다음 날에 업그레이드 진행

5.2.1 모 아니면 도
-----------------

> 테스트용 UserService 대역   

 테스트용으로 특별히 만든 UserService로 진행   
 만드는 법: UserService를 상혹하여 테스트에 필요한 기능을 추가하도록 일부 메소드를 오버라이딩 수행   
 
 테스트를 위해 직접 소스를 수정하는건 권고 하지 않으나 상속하여 사용해야하기에 upgradeLevel()의 메소드 권한을 protected로 변경
 
 오버라이드된 upgradeLevel()의 경우 기존 기능을 수행하데 미리 지정된 id에서만 강제로 예외를 던지도록 수정   
 
> 강제 예외 발생을 통한 테스트   

 테스트 결과, 네번째일 때 사용자 처리중 예외 발생 했지만 두번째 사용자의 경우 레벨이 BASIC에서 SILVER로 변경 됨
 따라서 실패   

> 테스트 실패의 원인     

 원인: 모든 사용자 레벨 업그레이드 작업이 하나의 트랜잭션 안에서 동작하지 않아서   
 
5.2.2 트랜잭션 경계설정
----------------------
 여러 작업 수행시 뒤에 작업에서 에러 발생시 앞의 작업을 되돌리기 위해 트랜잭션 롤백 수행   
 모든 작업이 완벽히 수행시에는 트랜잭션 커밋 수행
 
> JDBC 트랜잭션의 트랜잭션 경계설정   

 작업 시작시 setAutoCommit(false);로 선언하고 commit() 또는 rollback()으로 트랜잭션 종료하는 것을 트랜잭션의 경계설정이라 부름   
 하나의 DB 커넥션 안에서 만들어지는 트랜잭션을 로컬 트랜잭션이라고 부름   
 
> UserService와 UserDao의 트랜잭션 문제     

 DAO 메소드에서 DB 커넥션을 매번 만들기에 위으 에러가 발생   
 결국 DAO 사용시 비즈니스 로직을 담고 있는 UserService 내에서 진행되는 여러 가지 작업을 하나의 트랜잭션으로 묶는게 불가능해짐   
 
> 비즈니스 로직 내의 트랜잭션 경계 설정   

 문제 해결을 위해 DAO 메소드 안으로 upgradeLevels()메소드 기능을 옮기는 생각 고려 X -> 비즈니스 로직과 데이터 로직을 한군데에 묶음   
 따라서 트랜잭션 경계설정 작업을 UserService로 가져오도록 함
```java
	
private void upgradeLevels() throws Exception {
	
	(1) DB connection 생성
	(2) 트랜잭션 시작
	try {
	  (3) DAO 메소드 호출
	  (4) 트랜잭션 커밋
	}
	catch(Exception e) {
	  (5) 트랜잭션 롤백
	  throw e;
	}
	finally {
	  (6) DB connection 종료
	}
}
```

위의 구조를 유지하기 위해 Connection 오브젝트를 직접 넘기고 주고 받고 해야함   

> UserService 트랜잭션 경계설정의 문제점   

 + DB 커넥션을 비롯한 리소스의 깔끔한 처리를 가능하게 했던 JdbcTemplate 를 더 이상 활용 불가   
 + UserService의 모든 메소드에 Connection 파라미터가 추가되어야함   
 + UserDao는 더이상 데이터 액세스 기술에 독립적일 수가 없음   
 + 테스트 코드에 직접 Connection 오브젝트를 일일이 만들어서 DAO 메소드를 호출하도록 변경 해야함   
 
5.2.3 트랜잭션 동기화
----------------------

> Connection 파라미터 제거   
 트랜잭션 동기화란 UserService에서 트랜잭션을 시작하기 위해 만든 Connection 오브젝트를 특별한 저장소에 보관하고, 이후 호출되는 DAO의 메소드에서는 저장된 Connection을 가져다 사용함   
 해당 방식의 경우 작업 스레드마다 독립적으로 Connection 오브젝트를 저장하고 관리하기에 다중 사용자가 처리하는 서버의 멀티스레드 환경에서도 충돌 X
 
> 트랜잭션 동기화 적용
```java

private DataSource dataSource;  			

public void setDataSource(DataSource dataSource) {
	this.dataSource = dataSource;
}

public void upgradeLevels() throws Exception {
	TransactionSynchronizationManager.initSynchronization();  
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
		TransactionSynchronizationManager.unbindResource(this.dataSource);  
		TransactionSynchronizationManager.clearSynchronization();  
	}
}

```
 DB 커넥션을 직접 다룰 떄 DataSource가 필요함으로 DataSource 빈에 대한 DI 설정을 해둬야함   
 스프링이 제공하는 트랜잭션 동기화 클래스인 TransactionSynchronizationManager를 통해 트랜잭션 동기화 작업을 초기화 하도록 함   
 DataSourceUtils에서 제공하는 getConnection() 메소드를 통해 DB 커넥션 생성    
 &nbsp;&nbsp;&nbsp;- getConnection() 사용이유: 오브젝트 생성 + 트랜잭션 동기화에 사용하도록 저장소에 바인딩 수행   
 작업 수행 및 정상처리 Commit, 비정상 rollback 수행
 

 