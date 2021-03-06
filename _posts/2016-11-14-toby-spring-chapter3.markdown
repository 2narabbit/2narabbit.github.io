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

## JDBC 조회 기능의 예외처리
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

이런 코드를 효과적으로 다룰 수 있는 방법은 없을까? 물론 이런 문제를 효과적으로 다룰 수 있는 방법이 있다. 이 문제의 핵심은 변하지 않지만 많은 곳에서 중복되는 코드와 로직에 따라 자꾸 확장되고 자주 변하는 코드를 잘 분리해내는 작업이다. 자세히 살펴보면 1장에서 살펴봤던 것과 비슷한 문제이고 같은 방법으로 접근하면 된다.

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

deleteAll() 메소드에서 변하지 않는 부분이라고 명시한 것이 바로 이 contextMethod()가 된다. deleteAll()의 컨텍스트를 정리해보면 다음과 같다.

* DB 커넥션 가져오기
* PreparedStatement를 만들어줄 외부 기능 호출하기
* 전달받은 PreparedStatement 실행하기
* 예외가 발생하면 이를 다시 메소드 밖으로 던지기
* 모든 경우에 만들어진 PreparedStatement와 Connection을 적절히 닫아주기

두 번째 작업에서 사용하는 PreparedStatement를 만들어주는 외부 기능이 바로 전략 패턴에서 말하는 *전략*이라고 볼 수 있다. 전략 패턴의 구조를 따라 이 기능을 인터페이스로 만들어두고 인터페이스의 메소드를 통해 PreparedStatement 생성 전략을 호출해주면 된다.

PreparedStatement를 만드는 전략의 인터페이스는 컨텍스트가 만들어둔 Connection을 전달받아서, PreparedStatement를 만들고 만들어진 PreparedStatement 오브젝트를 돌려준다.

```
public interface StatementStrategy {
	PreparedStatement makePreparedStatement(Connection c) throws SQLException;
}
```

이 인터페이스를 상속해서 실제 전략, 즉 PreparedStatement를 생성하는 클래스를 만들어보자.

```
public class DeleteAllStatement implements StatementStrategy {
	public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
		PreparedStatement ps = c.prepareStatement("delete from users");
		return ps;
	}
}
```

이것을 contextMethod()에 해당하는 UserDao의 deleteAll() 메소드에서 사용하면 그럭저럭 전략 패턴을 적용했다고 볼 수 있다.

```
public void deleteAll() throws SQLException {
	....
	try {
		c = dataSource.getConnection();

		StatementStrategy strategy = new DeleteAllStatement();
		ps = strategy.makePreparedStatement(c);

		ps.executeUpdate();
	} catch (SQLException e) {
	....
}
```

하지만 전략 패턴은 필요에 따라 컨텍스트는 그대로 유지되면서 전략을 바꿔 쓸 수 있다는 것인데, 이렇게 컨텍스트 안에서 이미 구체적인 전략 클래스인 DeleteAllStatement를 사용하도록 고정되어 있다면 뭔가 이상하다. 컨텍스트가 인터페이스뿐 아니라 특정 구현 클래스를 직접 알고 있다는 건, 전략패턴에도 OCP에도 잘 들어맞는다고 볼 수 없기 때문이다.

### DI 적용을 위한 클라이언트/컨텍스트 분리
전략 패턴에 따르면 Context가 어떤 전략을 사용하게 할 것인가는 Context를 사용하는 앞단의 Client가 결정하는 게 일반적이다. Client가 구체적인 전략의 하나를 선택하고 오브젝트로 만들어서 Context에 전달하는 것이다. Context는 전달받은 그 Strategy 구현 클래스의 오브젝트를 사용한다.

바로 1장에서 처음 UserDao와 ConnectionMaker를 독립시키고 나서 UserDao가 구체적인 ConnectionMaker 구현 클래스를 만들어 사용하는 데 문제가 있다고 판단됐을 때 적용했던 그 방법이다.

결국 이 구조에서 전략 오브젝트 생성과 컨텍스트로의 전달을 담당하는 책임을 분리시킨 것이 바로 ObjectFactory이며, 이를 일반화한 것이 앞에서 살펴봤던 의존관계 주입이었다. 결국 *DI란 이러한 전략 패턴의 장점을 일반적으로 활용할 수 있도록 만든 구조*라고 볼 수 있다.

이 패턴 구조를 코드에 적용해보자. 현재 deleteAll() 메소드에서 다음 코드는 클라이언트에 들어가야 할 코드다. deleteAll()의 나머지 코드는 컨텍스트 코드이므로 분리해야 한다.

```
StatementStrategy strategy = new DeleteAllStatement();
```

컨텍스트에 해당하는 부분은 별도의 메소드로 독립시켜보자.

```
public void jdbcContextWithStatementStrategy(StatementStrategy stmt) throws SQLException {
	Connection c = null;
	PreparedStatement ps = null;

	try {
		c = dataSource.getConnection();

		ps = stmt.makePreparedStatement(c);

		ps.executeUpdate();
	} catch (SQLException e) {
		throw e;
	} finally {
		....
	}
}
```

이 메소드는 컨텍스트의 핵심적인 내용을 잘 담고 있다. 클라이언트로부터 StatementStrategy 타입의 전략 오브젝트를 제공받고 JDBC try/catch/finally 구조로 만들어진 컨텍스트 내에서 작업을 수행한다. 제공받은 전략 오브젝트는 PreparedStatement 생성이 필요한 시점에 호출해서 사용한다. 

다음은 클라이언트에 해당하는 부분을 살펴보자. 컨텍스트를 별도의 메소드로 분리했으니 deleteAll() 메소드가 클라이언트가 된다. deleteAll()은 전략 오브젝트를 만들고 컨텍스트를 호출하는 책임을 지고 있다.

```
public void deleteAll() throws SQLException {
	// 선정한 전략 클래스의 오브젝트 생성
	StatementStrategy st = new DeleteAllStatement();

	// 컨텍스트 호출. 전략 오브젝트 전달
	jdbcContextWithStatementStrategy(st);
}
```

이제 구조로 볼 때 완벽한 전략 패턴의 모습을 갖췄다. 비록 클라이언트와 컨텍스트는 클래스를 분리하진 않았지만, 의존관계와 책임으로 볼 때 이상적인 클라이언트/컨텍스트 관계를 갖고 있다.

아직까지는 이렇게 분리한 것에서 크게 장점이 보이지 않는다. 하지만 지금까지 해온 관심사를 분리하고 유연한 확장관계를 유지하도록 만든 작업은 매우 중요하다. 이 구조가 기반이 돼서 앞으로 진행할 UserDao코드의 본격적인 개선 작업이 가능한 것이다.

> 의존관계 주입은 다양한 형태로 적용할 수 있다. DI의 가장 중요한 개념은 제 3자의 도움을 통해 두 오브젝트 사이의 유연한 관계가 설정되도록 만든다는 것이다. 이 개념만 따른다면 DI를 이루는 오브젝트와 구성요소의 구조나 관계는 다양하게 만들 수 있다.
>
> 일반적으로 DI는 의존관계에 있는 두 개의 오브젝트와 이 관계를 다이나믹하게 설정해주는 오브젝트 팩토리(DI 컨테이너), 그리고 이를 사용하는 클라이언트라는 4개의 오브젝트 사이에서 일어난다. 하지만 때로는 원시적인 전략 패턴 구조를 따라 클라이언트가 오브젝트 팩토리의 책임을 함께 지고 있을 수도 있다. 또는 클라이언트와 전략(의존 오브젝트)이 결합될 수도 있다. 심지어는 클라이언트와 DI 관계에 있는 두 개의 오브젝트가 모두 하나의 클래스 안에 담길 수도 있다.

