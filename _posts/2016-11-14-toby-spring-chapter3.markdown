---
layout: default
title:  "[토비의 스프링] 3장 - 템플릿"
date:   2016-11-14 00:00:00
categories: main
---

# 다시 보는 초난감 DAO
UserDao의 코드에는 아직 문제점이 남아 있다. 바로 예외상황에 대한 처리다.

## 예외처리 기능을 갖춘 DAO

### JDBC 수정 기능의 예외처리 코드

```
public void deleteAll() throws SQLException {
	Connection c = dataSource.getConnection();

	PreparedStatement ps = c.prepareStatement("delete from users");
	ps.executeUpdate();

	ps.close();
	c.close();
}
```

이 메소드에서는 Connection과 PreparedStatement라는 두 개의 공유 리소스를 가져와서 사용한다. 물론 정상적으로 처리되면 메소드를 마치기 전에 각각 close()를 호출해 리소스를 반환한다. 그런데 PreparedStatement를 처리하는 중에 예외가 발생하면 어떻게 될까? 이때는 메소드 실행을 끝마치지 못하고 바로 메소드를 빠져나가게 된다. 이 때 **문제는 Connection과 PreparedStatement의 close() 메소드가 실행되지 않아서 제대로 리소스가 반환되지 않을 수 있다는 점**이다.

그래서 이런 JDBC 코드에서는 어떤 상황에서도 가져온 리소스를 반환하도록 try/catch/finally 구문 사용을 권장하고 있다.

```
public void deleteAll() throws SQLException {
	Connection c = null;
	PreparedStatement ps = null;
	
	try {
		// 예외가 발생할 가능성이 있는 코드를 모두 try 블록으로 묶어준다.
		c = dataSource.getConnection();
		ps = c.prepareStatement("delete from users");
		ps.executeUpdate();
	} catch (SQLException e) {
		throw e;
	} finally {
		if (ps != null) {
			try {
				ps.close();
			} catch (SQLException e) {
				// ps.close() 메소드에서도 SQLException이 발생할 수 있기 때문에 이를 잡아준다.
				// 그렇지 않으면 Connection을 close()하지 못하고 메소드를 빠져나갈 수 있다.
			}
		}

		if (c != null) {
			try {
				c.close();
			} catch (SQLException e) {
			}
		}
	}
}
```

## JDBC 조회 기능의 에외처리
조회를 위한 JDBC 코드는 좀 더 복잡해진다. Connection, PreparedStatement 외에도 ResultSet이 추가되기 때문이다.

```
public void deleteAll() throws SQLException {
	Connection c = null;
	PreparedStatement ps = null;
	ResultSet rs = null;
	
	try {
		c = dataSource.getConnection();

		ps = c.prepareStatement("select count(*) from users");

		// ResultSet도 SQLException이 발생할 수 있는 코드이므로 try 블록 안에 둔다.
		rs = ps.executeQuery();
		rs.next();
		return rs.getInt(1);
	} catch (SQLException e) {
		throw e;
	} finally {
		// 만들어진 ResultSet을 닫아주는 기능
		// close()는 만들어진 순서의 반대로 하는 것이 원칙이다.
		if (rs != null) {
			try {
				rs.close();
			} catch (SQLException e) {
			}
		}

		if (ps != null) {
			try {
				ps.close();
			} catch (SQLException e) {
			}
		}

		if (c != null) {
			try {
				c.close();
			} catch (SQLException e) {
			}
		}
	}
}
```

# 변하는 것과 변하지 않는 것

## JDBC try/catch/finally 코드의 문제점
이제 try/catch/finally 블록도 적용돼서 완성도 높은 DAO 코드가 된 UserDao이지만, 막상 코드를 훑어보면 한숨부터 나온다. 복잡한 try/catch/finally 블록이 2중으로 중첩까지 되어 나오는데다, 모든 메소드마다 반복된다. 

이런 코드를 효과적으로 다룰 수 있는 방법은 없을까? 물론 이런 문제를 효과적으로 다룰 수 있는 방법이 있다. 이 문제의 핵심은 변하지 않는, 그러나 많은 곳에서 중복되는 코드와 로직에 따라 자꾸 확장되고 자주 변하는 코드를 잘 분리해내는 작업이다. 자세히 살펴보면 1장에서 살펴봤던 것과 비슷한 문제이고 같은 방법으로 접근하면 된다.

## 분리와 재사용을 위한 디자인 패턴 적용
UserDao의 메소드를 개선하는 작업을 시작해보자. 가장 먼저 할 일은 변하는 성격이 다른 것을 찾아내는 것이다. 

```
public void deleteAll() throws SQLException {
	Connection c = null;
	PreparedStatement ps = null;
	
	try {
		c = dataSource.getConnection();

		// 변하는 부분. 그외는 변하지 않는 부분
		ps = c.prepareStatement("delete from users");
		
		ps.executeUpdate();
	} catch (SQLException e) {
		throw e;
	} finally {
		if (ps != null) {
			try {
				ps.close();
			} catch (SQLException e) {
			}
		}

		if (c != null) {
			try {
				c.close();
			} catch (SQLException e) {
			}
		}
	}
}
```

