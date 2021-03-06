---
layout: default
title:  "[토비의 스프링] 1장 - 오브젝트와 의존관계"
date:   2016-11-03 00:00:00
categories: main
---

스프링이 가장 관심을 많이 두는 대상은 **오브젝트**다. 스프링을 이해하려면 먼저 오브젝트에 깊은 관심을 가져야 한다.
1장에서는 스프링이 어떤 것이고, 무엇을 제공하는지보다는 스프링이 관심을 갖는 대상인 오브젝트의 설계와 구현, 동작원리에 더 집중하자.

# 초난감 DAO
사용자 정보를 JDBC API를 통해 DB에 저장하고 조회할 수 있는 간단한 DAO를 하나 구현해보자.

## User
사용자 정보를 저장하기위해 자바빈 규약을 따르는 클래스를 선언하자.

```
public class User {
	String id;
	String name;
	String password;

	public String getId() {
		return id;
	}

	public String getName() {
		return name;
	}

	public String getPassword() {
		return password;
	}

	public void setId(String id) {
		this.id = id;
	}

	public void setName(String name) {
		this.name = name;
	}

	public void setPassword(String password) {
		this.password = password;
	}
}
```

> **자바빈?**
>
> 다음 두가지 관례를 따라 만들어진 오브젝트를 의미하며 간단히 빈이라고 부르기도함
>
> * 디폴트 생성자 
>    * 자바빈은 파라미터가 없는 디폴트 생성자를 가져야함
>    * 툴이나 프레임워크에서 리플렉션을 이용해 오브젝트를 생성하기 때문
> * 프로퍼티
>    * 자바빈이 노출하는 이름을 가진 속성을 의미
>    * 프로퍼티는 set으로 시작하는 수정자 메소드(setter)와 get으로 시작하는 접근자 메소드(getter)를 이용해 수정 또는 조회 가능

## UserDao
사용자 정보를 DB에 넣고 관리할 수 있는 DAO 클래스를 만들어보자.

```
public class UserDao {
	public void add(User user) throws ClassNotFoundException, SQLException {
		// DB 연결을 위한 Connection을 가져온다.
		Class.forName("com.mysql.jdbc.Driver");
		Connection c = DriverManager.getConnection("jdbc:mysql://localhost/test", "mysql", null);

		// SQL을 담은 statement를 만든다.
		PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
		ps.setString(1, user.getId());
		ps.setString(2, user.getName());
		ps.setString(3, user.getPassword());

		// statement를 실행한다.
		ps.executeUpdate();

		// 작업을 마친 뒤 리소스를 정리한다.
		ps.close();
		c.close();
	}

	public User get(String id) throws ClassNotFoundException, SQLException {
		// DB 연결을 위한 Connection을 가져온다.
		Class.forName("com.mysql.jdbc.Driver");
		Connection c = DriverManager.getConnection("jdbc:mysql://localhost/test", "mysql", null);

		// SQL을 담은 statement를 만든다.
		PreparedStatement ps = c.prepareStatement("select * from users where id=?");
		ps.setString(1, id);

		// statement를 실행한다.
		ResultSet rs = ps.executeQuery();
		rs.next();

		// 쿼리 실행 결과를 받아서 오브젝트에 옮겨준다.
		User user = new User();
		user.setId(rs.getString("id"));
		user.setName(rs.getString("name"));
		user.setPassword(rs.getString("password"));

		// 작업을 마친 뒤 리소스를 정리한다.
		rs.close();
		ps.close();
		c.close();

		return user;
	}
}
```

## main()을 이용한 DAO 테스트 코드
만들어진 코드의 기능을 검증하기 위해 테스트 코드를 작성하자.

```
public static void main(String[] args) throws ClassNotFoundException, SQLException {
	UserDao dao = new UserDao();

	User user = new User();
	user.setId("ellie");
	user.setName("스프링");
	user.setPassword("12345678");

	dao.add(user);
	System.out.println(user.getId() + " 등록 성공");

	User user2 = dao.get(user.getId());
	System.out.println(user2.getName());
	System.out.println(user2.getPassword());

	System.out.println(user2.getId() + " 조회 성공");
}
```

이렇게 해서 사용자 정보의 등록과 조회가 되는 초간단 DAO를 구현하였으나 이 코드는 여러가지 문제를 가진 **초난감 DAO** 코드임!

# DAO의 분리
오브젝트에 대한 설계와 이를 구현한 코드는 계속 변한다. 소프트웨어 개발에서 끝이란 개념은 없다ㅠㅠ
그래서 개발자는 미래의 변화를 어떻게 대비할 것인가를 염두하여 객체를 설계하여야 한다. 가장 좋은 대책은 **변화의 폭을 최소한으로 줄여주는 것**이다. 어떻게 변경이 일어날 때 필요한 작업은 최소화하고, 그 변경이 다른 곳에 문제를 일으키지 않게 할 수 있을까?

**모든 변경과 발전은 한 번에 한 가지 관심사항에 집중해서 일어나는데, 문제는 그에 따른 작업은 한 곳에 집중되지 않는 경우가 많다는 것이다.**
변화가 한 번에 한 가지 관심에 집중돼서 일어난다면, 한가지 관심이 한 군데에 집중되게 하자!

## 커넥션 만들기의 추출
UserDao의 구현된 메소드를 자세히 들여다보면 add() 메소드 하나에서만 적어도 세 가지 관심사항을 발견할 수 있다.

1. DB 연결을 위한 커넥션을 어떻게 가져올까
2. SQL 문장을 담을 statement를 만들고 실행
3. 작업이 끝난 뒤 사용한 리소스 처리

가장 먼저 커넥션을 가져오는 중복된 코드를 분리하자.

```
public void add(User user) throws ClassNotFoundException, SQLException {
	Connection c = getConnection();
	....
}

public User get(String id) throws ClassNotFoundException, SQLException {
	Connection c = getConnection();
	....
}

private Connection getConnection() throws ClassNotFoundException, SQLException {
	Class.forName("com.mysql.jdbc.Driver");
	Connection c = DriverManager.getConnection("jdbc:mysql://localhost/test", "mysql", null);
	return c;
}
```

추후 DB연결과 관련된 부분에 변경이 일어났을 경우, getConnection() 메소드 하나만 수정하면 된다. 관심의 종류에 따라 코드를 구분해놓았기 때문에 한 가지 관심에 대한 변경이 일어날 경우 그 관심이 집중되는 부분의 코드만 수정하면 된다.

## 커넥션 만들기의 독립
UserDao 소스코드를 제공하지 않고도 원하는 DB 커넥션 생성 방식을 적용해가면서 UserDao를 사용하게 할 수 있을까? ㅇㅇ UserDao 코드를 한 단계 더 분리하자.

기존에는 같은 클래스에 다른 메소드로 분리했던 DB 커넥션 연결이라는 관심을 이번에는 **상속을 통해 서브클래스로 분리**해보자.

```
public abstract class UserDao {
    public void add(User user) throws ClassNotFoundException, SQLException {
        Connection c = getConnection();
		....
    }

    public User get(String id) throws ClassNotFoundException, SQLException {
        Connection c = getConnection();
		....
    }

    // 구현 코드는 제거되고 추상 메소드로 바뀌었다.
    // 메소드의 구현은 서브클래스가 담당한다.
    public abstract Connection getConnection() throws ClassNotFoundException, SQLException;
}
```

UserDao의 소스코드가 없어도 UserDao 클래스 상속을 통해 getConnection() 메소드를 원하는 방식으로 확장하여 UserDao의 기능을 사용할 수 있게 되었다.

```
pulic class NUserDao extends UserDao {
	// 상속을 통해 확장된 DB 커넥션 생성 메소드
	public Connection getConnection() throws ClassNotFoundException, SQLException {
		....
	}
}
```

클래스 계층구조를 통해 두 개의 관심이 독립적으로 분리되면서 변경 작업은 한층 용이해졌다. 이제는 UserDao의 코드는 한 줄도 수정할 필요 없이 DB연결 기능을 새롭게 정의한 클래스를 만들 수 있다. 이제 UserDao는 단순히 변경이 용이하다는 수준을 넘어서 **손쉽게 확장된다**라고 말할 수 있게 되었다.

이렇게 슈퍼클래스에 기본적인 로직의 흐름을 만들고, 그 기능의 일부를 추상 메소드나 오버라이딩이 가능한 protected 메소드 등으로 만든 뒤 서브클래스에서 이런 메소드를 필요에 맞게 구현해서 사용하도록 하는 방법을 **템플릿 메소드 패턴**이라고 한다.

> **템플릿 메소드 패턴?**
>
> 상속을 통해 슈퍼클래스의 기능을 확장할 때 사용하는 가장 대표적인 방법.
> 변하지 않는 기능은 슈퍼클래스에 만들어두고 자주 변경되며 확장할 기능은 서브클래스에서 만들도록함.

그리고 UserDao의 서브클래스의 getConnection() 메소드는 어떤 Connection 클래스의 오브젝트를 어떻게 생성할 것인지를 결정하는 방법이라고도 볼 수 있다. 이렇게 서브클래스에서 구체적인 오브젝트 생성 방법을 결정하게 하는 것을 **팩토리 메소드 패턴**이라고 부르기도 한다.

