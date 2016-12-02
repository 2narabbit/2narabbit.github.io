---
layout: default
title:  "[토비의 스프링] 2장 - 테스트"
date:   2016-11-10 00:00:00
categories: main
---

어플리케이션은 계속 변하고 복잡해져 간다. 그 변화에 대응하는 첫 번째 전략이 확장과 변화를 고려한 객체지향적 설계와 그것을 효과적으로 담아낼 수 있는 IoC/DI 같은 기술이라면, 두 번째 전략은 만들어진 코드를 확신할 수 있게 해주고, 변화에 유연하게 대처할 수 있는 자신감을 주는 테스트 기술이다.

# UserDaoTest 다시 보기

## 테스트의 유용성
이전에 만들었던 테스트 코드는 main() 메소드를 이용해 UserDao 오브젝트의 add(), get() 메소드를 호출하고, 그 결과를 화면에 출력해서 그 값을 눈으로 확인시켜준다. 이렇게 만든 테스트용 main() 메소드를 반복적으로 실행해가면서 처음 설계한 대로 기능이 동작하는지를 매 단계 확인한 덕분에, 다양한 방법으로 초난감 UserDao 코드의 설계와 코드를 개선했고, 심지어 스프링을 적용해서 동작하게 만들 수도 있었다.

테스트란 결국 내가 예상하고 의도했던 대로 코드가 정확히 동작하는지를 확인해서, 만든 코드를 확신할 수 있게 해주는 작업이다. 또한 테스트의 결과가 원하는 대로 나오지 않는 경우에는 코드나 설계에 결함이 있음을 알 수 있다. 이를 통해 코드의 결함을 제거해가는 작업, 일명 디버깅을 거치게 되고, 결국 최종적으로 테스트가 성공하면 모든 결함이 제거됐다는 확신을 얻을 수 있다.

## UserDaoTest의 특징

### 웹을 통한 DAO 테스트 방법의 문제점
보통 웹 프로그램에서 사용하는 DAO를 테스트하는 방밥은 다음과 같다. DAO를 만든 뒤 바로 테스트하지 않고, 서비스 계층, MVC 프레젠테이션 계층까지 포함한 모든 입출력 기능을 대충이라도 코드로 다 만든다. 이렇게 만들어진 테스트용 웹 어플리케이션을 서버에 배치한 뒤 웹 화면을 통해 값을 입력하고, 기능을 수행하고, 결과를 확인한다. 이 방법은 가장 흔히 쓰이는 방법이지만, DAO에 대한 테스트로서는 단점이 너무 많다.

일단 모든 레이어의 기능을 다 만들고 나서야 테스트가 가능하다는 것이 가장 큰 문제다. 또한 테스트를 하는 중에 에러가 나거나 테스트가 실패했다면, 과연 어디에서 문제가 발생했는지를 찾아내야 하는 수고도 필요하다. 하나의 테스트를 수행하는 데 참여하는 클래스와 코드가 너무 많기 때문이다. 테스트하고 싶었던 건 UserDao였는데 다른 계층의 코드와 컴포넌트, 심지어 서버의 설정 상태까지 모두 테스트에 영향을 줄 수 있기 때문에 이런 방식으로 테스트하는 것은 번거롭고, 오류가 있을 때 빠르고 정확하게 대응하기가 힘들다는 문제가 있다.

### 작은 단위의 테스트
테스트하고자 하는 대상이 명확하다면 그 대상에만 집중해서 테스트하는 것이 바람직하다. 한꺼번에 너무 많은 것을 몰아서 테스트하면 테스트 수행 과정도 복잡해지고, 오류가 발생했을 때 정확한 원인을 찾기가 힘들어진다.

UserDaoTest는 한 가지 관심에 집중할 수 있게 작은 단위로 만들어진 테스트다. 이렇게 작은 단위의 코드에 대해 테스트를 수행한 것을 **단위 테스트**라고 한다. 일반적으로 단위는 작을수록 좋다. 단위를 넘어서는 다른 코드들은 신경 쓰지 않고, 참여하지도 않고 테스트가 동작할 수 있으면 좋다.

때로는 웹 사용자 인터페이스부터 시작해서 DB에 이르기까지의 어플리케이션 전 계층이 참여하고, 각종 기능을 모두 사용하는 전 과정을 하나로 묶어서 테스트할 필요도 있다. 아마도 수많은 에러를 만나거나 에러는 안 나지만 제대로 기능이 동작하지 않는 경험을 하게 될 것이다. 이때는 문제의 원인을 찾기가 매우 힘들다. 예외가 발생해도 그 이유를 찾는 데 많은 시간이 걸릴 수 있다. 그런데 각 단위별로 테스트를 먼저 모두 진행하고 나서 이런 긴 테스트를 시작했다면 어떨까? 그래도 역시 예외가 발생하거나 테스트가 실패할 수는 있겠지만, 이미 각 단위별로 충분한 검증을 마치고 오류를 잡았으므로 훨씬 나을 것이다.

### 자동수행 테스트 코드
테스트는 자동으로 수행되도록 코드로 만들어지는 것이 중요하다. 테스트 자체가 사람의 수작업을 거치는 방법을 사용하기보다는 코드로 만들어져서 자동으로 수행될 수 있어야 한다는 건 매우 중요하다.

자동으로 수행되는 테스트의 장점은 자주 반복할 수 있다는 것이다. 번거로운 작업이 없고 테스트를 빠르게 실행할 수 있기 때문에 언제든 코드를 수정하고 나서 테스트를 해볼 수 있다.

### 지속적인 개선과 점진적인 개발을 위한 테스트
처음 만든 초난감 DAO 코드를, 스프링을 이용한 깔끔하고 완성도 높은 객체지향적 코드로 발전시키는 과정의 일등 공신은 바로 이 테스트였다. 테스트가 없었다면, 다양한 방법을 동원해서 코드를 수정하고 설계를 개선해나가는 과정이 그다지 미덥지 않을 수도 있고, 그래서 마음이 불편해지면 이쯤에서 그만두자는 생각이 들 수도 있었기 때문이다.

하지만 일단은 단순 무식한 방법으로 정상동작하는 코드를 만들고, 테스트를 만들어뒀기 때문에 매우 작은 단계를 거쳐가면서 계속 코드를 개선해나갈 수 있었다. 오히려 그렇게 작은 단계를 거치는 동안 테스트를 수행해서 확신을 가지고 코드를 변경해갔기 때문에 전체적으로 코드를 개선하는 작업에 속도가 붙고 더 쉬워졌을 수도 있다.

