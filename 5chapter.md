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
But 오타에의한 에러 발생
SQL의 경우 컴파일 과정에서 에러를 못찾음. 따라서 실행을 해보지 않으면 오류를 찾기 힘듬
그렇기에 빠리게 실행 가능한 포괄적인 테스트를 만들어두어야함

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