> **팩토리 메소드 패턴?**
>
> 템플릿 메소드 패턴과 마찬가지로 상속을 통해 기능을 확장하게 하는 패턴.
>
> 슈퍼클래스는 서브클래스에서 구현할 메소드를 호출해서 필요한 타입의 오브젝트를 가져와 사용하는 방식으로 구현. 이렇게 서브클래스에서 오브젝트 생성 방법과 클래스를 결정할 수 있도록 미리 정의해둔 메소드를 **팩토리 메소드**라고 하고, 이 방식을 통해 오브젝트 생성 방법을 슈퍼클래스의 기본 코드에서 독립시키는 방법을 팩토리 메소드 패턴이라고 한다.
>
> **(주의)**
>
> 자바에서는 오브젝트를 생성하는 기능을 가진 메소드를 일반적으로 **팩토리 메소드**라고 부름. 이 때 말하는 팩토리 메소드와 팩토리 메소드 패턴의 팩토리 메소드는 의미가 다르므로 혼동하지 말아야함.

이렇게 템플릿 메소드 패턴 또는 팩토리 메소드 패턴으로 관심사항이 다른 코드를 분리해내고, 서로 독립적으로 변경 또는 확장할 수 있도록 만드는 것은 간단하면서도 매우 효과적인 방법이다. 하지만 이 방법은 **상속을 사용했다는 단점**이 있다. 상속 자체는 간단하고 사용하기도 편리하지만 많은 한계점이 있다. 

* 자바는 클래스의 다중상속을 허용하지 않는다. 단지 커넥션 객체를 가져오는 방법을 분리하기 위해 상속구조로 만들어버리면, 후에 다른 목적으로 UserDao에 상속을 적용하기 힘들다.
* 또 다른 문제는 상속을 통한 상하위 클래스의 관계는 생각보다 밀접하다는 점이다. 서브클래스는 슈퍼클래스의 기능을 직접 사용할 수 있다. 그래서 슈퍼클래스 내부의 변경이 있을 때 모든 서브클래스를 함께 수정하거나 다시 개발해야 할 수도 있다. 
* 확장된 기능인 DB 커넥션을 생성하는 코드를 다른 DAO 클래스에 적용할 수 없다는 것도 큰 단점이다. 만약 UserDao외에 DAO 클래스들이 계속 만들어진다면 그때는 상속을 통해서 만들어진 getConnection()의 구현 코드가 매 DAO 클래스마다 중복돼서 나타나는 심각한 문제가 발생할 것이다.

# DAO의 확장
지금까지는 성격이 다른, 그래서 *다르게 변할 수 있는 관심사를 분리하는 작업*을 점진적으로 진행해왔다.

* 처음에는 독립된 메소드를 만들어 분리했고,
* 다음에는 상하위 클래스로 분리했다.

이번에는 아예 상속관계도 아닌 완전히 독립적인 클래스로 만들어보자.

## 클래스의 분리
방법은 간단하다. DB 커넥션과 관련된 부분을 서브클래스가 아니라, 아예 별도의 클래스에 담는다. 그리고 이렇게 만든 클래스를 UserDao가 이용하게 한다.

```
// DB 커넥션 생성 기능을 독립시킨 클래스를 선언한다.
public class SimpleConnectionMaker {
	public Connection makeNewConnection() throws ClassNotFoundException, SQLException {
		Class.forName("com.mysql.jdbc.Driver");
		return DriverManager.getConnection("jdbc:mysql://localhost/test", "mysql", null);
	}
}
```

```
public class UserDao {
    private SimpleConnectionMaker simpleConnectionMaker;

    // SimpleConnectionMaker 클래스의 오브젝트를 미리 만들어 두고, 각 메소드에서 사용한다.
    public UserDao() {
        simpleConnectionMaker = new SimpleConnectionMaker();
    }

    public void add(User user) throws ClassNotFoundException, SQLException {
        Connection c = simpleConnectionMaker.makeNewConnection();
		....
    }

    public User get(String id) throws ClassNotFoundException, SQLException {
        Connection c = simpleConnectionMaker.makeNewConnection();
		....
    }
}
```

성격이 다른 코드를 화끈하게 분리하기는 잘한 것 같은데, 다른 문제가 발생했다. UserDao의 코드가 SimpleConnectionMaker라는 특정 클래스에 종속되어 있기 때문에 상속을 사용했을 때처럼 UserDao 코드의 수정 없이 DB 커넥션 생성 기능을 변경할 방법이 없다ㅠㅠ

이런 문제의 근본적인 원인은 UserDao가 바뀔 수 있는 정보, 즉 DB 커넥션을 가져오는 클래스에 대해 너무 많이 알고 있기 때문이다. 어떤 클래스가 쓰일지, 그 클래스에서 커넥션을 가져오는 메소드는 이름이 뭔지까지 일일이 알고 있어야 한다. 따라서 **UserDao는 DB 커넥션을 가져오는 구체적인 방법에 종속**되어 버린다.

## 인터페이스의 도입
클래스를 분리하면서도 이런 문제를 해결할 수는 없을까? ㅇㅇ 가장 좋은 해결책은 두 개의 클래스가 서로 긴밀하게 연결되어 있지 않도록 중간에 추상적인 느슨한 연결고리를 만들어주는 것이다. 추상화란 어떤 것들의 공통적인 성격을 뽑아내어 분리해내는 작업이다. 자바가 추상화를 위해 제공하는 가장 유용한 도구는 바로 인터페이스다.

인터페이스는 자신을 구현한 클래스에 대한 구체적인 정보는 모두 감춰버린다. 결국 오브젝트를 만들려면 구체적인 클래스 하나를 선택해야겠지만 인터페이스로 추상화해놓은 최소한의 통로를 통해 접근하는 쪽에서는 오브젝트를 만들때 사용할 클래스가 무엇인지 몰라도 된다. **인터페이스를 통해 접근하게 하면 실제 구현 클래스를 바꿔도 신경 쓸 일이 없다.**

```
// UserDao가 사용할 인터페이스
public interface ConnectionMaker {
	public Connection makeConnection() throws ClassNotFoundException, SQLException;
}

// 인터페이스 구현체
public class DConnectionMaker implements ConnectionMaker {
	public Connection makeConnection() throws ClassNotFoundException, SQLException {
		// 원하는 방법으로 Connection을 생성하는 코드
	}
	....
}
```

```
public class UserDao {
	// 인터페이스를 통해 오브젝트에 접근하므로 구체적인 클래스 정보를 알 필요가 없다.
	private ConnectionMaker connectionMaker;

	public UserDao() {
		// 앗! 그런데 여기에는 클래스 이름이 나오네!!
		connectionMaker = new DConnectionMaker();
	}

	public void add(User user) throws ClassNotFoundException, SQLException {
		// 인터페이스에 정의된 메소드를 사용하므로 클래스가 바뀐다고 해도 메소드 이름이 변경될 걱정은 없다.
		Connection c = connectionMaker.makeConnection();
	}

	public User get(String id) throws ClassNotFoundException, SQLException {
		// 인터페이스에 정의된 메소드를 사용하므로 클래스가 바뀐다고 해도 메소드 이름이 변경될 걱정은 없다.
		Connection c = connectionMaker.makeConnection();
	}
}
```

UserDao의 다른 모든 곳에서는 인터페이스를 이용하게 만들어서 DB커넥션을 제공하는 클래스에 대한 구체적인 정보는 모두 제거가 가능했지만, 초기에 한 번 어떤 클래스의 오브젝트를 사용할지를 결정하는 생성자 코드는 제거되지 않고 남아있다.

## 관계설정 책임의 분리
두 개의 관심을 인터페이스를 써가면서까지 거의 완벽하게 분리했는데도 왜 문제가 발생하는 것일까? 그 이유는 UserDao 안에 **분리되지 않은 또 다른 관심사항이 존재**하고 있기 때문이다.

```
connectionMaker = new DConnectionMaker();
```

이 코드는 매우 짧고 간단하지만 그 자체로 충분히 독립적인 관심사를 담고 있다. 바로 UserDao가 어떤 ConnectionMaker 구현 클래스의 오브젝트를 이용하게 할지를 결정하는 것이다. 간단히 말하자면 **UserDao와 UserDao가 사용할 ConnectionMaker의 특정 구현 클래스 사이의 관계를 설정해주는 것에 관한 관심**이다.

UserDao를 사용하는 클라이언트가 적어도 하나는 존재할 것이다. 두 개의 오브젝트가 있고 한 오브젝트가 다른 오브젝트의 기능을 사용한다면, 사용되는 쪽이 사용하는 쪽에게 서비스를 제공하는 셈이다. 따라서 **사용되는 오브젝트를 서비스, 사용하는 오브젝트를 클라이언트**라고 부를 수 있다. UserDao의 클라이언트라고 하면 UserDao를 사용하는 오브젝트를 가리킨다. 이 UserDao의 클라이언트 오브젝트가 바로 제 3의 관심사항인 UserDao와 ConnectionMaker 구현 클래스의 관계를 결정해주는 기능을 분리해서 두기에 적절한 곳이다.

클라이언트는 자기가 UserDao를 사용해야 할 입장이기 때문에 UserDao의 세부 전략이라고도 볼 수 있는 ConnectionMaker의 구현 클래스를 선택하고, 선택한 클래스의 오브젝트를 생성해서 UserDao와 연결해줄 수 있다. 기존의 UserDao에서는 생성자에게 이 책임이 있었다. 자, 이제 이 관심을 분리해서 클라이언트에게 떠넘겨보자.

```
// 클라이언트와 같은 제 3의 오브젝트가 UserDao 오브젝트가 사용할 ConnectionMaker 오브젝트를 전달해주도록 만듬
public UserDao(ConnectionMaker connectionMaker) {
	this.connectionMaker = connectionMaker;
}
```