# JDBC 전략 패턴의 최적화

## 전략 클래스의 추가 정보
이번엔 add() 메소드에도 적용해보자.

```
public class AddStatement implements StatementStrategy {
	public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
		PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?, ?, ?)");

		// 그런데 user는 어디서 가져올까??
		ps.setString(1, user.getId());
		ps.setString(1, user.getName());
		ps.setString(1, user.getPassword());

		return ps;
	}
}
```

이렇게 클래스를 분리하고 나니 컴파일 에러가 난다. deleteAll()과는 달리 add()에서는 PreparedStatement를 만들 때 user라는 부가정보가 필요하기 때문이다.

클라이언트로부터 User 타입 오브젝트를 받을 수 있도록 AddStatement의 생성자를 통해 제공받게 만들자.

```
public class AddStatement implements StatementStrategy {
	User user;

	public AddStatement(User user) {
		this.user = user;
	}

	public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
		....
		ps.setString(1, user.getId());
		ps.setString(1, user.getName());
		ps.setString(1, user.getPassword());
		....
	}
}
```

이제 컴파일 에러는 나지 않을 것이다. 다음은 클라이언트인 UserDao의 add() 메소드에 user 정보를 생성자를 통해 전달해주도록 수정한다.

```
public void add(User user) throws SQLException{
	StatementStrategy st = new AddStatement(user);
	jdbcContextWithStatementStrategy(st);
}
```

이렇게 해서 deleteAll()과 add() 두 군데에서 모두 PreparedStatement를 실행하는 JDBC try/catch/finally 컨텍스트를 공유해서 사용할 수 있게 됐다. 앞으로 비슷한 기능의 DAO 메소드가 필요할 때마다 try/catch/finally 로 범벅된 코드를 만들 필요가 없어졌다.

## 전략과 클라이언트의 동거
현재 만들어진 구조에는 2가지 불만이 있다.

1. DAO 메소드마다 새로운 StatementStrategy 구현 클래스를 만들어야 한다는 점이다. 이렇게 되면 기존 UserDao 때보다 클래스 파일의 개수가 많이 늘어난다. 이래서는 템플릿 메소드 패턴을 적용했을 때 보다 그다지 나을 게 없다.
2. DAO 메소드에서 StatementStrategy에 전달할 User와 같은 부가적인 정보가 있는 경우, 이를 위해 오브젝트를 전달받는 생성자와 이를 저장해둘 인스턴스 변수를 번거롭게 만들어야 한다는 점이다. 

이 두 가지 문제를 해결할 수 있는 방법을 생각해보자.

### 로컬 클래스
클래스 파일이 많아지는 문제는 간단한 해결 방법이 있다. StatementStrategy 전략 클래스를 매번 독립된 파일로 만들지 말고 UserDao 클래스 안에 내부 클래스로 정의해버리는 것이다. DeleteAllStatement나 AddStatement는 UserDao 밖에서는 사용되지 않는다. 특정 메소드에서만 사용되는 것이라면 **로컬 클래스**로 만들 수 있다.

```
public void add(User user) throws SQLException{
	// add() 메소드 내부에 로컬 클래스를 선언한다.
	class AddStatement implements StatementStrategy {
		User user;

		public AddStatement(User user) {
			this.user = user;
		}

		public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
			....
			ps.setString(1, user.getId());
			ps.setString(1, user.getName());
			ps.setString(1, user.getPassword());
			....
		}
	}

	StatementStrategy st = new AddStatement(user);
	jdbcContextWithStatementStrategy(st);
}
```

로컬 클래스는 선언된 메소드 내에서만 사용할 수 있다. AddStatement가 사용될 곳이 add() 메소드뿐이라면, 이렇게 사용하기 전에 바로 정의해서 쓰는 것도 나쁘지 않다. 덕분에 클래스 파일이 하나 줄었고, add() 메소드 안에서 PreparedStatement 생성 로직을 함께 볼 수 있으니 코드를 이해하기도 좋다.

로컬 클래스에는 또 한 가지 장점이 있다. 바로 로컬 클래스는 클래스가 내부 클래스이기 때문에 자신이 선언된 곳의 정보에 접근할 수 있다는 점이다. AddStatement는 User 정보를 필요로 한다. 이를 위해 생성자를 만들어서 add() 메소드에서 이를 전달해주도록 했다. 그런데 이렇게 add() 메소드 내에 AddStatement 클래스를 정의하면 번거롭게 생성자를 통해 User 오브젝트를 전달해줄 필요가 없다.

내부 메소드는 자신이 정의된 메소드의 로컬 변수에 직접 접근할 수 있기 때문이다. 다만, 내부 클래스에서 외부의 변수를 사용할 때는 외부 변수는 반드시 final로 선언해줘야 한다. 

이렇게 내부 클래스의 장점을 이용하면 User 정보를 전달받기 위해 만들었던 생성자와 인스턴스 변수를 제거할 수 있다.

```
public void add(final User user) throws SQLException{
	class AddStatement implements StatementStrategy {
		public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
			....
			// 로컬(내부) 클래스의 코드에서 외부의 메소드 로컬 변수에 직접 접근할 수 있다.
			ps.setString(1, user.getId());
			ps.setString(1, user.getName());
			ps.setString(1, user.getPassword());
			....
		}
	}

	// 생성자 파라미터로 User를 전달하지 않아도 된다.
	StatementStrategy st = new AddStatement();
	jdbcContextWithStatementStrategy(st);
}
```

로컬 클래스로 만들어두니 장점이 많다. 메소드마다 추가해야 했던 클래스 파일을 하나 줄일 수 있다는 것도 장점이고, 내부 클래스의 특징을 이용해 로컬 변수를 바로 가져다 사용할 수 있다는 것도 큰 장점이다.

> 중첩 클래스의 종류
>
> 다른 클래스 내부에 정의되는 클래스를 중첩 클래스라고 한다. 중첩 클래스는 독립적으로 오브젝트로 만들어질 수 있는 **스태틱 클래스**와 자신이 정의된 클래스의 오브젝트 안에서만 만들어질 수 있는 **내부 클래스**로 구분된다.
>
> 내부 클래스는 다시 범위에 따라 세 가지로 구분된다. 멤버 필드처럼 오브젝트 레벨에 정의되는 **멤버 내부 클래스**와 메소드 레벨에 정의되는 **로컬 클래스**, 그리고 이름을 갖지 않는 **익명 내부 클래스**다. 익명 내부 클래스의 범위는 선언된 위치에 따라서 다르다.

### 익명 내부 클래스
한 가지 더 욕심을 내보자. AddStatement 클래스는 add() 메소드에서만 사용할 용도로 만들어졌다. 그렇다면 좀 더 간결하게 클래스 이름도 제거할 수 있다.