## UserDaoTest의 문제점
UserDaoTest가 UI까지 동원되는 번거로운 수동 테스트에 비해 장점이 많은 건 사실이다. 하지만 만족스럽지 못한 부분도 있다.

* 수동 확인 작업의 번거로움
    * add()에서 User 정보를 DB에 등록하고, 이를 다시 get()을 이용해 가져왔을 때 입력한 값과 가져온 값이 일치하는지를 테스트 코드는 확인해주지 않는다. 단지 콘솔에 값만 출력해줄 뿐이다.
	* 결국 그 콘솔에 나온 값을 보고 등록과 조회가 성공적으로 되고 있는지를 확인하는 건 사람의 책임이다.
* 실행 작업의 번거로움
    * 만약 DAO가 수백 개가 되고 그에 대한 main() 메소드도 그만큼 만들어진다면, 전체 기능을 테스트해보기 위해 main() 메소드를 수백 번 실행하는 수고가 필요하다.

이 두 가지 문제점을 개선해보자.

# UserDaoTest 개선

## 테스트 검증의 자동화
먼저 테스트 결과의 검증 부분을 코드로 만들어보자.

모든 테스트는 성공과 실패의 두 가지 결과를 가질 수 있다. 또 테스트의 실패는 테스트가 진행되는 동안에 에러가 발생해서 실패하는 경우와, 테스트 작업 중에 에러가 발생하진 않았지만 그 결과가 기대한 것과 다르게 나오는 경우로 구분해볼 수 있다. 여기서 전자를 테스트 에러, 후자를 테스트 실패로 구분해서 부르겠다.

테스트 중에 에러가 발생하는 것은 쉽게 확인이 가능하다. 콘솔에 에러 메시지와 긴 호출 스택 정보가 출력되기 때문이다. 하지만 테스트가 실패하는 것은 별도의 확인 작업과 그 결과가 있어야만 알 수 있다.

그러고보면 기존에 출력했던 "조회 성공"이라는 메세지는 사실 get() 메소드가 에러없이 끝났다는 의미이지 조회 테스트가 모두 성공했다는 뜻은 아니었다.

```
// 수정 전 테스트 코드
System.out.println(user2.getName());
System.out.println(user2.getPassword());
System.out.println(user2.getId() + "조회 성공");

// 수정 후 테스트 코드
if (!user.getName().equals(user2.getName())) {
	System.out.println("테스트 실패 (name)");
}
else if (!user.getPassword().equals(user2.getPassword())) {
	System.out.println("테스트 실패 (password)");
}
else {
	System.out.println("조회 테스트 성공");
}
```

이렇게 해서 테스트의 수행과 테스트 값 적용, 그리고 결과를 검증하는 것까지 모두 자동화했다. 이제 거의 모든 과정을 자동화한 테스트가 만들어졌다. 테스트를 수행하고 나서 할 일은 마지막 출력 메세지가 "테스트 성공"이라고 나오는지 확인하는 것 뿐이다.

> 자동화된 테스트를 위한 xUnit 프레임워크 개발자 켄트 백은 **테스트란 개발자가 마음 편하게 잠자리에 들 수 있게 해주는 것**이라고 했다.

## 테스트의 효율적인 수행과 결과 관리
이제 main() 메소드로 만든 테스트는 테스트로서 필요한 기능은 모두 갖춘 셈이다. 하지만 좀 더 편리하게 테스트를 수행하고 편리하게 결과를 확인하려면 단순한 main() 메소드로는 한계가 있다.

이미 자바에는 단순하면서도 실용적인 테스트를 위한 도구가 여러 가지 존재한다. 그 중에서도 프로그래머를 위한 자바 테스팅 프레임워크라고 불리는 JUnit은 유명한 테스트 지원 도구다. 지금까지 만들었던 main() 메소드 테스트를 JUnit을 이용해 다시 작성해보자.

### 테스트 메소드 전환
가장 먼저 할 일은 main() 메소드에 있던 테스트 코드를 일반 메소드로 옮기는 것이다. 새로 만들 테스트 메소드는 JUnit 프레임워크가 요구하는 조건 두 가지를 따라야 한다. 첫째는 메소드가 public으로 선언되야 하는 것이고, 다른 하나는 메소드에 @Test라는 애노테이션을 붙여주는 것이다.

```
public class UserDaoTest() {
	@Test
	public void addAndGet() throws SQLException {
		ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");

		UserDao dao = context.getBean("userDao", UserDao.class);
		....
	}
}
```

### 검증 코드 전환
다음 if 문은 처음 add()에 사용했던 user 오브젝트의 name 값과 get()에서 가져온 user2 오브젝트의 name 값이 같으면 다음으로 넘어가고, 아니면 테스트가 실패하게 한다. 이 if 문장의 기능을 JUnit이 제공해주는 assertThat이라는 스태틱 메소드를 이용해 변경할 수 있다.

```
if (!user.getName().equals(user2.getName())) {
	...
}

--> assertThat(user2.getName(), is(user.getName()));
```

assertThat()은 첫 번째 파라미터의 값을 뒤에 나오는 매처라고 불리는 조건으로 비교해서 일치하면 다음으로 넘어가고, 아니면 테스트가 실패하도록 만들어준다. is()는 매처의 일종으로 equals()로 비교해주는 기능을 가졌다.

JUnit은 예외가 발생하거나 assertThat()에서 실패하지 않고 테스트 메소드의 실행이 완료되면 테스트가 성공했다고 인식한다. "테스트 성공"이라는 메세지를 굳이 출력할 필요가 없다.

```
public class UserDaoTest() {
	@Test
	public void addAndGet() throws SQLException {
		ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");

		UserDao dao = context.getBean("userDao", UserDao.class);
		User user = new User();
		user.setId("gyumee");
		user.setName("박성철");
		user.setPassword("springno1");

		dao.add(user);

		User user2 = dao.get(uesr.getId());

		assertThat(user2.getName(), is(user.getName()));
		assertThat(user2.getPassword(), is(user.getPassword()));
	}
}
```