```
public class UserDaoTest() {
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		// UserDao가 사용할 ConnectionMaker 구현 클래스를 결정하고 오브젝트를 만든다.
		ConnectionMaker connectionMaker = new DConnectionMaker();

		// 1. UserDao 생성
		// 2. 사용할 ConnectionMaker 타입의 오브젝트 제공
		//    결국 두 오브젝트 사이의 의존관계 설정 효과
		UserDao dao = new UserDao(connectionMaker);
	}
}
```

> 클래스 사이의 관계와 오브젝트 사이의 관계의 차이를 구분할 수 있어야 한다. **클래스 사이의 관계는 코드에 다른 클래스 이름이 나타나기 때문에 만들어지는 것**이다. 하지만 **오브젝트 사이의 관계**는 모델링 시에는 없었던, 그래서 **코드에는 보이지 않던 관계가 런타임시 오브젝트로 만들어진 후에 생성**되는 것이다. 

인터페이스를 사용하므로써 상속을 통한 확장 방법보다 더 깔끔하고 유연한 방법으로 UserDao와 ConnectionMaker 클래스들을 분리하고, 서로 영향을 주지 않으면서도 필요에 따라 자유롭게 확장할 수 있는 구조가 됐다.

## 원칙과 패턴
지금까지 초난감 DAO 코드를 개선해온 결과를 객체지향 기술의 여러 가지 이론을 통해 설명하려고 한다.

### 개방 폐쇄 원칙
개방 폐쇄 원칙(OCP, Open-Closed Principle)은 깔끔한 설계를 위해 적용 가능한 객체지향 설계 원칙 중의 하나다. 이 원칙을 간단히 정의하자면 **클래스나 모듈은 확장에는 열려 있어야 하고 변경에는 닫혀 있어야한다**라고 할 수 있다.

인터페이스를 통해 수정한 UserDao는 개방폐쇄 원칙을 잘 따르고 있다. 인터페이스를 통해 제공되는 확장 포인트는 확장을 위해 활짝 개방되어 있다. 반면 인터페이스를 이용하는 클래스는 자신의 변화가 불필요하게 일어나지 않도록 굳게 폐쇄되어 있다.

인터페이스를 사용해 확장 기능을 정의한 대부분의 API는 바로 이 개방 폐쇄 원칙을 따른다고 볼 수 있다.

즉, 변경없이 확장 가능한 모습으로 설계하자!

### 높은 응집도와 낮은 결합도 
응집도가 높다는 건 하나의 모듈, 클래스가 하나의 책임 또는 관심사에만 집중되어 있다는 것이다. 또한, 응집도가 높다는 것은 변화가 일어날 때 해당 모듈에서 변하는 부분이 크다는 것으로 설명할 수도 있다. 즉 **변경이 일어날 때 모듈의 많은 부분이 함께 바뀐다면 응집도가 높다**고 말할 수 있다. 만약 모듈의 일부분에만 변경이 일어나도 된다면, 모듈 전체에서 어떤 부분이 바뀌어야 하는지 파악해야 하고, 또 그 변경으로 인해 바뀌지 않는 부분에는 다른 영향을 미치지는 않는지 확인해야 하는 이중의 부담이 생긴다.

낮은 결합도는 높은 응집도보다 더 민감한 원칙이다. 책임과 관심사가 다른 오브젝트 또는 모듈과는 낮은 결합도, 즉 느슨하게 연결된 형태를 유지하는 것이 바람직하다. 느슨한 연결은 관계를 유지하는 데 꼭 필요한 최소한의 방법만 간접적인 형태로 제공하고, 나머지는 서로 독립적이고 알 필요도 없게 만들어주는 것이다(eg. 인터페이스). **결합도가 낮아지면 변화에 대응하는 속도가 높아지고, 구성이 깔끔해진다.**

즉, 관심사가 같은 애들끼리 묶고(=높은 응집도) 관심사가 다른 애들하고는 간접적으로 연결(=낮은 결합도)시키자! 

### 전략 패턴
전략 패턴은 자신의 기능맥락에서 필요에 따라 변경이 필요한 알고리즘을 인터페이스를 통해 통째로 외부로 분리시키고, 이를 구현한 구체적인 알고리즘 클래스를 필요에 따라 바꿔서 사용할 수 있게 하는 디자인 패턴이다. 개방 폐쇄 원칙의 실현에도 가장 잘 들어맞는 패턴이라고 볼 수 있다.

UserDao는 전략 패턴의 컨텍스트에 해당한다. 컨텍스트는 자신의 기능을 수행하는데 필요한 기능 중에서 변경 가능한, DB 연결 방식이라는 알고리즘을 ConnectionMaker라는 인터페이스로 정의하고, 이를 구현한 클래스, 즉 전략을 바꿔가면서 사용할 수 있게 분리했다.

전략 패턴은 클라이언트의 필요성에 대해서도 잘 설명하고 있다. 컨텍스트(UserDao)를 사용하는 클라이언트(UserDaoTest)는 컨텍스트가 사용할 전략(ConnectionMaker를 구현한 클래스)을 컨택스트의 생성자 등을 통해 제공해주는 게 일반적이다.

이렇게 보면 UserDao는 개방 폐쇄 원칙을 잘 따르고 있으며, 응집력이 높고 결합도는 낮으며, 전략 패턴을 적용했음을 알 수 있다. 지금은 이렇게 말하기엔 낯간지러울 만큼 간단한 코드지만, 이 구조가 점점 복잡하게 발전해나가면 이 원칙과 패턴의 장점이 확연하게 드러날 것이다.

# 제어의 역전(IoC)
IoC라는 약자로 많이 사용되는 제어의 역전(Inversion of Control)이라는 용어가 있다. 이 IoC가 무엇인지 살펴보기 위해 UserDao 코드를 좀 더 개선해보겠다.

## 오브젝트 팩토리
문제가 많은 초난감 DAO를 깔끔한 구조로 리팩토링하는 작업을 수행하는 과정에서 얼렁뚱땅 넘긴 게 하나 있다. 바로 클라이언트인 UserDaoTest다. UserDaoTest는 기존에 UserDao가 직접 담당하는 기능을 엉겁결에 떠맡았다. 그런데 원래 UserDaoTest는 UserDao의 기능이 잘 동작하는지를 테스트하려고 만든것이 아닌가? 그런데 지금은 또 다른 책임까지 떠맡고 있으니ㅠㅠ 문제가 있다. 성격이 다른 책임이나 관심사는 분리해버리는 것이 지금까지 해왔던 주요한 작업이다. 그러니 이것도 분리하자!

### 팩토리
분리시킬 기능을 담당할 클래스를 하나 만들어보자. 이 클래스의 역할은 객체의 생성 방법을 결정하고 그렇게 만들어진 오브젝트를 돌려주는 것인데, 이런 일을 하는 오브젝트를 흔히 **팩토리**라고 부른다.

```
// UserDao의 생성 책임을 맡은 팩토리 클래스
public class DaoFactory {
	public UserDao userDao() {
		// 팩토리의 메소드는 UserDao 타입의 오브젝트를 어떻게 만들고, 어떻게 준비시킬지를 결정한다.
		ConnectionMaker connectionMaker = new DConnectionMaker();
		UserDao userDao = new UserDao(connectionMaker);
		return userDao;
	}
}
```

```
// 팩토리를 사용하도록 수정
public class UserDaoTest {
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		UserDao dao = new DaoFactory().userDao();
	}
}
```

### 설계도로서의 팩토리
이렇게 분리된 오브젝트들의 역할과 관계를 분석해보자. UserDao와 ConnectionMaker는 각각 애플리케이션의 핵심적인 데이터 로직과 기술 로직을 담당하고 있고, DaoFactory는 이런 애플리케이션의 오브젝트들을 구성하고 그 관계를 정의하는 책임을 맡고 있다. **전자가 실질적인 로직을 담당하는 컴포넌트라면, 후자는 애플리케이션을 구성하는 컴포넌트의 구조와 관계를 정의한 설계도 같은 역할을 한다고 볼 수 있다.**

## 오브젝트 팩토리의 활용
DaoFactory에 UserDao가 아닌 다른 DAO의 생성 기능을 넣으면 어떻게 될까?

```
public class DaoFactory {
	public UserDao userDao() {
		return new UserDao(new DConnectionMaker());
	}

	public AccountDao accountDao() {
		return new AccountDao(new DConnectionMaker());
	}

	public MessageDao messageDao() {
		return new MessageDao(new DConnectionMaker());
	}
}
```

ConnectionMaker 구현 클래스를 선정하고 생성하는 코드의 중복이 발생했다. 중복 문제를 해결하려면 역시 분리해내는 게 가장 좋은 방법이다.

```
public class DaoFactory {
	public UserDao userDao() {
		return new UserDao(connectionMaker());
	}

	public AccountDao accountDao() {
		return new AccountDao(connectionMaker());
	}

	public MessageDao messageDao() {
		return new MessageDao(connectionMaker());
	}

	public ConnectionMaker connectionMaker() {
		return new DConnectionMaker();
	}
}
```

## 제어권의 이전을 통한 제어관계 역전
이제 제어의 역전이라는 개념에 대해 알아보자.