> 익명 내부 클래스
>
> 익명 내부 클래스는 이름을 갖지 않는 클래스다. 클래스 선언과 오브젝트 생성이 결합된 형태로 만들어지며, 상속할 클래스나 구현할 인터페이스를 생성자 대신 사용해서 다음과 같은 형태로 만들어 사용한다.
>
> new 인터페이스이름() { 클래스 본문 };
>
> 클래스를 재사용할 필요가 없고, 구현한 인터페이스 타입으로만 사용할 경우에 유용하다.

AddStatement를 익명 내부 클래스로 만들어보자. 이름이 없기 때문에 클래스 자신의 타입을 가질 수 없고, 구현한 인터페이스 타입의 변수에만 저장할 수 있다.

```
StatementStrategy st = new StatementStrategy() {
	public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
		PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?, ?, ?)");

		ps.setString(1, user.getId());
		ps.setString(1, user.getName());
		ps.setString(1, user.getPassword());

		return ps;
	}
}
```

만들어진 익명 내부 클래스의 오브젝트는 딱 한 번만 사용할 테니 굳이 변수에 담아두지 말고 jdbcContextWithStatementStrategy() 메소드의 파라미터에서 바로 생성하는 편이 낫다.

```
public void add(final User user) throws SQLException {
	jdbcContextWithStatementStrategy(
		new StatementStrategy() {
			public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
				PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?, ?, ?)");

				ps.setString(1, user.getId());
				ps.setString(1, user.getName());
				ps.setString(1, user.getPassword());

				return ps;
			}
	);
}
```

# 컨텍스트와 DI

## JdbcContext의 분리

JDBC의 일반적인 작업 흐름을 담고 있는 jdbcContextWithStatementStrategy()는 다른 DAO에서도 사용 가능하다. 그러니 jdbcContextWithStatementStrategy()를 UserDao 클래스 밖으로 독립시켜서 모든 DAO가 사용할 수 있게 해보자.

### 클래스 분리

```
public class JbdcContext {
	// DataSource 타입 빈을 DI 받을 수 있게 준비해둔다.
	private DataSource dataSource;

	public void setDataSource(DataSource dataSource) {
		this.dataSource = dataSource;
	}

	public void workWithStatementStrategy(StatementStrategy stmt) throws SQLException {
		Connection c = null;
		PreparedStatement ps = null;

		try {
			c = this.dataSource.getConnection();

			ps = stmt.makePreparedStatement(c);

			ps.executeUpdate();
		} catch (SQLException e) {
			throw e;
		} finally {
			....
		}
	}
}
```

UserDao가 분리된 JdbcContext를 DI 받아서 사용할 수 있게 만든다.

```
public class UserDao {
	....
	
	// jdbcContext를 DI받도록 만든다.
	private JdbcContext jdbcContext;

	public void setJdbcContext(JdbcContext jdbcContext) {
		this.jdbcContext = jdbcContext;
	}

	public void add(final User user) throws SQLException {
		// DI 받은 JdbcContext의 컨텍스트 메소드를 사용하도록 변경한다.
		this.jdbcContext.workWithStatementStrategy(
			new StatementStrategy() { ... }
		);
	}

	....
}
```

### 빈 의존관계 변경

새롭게 작성된 오브젝트 간의 의존관계를 스프링 설정에 적용해보자.

UserDao는 이제 JdbcContext에 의존하고 있다. 그런데 JdbcContext는 인터페이스인 DataSource와는 달리 구체 클래스다. 스프링의 DI는 기본적으로 인터페이스를 사이에 두고 의존 클래스를 바꿔서 사용하도록 하는 게 목적이다. 하지만 이 경우 JdbcContext는 그 자체로 독립적인 JDBC 컨텍스트를 제공해주는 서비스 오브젝트로서 의미가 있을 뿐이고 구현 방법이 바뀔 가능성은 없다. 따라서 인터페이스를 구현하도록 만들지않았고, UserDao와 JdbcContext는 인터페이스를 사이에 두지 않고 DI를 적용하는 특별한 구조가 된다.

스프링의 빈 설정은 클래스 레벨이 아니라 런타임 시에 만들어지는 오브젝트 레벨의 의존관계에 따라 정의된다. 기존에는 userDao 빈이 dataSource 빈을 직접 의존했지만 이제는 jdbcContext 빈이 그 사이에 끼게 된다.

빈 의존관계를 따라서 XML 설정파일을 수정하자.

```
<beans>
	<bean id="userDao" class="....UserDao">
		// UserDao 내에 아직 JdbcContext를 적용하지 않은 메소드가 있어서 제거하지 않았다.
		<property name="dataSource" ref="dataSource" />
		<property name="jdbcContext" ref="jdbcContext" />
	</bean>

	// 추가된 JdbcContext 타입 빈
	<bean id="jdbcContext" class="....JdbcContext">
		<property name="dataSource" ref="dataSource" />
	</bean>

	<bean id="dataSource" class="....SimpleDriverDataSource">
		....
	</bean>
</beans>
```

## JdbcContext의 특별한 DI
JdbcContext를 분리하면서 사용했던 DI 방법에 대해 좀 더 생각해보자. UserDao와 JdbcContext 사이에는 인터페이스를 사용하지 않고 DI를 적용했다. UserDao와 JdbcContext는 클래스 레벨에서 의존관계가 결정된다. 비록 런타임 시에 DI 방식으로 외부에서 오브젝트를 주입해주는 방식을 사용하긴 했지만, 의존 오브젝트의 구현 클래스를 변경할 수는 없다.

### 스프링 빈으로 DI
DI라는 개념을 충실히 따르자면, 인터페이스를 사이에 둬서 클래스 레벨에서는 의존관계가 고정되지 않게 하고, 런타임 시에 의존할 오브젝트와의 관계를 다이나믹하게 주입해주는 것이 맞다. 따라서 *인터페이스를 사용하지 않았다면 엄밀히 말해서 온전한 DI라고 볼 수는 없다.*

인터페이스를 사용해서 클래스를 자유롭게 변경할 수 있게 하지는 않았지만, JdbcContext를 UserDao와 DI 구조로 만들어야 할 이유를 생각해보자.

첫째는 JdbcContext가 스프링 컨테이너의 싱글톤 레지스트리에서 관리되는 싱글톤 빈이 되기 때문이다. JdbcContext는 JDBC 컨텍스트 메소드를 제공해주는 일종의 서비스 오브젝트로서 의미가 있고, 그래서 싱글톤으로 등록돼서 여러 오브젝트에서 공유해 사용되는 것이 이상적이다.

둘째는 JdbcContext가 DI를 통해 다른 빈에 의존하고 있기 때문이다. JdbcContext는 dataSource 프로퍼티를 통해 DataSource 오브젝트를 주입받도록 되어있다. DI를 위해서는 주입되는 오브젝트와 주입받는 오브젝트 양쪽 모두 스프링 빈으로 등록돼야 한다. 따라서 JdbcContext는 다른 빈을 DI 받기 위해서라도 스프링 빈으로 등록돼야 한다.

실제로 스프링에는 드물지만 이렇게 인터페이스를 사용하지 않는 클래스를 직접 의존하는 DI가 등장하는 경우도 있다.