### JUnit 테스트 실행
JUnit 프레임워크도 자바 코드로 만들어진 프로그램이므로 어디선가 한 번은 JUnit 프레임워크를 시작시켜 줘야 한다. 어디에든 main() 메소드를 하나 추가하고, 그 안에 JUnitCore 클래스의 main 메소드를 호출해주는 간단한 코드를 넣어주면 된다. 메소드 파라미터에는 @Test 테스트 메소드를 가진 클래스의 이름을 넣어준다.

```
public static void main(String[] args) {
	JUnitCore.main("springbook.user.dao.UserDaoTest");
}
```

이 클래스를 실행하면 테스트를 실행하는 데 걸린 시간과 테스트 결과, 그리고 몇 개의 테스트 메소드가 실행됐는지를 알려준다.

# 개발자를 위한 테스팅 프레임워크 JUnit
JUnit은 사실상 자바의 표준 테스팅 프레임워크라고 불릴 만큼 폭넓게 사용되고 있다. 스프링의 핵심 기능 중 하나인 스프링 테스트 모듈도 JUnit을 이용한다. 

## JUnit 테스트 실행 방법
*IDE와 빌드툴(앤트, 메이븐)을 이용하여 테스트를 실행하는 방법을 기술하므로 생략*

## 테스트 결과의 일관성
지금까지 테스트를 실행하면서 가장 불편했던 일은, 매번 UserDaoTest 테스트를 실행하기 전에 DB의 USER 테이블 데이터를 모두 삭제해줘야 할 때였다. 깜빡 잊고 그냥 테스트를 실행했다가는 이전 테스트를 실행했을 때 등록됐던 사용자 정보와 기본키가 중복된다면서 add() 메소드 실행 중에 에러가 발생할 것이다.

여기서 생각해볼 문제는 테스트가 외부 상태에 따라 성공하기도 하고 실패하기도 한다는 점이다. 이는 좋은 테스트라고 할 수가 없다. 코드에 변경사항이 없다면 테스트는 항상 동일한 결과를 내야 한다.

### deleteAll()과 getCount() 추가
일관성 있는 결과를 보장하는 테스트를 만들기 위해 UserDao에 새로운 기능을 추가해보자.

**deleteAll()**

첫 번째 추가할 것은 USER 테이블의 모든 레코드를 삭제해주는 deleteAll() 메소드이다.

```
public void deleteAll() throws SQLException {
	Connection c = dataSource.getConnection();

	PreparedStatement ps = c.prepareStatement("delete from users");
	ps.executeUpdate();

	ps.close();
	c.close();
}
```

**getCount()**

두 번째 추가할 것은 USER 테이블의 레코드 개수를 돌려주는 getCount() 메소드다.

```
public int getCount() throws SQLException {
	Connection c = dataSource.getConnection();

	PreparedStatement ps = c.prepareStatement("select count(*) from users");

	ResultSet rs = ps.executeQuery();
	rs.next();
	int count = rs.getInt(1);

	rs.close();
	ps.close();
	c.close();

	return count;
}
```

### deleteAll()과 getCount()의 테스트
기존의 addAndGet() 테스트의 불편한 점은 실행 전에 수동으로 USER 테이블의 내용을 모두 삭제해줘야 하는 것이었다. deleteAll()을 이용하면 테이블의 모든 내용을 삭제할 수 있으니 이 메소드를 테스트가 시작될 때 실행해주자.

deleteAll()을 넣는 것만으로는 조금 부족하다. deleteAll() 자체도 아직 검증이 안 됐는데 무턱대고 다른 테스트에 적용할 수는 없다. 그래서 getCount()를 함께 적용해보자. deleteAll()이 기대한 대로 동작한다면, getCount()로 레코드의 개수를 가져올 경우 0이 나와야 한다.

```
public class UserDaoTest() {
	@Test
	public void addAndGet() throws SQLException {
		ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");

		UserDao dao = context.getBean("userDao", UserDao.class);

		dao.deleteAll();
		assertThat(dao.getCount(), is(0));

		User user = new User();
		user.setId("gyumee");
		user.setName("박성철");
		user.setPassword("springno1");

		dao.add(user);
		assertThat(dao.getCount(), is(1));

		User user2 = dao.get(uesr.getId());

		assertThat(user2.getName(), is(user.getName()));
		assertThat(user2.getPassword(), is(user.getPassword()));
	}
}
```

### 동일한 결과를 보장하는 테스트
단위 테스트는 항상 일관성 있는 결과가 보장돼야 한다는 점을 잊어선 안된다. DB에 남아 있는 데이터와 같은 외부 환경에 영향을 받지 말아야 하는 것은 물론이고, 테스트를 실행하는 순서를 바꿔도 동일한 결과가 보장되도록 만들어야 한다.

## 포괄적인 테스트

### getCount() 테스트
getCount()에 대한 좀 더 꼼꼼한 테스트를 만들어보자. 이번에는 여러 개의 User를 등록해가면서 getCount()의 결과를 매번 확인해보겠다. 이 테스트 기능을 기존의 addAndGet() 메소드에 추가하는 건 별로 좋은 생각이 아니다. 테스트 메소드는 한 번에 한 가지 검증 목적에만 충실한 것이 좋다.

테스트를 만들기 전에 먼저 User 클래스에 한 번에 모든 정보를 넣을 수 있도록 초기화가 가능한 생성자를 추가하자.

```
public User(String id, String name, String password) {
	this.id = id;
	this.name = name;
	this.password = password;
}

// 자바빈 규약을 따르는 클래스에 생성자를 명시적으로 추가했을 때는
// 파라미터가 없는 디폴트 생성자도 함께 정의해주는 것을 잊지 말자
public User() {
}
```

```
@Test
public void count() throws SQLException {
	ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");

	UserDao dao = context.getBean("userDao", UserDao.class);
	User user1 = new User("gyumee", "박성철", "springno1");
	User user2 = new User("leegw700", "이길원", "springno2");
	User user3 = new User("bumjin", "박범진", "springno3");

	dao.deleteAll();
	assertThat(dao.getCount(), is(0));

	dao.add(user1);
	assertThat(dao.getCount(), is(1));

	dao.add(user2);
	assertThat(dao.getCount(), is(2));

	dao.add(user3);
	assertThat(dao.getCount(), is(3));
}
```

주의해야 할 점은 두 개의 테스트가 어떤 순서로 실행될지는 알 수 없다는 것이다. JUnit은 특정한 테스트 메소드의 실행 순서를 보장해주지 않는다. 테스트의 결과가 테스트 실행 순서에 영향을 받는다면 테스트를 잘못 만든 것이다.