일반적으로 프로그램의 흐름은 main()메소드와 같이 프로그램이 시작되는 지점에서 다음에 사용할 오브젝트를 결정하고, 결정한 오브젝트를 생성하고, 만들어진 오브젝트에 있는 메소드를 호출하고, 그 오브젝트 메소드 안에서 다음에 사용할 것을 결정하고 호출하는 식의 작업이 반복된다. **모든 오브젝트가 능동적으로 자신이 사용할 클래스를 결정하고, 언제 어떻게 그 오브젝트를 만들지를 스스로 관장한다.** 모든 종류의 작업을 사용하는 쪽에서 제어하는 구조다.

제어의 역전이란 이런 제어 흐름의 개념을 거꾸로 뒤집는 것이다. 제어의 역전에서는 오브젝트가 자신이 사용할 오브젝트를 스스로 선택하지 않는다. **모든 제어 권한을 자신이 아닌 대른 대상에게 위임한다.** 프로그램의 시작을 담당하는 main()과 같은 엔트리 포인트를 제외하면 모든 오브젝트는 이렇게 위임받은 제어 권한을 갖는 특별한 오브젝트에 의해 결정되고 만들어진다.

제어의 역전 개념은 사실 이미 폭넓게 적용되어 있다.

* 서블릿을 생각해보자. 서블릿은 개발해서 서버에 배포할 수는 있지만, 그 실행을 개발자가 직접 제어할 수 있는 방법은 없다. 대신 서블릿에 대한 제어 권한을 가진 컨테이너가 적절한 시점에 서블릿 클래스의 오브젝트를 만들고 그 안의 메소드를 호출한다.

* UserDao 개선 초기에 적용했던 템플릿 메소드 패턴을 생각해보자. 추상 UserDao를 상속한 서브클래스는 getConnection()을 구현한다. 하지만 이 메소드가 언제 어떻게 사용될지 자신은 모른다. 일단 구현해놓으면, 슈퍼클래스인 UserDao가 필요할 때 호출해서 사용하는 것이다. 즉 제어권을 상위 템플릿에 넘기고 자신은 필요할 때 호출되어 사용되도록 한다는, 제어의 역전 개념을 발견할 수 있다.

* 프레임워크도 제어의 역전 개념이 적용된 대표적인 기술이다. 프레임워크는 라이브러리와 다르다. 라이브러리를 사용하는 애플리케이션 코드는 애플리케이션 흐름을 직접 제어한다. 단지 동작하는 중에 필요한 기능이 있을 때 능동적으로 라이브러리를 사용할 뿐이다. 반면에 프레임워크는 거꾸로 애플리케이션 코드가 프레임워크에 의해 사용된다. 보통 프레임워크 위에 개발한 클래스를 등록해두고, 프레임워크가 흐름을 주도하는 중에 개발자가 만든 애플리케이션 코드를 사용하도록 만드는 방식이다.

* 우리가 만든 UserDao와 DaoFactory에도 제어의 역전이 적용되어 있다. 원래 ConnectionMaker의 구현 클래스를 결정하고 오브젝트를 만드는 제어권은 UserDao에게 있었다. 그런데 지금은 DaoFactory에게 있다. 자연스럽게 관심을 분리하고 책임을 나누고 유연하게 확장 가능한 구조로 만들기 위해 DaoFactory를 도입했던 과장이 바로 IoC를 적용하는 작업이었다고 볼 수 있다.

그러고 보니, 우리는 대표적인 IoC 프레임워크라고 불리는 스프링 없이도 IoC 개념을 이미 적용한 셈이다!

제어의 역전에서는 프레임워크 또는 컨테이너와 같이 애플리케이션 컴포넌트의 생성과 관계설정, 사용, 생명주기 관리 등을 관장하는 존재가 필요하다. DaoFactory는 오브젝트 수준의 가장 단순한 IoC 컨테이너 내지는 IoC 프레임워크라고 불릴 수 있다. 단순한 적용이라면 이런 방법으로 충분하겠지만, IoC를 애플리케이션 전반에 걸쳐 본격적으로 적용하려면 스프링과 같은 IoC 프레임워크의 도움을 받는 편이 훨씬 유리하다. **스프링은 IoC를 모든 기능의 기초가 되는 기반기술로 삼고 있으며, IoC를 극한까지 적용하고 있는 프레임워크**다.

# 스프링의 IoC
스프링은 애플리케이션 개발의 다양한 영역과 기술에 관여하며 매우 많은 기능을 제공한다. 하지만 스프링의 핵심을 담당하는 건 바로 **빈 팩토리** 또는 **애플리케이션 컨텍스트**라고 불리는 것이다. 

## 오브젝트 팩토리를 이용한 스프링 IoC

### 애플리케이션 컨텍스트와 설정정보
스프링에서는 스프링이 제어권을 가지고 **직접 만들고 관계를 부여하는 오브젝트를 빈**이라고 부른다. **빈의 생성과 관계설정 같은 제어를 담당하는 IoC 오브젝트를 빈 팩토리**라고 부른다. 보통 빈 팩토리보다는 이를 좀 더 확장한 **애플리케이션 컨텍스트**를 주로 사용한다.

애플리케이션 컨텍스트는 별도의 정보를 참고해서 빈의 생성, 관계설정 등의 제어 작업을 총괄한다. 애플리케이션 컨텍스트가 사용하는 설정정보를 만드는 방법은 여러가지가 있는데, 우리가 앞에서 만든 오브젝트 팩토리인 DaoFactory도 조금만 손을 보면 설정정보로 사용할 수 있다.

### DaoFactory를 사용하는 애플리케이션 컨텍스트

```
@Configuration    // 애플리케이션 컨텍스트가 사용할 설정정보라는 표시
public class DaoFactory {
	@Bean         // 오브젝트 생성을 담당하는 IoC용 메소드라는 표시
	public UserDao userDao() {
		return new UserDao(connectionMaker());
	}

	@Bean
	public ConnectionMaker connectionMaker() {
		return new DConnectionMaker();
	}
}
```

```
// 애플리케이션 컨텍스트를 적용한 UserDaoTest
public class UserDaoTest {
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		// 애플리케이션 컨텍스트를 구현한 클래스는 여러가지가 있는데, 
		// @Configration이 붙은 자바코드를 설정정보로 사용하려면 
		// AnnotationConfigApplicationContext를 이용하면 된다.
		ApplicationContext context = new AnnotationConfigApplicationContext(DaoFactory.class);
		UserDao dao = context.getBean("userDao", UserDao.class);
	}
}
```

드디어 첫 번째 스프링 애플리케이션을 성공적으로 만들었다!

## 애플리케이션 컨텍스트의 동작방식
DaoFactory가 UserDao를 비롯한 DAO 오브젝트를 생성하고 관계를 맺어주는 제한적인 역할을 하는데 반해, 애플리케이션 컨텍스트를 애플리케이션에서 IoC를 적용해서 관리할 모든 오브젝트에 대한 생성과 관계설정을 담당한다. 대신 DaoFactory 처럼 직접 오브젝트를 생성하고 관계를 맺어주는 코드가 없고, 그런 생성정보와 연관관계 정보를 별도의 설정정보를 통해 얻는다.

DaoFactory를 오브젝트 팩토리로 직접 사용했을 때와 비교해서 애플리케이션 컨텍스트를 사용했을 때 얻을 수 있는 장점은 다음과 같다.

* 클라이언트는 구체적인 팩토리 클래스를 알 필요가 없다. 애플리케이션 컨텍스트를 사용하면 오브젝트 팩토리가 아무리 많아져도 일관된 방식으로 원하는 오브젝트를 가져올 수 있다.
* 애플리케이션 컨텍스트는 종합 IoC 서비스를 제공해준다.
* 애플리케이션 컨텍스트는 빈을 검색하는 다양한 방법을 제공한다.

## 스프링 IoC의 용어 정리

### 빈
스프링이 IoC 방식으로 관리하는 오브젝트. 스프링 애플리케이션에서 만들어지는 모든 오브젝트가 다 빈은 아님! 그 중에서 **스프링이 직접 그 생성과 제어를 담당하는 오브젝트만을 빈**이라고 함

### 빈 팩토리
스프링의 IoC를 담당하는 핵심 컨테이너를 의미. 빈을 등록하고, 생성하고, 조회하여 돌려주고, 그 외에 부가적인 빈 관리 기능을 담당. 보통은 빈 팩토리 보다 이를 확장한 애플리케이션 컨택스트를 이용.

### 애플리케이션 컨텍스트
빈 팩토리를 확장한 IoC 컨테이너를 의미. 빈을 관리하는 기본적인 기능은 빈팩토리와 동일. 여기에 스프링이 제공하는 각종 부가 서비스를 추가로 제공.

### 설정정보/설정 메타정보
스프링의 설정정보란 애플리케이션 컨텍스트가 IoC를 적용하기 위해 사용하는 메타정보를 말함.

### 컨테이너/IoC 컨테이너
IoC 방식으로 빈을 관리한다는 의미에서 애플리케이션 컨텍스트나 빈팩토리를 IoC 컨테이너라고 함. 컨테이너라는 말 자체가 IoC 개념을 담고 있기 때문에 애플리케이션 컨텍스트 대신 스프링 컨테이너라고 부르기도 함. 

### 스프링 프레임워크
IoC 컨테이너, 애플리케이션 컨텍스트를 포함해서 스프링이 제공하는 모든 기능을 통틀어 말함.

# 싱글톤 레지스트리와 오브젝트 스코프
우리가 만들었던 오브젝트 팩토리와 스프링의 애플리케이션 컨텍스트의 동작방식에는 차이점이 있다. **스프링은 여러 번에 걸쳐 빈을 요청하더라도 매번 동일한 오브젝트를 돌려준다는 것**이다. 단순하게 getBean()을 실행할 때마다 매번 new에 의해 새로운 UserDao가 만들어지지 않는다는 뜻이다. 왜 그럴까?