비슷한 기능의 메소드에서 동일하게 나타날 수 있는 *변하지 않고 고정되는 부분*과, 각 메소드마다 로직에 따라 *변하는 부분*을 구분해볼 수 있다. 그렇다면 이 로직에 따라서 변하는 부분을 변하지 않는 나머지 코드에서 분리하는 것이 어떨까? 그렇게 할 수 있다면 변하지 않는 부분을 재사용할 수 있는 방법이 있지 않을까?


### 메소드 추출
먼저 생각해볼 수 있는 방법은 변하는 부분을 메소드로 빼는 것이다. 변하지 않는 부분이 변하는 부분을 감싸고 있어서 변하지 않는 부분을 추출하기가 어려워 보이기 때문에 반대로 해봤다.

```
public void deleteAll() throws SQLException {
	....
	try {
		c = dataSource.getConnection();

		// 변하는 부분을 메소드로 추출하고 변하지 않는 부분에서 호출하도록 함
		ps = makeStatement(c);

		ps.executeUpdate();
	} catch (SQLException e) {
	....
}

private PreparedStatement makeStatement(Connection c) throws SQLException {
	PreparedStatement ps;
	ps = c.prepareStatement("delete from users");
	return ps;
}
```

자주 바뀌는 부분을 메소드로 독립시켰는데 당장 봐서는 별 이득이 없어 보인다. 왜냐하면 보통 메소드 추출 리팩토링을 적용하는 경우에는 분리시킨 메소드를 다른 곳에서 재사용할 수 있어야 하는데, 이건 반대로 분리시키고 남은 메소드가 재사용이 필요한 부분이고, 분리된 메소드는 DAO 로직마다 새롭게 만들어서 확장돼야 하는 부분이기 때문이다. 뭔가 반대로 됐다.

### 템플릿 메소드 패턴의 적용
다음은 템플릿 메소드 패턴을 이용해서 분리해보자. 템플릿 메소드 패턴은 상속을 통해 기능을 확장해서 사용하는 부분이다. 변하지 않는 부분은 슈퍼클래스에 두고 변하는 부분은 추상 메소드로 정의해둬서 서브클래스에서 오버라이드하여 새롭게 정의해 쓰도록 하는 것이다.

추출해서 별도의 메소드로 독립시킨 makeStatement() 메소드를 다음과 같이 추상메소드 선언으로 변경한다. 물론 UserDao 클래스도 추상 클래스가 돼야 할 것이다.

```
abstract protected PreparedStatement makeStatement(Connection c) throws SQLException
```

그리고 이를 상속하는 서브클래스를 만들어서 이 메소드를 구현한다. 고정된 JDBC try/catch/finally 블록을 가진 슈퍼클래스 메소드와 필요에 따라서 상속을 통해 구체적인 PreparedStatement를 바꿔서 사용할 수 있게 만드는 서브클래스로 깔끔하게 분리할 수 있다.

```
public class UserDaoDeleteAll extends UserDao {
	protected PreparedStatement makeStatement(Connection c) throws SQLException {
		PreparedStatement ps = c.prepareStatement("delete from users");
		return ps;
	}
}
```

이제 UserDao 클래스의 기능을 확장하고 싶을 때마다 상속을 통해 자유롭게 확장할 수 있고, 확장 때문에 기존의 상위 DAO 클래스에 불필요한 변화는 생기지 않도록 할 수 있으니 OCP를 그럭저럭 지키는 구조를 만들어낼 수는 있는 것 같다. 

하지만 템플릿 메소드 패턴으로의 접근은 제한이 많다. 가장 큰 문제는 DAO 로직마다 상속을 통해 새로운 클래스를 만들어야 한다는 점이다. 만약 이 방식을 사용한다면 UserDao의 JDBC 메소드가 4개일 경우 4개의 서브클래스를 만들어서 사용해야 한다.

또 확장구조가 이미 클래스를 설계하는 시점에서 고정되어 버린다는 점도 문제다. 따라서 그 관계에 대한 유연성이 떨어져 버린다. 상속을 통해 확장을 꾀하는 템플릿 메소드 패턴의 단점이 고스란히 드러난다.

### 전략 패턴의 적용
OCP를 잘 지키는 구조이면서도 템플릿 메소드 패턴보다 유연하고 확장성이 뛰어난 것이, 오브젝트를 아예 둘로 분리하고 클래스 레벨에서는 인터페이스를 통해서만 의존하도록 만드는 전략 패턴이다. 전략 패턴은 OCP 관점에서 보면 확장에 해당하는 변하는 부분을 별도의 클래스로 만들어 추상화된 인터페이스를 통해 위임하는 방식이다.

다음은 전략 패턴의 구조를 나타낸다.

```
---------------------                           ---------------------
| 컨텍스트(context)   |-------------------------->|  전략(strategy)    |
---------------------                           ---------------------
| contextMethod()   |                           | algorithmMethod() |
---------------------                           ---------------------
                                                          ^
                                                          |
                                                   ---------------
                                                   |             |
                                  ---------------------        ---------------------
                                  | ConcreteStrategyA |        | ConcreteStrategyB |
                                  ---------------------        ---------------------
                                  | algorithMethod()  |        | algorithMethod()  |
                                  ---------------------        ---------------------
```

좌측에 있는 Context의 contextMethod()에서 일정한 구조를 가지고 동작하다가 특정 확장 기능은 Strategy 인터페이스를 통해 외부의 독립된 전략 클래스에 위임하는 것이다.