### get() 예외조건에 대한 테스트
한 가지 더 생각해볼 문제가 있다. get() 메소드에 전달된 id 값에 해당하는 사용자 정보가 없다면 어떻게 될까? 이럴 땐 어떤 결과가 나오면 좋을까? 두 가지 방법이 있을 것이다. 하나는 null과 같은 특별한 값을 리턴하는 것이고, 다른 하나는 id에 해당하는 정보를 찾을 수 없다고 예외를 던지는 것이다. 각기 장단점이 있다. 여기서는 후자의 방법을 써보자

주어진 id에 해당하는 정보가 없다는 의미를 가진 예외를 하나 정의할 수도 있지만, 그냥 스프링이 미리 정의해놓은 EmptyResultDataAccessException 예외를 가져다 쓰도록 하겠다.

코드를 만들기 전에, 먼저 이런 경우를 어떻게 테스트 코드로 만들지 생각해보자. 일반적으로는 테스트 중에 예외가 던져지면 테스트 메소드의 실행은 중단되고 테스트는 실패한다. 그런데 이번에는 반대로 테스트 진행 중에 특정 예외가 던져지면 테스트가 성공한 것이고, 예외가 던져지지 않고 정상적으로 작업을 마치면 테스트가 실패했다고 판단해야 한다.

바로 이런 경우를 위해 JUnit은 예외조건 테스트를 위한 특별한 방법을 제공해준다.

```
// 테스트 중에 발생할 것으로 기대하는 예외 클래스를 지정해준다.
@Test(expected=EmptyResultDataAccessException.class)
public void getUserFailure() throws SQLException {
	ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");

	UserDao dao = context.getBean("userDao", UserDao.class);
	dao.deleteAll();
	assertThat(dao.getCount(), is(0));

	// 이 메소드 실행 중에 예외가 발생해야 한다.
	// 예외가 발생하지 않으면 테스트가 실패한다.
	dao.get("unknown_id");
}
```

이 테스트를 실행시키면 어떻게 될까? 당연히 테스트는 실패한다. 아직 UserDao 코드에는 손을 대지 않았으니 실패하는 것이 당연하다.

### 테스트를 성공시키기 위한 코드의 수정

```
public User get(String id) throws SQLException {
	...
	ResultSet rs = ps.executeQuery();

	// uUer는 null 상태로 초기화해놓는다.
	User user = null;
	if (rs.next()) {
		// id를 조건으로 한 쿼리의 결과가 있으면 User 오브젝트를 만들고 값을 넣어준다.
		user = new User();
		user.setId(rs.getString("id"));
		user.setName(rs.getString("name"));
		user.setPassword(rs.getString("password"));
	}

	rs.close();
	ps.close();
	c.close();
	
	// 결과가 없으면 User는 null 상태 그대로일 것이다.
	// 이를 확인해서 예외를 던져준다.
	if (user == null) throw new EmptyResultDataAccessException();

	return user;
}
```

테스트를 다시 실행해보면 이번엔 분명히 성공할 것이다.

### 포괄적인 테스트
개발자가 테스트를 직접 만들 때 자주 하는 실수가 하나 있다. 바로 성공하는 테스트만 골라서 만드는 것이다. 개발자는 머릿속으로 이 코드가 잘 돌아가는 케이스를 상상하면서 코드를 만드는 경우가 일반적이다. 

하지만 개발자도 조금만 신경을 쓰면 자신이 만든 코드에서 발생할 수 있는 다양한 상황과 입력 값을 고려하는 포괄적인 테스트를 만들 수 있다. 스프링의 창시자인 로드 존슨은 **항상 네거티브 테스트를 먼저 만들라**는 조언을 했다.

그래서 테스트를 작성할 때 부정적인 케이스를 먼저 만드는 습관을 들이는게 좋다.

## 테스트가 이끄는 개발
get() 메소드의 예외 테스트를 만드는 과정을 다시 돌아보면 한 가지 흥미로운 점을 발견할 수 있다. 작업한 순서를 잘 생각해보면 새로운 기능을 넣기 위해 UserDao 코드를 수정하고, 그런 다음 수정한 코드를 검증하기 위해 테스트를 만드는 순서로 진행한 것이 아니다. 반대로 테스트를 먼저 만들어 테스트가 실패하는 것을 보고 나서 UsreDao의 코드에 손을 대기 시작했다. 테스트할 코드도 안 만들어놓고 테스트 코드부터 만드는 것은 좀 이상하다고 생각할지 모르겠다. 그런데 이런 순서를 따라서 개발을 진행하는 구체적인 개발 전략이 실제로 존재한다.

### 테스트 주도 개발
만들고자 하는 기능의 내용을 담고 있으면서 만들어진 코드를 검증도 해줄 수 있도록 테스트 코드를 먼저 만들고, 테스트를 성공하게 해주는 코드를 작성하는 방식의 개발 방법이 있다. 이를 **테스트 주도 개발**(TDD, Test Driven Development)이라고 한다. *실패한 테스트를 성공시키기 위한 목적이 아닌 코드는 만들지 않는다*는 것이 TDD의 기본 원칙이다. 이 원칙을 따랐다면 만들어진 모든 코드는 빠짐없이 테스트로 검증된 것이라고 볼 수 있다.

TDD에서는 테스트를 작성하고 이를 성공시키는 코드를 만드는 작업의 주기를 가능한 한 짧게 가져가도록 권장한다. 테스트를 반나절 동안이나 만들고 오후 내내 테스트를 통과시키는 코드를 만드는 식의 개발은 그다지 좋은 방법이 아니다.

TDD를 하면 자연스럽게 단위 테스트를 만들 수 있다. 빠르게 자동으로 실행할 수 있는 단위 테스트가 아니고서는 이런 식의 개발은 거의 불가능하다.

## 테스트 코드 개선
지금까지 세 개의 테스트 메소드를 만들었다. 이쯤 해서 테스트 코드를 리팩토링해보자. UserDaoTest 코드를 잘 살펴보면 기계적으로 반복되는 부분이 눈에 띈다.

```
ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");
UserDao dao = context.getBean("userDao", UserDao.class);
```

중복된 코드는 별도의 메소드로 뽑아내는 것이 가장 손쉬운 방법이다. 그런데 이번에는 일반적으로 사용했던 메소드 추출 리팩토링 방법 말고 JUnit이 제공하는 기능을 활용해보겠다.