## 싱글톤 레지스트리로서의 애플리케이션 컨텍스트
스프링은 기본적으로 별다른 설정을 하지 않으면 내부에서 생성하는 빈 오브젝트를 모두 **싱글톤**으로 만든다.

> **싱글톤 패턴**
>
> 싱글톤 패턴은 어떤 클래스를 애플리케이션 내에서 제한된 인스턴스 개수, 이름처럼 주로 하나만 존재하도록 강제하는 패턴이다. 이렇게 하나만 만들어지는 클래스의 오브젝트는 애플리케이션 내에서 전역적으로 접근이 가능하다.

왜 스프링은 싱글톤으로 빈을 만드는 것일까? 이는 스프링이 주로 적용되는 대상이 자바 엔터프라이즈 기술을 사용하는 서버환경이기 때문이다. 대규모의 엔터프라이즈 서버환경은 서버 하나당 최대로 초당 수백 번씩 요청을 받게된다. 그런데 매번 클라이언트에서 요청이 올 때마다 각 로직을 담당하는 오브젝트를 새로 만들어서 사용한다면, 아무리 자바의 오브젝트 생성과 가비지 컬렉션의 성능이 좋아졌다고 한들 서버가 감당하기 힘들다. 그래서 엔터프라이즈 분야에서는 스펙에서 강제하지는 않지만 대부분 멀티스레드 환경에서 싱글톤으로 동작한다.

### 싱글톤 패턴의 한계
자바에서 싱글톤을 구현하는 방법은 보통 이렇다.

```
public class UserDao() {
	// 생성된 싱글톤 오브젝트를 저장할 수 있는 자신과 같은 타입의 스태틱 필드를 정의한다.
	private static UserDao INSTANCE;

	// 클래스 밖에서는 오브젝트를 생성하지 못하도록 생성자를 private으로 만든다.
	private UserDao(ConnectionMaker connectionMaker) {
		this.connectionMaker = connectionMaker;
	}

	// 스태틱 팩토리 메소드를 만들고, 이 메소드가 최초로 호출되는 시점에서 한 번만 오브젝트가 만들어지게 한다.
	public static synchronized UserDao getInstance() {
		if (INSTANCE == null)  INSTANCE = new UserDao(???);
		return INSTANCE;
	}
}
```

싱글톤 패턴의 구현 방식에는 다음과 같은 문제가 있다.

* private 생성자를 갖고 있기 때문에 상속할 수 없다.
* 싱글톤은 테스트하기가 힘들다.
* 서버환경에서는 싱글톤이 하나만 만들어지는 것을 보장하지 못한다.
* 싱글톤의 사용은 전역 상태를 만들 수 있기 때문에 바람직하지 못하다.

### 싱글톤 레지스트리
자바의 기본적인 싱글톤 패턴의 구현 방식은 여러 가지 단점이 있기 때문에, 스프링은 직접 싱글톤 형태의 오브젝트를 만들고 관리하는 기능을 제공한다. 이것이 바로 **싱글톤 레지스트리**다. 싱글톤 레지스트리의 장점은 스태틱 메소드와 private 생성자를 사용해야 하는 비정상적인 클래스가 아니라 평범한 자바 클래스를 싱글톤으로 활용하게 해준다는 점이다.

가장 중요한 것은 싱글톤 패턴과 달리 스프링이 지지하는 객체지향적인 설계 방식과 원칙 등을 적용하는데 아무런 제약이 없다는 점이다. 스프링은 IoC 컨테이너일 뿐만 아니라, 고전적인 싱글톤 패턴을 대신해서 싱글톤을 만들고 관리해주는 싱글톤 레지스트리라는 점을 기억해두자.

### 싱글톤과 오브젝트의 상태
싱글톤은 멀티스레드 환경이라면 여러 스레드가 동시에 접근해서 사용할 수 있다. 따라서 상태 관리에 주의를 기울여야 한다. 기본적으로 싱글톤이 멀티스레드 환경에서 서비스 형태의 오브젝트로 사용되는 경우에는 상태정보를 내부에 갖고 있지 않은 **무상태 방식**으로 만들어져야 한다.

### 스프링 빈의 스코프
빈이 생성되고, 존재하고, 적용되는 범위를 스코프(scope)라고 한다. 스프링 빈의 기본 스코프는 싱글톤이다. 싱글톤 스코프는 컨테이너 내에 한 개의 오브젝트만 만들어져서, 강제로 제거하지 않는 한 스프링 컨테이너가 존재하는 동안 계속 유지된다. 스프링에서 만들어지는 대부분의 빈은 싱글톤 스코프를 갖는다.

# 의존관계 주입(DI)

## 제어의 역전(IoC)과 의존관계 주입
IoC는 매우 느슨하게 정의되어 폭넓게 사용되는 용어다. 때문에 스프링을 IoC 컨테이너라고만 해서는 스프링이 제공하는 기능의 특징을 명확하게 설명하지 못한다. 그래서 스프링이 제공하는 IoC방식의 핵심을 짚어주는 의존관계 주입(Dependency Injection)이라는, 좀 더 의도가 명확히 드러나는 이름을 사용하기 시작했다.

스프링이 컨테이너이고 프레임워크이니 기본적인 동작원리가 모두 IoC 방식이라고 할 수 있지만, 스프링이 여타 프레임워크와 차별화돼서 제공해주는 기능은 의존관계 주입이라는 새로운 용어를 사용할 때 분명하게 드러난다.

그래서 초기에는 주로 IoC 컨테이너라고 불리던 스프링이 지금은 **DI 컨테이너**라고 더 많이 불리고 있다.

## 런타임 의존관계 설정
의존관계란 무엇인가? 두 개의 클래스 또는 모듈이 의존관계에 있다고 말할 때는 항상 **방향성**을 부여해줘야 한다. 즉 누가 누구에게 의존하는 관계에 있다는 식이어야 한다. 그렇다면 의존하고 있다는 건 무슨 의미일까? 의존한다는 건, 의존대상이 변할 때 그 변화에 영향을 받는다는 것이다. 다시 말하지만 의존관계에는 방향성이 있다. 의존하지 않는다는 말은 대상이 변할 때 그 변화에 영향을 받지 않는다는 뜻이다.

### UserDao의 의존관계
UserDao의 예를 보자. UserDao가 ConnectionMaker에 의존하고 있는 형태다. 따라서 ConnectionMaker 인터페이스가 변한다면 그 영향을 UserDao가 직접적으로 받게 된다. 하지만 ConnectionMaker 인터페이스를 구현한 클래스가 다른 것으로 바뀌거나 그 내부에서 사용하는 메소드에 변화가 생겨도 UserDao에 영향을 주지 않는다. 이렇게 인터페이스에 대해서만 의존관계를 만들어두면 인터페이스 구현 클래스와의 관계는 느슨해지면서 변화에 영향을 덜 받는 상태가 된다. *결합도가 낮다고 설명할 수 있다.*

인터페이스를 통해 설계 시점에 느슨한 의존관계를 갖는 경우에는 런타임 시에 사용할 오브젝트가 어떤 클래스로 만든 것인지 미리 알 수가 없다. 프로그램이 시작되고 나서 런타임 시에 의존관계를 맺는 대상, 즉 실제 사용대상인 오브젝트를 **의존 오브젝트**라고 말한다.

의존관계 주입은 이렇게 구체적인 의존 오브젝트와 그것을 사용할 주체를 런타임 시에 연결해주는 작업을 말한다.

정리하면 의존관계 주입이란 다음과 같은 세 가지 조건을 충족하는 작업을 말한다.

1. 클래스 모델이나 코드에는 런타임 시점의 의존관계가 드러나지 않는다. 그러기 위해서는 인터페이스에만 의존하고 있어야 한다.
2. 런타임 시점의 의존관계는 컨테이너나 팩토리 같은 제 3의 존재가 결정한다.
3. 의존관계는 사용할 오브젝트에 대한 레퍼런스를 외부에서 제공(주입)해줌으로써 만들어진다.

의존관계 주입의 핵심은 설계 시점에는 알지 못했던 두 오브젝트의 관계를 맺도록 도와주는 제 3의 존재가 있다는 것이다. 전략패턴에 등장하는 클라이언트나 앞에서 만들었던 DaoFactory, 또 DaoFactory와 같은 작업을 일반화해서 만들어졌다는 스프링의 애플리케이션 컨텍스트, 빈 팩토리, IoC 컨테이너 등이 모두 외부에서 오브젝트 사이의 런타임 관계를 맺어주는 책임을 지닌 제 3의 존재라고 볼 수 있다.

### UserDao의 의존관계 주입
인터페이스를 사이에 두고 UserDao와 ConnectionMaker 구현 클래스 간에 의존관계를 느슨하게 만들긴 했지만, 마지막으로 남은 문제가 있었는데 그것은 UserDao가 사용할 구체적인 클래스를 알고 있어야 한다는 점이였다.

```
public UserDao() {
	connectionMaker = new DConnectionMaker();
}
```