여기서 중요한 것은 인터페이스의 사용 여부다. 왜 인터페이스를 사용하지 않았을까? 인터페이스가 없다는 건 UserDao와 JdbcContext가 매우 긴밀한 관계를 가지고 강하게 결합되어 있다는 의미다. 비록 클래스는 구분되어 있지만 이 둘은 강한 응집도를 가지고 있다. UserDao가 JDBC 방식 대신 JAP나 하이버네이트 같은 ORM을 사용해야 한다면 JdbcContext도 통째로 바뀌어야 한다. JdbcContext는 DataSource와 달리 테스트에서도 다른 구현으로 대체해서 사용할 이유가 없다. 

이런 경우는 굳이 인터페이스를 두지 말고 강력한 결합을 가진 관계를 허용하면서 위에서 말한 두 가지 이유인, 싱글톤으로 만드는 것과 JdbcContext에 대한 DI 필요성을 위해 스프링의 빈으로 등록해서 UserDao에 DI되도록 만들어도 좋다.

단, 이런 클래스를 바로 사용하는 코드 구성을 DI에 적용하는 것은 가장 마지막 단계에서 고려해볼 사항임을 잊지 말자. 그저 인터페이스를 만들기가 귀찮으니까 그냥 클래스를 사용하자는 건 잘못된 생각이다. 

### 코드를 이용하는 수동 DI
JdbcContext를 스프링의 빈으로 등록해서 UserDao에 DI하는 대신 사용할 수 있는 방법이 있다. UserDao 내부에서 직접 DI를 적용하는 방법이다. 이 방법을 쓰려면 JdbcContext를 싱글톤으로 만들려는 것은 포기해야 한다.

JdbcContext를 스프링 빈으로 등록하지 않았으므로 다른 누군가가 JdbcContext의 생성과 초기화를 책임져야 한다. JdbcContext의 제어권은 UserDao가 갖는 것이 적당하다. 

JdbcContext는 다른 빈을 인터페이스를 통해 간접적으로 의존하고 있다. 다른 빈을 의존하고 있다면, 의존 오브젝트를 DI를 통해 제공받기 위해서라도 자신도 빈으로 등록돼야 한다고 했다. 그렇다면 UserDao에서 JdbcContext를 직접 생성해서 사용하는 경우에는 어떠게 해야 할까? JdbcContext가 스프링의 빈이 아니니 DI 컨테이너를 통해 DI 받을 수는 없다. UserDao에게 DI까지 맡겨버리자. JdbcContext에 주입해줄 의존 오브젝트인 DataSource는 UserDao가 대신 DI 받도록 하면 된다.

먼저 설정파일에 등록했던 JdbcContext 빈을 제거한다. UserDao의 jdbcContext 프로퍼티도 제거한다. 그리고 UserDao는 DataSource 프로퍼티만 갖도록한다.

```
<beans>
	<bean id="userDao" class="....UserDao">
		<property name="dataSource" ref="dataSource" />
	</bean>

	<bean id="dataSource" class="....SimpleDriverDataSource">
		....
	</bean>
</beans>
```

설정파일만 보자면 UserDao가 직접 DataSource를 의존하고 있는 것 같지만, 내부적으로는 JdbcContext를 통해 간접적으로 DataSource를 사용하고 있을 뿐이다.

UserDao는 이제 JdbcContext를 외부에서 주입받을 필요가 없으니 setJdbcContext()는 제거한다. 그리고 setDataSource()를 다음과 같이 수정한다.

```
public class UserDao {
	....
	private JdbcContext jdbcContext;

	// 수정자 메소드이면서 JdbcContext에 대한 생성, DI 작업을 동시에 수행
	public void setDataSource(DataSource dataSource) {
		// JdbcContext 생성(IoC)
		this.jdbcContext = new JdbcContext();

		// 의존 오브젝트 주입 (DI)
		this.jdbcContext.setDataSource(dataSource);

		// 아직 JdbcContext를 적용하지 않은 메소드를 위해 저장해둔다.
		this.dataSouce = dataSource;
	}
}
```

setDataSource() 메소드는 DI 컨테이너가 DataSource 오브젝트를 주입해줄 때 호출된다. 이때 JdbcContext에 대한 수동 DI작업을 진행하면 된다.

이 방법의 장점은 굳이 인터페이스를 두지 않아도 될 만큼 긴밀한 관계를 갖는 DAO클래스와 JdbcContext를 어색하게 따로 빈으로 분리하지 않고 내부에서 직접 만들어 사용하면서도 다른 오브젝트에 대한 DI를 적용할 수 있다는 점이다. 이렇게 한 오브젝트의 수정자 메소드에서 다른 오브젝트를 초기화하고 코드를 이용해 DI하는 것은 스프링에서도 종종 사용되는 기법이다.


지금까지 JdbcContext와 같이 인터페이스를 사용하지 않고 DAO와 밀접한 관계를 갖는 클래스를 DI에 적용하는 방법 두 가지를 알아봤다. 두 가지 방법 모두 장단점이 있다.

**스프링의 DI를 이용한 방법**

* 장점 : 오브젝트 사이의 실제 의존관계가 설정파일에 명확하게 드러남
* 단점 : DI의 근본적인 원칙에 부합하지 않는 구체적인 클래스와의 관계가 설정에 직접 노출됨

**코드를 이용해 수동으로 DI를 하는 방법**

* 장점 : JdbcContext가 UserDao의 내부에서 만들어지고 사용되면서 그 관계를 외부에는 드러내지 않음. 필요에 따라 내부에서 은밀히 DI를 수행하고 그 전략을 외부에는 감출 수 있음
* 단점 : JdbcContext를 여러 오브젝트가 사용하더라도 싱글톤으로 만들 수 없고, DI 작업을 위한 부가적인 코드가 필요함

일반적으로 어떤 방법이 더 낫다고 말할 수는 없다. 상황에 따라 적절하다고 판단되는 방법을 선택해서 사용하면 된다. 다만 왜 그렇게 선택했는지에 대한 분명한 이유와 근거는 있어야 한다. *분명하게 설명할 자신이 없다면 차라리 인터페이스를 만들어서 평범한 DI 구조로 만드는 게 나을 수도 있다.*

# 템플릿과 콜백
지금까지 UserDao와 StatementStrategy, JdbcContext를 이용해 만든 코드는 일종의 전략 패턴이 적용된 것이라고 볼 수 있다. 복잡하지만 바뀌지 않는 일정한 패턴을 갖는 작업 흐름이 존재하고 그 중 일부분만 자주 바꿔서 사용해야 하는 경우에 적합한 구조다. 전략 패턴의 기본 구조에 익명 내부 클래스를 활용한 방식이다. 이런 방식을 스프링에서는 **템플릿/콜백 패턴**이라고 부른다. 전략 패턴의 컨텍스트를 템플릿이라 부르고, 익명 내부 클래스로 만들어지는 오브젝트를 콜백이라고 부른다.

## 템플릿/콜백의 동작원리
템플릿은 고정된 작업 흐름을 가진 코드를 재사용한다는 의미에서 붙인 이름이다. 콜백은 템플릿 안에서 호출되는 것을 목적으로 만들어진 오브젝트를 말한다.

### 템플릿/콜백의 특징
여러 개의 메소드를 가진 일반적인 인터페이스를 사용할 수 있는 전략 패턴의 전략과 달리 템플릿/콜백 패턴의 콜백은 보통 단일 메소드 인터페이스를 사용한다. 하나의 템플릿에서 여러 가지 종류의 전략을 사용해야 한다면 하나 이상의 콜백 오브젝트를 사용할 수도 있다. 콜백은 일반적으로 하나의 메소드를 가진 인터페이스를 구현한 익명 내부 클래스로 만들어진다고 보면 된다.

