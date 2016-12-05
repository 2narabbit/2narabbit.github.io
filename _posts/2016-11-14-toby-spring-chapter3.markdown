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

*스프링의 DI를 이용한 방법*
* 장점 : 오브젝트 사이의 실제 의존관계가 설정파일에 명확하게 드러남
* 단점 : DI의 근본적인 원칙에 부합하지 않는 구체적인 클래스와의 관계가 설정에 직접 노출됨

*코드를 이용해 수동으로 DI를 하는 방법*
* 장점 : JdbcContext가 UserDao의 내부에서 만들어지고 사용되면서 그 관계를 외부에는 드러내지 않음. 필요에 따라 내부에서 은밀히 DI를 수행하고 그 전략을 외부에는 감출 수 있음
* 단점 : JdbcContext를 여러 오브젝트가 사용하더라도 싱글톤으로 만들 수 없고, DI 작업을 위한 부가적인 코드가 필요함

일반적으로 어떤 방법이 더 낫다고 말할 수는 없다. 상황에 따라 적절하다고 판단되는 방법을 선택해서 사용하면 된다. 다만 왜 그렇게 선택했는지에 대한 분명한 이유와 근거는 있어야 한다. *분명하게 설명할 자신이 없다면 차라리 인터페이스를 만들어서 평범한 DI 구조로 만드는 게 나을 수도 있다.*