이 코드의 문제는 이미 런타임 시의 의존관계가 코드 속에 다 미리 결정되어 있다는 점이다. 그래서 IoC 방식을 써서 UserDao로부터 런타임 의존관계를 드러내는 코드를 제거하고, 제 3의 존재에 런타임 의존관계 결정 권한을 위임한다. 그래서 최종적으로 만들어진 것이 DaoFactory다. DaoFactory는 런타임 시점에 UserDao가 사용할 ConnectionMaker 타입의 오브젝트를 결정하고, 이를 생성한 후에 UserDao의 생성자 파라미터로 주입해서 UserDao가 DConnectionMaker의 오브젝트와 런타임 의존관계를 맺게 해준다. 따라서 의존관계 주입의 세 가지 조건을 모두 충족한다고 볼 수 있고, 이미 DaoFactory를 만든 시점에서 의존관계 주입(DI)을 이용한 셈이다. 따라서 DaoFactory는 의존관계 주입을 담당하는 컨테이너라고 볼 수 있고, 줄여서 DI 컨테이너라고 불러도 된다.

주입이라는 건 외부에서 내부로 무엇인가를 넘겨줘야 하는 것인데, 자바에서 오브젝트에 무엇인가를 넣어준다는 개념은 메소드를 실행하면서 파라미터로 오브젝트의 레퍼런스를 전달해주는 방법뿐이다. 가장 손쉽게 사용할 수 있는 파라미터 전달이 가능한 메소드는 바로 생성자다.

```
public class UserDao() {
	private ConnectionMaker connectionMaker;

	public UserDao(ConnectionMaker connectionMaker) {
		this.connectionMaker = connectionMaker;
	}
}
```

이렇게 DI 컨테이너에 의해 런타임 시에 의존 오브젝트를 사용할 수 있도록 그 래퍼런스를 전달받는 과정이 마치 메소드(생성자)를 통해 DI 컨테이너가 UserDao에게 주입해주는 것과 같다고 해서 이를 의존관계 주입이라고 부른다.

DI는 자신이 사용할 오브젝트에 대한 선택과 생성 제어권을 외부로 넘기고 자신은 수동적으로 주입받은 오브젝트를 사용한다는 점에서 IoC의 개념에 잘 들어맞는다. 그래서 스프링을 IoC 컨테이너 외에도 DI 컨테이너라고 부르는 것이다.

## 의존관계 검색과 주입
스프링이 제공하는 IoC 방법에는 의존관계 주입만 있는 것이 아니다. 코드에서는 구체적인 클래스에 의존하지 않고, 런타임 시에 의존관계를 결정한다는 점에서 의존관계 주입과 비슷하지만, 의존관계를 맺는 방법이 외부로부터의 주입이 아니라 스스로 검색을 이용하기 때문에 **의존관계 검색**이라고 불리는 것도 있다.

의존관계 검색은 런타임 시 의존관계를 맺을 오브젝트를 결정하는 것과 오브젝트의 생성 작업은 외부 컨테이너에게 IoC로 맡기지만, 이를 가져올 때는 메소드나 생성자를 통한 주입 대신 스스로 컨테이너에게 요청하는 방법을 사용한다.

```
public UserDao() {
	DaoFactory daoFactory = new DaoFactory();
	this.connectionMaker = daoFactory.connectionMaker();
}
```

이렇게 해도 UserDao는 여전히 자신이 어떤 ConnectionMaker 오브젝트를 사용할지 미리 알 지 못한다. 여전히 코드의 의존대상은 ConnectionMaker 인터페이스뿐이다. 런타임 시에 DaoFactory가 만들어서 돌려주는 오브젝트와 다이나믹하게 런타임 의존관계를 맺는다. 따라서 IoC 개념을 잘 따르고 있으며, 그 혜택을 받고 있는 코드다. 하지만 적용 방법은 외부로부터의 주입이 아니라 스스로 IoC 컨테이너인 DaoFactory에게 요청하는 것이다. 

어플리케이션 컨텍스트를 사용해서 의존관계 검색 방식으로 ConnectionMaker 오브젝트를 가져오게 만들 수도 있다.

```
public UserDao() {
	AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(DaoFactory.class);
	this.connectionMaker = context.getBean("connectionMaker", ConnectionMaker.class);
}
```

의존관계 검색은 기존 의존관계 주입의 거의 모든 장점을 가지고 있다. 단, 방법만 조금 다를 뿐이다.

그렇다면 의존관계 검색과 의존관계 주입 중 어떤 것이 더 나을까? 코드를 보면 느낄 수 있겠지만 의존관계 주입 쪽이 훨씬 단순하고 깔끔하다. 그런데 의존관계 검색을 사용해야 할 때가 있다. 

```
public class UserDaoTest {
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		ApplicationContext context = new AnnotationConfigApplicationContext(DaoFactory.class);
		UserDao dao = context.getBean("userDao", UserDao.class);
	}
}
```

앞에서 만들었던 테스트 코드에서는 이미 의존관계 검색 방식인 getBean()을 사용했다. 스태틱 메소드인 main()에서는 DI를 이용해 오브젝트를 주입받을 방법이 없기 때문이다.

의존관계 검색과 의존관계 주입을 적용할 때 발견할 수 있는 중요한 차이점이 하나 있다. 의존관계 검색 방식에서는 검색하는 오브젝트는 자신이 스프링의 빈일 필요가 없다는 점이다. UserDao에 스프링의 getBean()을 사용한 의존관계 검색 방법을 적용했다고 해보자. 이 경우 UserDao는 굳이 스프링이 만들고 관리하는 빈일 필요가 없다. 그냥 어딘가에서 직접 new UserDao() 해서 만들어서 사용해도 된다. 이 때는 ConnectionMaker만 스프링의 빈이기만 하면 된다.

반면에 의존관계 주입에서는 UserDao와 ConnectionMaker 사이에 DI가 적용되려면 UserDao도 반드시 컨테이너가 만드는 빈 오브젝트여야 한다. 컨테이너가 UserDao에 ConnectionMaker 오브젝트를 주입해주려면 UserDao에 대한 생성과 초기화 권한을 갖고 있어야 하고, 그러려면 UserDao는 IoC 방식으로 컨테이너에서 생성되는 오브젝트, 즉 빈이어야 하기 때문이다. 이런 점에서 DI와 DL(의존관계 검색)은 적용 방법에 차이가 있다.

DI를 원하는 오브젝트는 먼저 자기 자신이 컨테이너가 관리하는 빈이 돼야 한다는 사실을 잊지 말자.

> DI의 동작방식은 이름 그대로 외부로부터의 주입이다. 하지만 단지 외부에서 파라미터로 오브젝트를 넘겨줬다고 해서 다 DI가 아니라는 점을 주의해야 한다. 주입받는 메소드 파라미터가 이미 특정 클래스 타입으로 고정되어 있다면 DI가 일어날 수 없다. DI에서 말하는 주입은 다이나믹하게 구현 클래스를 결정해서 제공받을 수 있도록 인터페이스 타입의 파라미터를 통해 이뤄져야 한다.

## 의존관계 주입의 응용
스프링이 제공하는 기능의 99%가 DI의 혜택을 이용하고 있다. DI 없이는 스프링도 없다. 그만큼 DI를 활용할 방법은 다양하다.

### 기능 구현의 교환
개발 중에는 로컬 DB를 사용하고, 배포시에 리얼 DB를 사용하도록 하는 어플리케이션 개발을 생각해보자. 

DI 방식을 적용하지 않았을 경우, 로컬 DB에 대한 연결 기능이 있는 클래스를 만들고, 모든 DAO에서 이 클래스의 오브젝트를 매번 생성해서 사용하게 했을 것이다(마치 초난감 DAO처럼..). 배포를 위해 DB 연결 클래스를 변경하게 되면 모든 DAO에서 클래스 정보를 변경해주어야 하는 수고가 발생한다. 

반면에 DI 방식을 적용해서 만들었다고 해보자. 모든 DAO는 생성시점에 ConnectionMaker 타입의 오브젝트를 컨테이너로부터 제공받는다. 구체적인 사용 클래스 이름은 컨테이너가 사용할 설정정보에 들어 있다. 개발자 PC에서는 DaoFactory의 코드를 다음과 같이 만들어서 사용하면 된다.

```
@Bean
public ConnectionMaker connectionMaker() {
	return new LocalDBConnectionMaker();
}
```

이를 서버에 배포할 때는 어떤 DAO 클래스와 코드도 수정할 필요가 없다. 단지 아래와 같이 딱 한줄만 수정하면 된다.

```
@Bean
public ConnectionMaker connectionMaker() {
	return new ProductionDBConnectionMaker();
}
```

### 부가기능 추가
DAO가 DB를 얼마나 많이 연결해서 사용하는지 파악하고 싶다고 해보자.

무식한 방법으로 모든 DAO의 makeConnection() 메소드를 호출하는 부분에 새로 추가한 카운터를 증가시키는 코드를 넣어야 할까? 이것은 엄청난 낭비다. 게다가 DAO 코드를 수정하는건 지금까지 그렇게 피하려고 했던 일이 아닌가. 또한 DB 연결횟수를 세는 일은 DAO의 관심사항이 아니다. 어떻게든 분리돼야 할 책임이기도 하다.

DI 컨테이너에서라면 아주 간단한 방법으로 가능하다. DAO와 DB 커넥션을 만드는 오브젝트 사이에 연결횟수를 카운팅하는 오브젝트를 하나 더 추가하는 것이다. DI를 이용한다고 했으니 **당연히 기존 코드는 수정하지 않아도 된다**. 단지 컨테이너가 사용하는 설정정보만 수정해서 런타임 의존관계만 새롭게 정의해주면 된다.