콜백 인터페이스의 메소드에는 보통 파라미터가 있다. 이 파라미터는 템플릿의 작업 흐름 중에 만들어지는 컨텍스트 정보를 전달받을 때 사용된다. 다음은 템플릿/콜백 패턴의 일반적인 작업 흐름을 보여준다.

![](http://t1.daumcdn.net/section/oc/465408b42c7e404598597ad893268ca6)

조금 복잡해 보이지만 DI 방식의 전략 패턴 구조라고 생각하고 보면 간단하다. 클라이언트가 템플릿 메소드를 호출하면서 콜백 오브젝트를 전달하는 것은 메소드 레벨에서 일어나는 DI다. 일반적인 DI라면 템플릿에 인스턴스 변수를 만들어두고 사용할 의존 오브젝트를 수정자 메소드로 받아서 사용할 것이다. 반면에 템플릿/콜백 방식에서는 매번 메소드 단위로 사용할 오브젝트를 새롭게 전달 받는다는 것이 특징이다.

콜백 오브젝트가 내부 클래스로서 자신을 생성한 클라이언트 메소드 내의 정보를 직접 참조한다는 것도 템플릿/콜백의 고유한 특징이다. 클라이언트와 콜백이 강하게 결합된다는 면에서도 일반적인 DI와 조금 다르다.

템플릿/콜백 방식은 전략 패턴과 DI의 장점을 익명 내부 클래스 사용 전략과 결합한 독특한 활용법이라고 이해할 수 있다. 

### JdbcContext에 적용된 템플릿/콜백
앞에서 만들었던 UserDao, JdbcContext와 StatementStrategy의 코드에 적용된 템플릿/콜백 패턴을 한 번 살펴보자. 템플릿과 클라이언트가 메소드 단위인 것이 특징이다.

![](http://t1.daumcdn.net/section/oc/bc7a262084cf467dabdbffa75866498b)

## 편리한 콜백의 재활용
템플릿/콜백 방식은 템플릿에 담긴 코드를 여기저기서 반복적으로 사용하는 원시적인 방법에 비해 많은 장점이 있다. 그런데 템플릿/콜백 방식에서 한 가지 아쉬운 점이 있다. DAO 메소드에서 매번 익명 내부 클래스를 사용하기 때문에 상대적으로 코드를 작성하고 읽기가 조금 불편하다는 점이다.

### 콜백의 분리와 재활용
그래서 이번에는 복잡한 익명 내부 클래스의 사용을 최소화할 수 있는 방법을 찾아보자.

```
public void deleteAll() throws SQLException {
	this.jdbcContext.workWithStatementStrategy(
		new StatementStrategy() {
			public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
				return c.prepareStatement("delete from users");    // 변하는 SQL 문장
			}
		}
	);
}
```

StatementStrategy 인터페이스의 makePreparedStatement() 메소드를 구현한 콜백 오브젝트 코드를 살펴보면 그 내용은 간단하다. 고정된 SQL 쿼리 하나를 담아서 PreparedStatement를 만드는 게 전부다. 그렇다면, 언제나 그랬듯이 중복될 가능성이 있는 자주 바뀌지 않는 부분을 분리해보자.

```
public void deleteAll() throws SQLException {
    executeSql("delete from users");    // 변하는 SQL 문장
}

private void executeSql(final String query) throws SQLException {
	this.jdbcContext.workWithStatementStrategy(
		// 변하지 않는 콜백 클래스 정의와 오브젝트 생성
		new StatementStrategy() {
			public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
				return c.prepareStatement(query);
			}
		}
	);
}
```

이렇게 해서 재활용 가능한 콜백을 담은 메소드가 만들어졌다. 이제 모든 고정된 SQL을 실행하는 DAO 메소드는 deleteAll() 메소드처럼 executeSql()을 호출하는 한 줄이면 끝이다. 복잡한 익명 내부 클래스인 콜백을 직접 만들 필요조차 없어졌다.

### 콜백과 템플릿의 결합
executeSql() 메소드는 UserDao만 사용하기는 아깝다. 이렇게 재사용 가능한 콜백을 담고있는 메소드라면 DAO가 공유할 수 있는 템플릿 클래스 안으로 옮겨도 된다. 엄밀히 말해서 템플릿은 JdbcContext 클래스가 아니라 workWithStatementStrategy() 메소드이므로 JdbcContext 클래스로 콜백 생성과 템플릿 호출이 담긴 executeSql() 메소드를 옮긴다고 해도 문제 될 것은 없다.

```
public class JdbcContext {
	....
		
	public void executeSql(final String query) throws SQLException {
		workWithStatementStrategy(
			new StatementStrategy() {
				public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
					return c.prepareStatement(query);
				}
			}
		);
	}
}
```

이제 모든 DAO 메소드에서 executeSql() 메소드를 사용할 수 있게 됐다. 익명 내부 클래스의 사용으로 조금 복잡해 보였던 클라이언트 메소드는 이제 깔끔하고 단순해졌다. 결국 JdbcContext 안에 클라이언트와 템플릿, 콜백이 모두 함께 공존하면서 동작하는 구조가 됐다.

![](http://t1.daumcdn.net/section/oc/770a80d4ac1f428daf857fc3413e2202)

일반적으로는 성격이 다른 코드들은 가능한 한 분리하는 편이 낫지만, 이 경우는 반대다. 하나의 목적을 위해 서로 긴밀하게 연관되어 동작하는 응집력이 강한 코드들이기 때문에 한 군데 모여 있는 게 유리하다. 구체적인 구현과 내부의 전략 패턴, 코드에 의한 DI, 익명 내부 클래스 등의 기술은 최대한 감춰두고, 외부에는 꼭 필요한 기능을 제공하는 단순한 메소드만 노출해주는 것이다.

## 템플릿/콜백의 응용
지금까지 살펴본 템플릿/콜백 패턴은 사실 스프링에서만 사용할 수 있다거나 스프링이 제공해주는 독점적인 기술은 아니다. 하지만 스프링만큼 이 패턴을 적극적으로 활용하는 프레임워크는 없다. 스프링의 많은 API나 기능을 살펴보면 템플릿/콜백 패턴을 적용한 경우를 많이 발견할 수 있다.

스프링을 사용하는 개발자라면 당연히 스프링이 제공하는 템플릿/콜백 기능을 잘 사용할 수 있어야 한다. 동시에 템플릿/콜백이 필요한 곳이 있으면 직접 만들어서 사용할 줄도 알아야 한다. 스프링에 내장된 것을 원리도 알지 못한 채로 기계적으로 사용하는 경우와 적용된 패턴을 이해하고 사용하는 경우는 큰 차이가 있다. 그런 면에서 스프링의 기본이 되는 전략 패턴과 DI는 물론이고 템플릿/콜백 패턴도 익숙해지도록 학습할 필요가 있다.

고정된 작업 흐름을 갖고 있으면서 여기저기서 자주 반복되는 코드가 있다면, 중복되는 코드를 분리할 방법을 생각해보는 습관을 기르자. 중복된 코드는 먼저 **메소드로 분리**하는 간단한 시도를 해본다. 그중 일부 작업을 필요에 따라 바꾸어 사용해야 한다면 인터페이스를 사이에 두고 분리해서 **전략 패턴**을 적용하고 DI로 의존관계를 관리하도록 만든다. 그런데 바뀌는 부분이 한 어플리케이션 안에서 동시에 여러 종류가 만들어질 수 있다면 이번엔 **템플릿/콜백 패턴**을 적용하는 것을 고려해볼 수 있다.

### 테스트와 try/catch/finally
가장 전형적인 템플릿/콜백 패턴의 후보는 try/catch/finally 블록을 사용하는 코드다. 간단한 예제를 하나 만들어 보자.

파일을 하나 열어서 모든 라인의 숫자를 더한 합을 돌려주는 코드를 만들어보겠다.

```
public class Calculator {
	public Integer calcSum(String filepath) throws IOException {
		BufferedReader br = new BufferedReader(new FileReader(filepath));
		Integer sum = 0;
		String line = null;

		while((line = br.readLine()) != null) {
			sum += Integer.valueOf(line);
		}

		br.close();
		return sum;
	}
}
```

이 코드는 초난감 DAO와 마찬가지로 calcSum() 메소드도 파일을 읽거나 처리하다가 예외가 발생하면, 파일이 정상적으로 닫히지 않고 메소드를 빠져나가는 문제가 발생한다. 따라서 try/finally 블록을 적용해서 어떤 경우에라도 파일이 열렸으면 반드시 닫아주도록 만들어야 한다.

```
public Integer calcSum(String filepath) throws IOException {
	BufferedReader br = null;
	
	try {
		br = new BufferedReader(new FileReader(filepath));
		Integer sum = 0;
		String line = null;

		while((line = br.readLine()) != null) {
			sum += Integer.valueOf(line);
		}

		br.close();
		return sum;
	} catch (IOException e) {
		System.out.println(e.getMessage());
		throw e;
	} finally {
		if (br != null) {
			try { br.close(); }
			catch(IOException e) { System.out.println(e.getMessage()); }
		}
	}
}
```

### 중복의 제거와 템플릿/콜백 설계
그런데 이번에는 파일에 있는 모든 숫자의 곱을 계산하는 기능을 추가해야 한다는 요구가 발생했다. 

파일을 읽어서 처리하는 비슷한 기능이 새로 필요할 때마다 앞에서 만든 코드를 복사해서 사용할 것인가? 물론 아니어야 한다. 한두 번까지는 어떻게 넘어간다고 해도, 세 번 이상 반복된다면 본격적으로 코드를 개선할 시점이라고 생각해야 한다. 객체지향 언어를 사용하고 객체지향 설계를 통해 코드를 작성하는 개발자의 기본적인 자세다.

템플릿/콜백 패턴을 적용해보자. 먼저 템플릿에 담을 반복되는 작업 흐름은 어떤 것인지 살펴보자. 템플릿이 콜백에게 전달해줄 내부의 정보는 무엇이고, 콜백이 템플릿에게 돌려줄 내용은 무엇인지도 생각해보자. 이번에는 템플릿이 작업을 마친 뒤 클라이언트에게 전달해줘야 할 것도 있을 것이다.

템플릿/콜백을 적용할 때는 **템플릿과 콜백의 경계를 정하고 템플릿이 콜백에게, 콜백이 템플릿에게 각각 전달하는 내용이 무엇인지 파악하는게 가장 중요**하다. 그에 따라 콜백의 인터페이스를 정의해야 하기 때문이다.

가장 쉽게 생각해볼 수 있는 구조는 템플릿이 파일을 열고 각 라인을 읽어올 수 있는 BufferedReader를 만들어서 콜백에게 전달해주고, 콜백이 각 라인을 읽어서 알아서 처리한 후에 최종 결과만 템플릿에게 돌려주는 것이다.

```
public interface BufferedReaderCallback {
	Integer doSomethingWithReader (BufferedReader br) throws IOException;
}
```

이제 템플릿 부분을 메소드로 분리해보자. 

```
// BufferedReaderCallback을 사용하는 템플릿 메소드
public Integer fileReadTemplate(String filepath, BufferedReaderCallback callback) throws IOException {
	BufferedReader br = null;
	
	try {
		br = new BufferedReader(new FileReader(filepath));
		// 콜백 오브젝트 호출
		int ret = callback.doSomethingWithReader(br);
		return ret;
	} catch (IOException e) {
		System.out.println(e.getMessage());
		throw e;
	} finally {
		if (br != null) {
			try { br.close(); }
			catch(IOException e) { System.out.println(e.getMessage()); }
		}
	}
}

// 템플릿/콜백을 적용한 calcSum() 메소드
public Integer calcSum(String filepath) throws IOException {
	BufferedReaderCallback sumCallback = 
		new BufferedReaderCallback() {
			public Integer doSomethingWithReader(BufferedReader br) throws IOException {
				Integer sum = 0;
				String line = null;
				while((line = br.readLine()) != null) {
					sum += Integer.valueOf(line);
				}
				return sum;
			}
		};
	return fileReadTemplate(filepath, sumCallback);
}
```

이제 파일에 있는 숫자의 곱을 구하는 메소드도 이 템플릿/콜백을 이용해 만들면 된다.

```
public Integer calcMultiply(String filepath) throws IOException {
	BufferedReaderCallback multiplyCallback = 
		new BufferedReaderCallback() {
			public Integer doSomethingWithReader(BufferedReader br) throws IOException {
				Integer multiply = 0;
				String line = null;
				while((line = br.readLine()) != null) {
					multiply *= Integer.valueOf(line);
				}
				return multiply;
			}
		};
	return fileReadTemplate(filepath, multiplyCallback);
}
```

### 템플릿/콜백의 재설계
템플릿/콜백 패턴을 적용해서 파일을 읽어 처리하는 코드를 상당히 깔끔하게 정리할 수 있게 되었다.

그런데 위에서 만든 calcSum()과 calcMultiply()에 나오는 두 개의 콜백을 비교해보자. 조금만 살펴봐도 두 개의 코드의 아주 유사함을 알 수 있다. 

템플릿과 콜백을 찾아낼 때는, 변하는 코드의 경계를 찾고 그 경계를 사이에 두고 주고받는 일정한 정보가 있는지 확인하면 된다고 했다. 여기서 바뀌는 코드는 실제로 한줄 뿐이다.

```
sum += Integer.valueOf(line);
==================================
multiply *= Integer.valueOf(line);
```

앞에서 해당 라인으로 전달하는 정보는 처음에 선언한 변수 값인 multiply 또는 sum이다. 이후 해당 라인을 처리하고 다시 외부로 전달되는 것은 multiply 또는 sum과 각 라인의 숫자 값을 가지고 계산한 결과다. 이를 콜백 인터페이스로 정의해보면 다음과 같다.

```
public interface LineCallback {
	Integer doSumethingWithLine(String line, Integer value);
}
```

LineCallback은 파일의 각 라인과 현재까지 계산한 값을 넘겨주도록 되어 있다. 그리고 새로운 계산 결과를 리턴 값을 통해 다시 전달받는다. 이 콜백을 기준으로 코드를 다시 정리해보면 템플릿에 포함되는 작업 흐름은 더 많아지고 콜백은 단순해질 것이다.

```
public Integer lineReadTemplate(String filepath, LineCallback callback, int initVal) throws IOException {
	BufferedReader br = null;
	
	try {
		br = new BufferedReader(new FileReader(filepath));

		// 파일의 각 라인을 루프를 돌면서 가져오는 것도 템플릿이 담당한다.
		Integer res = initVal;
		String line = null;
		while((line = br.readLine()) != null) {
			// 각 라인의 내용을 가지고 계산하는 작업만 콜백에게 맡긴다.
			res = callback.doSomethingWithLine(line, res);
		}
		return res;
	} catch (IOException e) {
		System.out.println(e.getMessage());
		throw e;
	} finally {
		if (br != null) {
			try { br.close(); }
			catch(IOException e) { System.out.println(e.getMessage()); }
		}
	}
}
```

템플릿에 파일의 각 라인을 읽는 작업이 추가됐다. 

이번엔 이렇게 수정한 템플릿을 사용하는 코드를 만들어보자.

```
public Integer calcSum(String filepath) throws IOException {
	LineCallback sumCallback = 
		new LineCallback() {
			public Integer doSomethingWithLine(String line, Integer value) throws IOException {
				return value + Integer.valueOf(line);
			}
		};
	return lineReadTemplate(filepath, sumCallback, 0);
}

public Integer calcMultiply(String filepath) throws IOException {
	LineCallback multiplyCallback = 
		new LineCallback() {
			public Integer doSomethingWithLine(String line, Integer value) throws IOException {
				return value * Integer.valueOf(line);
			}
		};
	return lineReadTemplate(filepath, multiplyCallback, 1);
}
```

여타 로우레벨의 파일 처리 코드가 템플릿으로 분리되고 순수한 계산 로직만 남아 있기 때문에 코드의 관심이 무엇인지 명확하게 보인다. Calculator 클래스와 메소드는 데이터를 가져와 계산한다는 핵심 기능에 충실한 코드만 갖고 있게 됐다.

### 제네릭스를 이용한 콜백 인터페이스
지금까지 사용한 LineCallback과 lineReadTemplate()은 템플릿과 콜백이 만들어내는 결과가 Integer 타입으로 고정되어 있다. 만약 결과의 타입을 다양하게 가져가고 싶다면, 자바 언어에 타입 파라미터라는 개념을 도입한 제네릭스를 이용하면 된다. 

파일의 각 라인에 있는 문자를 모두 연결해서 하나의 스트링으로 돌려주는 기능을 만든다고 생각해보자. 이번에는 템플릿이 리턴하는 타입이 스트링이어야 한다. 콜백의 작업 결과도 스트링이어야 한다. 기존에 만들었던 Integer 타입의 결과만 다루는 콜백과 템플릿을 스트링 타입의 값도 처리할 수 있도록 확장해보자.

```
public interface LineCallback<T> {
	T doSomethingWithLine(String line, T value);
}
```

```
public <T> T lineReadTemplate(String filepath, LineCallback<T> callback, T initVal) throws IOException {
	BufferedReader br = null;
	
	try {
		br = new BufferedReader(new FileReader(filepath));
		T res = initVal;
		String line = null;
		while((line = br.readLine()) != null) {
			res = callback.doSomethingWithLine(line, res);
		}
		return res;
	} catch (IOException e) {
		System.out.println(e.getMessage());
		throw e;
	} finally {
		if (br != null) {
			try { br.close(); }
			catch(IOException e) { System.out.println(e.getMessage()); }
		}
	}
}
```

lineReadTemplate() 메소드는 이제 타입 파라미터 T를 갖는 인터페이스 LineCallback 타입의 오브젝트와 T 타입의 초기값 initVal을 받아서, T 타입의 변수 res를 정의하고, T 타입 파라미터로 선언된 LineCallback의 메소드를 호출해서 처리한 후에 T 타입의 결과를 리턴하는 메소드가 되는 것이다. 이제 LineCallback 콜백과 lineReadTemplate() 템플릿은 파일의 라인을 처리해서 T 타입의 결과를 만들어내는 범용적인 템플릿/콜백이 됐다.

이제 파일의 모든 라인의 내용을 하나의 문자열로 길게 연결하는 기능을 가진 메소드를 추가해보자.

```
public String concatenate(String filepath) throws IOException {
	LineCallback<String> concatenateCallback = 
		new LineCallback<String>() {
			public String doSomethingWithLine(String line, String value) {
				return value + line;
			}
		};
	return lineReadTemplate(filepath, concatenateCallback, "");
}
```

이렇게 범용적으로 만들어진 템플릿/콜백을 이용하면 파일을 라인 단위로 처리하는 다양한 기능을 편리하게 만들 수 있다.

# 스프링의 JdbcTemplate
이번에는 스프링이 제공하는 템플릿/콜백 기술을 살펴보자. 스프링은 JDBC를 이용하는 DAO에서 사용할 수 있도록 준비된 다양한 템플릿과 콜백을 제공한다. 자주 사용되는 패턴을 가진 콜백은 다시 템플릿에 결합시켜서 간단한 메소드 호출만으로 사용이 가능하도록 만들어져 있기 때문에 템플릿/콜백 방식의 기술을 사용하고 있는지 모르고도 쓸 수 있을 정도로 편리하다. 

스프링이 제공하는 JDBC 코드용 기본 템플릿은 JdbcTemplate이다. 지금까지 만들었던 JdbcContext를 스프링의 JdbcTemplate으로 바꿔보자.

현재 UserDao는 DataSource를 DI받아서 JdbcContext에 주입해 템플릿 오브젝트로 만들어서 사용한다. 이제 JdbcContext를 다음과 같이 JdbcTemplate으로 변경하자. JdbcTemplate은 생성자의 파라미터로 DataSource를 주입하면 된다.

```
public class UserDao {
	....
	private JdbcTemplate jdbcTemplate;

	public void setDataSource(DataSource dataSource) {
		this.jdbcTemplate = new JdbcTemplate(dataSource);

		this.dataSource = dataSource;
	}
}
```

이제 템플릿을 사용할 준비가 됐다.

## update()
deleteAll()에 먼저 적용해보자. deleteAll()에 처음 적용했던 콜백은 StatementStrategy 인터페이스의 makePreparedStatement() 메소드다. 이에 대응되는 JdbcTemplate의 콜백은 **PreparedStatementCreator 인터페이스**의 **createPreparedStatement()** 메소드다. 템플릿으로부터 Connection을 제공받아서 PreparedStatement를 만들어 돌려준다는 면에서 구조는 동일하다. PreparedStatementCreator 타입의 콜백을 받아서 사용하는 JdbcTemplate의 템플릿 메소드는 **update()**다.

```
public void deleteAll() {
	this.jdbcTemplate.update(
		new PreparedStatementCreator() {
			public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
				return con.preparedStatement("delete from users");
			}
		}
	);
}
```

앞에서 만들었던 executeSql()은 SQL 문장만 전달하면 미리 준비된 콜백을 만들어서 템플릿을 호출하는 것까지 한 번에 해주는 편리한 메소드였다. JdbcTemplate에도 기능이 비슷한 메소드가 존재한다. 콜백을 받는 update() 메소드와 이름은 동일한데 파라미터로 SQL 문장을 전달한다는 것만 다르다.

```
public void deleteAll() {
	this.jdbcTemplate.update("delete from users");
}
```

JdbcTemplate은 앞에서 구상만 해보고 만들지는 못했던 add() 메소드에 대한 편리한 메소드도 제공된다. 치환자(= ?)를 가진 SQL로 PreparedStatement를 만들고 함께 제공하는 파라미터를 순서대로 바인딩해주는 기능을 가진 update() 메소드를 사용할 수 있다. SQL과 함께 가변인자로 선언된 파라미터를 제공해주면 된다.

```
this.jdbcTemplate.update("insert into users(id, name, password) values(?,?,?)", user.getId(), user.getName(), user.getPassword());
```

## queryForInt()
다음은 아직 템플릿/콜백 방식을 적용하지 않았던 getCount() 메소드에 JdbcTemplate을 적용해보자.

getCount()에서 사용할 수 있는 템플릿은 PreparedStatementCreator 콜백과 ResultSetExtractor 콜백을 파라미터로 받는 query() 메소드다.

콜백이 두 개 등장하는 조금 복잡해 보이는 구조이지만 템플릿/콜백의 동작방식을 잘 생각해보면 어렵지 않게 이해할 수 있다. 첫 번째 PreparedStatementCreator 콜백은 템플릿으로부터 Connection을 받고 PreparedStatement를 돌려준다. 두 번째 ResultSetExtractor는 템플릿으로부터 ResultSet을 받고 거기서 추출한 결과를 돌려준다.

```
public int getCount() {
	return this.jdbcTemplate.query(
				new PreparedStatementCreator() {
					public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
						return con.prepareStatement("select count(*) from users");
					}
				},
				new ResultSetExtractor<Integer>() {
					public Integer extractData(ResultSet rs) throws SQLException, DataAccessException {
						rs.next();
						return rs.getInt(1);
					}
				}
			);
}
```

한 가지 눈여겨볼 것은 ResultSetExtractor는 제네릭스 타입 파라미터를 갖는다는 점이다. ResultSet에서 추출할 수 있는 값의 타입은 다양하기 때문에 타입 파라미터를 사용한 것이다. ResultSetExtractor 콜백에 지정한 타입은 제네릭 메소드에 적용돼서 query() 템플릿의 리턴 타입도 함께 바뀐다.

위 코드는 재사용하기 좋은 구조다. SQL을 가지고 PreparedStatement를 만드는 첫 번째 콜백은 이미 재사용 방법을 알아봤다. 두 번째 콜백도 간단하다. SQL의 실행 결과가 하나의 정수 값이 되는 경우는 자주 볼 수 있다. 클라이언트에서 콜백의 작업을 위해 특별히 제공할 값도 없어서 단순하다. 손쉽게 ResultSetExtractor 콜백을 템플릿 안으로 옮겨 재활용할 수 있다.

JdbcTemplate은 이런 기능을 가진 콜백을 내장하고 있는 queryForInt()라는 편리한 메소드를 제공한다. Integer 타입의 결과를 가져올 수 있는 SQL 문장만 전달해주면 된다.

```
public int getCount() {
	return this.jdbcTemplate.queryForInt("select count(*) from users");
}
```

## queryForObject()
이번엔 get() 메소드에 JdbcTemplate을 적용해보자. get()메소드는 지금까지 만들었던 것 중에서 가장 복잡하다. 일단 SQL은 바인딩이 필요한 치환자를 갖고 있다. 이것까지는 add()에서 사용했던 방법을 적용하면 될 것 같다. 남은 것은 ResultSet에서 getCount()처럼 단순한 값이 아니라 복잡한 User 오브젝트로 만드는 작업이다.

이를 위해, ResultSetExtractor 콜백 대신 RowMapper 콜백을 사용하겠다. ResultSetExtractor는 ResultSet을 한 번 전달받아 알아서 추출작업을 모두 진행하고 최종 결과만 리턴해주면 되는 데 반해, RowMapper는 ResultSet의 로우 하나를 매핑하기 위해 사용되기 때문에 여러 번 호출될 수 있다는 점이다.

기본키 값으로 조회하는 get() 메소드 SQL의 실행 결과는 로우가 하나인 ResultSet이다. ResultSet의 첫 번째 로우에 RowMapper를 적용하도록 만들면 된다. 이번에 사용할 템플릿 메소드는 queryForObject()다.

```
public User get(String id) {
	return this.jdbcTemplate.queryForObject("select * from users where id = ?",
				new Object[] {id},
				new RowMapper<User>() {
					public User mapRow(ResultSet rs, int rowNum) throws SQLException {
						User user = new User();
						user.setId(rs.getString("id"));
						user.setName(rs.getString("name"));
						user.setPassword(rs.getString("password"));
						return user;
					}
				}
			)
}
```

첫 번째 파라미터는 PreparedStatement를 만들기 위한 SQL이고, 두 번째는 여기에 바인딩할 값들이다. update()에서처럼 가변인자를 사용하면 좋겠지만 뒤에 다른 파라미터가 있기 때문에 이 경우엔 가변인자 대신 Object 타입 배열을 사용해야 한다. queryForObject() 내부에서 이 두 가지 파라미터를 사용하는 PreparedStatement 콜백이 만들어질 것이다.

queryForObject()는 SQL을 실행하면 한 개의 로우만 얻을 것이라고 기대한다. 그리고 ResultSet의 next()를 실행해서 첫 번째 로우로 이동시킨 후에 RowMapper 콜백을 호출한다. RowMapper에서는 현재 ResultSet이 가리키고 있는 로우의 내용을 User 오브젝트에 그대로 담아서 리턴해주기만 하면 된다. 

## query()
RowMapper를 좀 더 사용해보자. 현재 등록되어 있는 모든 사용자 정보를 가져오는 getAll() 메소드를 추가한다.

이번에는 JdbcTemplate의 query() 메소드를 사용하겠다. 앞에서 사용한 queryForObject()는 쿼리의 결과가 로우 하나일 때 사용하고, query()는 여러 개의 로우가 결과로 나오는 일반적인 경우에 쓸 수 있다. query()의 리턴 타입은 List<T>다. query()는 제네릭 메소드로 타입은 파라미터로 넘기는 RowMapper<T> 콜백 오브젝트에서 결정된다.

```
public List<User> getAll() {
	return this.jdbcTemplate.query("select * from users order by id",
				new RowMapper<User>() {
					public User mapRow(ResultSet rs, int rowNum) throws SQLException {
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

첫 번째 파라미터에는 실행할 SQL 쿼리를 넣는다. 바인딩할 파라미터가 있다면 두번째 파라미터에 추가할 수도 있다. 없다면 생략할 수 있다. 마지막 파라미터는 RowMapper 콜백이다. query() 템플릿은 SQL을 실행해서 얻은 ResultSet의 모든 로우를 열람하면서 로우마다 RowMapper 콜백을 호출한다. RowMapper는 현재 로우의 내용을 User 타입 오브젝트에 매핑해서 돌려준다. 이렇게 만들어진 User 오브젝트는 템플릿이 미리 준비한 List<User> 컬렉션에 추가된다. 모든 로우에 대한 작업을 마치면 모든 로우에 대한 User 오브젝트를 담고 있는 List<User> 오브젝트가 리턴된다.