JUnit 프레임워크는 테스트 메소드를 실행할 때 부가적으로 해주는 작업이 몇 가지 있다. 그중에서 테스트를 실행할 때마다 반복되는 준비 작업을 별도의 메소드에 넣게 해주고, 이를 매번 테스트 메소드를 실행하기 전에 먼저 실행시켜주는 기능이다.

### @Before

```
public class UserDaoTest {
	// setUp() 메소드에서 만드는 오브젝트를 테스트 메소드에서 사용할 수 있도록 인스턴스 변수로 선언한다.
	private UserDao dao;

	// JUnit이 제공하는 에노테이션
	// @Test 메소드가 실행되기 전에 먼저 실행돼야 하는 메소드를 정의한다.
	@Before
	public void setUp() {
		ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");
		this.dao = context.getBean("userDao", UserDao.class);
	}

	@Test
	public void addAndGet() throws SQLException {
		....
	}


	@Test
	public void count() throws SQLException {
		....
	}

	@Test(expected=EmptyResultDataAccessException.class)
	public void getUserFailure() throws SQLException {
		....
	}
}
```

JUnit 프레임워크가 테스트 메소드를 실행하는 과정을 알아보자.

1. 테스트 클래스에서 @Test가 붙은 public이고 void형이며 파라미터가 없는 테스트 메소드를 모두 찾는다.
2. 테스트 클래스의 오브젝트를 하나 만든다.
3. @Before가 붙은 메소드가 있으면 실행한다.
4. @Test가 붙은 메소드를 하나 호출하고 테스트 결과를 저장해둔다.
5. @After가 붙은 메소드가 있으면 실행한다.
6. 나머지 테스트 메소드에 대해 2~5번을 반복한다.
7. 모든 테스트의 결과를 종합해서 돌려준다.

실제로는 더 복잡한데, 간단히 정리하면 위의 7 단계를 거쳐서 진행된다고 볼 수 있다.

**각 테스트 메소드를 실행할 때마다 테스트 클래스의 오브젝트를 새로 만든다는 점**을 기억하자. 한 번 만들어진 테스트 클래스의 오브젝트는 하나의 테스트 메소드를 사용하고 나면 버려진다. JUnit 개발자는 각 테스트가 서로 영향을 주지 않고 독립적으로 실행됨을 보장해주기 위해 매번 새로운 오브젝트를 만들게 했다.

### 픽스쳐
테스트를 수행하는 데 필요한 정보나 오브젝트를 **픽스처**라고 한다. 일반적으로 픽스처는 여러 테스트에서 반복적으로 사용되기 때문에 @Before 메소드를 이용해 생성해두면 편리하다. UserDaoTest에서는 dao가 대표적인 픽스처다. 테스트 중에 add() 메소드에 전달하는 User 오브젝트들도 픽스처라고 볼 수 있다. 이 부분도 테스트 메소드에서 중복된 코드가 보인다. 중복을 제거하기 위해 @Before 메소드로 추출해보자.

```
public class UserDaoTest{
	private UserDao dao;
	private User user1;
	private User user2;
	private User user3;

	@Before
	public void setUp() {
		this.user1 = new User("gyumee", "박성철", "springno1");
		this.user2 = new User("leegw700", "이길원", "springno2");
		this.user3 = new User("bumjin", "박범진", "springno3");
	}
}
```

# 스프링 테스트 적용
테스트 코드를 어느 정도 깔끔하게 정리하였지만 아직 찜찜한 부분이 남아 있는데, 바로 애플리케이션 컨텍스트 생성 방식이다. @Before 메소드가 테스트 메소드 개수만큼 반복되기 때문에 애플리케이션 컨텍스트도 세 번 만들어진다. 애플리케이션 컨텍스트가 만들어질 때는 모든 싱글톤 빈 오브젝트를 초기화한다. 단순히 빈 오브젝트를 만드는 정도라면 상관없지만, 어떤 빈은 오브젝트가 생성될 때 자체적인 초기화 작업을 진행해서 제법 많은 시간을 필요로 한다.

테스트는 가능한 한 독립적으로 매번 새로운 오브젝트를 만들어서 사용하는 것이 원칙이다. 하지만 애플리케이션 컨텍스트처럼 생성에 많은 시간과 자원이 소모되는 경우에는 테스트 전체가 공유하는 오브젝트를 만들기도 한다. 이 때도 테스트는 일관성 있는 실행 결과를 보장해야 하고, 테스트의 실행 순서가 결과에 영향을 미치지 않아야 한다. 다행히도 애플리케이션 컨텍스트는 초기화되고 나면 내부의 상태가 바뀌는 일은 거의 없다. 따라서 애플리케이션 컨텍스트는 한 번만 만들고 여러 테스트가 공유해서 사용해도 된다.

## 테스트를 위한 애플리케이션 컨텍스트 관리
스프링은 JUnit을 이용하는 테스트 컨텍스트 프레임워크를 제공한다. 테스트 컨텍스트의 지원을 받으면 간단한 애노테이션 설정만으로 테스트에서 필요로 하는 애플리케이션 컨텍스트를 만들어서 모든 테스트가 공유하게 할 수 있다.

### 스프링 테스트 컨텍스트 프레임워크 적용
먼저 @Before 메소드에서 애플리케이션 컨텍스트를 생성하는 코드를 제거한다. 그리고 ApplicationContext 타입의 인스턴스 변수를 선언하고 스프링이 제공하는 @Autowired 애노테이션을 붙여준다. 

```
@RunWith(SpringJUnit4ClassRunner.class)	// 스프링의 테스트 컨텍스트 프레임워크의 JUnit 확장기능 지정
@ContextConfiguration(locations="/applicationContext.xml")	// 테스트 컨텍스트가 자동으로 만들어줄 애플리케이션 컨텍스트의 위치 지정
public class UserDaoTest {
	// 테스트 오브젝트가 만들어지고 나면 스프링 테스트 컨텍스트에 의해 자동으로 값이 주입된다.
	@Autowired
	private ApplicationContext context;
	....

	@Before
	public void setUp() {
		this.dao = this.context.getBean("userDao", UserDao.class);
		....
	}
}
```

@RunWith는 JUnit 프레임워크의 테스트 실행 방법을 확장할 때 사용하는 애노테이션이다. SpringJUnit4ClassRunner 라는 JUnit용 테스트 컨텍스트 프레임워크 확장 클래스를 지정해주면 JUnit이 테스트를 진행하는 중에 테스트가 사용할 애플리케이션 컨텍스트를 만들고 관리하는 작업을 진행해준다.

