# Spring/Core

> 스프링 핵심원리에 대해 공부한 내용입니다.

1. [IoC와 DI](#ioc와-di)

## IoC와 DI

IoC와 DI를 보여주기 위해 간단한 주문 애플리케이션을 개발했다. 

![UML](https://user-images.githubusercontent.com/83766322/224700912-2c1d5b01-4425-4a90-92aa-f39424c75277.png)

UML 다이어그램을 보면 OrderService 구현체는 MemberRepository와 DiscountPolicy 인터페이스를 참조하고 있다. 따라서 OrderService 구현체는 다음과 같이 구현된다.

```java
public class OrderServiceImpl implements OrderService {

	private MemberRepository memberRepository = new MemoryMemberRepository();
	private DiscountPolicy discountPolicy = new FixedDiscountPolicy();
	
    // ...

}
```

하지만 위 코드는 객체지향적이지 못한 코드이다. 

- SRP 단일 책임 원칙
  
  한 클래스는 하나의 책임만 가져야 한다. 그런데 OrderService 구현체는 직접 구현 객체를 생성하고 연결하고 실행까지 한다. 따라서 클라이언트 객체가 너무 많은 책임을 가지고 있다.

- DIP 의존관계 역전 원칙
  
  프로그래머는 추상화에 의존해야지 구체화에 의존해서는 안된다. UML을 보면 OrderService 구현체는 인터페이스에 의존함으로써 원칙을 지켰다. 하지만 코드에서는 인터페이스가 아닌 구현체에 의존하고 있으므로 원칙을 지키지 못했다.

- OCP 개방-폐쇄 원칙

  소프트웨어는 확장에는 열려있으나 변경에는 닫혀있어야 한다. 위 코드는 다형성을 사용하여 확장에는 열려있다. 하지만  ```FixedDiscountPolicy```에서 ```RatedDiscountPolicy```로 변경하려면 아래와 같이 클라이언트 코드를 수정해야 한다.
  ```java
  public class OrderServiceImpl implements OrderService {

	private MemberRepository memberRepository = new MemoryMemberRepository();
	private DiscountPolicy discountPolicy = new RatedDiscountPolicy();
	
    // ...

  }
  ```

어떻게 하면 객체 지향적으로 바꿀 수 있을까? 먼저 관심사를 분리해야 한다. 애플리케이션을 크게 사용 영역과 구성 영역으로 분리해야 한다.

```java
public class AppConfig {
	
	public MemberRepository memberRepository() {
		return new MemoryMemberRepository();
	}
	
	public DiscountPolicy discountPolicy() {
		return new FixedDiscountPolicy();
	}
	
	public OrderService orderService() {
		return new OrderServiceImpl(memberRepository(), discountPolicy());
	}
	
}

public class OrderServiceImpl implements OrderService {

	private MemberRepository memberRepository;
	private DiscountPolicy discountPolicy;
	
	public OrderServiceImpl(MemberRepository memberRepository, DiscountPolicy discountPolicy) {
		this.memberRepository = memberRepository;
		this.discountPolicy = discountPolicy;
	}

    // ...
}
```

구성 영역에 해당하는 AppConfig 클래스가 등장하면서 객체 지향 원칙을 모두 지킬 수 있게 되었다. 

- SRP 단일 책임 원칙
  
  AppConfig가 구현 객체를 생성하고 연결해주는 역할을 담당하게 되었다. 따라서 클라이언트 객체는 그저 실행하는 것에만 집중하면 되므로 역할이 분명해졌다. 

- DIP 의존관계 역전 원칙
  
  OrderService 구현체는 이제 인터페이스에만 의존하고 있다. 하지만 클라이언트 코드는 인터페이스만으로 실행할 수 없다. 따라서 AppConfig가 의존 관계를 클라이언트 코드에 주입하고 있다.

- OCP 개방-폐쇄 원칙

  이제 ```FixedDiscountPolicy```에서 ```RatedDiscountPolicy```로 변경하기 위해 구성 영역만 변경할 뿐, 사용 영역은 변경하지 않는다. 즉, 클라이언트 코드는 변경할 필요가 없다.

이전 코드는 클라이언트 구현 객체가 스스로 필요한 서버 구현 객체를 생성, 연결, 실행하였다. 하지만 AppConfig가 등장하면서 클라이언트 구현 객체는 자신의 로직 실행만 담당한다. 이제 AppConfig가 프로그램의 제어 흐름을 가져갔다. AppConfig가 인터페이스의 다른 구현 객체를 생성하고 연결할 수 있다. 

이렇게 프로그램의 제어 흐름을 직접이 아닌 외부에서 관리하는 것을 **제어의 역전(Inversion of Control)** 이라고 한다.

의존 관계는 **정적인 클래스 의존 관계**와 **동적인 클래스 의존 관계** 둘로 나뉜다.

- 정적인 클래스 의존 관계
  
  애플리케이션을 실행하지 않아도 의존 관계를 파악할 수 있다. 아래 코드에서 OrderService 구현 객체는 MemberRepository와 DiscountPolicy 인터페이스를 의존하고 있음을 알 수 있다.
  
  ```java
  public class OrderServiceImpl implements OrderService {

	private MemberRepository memberRepository;
	private DiscountPolicy discountPolicy;
	
	// ...
  }
  ```

- 동적인 클래스 의존 관계
  
  애플리케이션 **실행 시점(runtime)** 에 생성된 객체 인스턴스를 참조하는 의존 관계이다.

애플리케이션 **런타임**에 외부에서 구현 객체를 생성하고 클라이언트에게 전달함으로써 클라이언트와 서버의 의존 관계가 연결되는 것을 **의존 관계 주입(Dependency Injection)** 이라고 한다. 

의존 관계 주입을 정적인 클래스 의존 관계를 변경하지 않고 동적인 클래스 의존 관계를 쉽게 변경할 수 있다.

마지막으로 AppConfig처럼 구현 객체를 생성하고 관리하고 의존 관계를 연결해 주는 것을 **IoC 컨테이너, DI 컨테이너**라고 한다.  