```
// DAO가 의존할 대상이 될 것이기 때문에 ConnectionMaker 인터페이스를 구현해서 만든다.
public class CountingConnectionMaker implements ConnectionMaker {
	int counter = 0;
	private ConnectionMaker realConnectionMaker;

	public CountingConnectionMaker(ConnectionMaker realConnectionMaker) {
		this.realConnectionMaker = realConnectionMaker;
	}

	public Connection makeConnection() throws ClassNotFoundException, SQLException {
		// 직접 DB 커넥션을 만들지 않는다.
		// 자신의 관심사인 DB 연결횟수 카운팅 작업을 마치면 실제 DB 커넥션을 만들어주는 오브젝트의 메소드를 호출한다.
		this.counter++;
		return realConnectionMaker.makeConnection();
	}

	public int getCounter() {
		return this.counter;
	}
}
```

UserDao는 ConnectionMaker의 인터페이스에만 의존하고 있기 때문에, ConnectionMaker 인터페이스를 구현하고 있다면 어떤 것이든 DI가 가능하다. 그래서 UserDao 오브젝트가 DI 받는 대상의 설정을 조정해서 DConnection 오브젝트 대신 CountingConnectionMaker 오브젝트로 바꿔치기하는 것이다. 그렇다고 해서 DB 커넥션을 제공해주지 않으면 DAO가 동작하지 않을 테니 CountingConnectionMaker가 다시 실제 사용할 DB 커넥션을 제공해주는 DConnectionMaker를 호출하도록 만들어야 한다. 역시 DI를 사용하면 된다.

CountingConnectionMaker가 추가되면서 런타임 의존관계가 어떻게 바뀌는지 살펴보자. 

```
// CountingConnectionMaker를 사용하기 전의 런타임 의존관계
------------          ---------------------
|          |          |                   |
| :UserDao |--------->| :DConnectionMaker |
|          |          |                   |
------------          ---------------------

// 재구성된 새로운 런타임 의존관계
------------         ----------------------------          ---------------------
|          |         |                          |          |                   |
| :UserDao |-------->| :CountingConnectionMaker |--------->| :DConnectionMaker |
|          |         |                          |          |                   |
------------         ----------------------------          ---------------------
```

새로운 의존관계를 컨테이너가 사용할 설정정보를 이용해 만들어보자.

```
@Configuration
public class CountingDaoFactory {
	@Bean
	public UserDao userDao() {
		// 모든 DAO는 여전히 connectionMaker()에서 만들어지는 오브젝트를 DI 받는다.
		return new UserDao(connectionMaker());
	}

	@Bean
	public ConnectionMaker connectionMaker() {
		return new CountingConnectionMaker(realConnectionMaker());
	}

	@Bean
	public ConnectionMaker realConnectionMaker() {
		return new DConnectionMaker();
	}
}
```

지금은 DAO가 하나뿐이지만 DAO가 수십, 수백 개여도 상관없다. DI의 장점은 관심사의 분리를 통해 얻어지는 높은 응집도에서 나온다. 모든 DAO가 직접 의존해서 사용할 ConnectionMaker 타입 오브젝트는 connectionMaker() 메소드에서 만든다. 따라서 CountingConnectionMaker의 의존관계를 추가하려면 이 메소드만 수정하면 그만이다. 또한 CountingConnectionMaker를 이용한 분석 작업이 모두 끝나면, 다시 CountingDaoFactory 설정 클래스를 DaoFactory로 변경하거나 connectionMaker() 메소드를 수정하는 것만으로 DAO의 런타임 의존관계는 이전 상태로 복구된다.

바로 이런 것이 의존관계 주입의 매력을 잘 드러내는 응용 방법이다.

## 메소드를 이용한 의존관계 주입
지금까지는 UserDao의 의존관계 주입을 위해 생성자를 사용했다. 그런데 의존관계 주입 시 반드시 생성자를 사용해야 하는 것은 아니다. 생성자가 아닌 일반 메소드를 사용할 수도 있을 뿐만 아니라, 생성자를 사용하는 방법보다 더 자주 사용된다.

* 수정자 메소드(setter)를 이용한 주입
    * 수정자 메소드는 외부에서 오브젝트 내부의 애트리뷰트 값을 변경하려는 용도로 주로 사용된다.
    * 수정자 메소드의 핵심기능은 파라미터로 전달된 값을 보통 내부의 인스턴스 변수에 저장하는 것이다.
    * 수정자 메소드는 외부로부터 제공받은 오브젝트 레퍼런스를 저장해뒀다가 내부의 메소드에서 사용하게 하는 DI 방식에서 활용하기에 적당하다.

* 일반 메소드를 이용한 주입
    * 수정자 메소드처럼 한 번에 한 개의 파라미터만 가질 수 있다는 제약이 싫다면 여러 개의 파라미터를 갖는 일반 메소드를 DI용으로 사용할 수도 있다.
    * 하지만 파라미터의 개수가 많아지고 비슷한 타입이 여러 개라면 실수하기 쉽다.

스프링은 전통적으로 메소드를 이용한 DI 방법 중에서 수정자 메소드를 가장 많이 사용해왔다. 뒤에서 보겠지만, DaoFactory 같은 자바 코드 대신 XML을 사용하는 경우에는 자바빈 규약을 따르는 수정자 메소드가 가장 사용하기 편리하다.

UserDao도 수정자 메소드를 이용해 DI 하도록 만들어보자.

```
public class UserDao() {
	private ConnectionMaker connectionMaker;

	public void setConnectionMaker(ConnectionMaker connectionMaker) {
		this.connectionMaker = connectionMaker;
	}
}
```

```
@Configuration
public class DaoFactory {
	@Bean
	public UserDao userDao() {
		UserDao userDao = new UserDao();
		userDao.setConnectionMaker(connectionMaker());
		return userDao;
	}

	@Bean
	public ConnectionMaker connectionMaker() {
		return new DConnectionMaker();
	}
}
```

# XML을 이용한 설정
본격적인 범용 DI 컨테이너를 사용하면서 오브젝트 사이의 의존정보는 일일이 자바 코드로 만들어주려면 번거롭다. 스프링은 DaoFactory와 같은 자바 클래스를 이용하는 것 외에도, 다양한 방법을 통해 DI 의존관계 설정정보를 만들 수 있다. 가장 대표적인 것이 바로 XML이다.

## XML 설정
DI 정보가 담긴 XML 파일은 \<beans\>를 루트 엘리먼트로 사용한다. \<beans\>안에는 여러 개의 \<bean\>을 정의할 수 있다. XML 설정은 자바 클래스로 만든 설정과 내용이 동일하다. @Configuration을 \<beans\>, @Bean을 \<bean\>에 대응해서 생각하면 이해하기 쉬울 것이다.

하나의 @Bean 메소드를 통해 얻을 수 있는 빈의 DI 정보는 다음 세 가지다.

* 빈의 이름 : @Bean 메소드 이름이 빈의 이름이다. 이 이름은 getBean()에서 사용된다.
* 빈의 클래스 : 빈 오브젝트를 어떤 클래스를 이용해서 만들지를 정의한다.
* 빈의 의존 오브젝트 : 빈의 생성자나 수정 메소드를 통해 의존 오브젝트를 넣어준다. 의존 오브젝트도 하나의 빈이므로 이름이 있을 것이고, 그 이름에 해당하는 메소드를 호출해서 의존 오브젝트를 가져온다. 의존 오브젝트는 하나 이상일 수도 있다. ConnectionMaker 처럼 더 이상 의존하고 있는 오브젝트가 없는 경우에는 생략할 수 있다.

XML에서 <bean>을 사용해도 이 세 가지 정보를 정의할 수 있다.

### connectionMaker() 전환
connectionMaker()로 정의되는 빈은 의존하는 다른 오브젝트는 없으니 DI 정보 세 가지 중 두 가지만 있으면 된다.

| | 자바 코드 설정정보 | XML 설정정보 |
|---|---|---|
| 빈의 이름 | @Bean methodName() | <bean id="methodName" |
| 빈의 클래스 | return new BeanClass(); | class="a.b.c...BeanClass"> |

\<bean\> 태그의 class 애트리뷰트에 지정하는 것은 *자바 메소드의 리턴타입이 아니라 메소드에서 오브젝트를 만들 때 사용하는 클래스 이름*이라는 점에 주의하자. XML에서는 리턴 타입을 지정하지 않아도 된다. class 애트리뷰트에 넣을 클래스 이름은 패키지까지 모두 포함해야 한다.

```
@Bean ---------------------------------------------> <bean
public ConnectionMaker connectionMaker() { -------->       id="connectionMaker"
    return new DConnectionMaker() ----------------->       class="a.b.c..DConnectionMaker" />
}
```

### userDao() 전환
userDao()에는 DI 정보의 세 가지 요소가 모두 들어 있다. 여기서 관심을 가질 것은 수정자 메소드를 사용해 의존관계를 주입해주는 부분이다. XML로 의존관계 정보를 만들 때는 수정자 메소드를 사용하는 것이 편리하다.

XML에서는 \<property\> 태그를 사용해 의존 오브젝트와의 관계를 정의한다. \<property\> 태그는 name과 ref라는 두 개의 애트리뷰트를 갖는다. **name은 프로퍼티의 이름**이다. 이 프로퍼티 이름으로 수정자 메소드를 알 수 있다. **ref는 수정자 메소드를 통해 주입해줄 오브젝트의 빈 이름**이다. DI 할 오브젝트도 역시 빈이다. 그 빈의 이름을 지정해주면 된다.

