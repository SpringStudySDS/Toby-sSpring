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