@ContextConfiguration는 자동으로 만들어줄 애플리케이션 컨텍스트의 설정파일 위치를 지정한 것이다.

인스턴스 변수 context는 어디에서도 초기화해주는 코드가 없다. 스프링의 JUnit 확장기능은 테스트가 실행되기 전에 딱 한 번만 애플리케이션 컨텍스트를 만들어두고, 테스트 오브젝트가 만들어질 때마다 특별한 방법을 이용해 애플리케이션 컨텍스트 자신을 테스트 오브젝트의 특정 필드에 주입해준다.

### 테스트 클래스의 컨텍스트 공유
스프링 테스트 컨텍스트 프레임워크의 기능은 하나의 테스트 클래스 안에서 애플리케이션 컨텍스트를 공유해주는 것이 전부가 아니다. 여러 개의 테스트 클래스가 있는데 모두 같은 설정파일을 가진 애플리케이션 컨텍스트를 사용한다면, 스프링은 테스트 클래스 사이에서도 애플리케이션 컨텍스트를 공유하게 해준다.

물론 테스트 클래스마다 다른 설정파일을 사용하도록 만들어도 되고, 몇 개의 테스트에서만 다른 설정파일을 사용할 수도 있다. 스프링은 설정파일의 종류만큼 애플리케이션 컨텍스트를 만들고, 같은 설정파일을 지정한 테스트에서는 이를 공유하게 해준다.

### @Autowired
@Autowired는 스프링의 DI에 사용되는 특별한 애노테이션이다. @Autowired 가 붙은 인스턴스 변수가 있으면, 테스트 컨텍스트 프레임워크는 변수 타입과 일치하는 컨텍스트 내의 빈을 찾는다. 타입이 일치하는 빈이 있으면 인스턴스 변수에 주입해준다. 일반적으로는 주입을 위해서는 생성자나 수정자 메소드 같은 메소드가 필요하지만, 이 경우에는 메소드가 없어도 주입이 가능하다.

앞에서 만든 테스트 코드에서는 applicationContext.xml 파일에 정의된 빈이 아니라, ApplicationContext 라는 타입의 변수에 @Autowired 를 붙였는데 애플리케이션 컨텍스트가 DI됐다. 어떻게 된 일일까? 스프링 **애플리케이션 컨텍스트는 초기화할 때 자기 자신도 빈으로 등록**한다. 따라서 애플리케이션 컨텍스트에는 ApplicationContext 타입의 빈이 존재하는 셈이고 DI도 가능한 것이다.

@Autowired 를 이용해 애플리케이션 컨텍스트가 갖고 있는 빈을 DI 받을 수 있다면 굳이 컨텍스트를 가져와 getBean()을 사용하는 것이 아니라, 아예 UserDao 빈을 직접 DI 받을 수도 있을 것이다.

```
public class UserDaoTest {
	@Autowired
	UserDao dao;   // UserDao 타입 빈을 직접 DI 받는다.
	....
}
```

애플리케이션 컨텍스트를 DI 받아서 다시 DL 방식으로 UserDao를 가져올 때보다 테스트 코드가 더 깔끔해졌다.

@Autowired 는 변수에 할당 가능한 타입을 가진 빈을 자동으로 찾는다. 단, 같은 타입의 빈이 두 개 이상 있는 경우에는 타입만으로는 어떤 빈을 가져올지 결정할 수 없다. 타입으로 가져올 빈 하나를 선택할 수 없는 경우에는 변수의 이름과 같은 이름의 빈이 있는지 확인한다. 변수 이름으로도 빈을 찾을 수 없는 경우에는 예외가 발생한다.

## DI와 테스트

### 테스트 코드에 의한 DI
DI는 애플리케이션 컨텍스트 같은 스프링 컨테이너에서만 할 수 있는 작업이 아니다. UserDao에는 DI 컨테이너가 의존관계 주입에 사용하도록 수정자 메소드를 만들어뒀다. 이 수정자 메소드는 평범한 자바 메소드이므로 테스트 코드에서도 얼마든지 호출해서 사용할 수 있다. 따라서 테스트 코드 내에서 이를 이용해서 직접 DI해도 된다. UserDao가 사용할 DataSource 오브젝트를 테스트 코드에서 변경할 수 있다는 뜻이다.

테스트용 DB에 연결해주는 DataSource를 테스트 내에서 직접 만들어보자. DataSource 구현 클래스는 스프링이 제공하는 가장 빠른 DataSource인 SingleConnectionDataSource를 사용해보자. 

```
....
// 테스트 메소드에서 애플리케이션 컨텍스트의 구성이나 상태를 변경한다는 것을 
// 테스트 컨텍스트 프레임워크에게 알려준다.
@DirtiesContext
public class UserDaoTest {
	@Autowired
	UserDao dao;

	@Before
	public void setUp() {
		....
		// 테스트에서 UserDao가 사용할 DataSource 오브젝트를 직접 생성한다.
		DataSource dataSource = new SingleConnectionDataSource("jdbc:mysql://localhost/testdb", "spring", "book", true);

		// 코드에 의한 수동 DI
		dao.setDataSource(dataSource);
	}
}
```

이 방법의 장점은 XML 설정파일을 수정하지 않고도 테스트 코드를 통해 오브젝트 관계를 재구성할 수 있다는 것이다. 하지만 이 방식은 매우 주의해서 사용해야 한다. 이미 애플리케이션 컨텍스트에서 applicationContext.xml 파일의 설정정보를 따라 구성한 오브젝트를 가져와 의존관계를 강제로 변경했기 때문이다. 스프링 테스트 컨텍스트 프레임워크를 적용했다면 애플리케이션 컨텍스트는 테스트 중에 딱 한 개만 만들어지고 모든 테스트에서 공유해서 사용하므로 애플리케이션 컨텍스트의 구성이나 상태를 테스트 내에서 변경하지 않는 것이 원칙이다.

그래서 UserDaoTest에는 @DirtiesContext라는 애노테이션을 추가해줬다. 이 애노테이션은 스프링의 테스트 컨텍스트 프레임워크에게 해당 클래스의 테스트에서 애플리케이션 컨텍스트의 상태를 변경한다는 것을 알려준다. 테스트 컨텍스트는 이 애노테이션이 붙은 테스트 클래스에는 애플리케이션 컨텍스트 공유를 허용하지 않는다.