```
userDao.setConnectionMaker(connectionMaker());
                   |                    |
                   v                    v
<property name="connectionMaker" ref="connectionMaker" />
```

이 <\property\> 태그를 userDao 빈을 정의한 \<bean\> 태그 안에 넣어주면된다.

### XML의 의존관계 주입 정보
전환한 두 개의 \<bean\> 태그를 \<beans\>로 감싸주면 DaoFactory로부터 XML로의 전환 작업이 끝난다.

```
<beans>
	<bean id="connectionMaker" class="a.b.c..DConnectionMaker" />
	<bean id="userDao" class="a.b.c..UserDao">
		<property name="connectionMaker ref="connectionMaker" />
	</bean>
</beans>
```

\<property\> 태그의 name과 ref는 그 의미가 다르므로 이름이 같더라도 어떤 차이가 있는지 구별할 수 있어야 한다. name은 DI에 사용할 수정자 메소드의 프로퍼티 이름이며, ref는 주입할 오브젝트를 정의한 빈의 id다. 보통 프로퍼티 이름과 DI 되는 빈의 이름이 같은 경우가 많다. 프로퍼티 이름은 주입할 빈 오브젝트의 인터페이스를 따르는 경우가 많고, 빈 이름도 역시 인터페이스 이름을 사용하는 경우가 많기 때문이다. 바뀔 수 있는 클래스 이름보다는 대표적인 인터페이스 이름을 따르는 편이 자연스럽다.

때로는 같은 인터페이스를 구현한 의존 오브젝트를 여러 개 정의해두고 그 중에서 원하는 걸 골라서 DI 하는 경우도 있다. 이 때는 각 빈의 이름을 독립적으로 만들어두고 ref 애트리뷰터를 이용해 DI 받을 빈을 지정해주면 된다.

```
<beans>
	<bean id="localDBConnectionMaker" class="a.b.c..LocalDBConnectionMaker" />
	<bean id="testDBConnectionMaker" class="a.b.c..TestDBConnectionMaker" />
	<bean id="productionDBConnectionMaker" class="a.b.c..ProductionDBConnectionMaker" />

	<bean id="userDao" class="a.b.c..UserDao">
		// DAO에서 적절한 인터페이스 구현 클래스를 선택해서 사용한다.
		<property name="connectionMaker ref="localDBConnectionMaker" />
	</bean>
</beans>
```

## XML을 이용하는 애플리케이션 컨텍스트
이제 애플리케이션 컨텍스트가 DaoFactory 대신 XML 설정정보를 활용하도록 만들어보자.

```
// XML에서 빈의 의존관계 정보를 이용하는 IoC/DI 작업에는 GenericXmlApplicationContext를 사용한다.
//
// 생성자 파라미터로 XML 파일의 클래스패스를 지정해주면 된다.
// XML 설정파일은 클래스패스 최상단에 두면 편하다.
//
// 클래스패스를 시작하는 /는 넣을 수도 있고 생략할 수도 있다.
// 시작하는 /가 없는 경우에도 항상 루트에서부터 시작하는 클래스패스라는 점을 기억해두자.

ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");
```

## DataSource 인터페이스로 변환

### DataSource 인터페이스 적용
ConnectionMaker는 DB 커넥션을 생성해주는 기능 하나만을 정의한 매우 단순한 인터페이스다. 사실 자바에서는 DB 커넥션을 가져오는 오브젝트의 기능을 추상화해서 비슷한 용도로 사용할 수 있게 만들어진 DataSource라는 인터페이스가 이미 존재한다. 일반적으로 DataSource를 구현해서 DB 커넥션을 제공하는 클래스를 만들 일은 거의 없다. 이미 다양한 방법으로 DB 연결과 풀링 기능을 갖춘 많은 DataSource 구현 클래스가 존재하고, 이를 가져다 사용하면 충분하다.

우리가 DataSource 인터페이스에서 실제로 관심을 가질 것은 getConnection() 메소드 하나뿐이다. 이름만 다르지 ConnectionMaker 인터페이스의 makeConnection()과 목적이 동일한 메소드다. DAO에서는 DataSource의 getConnection() 메소드를 사용해 DB 커넥션을 가져오면 된다.

```
public class UserDao {
	private DataSource dataSource;

	// 의존 오브젝트 타입 변경
	public void setDataSource(DataSource dataSource) {
		this.dataSource = dataSource;
	}

	public void add(User user) throws SQLException {
		// 커넥션을 가져오는 메소드 변경
		Connection c = dataSource.getConnection();
	}
}
```

스프링이 제공해주는 DataSource 구현 클래스 중에 테스트환경에서 간단히 사용할 수 있는 SimpleDriverDataSource라는 것이 있다. 이 클래스를 사용하도록 DI를 재구성해보자.

### 자바 코드 설정 방식
먼저 DaoFactory 설정 방식을 이용해보자.

```
// 기존의 connectionMaker() 메소드를 dataSource()로 변경
@Bean
public DataSource dataSource() {
	SimpleDriverDataSource dataSource = new SimpleDriverDataSource();

	// DB 연결정보를 수정자 메소드를 통해 넣어준다.
	dataSource.setDriverClass(com.mysql.jdbc.Driver.class);
	dataSource.setUrl("jdbc:mysql://localhost/springbook");
	dataSource.setUsername("spring");
	dataSource.setPassword("book");

	return dataSource;
}

// UserDao는 이제 DataSource 타입의 dataSource()를 DI 받는다.
@Bean
public UserDao userDao() {
	UserDao userDao = new UserDao();
	userDao.setDataSource(dataSource());
	return userDao;
}
```

### XML 설정 방식
이번에는 XML 설정 방식으로 변경해보자.

```
// id가 connectionMaker인 <bean>을 없애고, dataSource라는 이름의 <bean>을 등록한다.
<bean id="dataSource" class="org.springframework.jdbc.datasource.SimpleDriverDataSource" />
```

그런데 문제는 이 \<bean\> 설정으로 SimpleDriverDataSource의 오브젝트를 만드는 것까지는 가능하지만, dataSource() 메소드에서 SimpleDriverDataSource 오브젝트의 수정자로 넣어준 DB 접속정보는 나타나 있지 않다는 점이다.

UserDao처럼 다른 빈에 의존하는 경우에는 \<property\> 태그에 의존할 빈 이름을 넣어주면 된다. 하지만 이 경우는 단순 Class 타입의 오브젝트나 텍스트 값이다. 어떻게 XML에서 DB 연결정보를 넣도록 설정을 만들 수 있을까?

## 프로퍼티 값의 주입

### 값 주입
다른 빈 오브젝트의 레퍼런스가 아닌 단순 정보도 오브젝트를 초기화하는 과정에서 수정자 메소드에 넣을 수 있다. 이때는 DI에서처럼 오브젝트의 구현 클래스를 다이나믹하게 바꿀 수 있게 해주는 것이 목적은 아니다. 대신 클래스 외부에서 DB 연결정보와 같이 변경 가능한 정보를 설정해줄 수 있도록 만들기 위해서다.

텍스트나 단순 오브젝트 등을 수정자 메소드에 넣어주는 것을 스프링에서는 *값을 주입한다*고 한다. 이것도 성격은 다르지만 일종의 DI라고 볼 수 있다. 사용할 오브젝트 자체를 바꾸지는 않지만 오브젝트의 특성은 외부에서 변경할 수 있기 때문이다.

스프링의 빈으로 등록될 클래스에 수정자 메소드가 정의되어 있다면 \<property\>를 사용해 주입할 정보를 지정할 수 있다는 점에서는 \<property ref=""\>와 동일하다. 하지만 다른 빈 오브젝트의 레퍼런스(ref)가 아니라 단순 값(value)을 주입해주는 것이기 때문에 ref 애트리뷰트 대신 value 애트리뷰트를 사용한다.

```
dataSource.setDriverClass(com.mysql.jdbc.Driver.class);   -> <property name="driverClass" value="com.mysql.jdbc.Driver" />
dataSource.setUrl("jdbc:mysql://localhost/springbook");   -> <property name="url" value="jdbc:mysql://localhost/springbook" />
dataSource.setUsername("spring");                         -> <property name="username" value="spring" />
dataSource.setPassword("book");                           -> <property name="password" value="book" />
```

value 애트리뷰트에 들어가는 것은 다른 빈의 이름이 아니라 실제로 수정자 메소드의 파라미터로 전달되는 스트링 그 자체다.

### value 값의 자동 변환
url, username, password는 모두 스트링 타입이니 원래 텍스트로 정의되는 value 애트리뷰트의 값을 사용하는 것은 문제없다. 그런데 driverClass는 스트링 타입이 아니라 java.lang.Class 타입인데, XML에서는 별다른 타입정보 없이 클래스의 이름이 텍스트 형태로 value에 들어가 있다.

이런 설정이 가능한 이유는 스프링이 프로퍼티의 값을, 수정자 메소드의 파라미터 타입을 참고로 해서 적절한 형태로 변환해주기 때문이다. setDriverClass() 메소드의 파라미터 타입이 Class임을 확인하고 "com.mysql.jdbc.Driver" 라는 텍스트 값을 com.mysql.jdbc.Driver.class 오브젝트로 자동 변경해주는 것이다. 내부적으로 다음과 같이 변환 작업이 일어난다고 생각하면 된다.

```
Class driverClass = Class.forName("com.mysql.jdbc.Driver");
dataSource.setDriverClass(driverClass);
```

이렇게 해서 DataSource 인터페이스로의 전환 작업을 모두 마쳤다.