### 테스트를 위한 별도의 DI 설정
테스트 코드에서 빈 오브젝트에 수동으로 DI 하는 방법은 장점보다 단점이 많다. 코드가 많아져 번거롭기도 하고 애플리케이션 컨텍스트도 매번 새로 만들어야 하는 부담이 있다.

아예 테스트에서 사용될 DataSource 클래스가 빈으로 정의된 테스트 전용 설정파일을 따로 만들어두는 방법을 이용해보자. 즉 두 가지 종류의 설정파일을 만들어서 하나에는 서버에서 운영용으로 사용할 DataSource를 빈으로 등록해두고, 다른 하나에는 테스트에 적합하게 준비된 DB를 사용하는 가벼운 DataSource가 빈으로 등록되게 만드는 것이다.

기존의 applicationContext.xml을 복사해서 다른 빈 설정은 그대로 두고 dataSource 빈의 설정을 테스트용으로 바꿔주자. 그리고 UserDaoTest의 @ContextConfiguration 애노테이션에 있는 locations 엘리먼트의 값을 새로 만든 테스트용 설정파일로 변경해준다. 이제 번거롭게 수동 DI하는 코드나 @DirtiesContext도 필요 없다.

### 컨테이너 없는 DI 테스트
마지막으로 살펴볼 DI를 테스트에 이용하는 방법은 아예 스프링 컨테이너를 사용하지 않고 테스트를 만드는 것이다.

UserDaoTest는 사실 UserDao 코드가 DAO로서 DB에 정보를 잘 등록하고 잘 가져오는지만 확인하면 된다. 스프링 컨테이너에서 UserDao가 동작함을 확인하는 일은 UserDaoTest의 기본적인 관심사가 아니다.

```
public class UserDaoTest {
	UserDao dao;   // @Autowired가 없다.

	@Before
	public void setUp() {
		....
		// 오브젝트 생성, 관계설정 등을 모두 직접 해준다.
		dao = new UserDao();
		DataSource dataSource = new SingleConnectionDataSource("jdbc:mysql://localhost/testdb", "spring", "book", true);
		dao.setDataSource(dataSource);
	}
}
```

테스트를 위한 DataSource를 직접 만드는 번거로움은 있지만 애플리케이션 컨텍스트를 아예 사용하지 않으니 코드는 더 단순해지고 이해하기 편해졌다.

이런 가볍고 깔끔한 테스트를 만들 수 있는 이유는 DI를 적용했기 때문이다. DI는 객체지향 프로그래밍 스타일이다. 따라서 DI를 위해 컨테이너가 반드시 필요한 것은 아니다. DI 컨테이너나 프레임워크는 DI를 편하게 적용하도록 도움을 줄 뿐, 컨테이너가 DI를 가능하게 해주는 것은 아니다.

> **침투적 기술과 비침투적 기술**
>
> 침투적 기술은 기술을 적용했을 때 어플리케이션 코드에 기술 관련 API가 등장하거나, 특정 인터페이스나 클래스를 사용하도록 강제하는 기술을 말한다. *침투적 기술을 사용하면 어플리케이션 코드가 해당 기술에 종속되는 결과*를 가져온다. 
>
> 반면에 *비침투적인 기술*은 어플리케이션 로직을 담은 코드에 아무런 영향을 주지 않고 적용이 가능하다. 따라서 *기술에 종속적이지 않은 순수한 코드를 유지*할 수 있게 해준다. 스프링은 이런 비침투적인 기술의 대표적인 예다. 그래서 스프링 컨테이너 없는 DI 테스트도 가능한 것이다.

### DI를 이용한 테스트 방법 선택
그렇다면 DI를 테스트에 이용하는 세가지 방법 중 어떤 것을 선택해야 할까? 세 가지 방법 모두 장단점이 있고 상황에 따라 유용하게 쓸 수 있다.

항상 스프링 컨테이너 없이 테스트할 수 있는 방법을 가장 우선적으로 고려하자. 이 방법이 테스트 수행 속도가 가장 빠르고 테스트 자체가 간결하다. 테스트를 위해 필요한 오브젝트의 생성과 초기화가 단순하다면 이 방법을 가장 먼저 고려해야 한다.

여러 오브젝트와 복잡한 의존관계를 갖고 있는 오브젝트를 테스트해야 할 경우가 있다. 이때는 스프링의 설정을 이용한 DI 방식의 테스트를 이용하면 편리하다.

테스트 설정을 따로 만들었다고 하더라도 때로는 예외적인 의존관계를 강제로 구성해서 테스트해야 할 경우가 있다. 이 때는 컨텍스트에서 DI받은 오브젝트에 다시 테스트 코드로 수동 DI해서 테스트하는 방법을 사용하면 된다. 테스트 메소드나 클래스에 @DirtiesContext 애노테이션을 붙이는 것을 잊지 말자.

# 학습 테스트로 배우는 스프링
개발자는 때로 자신이 만들지 않은 프레임워크나 다른 개발팀에서 만들어서 제공한 라이브러리 등에 대해서도 테스트를 작성해야 한다. 이런 테스트를 *학습 테스트*라고 한다.

학습 테스트의 목적은 자신이 사용할 API나 프레임워크의 기능을 테스트로 보면서 사용 방법을 익히려는 것이다. 따라서 테스트이지만 프레임워크나 기능에 대한 검증이 목적이 아니다. 오히려 자신이 테스트를 만들려고 하는 기술이나 기능에 대해 얼마나 제대로 이해하고 있는지, 그 사용 방법을 바로 알고 있는지를 검증하려는 게 목적이다.

## 학습 테스트의 장점

**다양한 조건에 따른 기능을 손쉽게 확인해볼 수 있다**

자동화된 테스트의 모든 장점이 학습 테스트에도 그대로 적용된다. 예제를 만들면서 학습하는 것은 수동 테스트와 성격이 비슷하다. 반면에 학습 테스트는 자동화된 테스트 코드로 만들어지기 때문에 다양한 조건에 따라 기능이 어떻게 동작하는지 빠르게 확인할 수 있다.

**학습 테스트 코드를 개발 중에 참고할 수 있다**

수동으로 예제를 만드는 방법은 코드를 계속 수정해가면서 기능을 확인해보기 때문에 결국 최종 수정한 예제 코드만 남아 있다. 반면에 학습 테스트는 다양한 기능과 조건에 대한 테스트 코드를 개별적으로 만들고 남겨둘 수 있다. 이렇게 테스트로 새로운 기술의 다양한 기능을 사용하는 코드를 만들어두면 실제 개발에서 샘플 코드로 참고할 수 있다.

**프레임워크나 제품을 업그레이드할 때 호환성 검증을 도와준다**

요즘은 모든 제품이 매우 빠르게 업데이트된다. 문제는 이렇게 새로운 버전으로 업그레이드를 할 때 API 사용법에 미묘한 변화가 생긴다거나, 기존에는 잘 동작하던 기능에 문제가 발생할 수도 있다는 점이다. 학습테스트에 새로운 버전의 프레임워크나 제품을 먼저 적용하여 기존에 사용하던 API가 기능에 문제가 없다는 사실을 미리 확인해 볼 수 있다.

**테스트 작성에 대한 좋은 훈련이 된다**

개발자가 테스트를 작성하는 데 아직 충분히 훈련되어 있지 않거나 부담을 갖고 있다면, 먼저 학습 테스트를 작성해보면서 테스트 코드 작성을 연습할 수 있다.

**새로운 기술을 공부하는 과정이 즐거워진다**

책이나 레퍼런스 문서 등을 그저 읽기만 하는 공부는 쉽게 지루해진다. 그에 비해 테스트 코드를 만들면서 하는 학습은 흥미롭고 재미있다.

## 학습 테스트 예제

### JUnit 테스트 오브젝트 테스트
JUnit은 테스트 메소드를 수행할 때마다 새로운 오브젝트를 만든다고 했다. 그런데 정말 매번 새로운 오브젝트가 만들어질까? JUnit으로 만드는 JUnit 자신에 대한 테스트를 만들어보자.

```
public class JUnitTest {
	// 테스트클래스 자신의 타입으로 스태틱 변수를 하나 선언
	static JUnitTest testObject;

	//
	// 매 테스트 메소드에서 현재 스태틱 변수에 담긴 오브젝트와 자신을 비교해서 같지 않다는 사실을 확인
	// 그리고 현재 오브젝트를 그 스태틱 변수에 저장
	//

	@Test public void test1() {
		assertThat(this, is(not(sameInstance(testObject))));
		testObject = this;
	}

	@Test public void test2() {
		assertThat(this, is(not(sameInstance(testObject))));
		testObject = this;
	}

	@Test public void test3() {
		assertThat(this, is(not(sameInstance(testObject))));
		testObject = this;
	}
}
```

한 가지 찜찜한 사실은 이 방식으로는 직전 테스트에서 만들어진 테스트 오브젝트와만 비교한다는 점이다. 만약 첫 번째와 세 번째 테스트 오브젝트가 같은 경우가 있다면 그것은 검증이 안된다.

```
public class JUnitTest {
	static Set<JUnitTest> testObjects = new HashSet<JUnitTest>();

	//
	// 테스트마다 현재 테스트 오브젝트가 컬렉션에 이미 등록되어 있는지 확인하고, 없으면 자기 자신을 추가한다.
	//

	@Test public void test1() {
		assertThat(testObjects, not(hasItem(this)));
		testObjects.add(this);
	}

	@Test public void test2() {
		assertThat(testObjects, not(hasItem(this)));
		testObjects.add(this);
	}

	@Test public void test3() {
		assertThat(testObjects, not(hasItem(this)));
		testObjects.add(this);
	}
}
```

### 스프링 테스트 컨텍스트 테스트
스프링의 테스트용 애플리케이션 컨텍스트는 테스트 개수에 상관없이 한 개만 만들어지고, 모든 테스트에서 공유된다고 했다. 정말 그런지 검증하는 학습 테스트를 만들어보자. 기존 JUnitTest에 테스트 컨텍스트의 컨텍스트 생성 방식을 확인하는 기능을 추가한다.

```
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration
public class JUnitTest {
	// 테스트 컨텍스트가 매번 주입해주는 애플리케이션 컨텍스트는 
	// 항상 같은 오브젝트인지 테스트로 확인해본다.
	@Autowired ApplicationContext context;

	static Set<JUnitTest> testObjects = new HashSet<JUnitTest>();
	static ApplicationContext contextObject = null;

	@Test public void test1() {
		assertThat(testObjects, not(hasItem(this)));
		testObjects.add(this);

		assertTrue(contextObject == null || contextObject == this.context);
		contextObject = this.context;
	}

	@Test public void test2() {
		assertThat(testObjects, not(hasItem(this)));
		testObjects.add(this);

		assertTrue(contextObject == null || contextObject == this.context);
		contextObject = this.context;
	}

	@Test public void test3() {
		assertThat(testObjects, not(hasItem(this)));
		testObjects.add(this);

		assertTrue(contextObject == null || contextObject == this.context);
		contextObject = this.context;
	}
}
```

## 버그 테스트
버그 테스트란 코드에 오류가 있을 때 그 오류를 가장 잘 드러내줄 수 있는 테스트를 말한다. 버그 테스트는 일단 실패하도록 만들어야 한다. 버그가 원인이 되서 테스트가 실패하는 코드를 만드는 것이다. 그러고 나서 버그 테스트가 성공할 수 있도록 애플리케이션 코드를 수정한다. 테스트가 성공하면 버그는 해결된 것이다. 버그 테스트의 장점은 다음과 같다.

**테스트의 완성도를 높여준다**

기존 테스트에서는 미처 검증하지 못했던 부분이 있기 때문에 오류가 발생한 것이다. 이에 대해 테스트를 만들면 불충분했던 테스트를 보완해준다.

**버그의 내용을 명확하게 분석하게 해준다**

버그가 있을 때 그것을 테스트로 만들어서 실패하게 하려면 어떤 이유 때문에 문제가 생겼는지 명확히 알아야 한다. 따라서 버그를 좀 더 효과적으로 분석할 수 있다.

**기술적인 문제를 해결하는 데 도움이 된다**

아무리 코드와 설정 등을 살펴봐도 별다른 문제가 없는 것 같이 느껴지거나 또는 기술적으로 다루기 힘든 버그를 발견하는 경우도 있다. 이럴 땐 동일한 문제가 발생하는 가장 단순한 코드와 그에 대한 버그 테스트를 만들어보면 도움이 된다.
