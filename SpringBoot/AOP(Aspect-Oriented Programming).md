## 핵심 관심사 vs 횡단 관심사(왜 분리해야 하는가?)

우리가 만드는 시스템의 요구사항은 성격에 따라 두 가지 관심사로 나뉜다.

- 핵심 관심사(비즈니스 로직)
    
    VIP회원 10%할인, 장바구니 총액 계산, 주문 생성 등 해당 클래스가 존재해야 하는 본질적인 이유이자 핵심 업무의 흐름이다.(OOP의 주 영역)
    
    “공통 기능들을 비즈니스 로직 안에 섞지 말고 아예 바깥으로 빼내서 하나의 독립된 모듈(Aspect)로 뭉쳐버리자. 그리고 나중에 프로그램이 실행될때 우리가 지정한 규칙에 맞춰서 스프링 프레임워크가 알아서 그 공통 기능들을 원래 코드 앞이나 뒤에 살짝 끼워넣게 만들자”
    
- 횡단 관심사(부가 기능)
    
    로깅, 트랜잭션 처리(@Transactional), 보안 및 권한 검사 등 특정 비즈니스에 속하지 않고 시스템 전반(Controller, Service, Repository)을 가로지르며 공통으로 필요한 기능이다.
    

#### 분리하지 않았을 때의 문제점

신규 개발자고 로직 이해하는게 목적이면 업무 흐름을 알아야 한다. 그러나 코드 열어보니 첫줄에 log.info(”주문 시작”) 찍히고 둘째줄에 try { db커넥션 열기 } 이런 로직들이 많이 있고 그 아래에도 권한 검사 코드나 다른 에러로그 finally{ 커넥션 닫기} 이런 로직들이 있으면 비즈니스 의도 자체를 파악하기 힘들다.

횡단 관심사를 OOP만으로 처리하려고 하면 모든 메서드에 `try-catch-finally`와 시간 측정, 로그 출력 코드를 수백 번 복사/붙여넣기 해야 한다.

## AOP의 구성요소

```jsx
// 나쁜 에시 : AOP가 적용되지 않아서 공통 코드가 비즈니스 로직에 침투한 케이스
@Service
public class OrderService {

    public void createOrder() {
		    // --- 공통 관심사 ---
        System.out.println("[LOG] 메서드 실행 시작: createOrder");
        long start = System.currentTimeMillis();

        try {
		        /// --- 핵심 비즈니스 로직 ---
            System.out.println("실제 주문 생성 로직 실행...");
            // db.save(order);

        } finally {
				    // --- 공통 관심사 ---
            long end = System.currentTimeMillis();
            System.out.println("[LOG] 메서드 실행 종료. 걸린 시간: " + (end - start) + "ms");
        }
    }

    public void cancelOrder() {
        // 똑같은 로그 코드와 try-finally 반복. 공통 관심사의 반복.
    }
}
```

이 코드를 보면 실제 중요한 주문 생성 로직은 한줄인데 로그를 찍고 성능 측정하고 하는 코드가 위 아래로 있다. 이런 메서드가 100개면 지저분한 코드가 100개가 될 수 있다. → 핵심 비즈니스 로직에 집중하기 어렵다.

```jsx
// AOP 적용한 예시
@Service
public class OrderService {

    public void createOrder() {
		    // 핵심 비즈니스 로직만 존재합니다.
        System.out.println("실제 주문 생성 로직 실행...");
        // db.save(order);

    }

    public void cancelOrder() {
        System.out.println("실제 주문 취소 로직 실행...");
    }
}
```

AOP관련한 코드는 한줄도 없다. 별도의 AOP관련 클래스를 만들고 거기에서 적용대상을 지정했기 때문이다.

**AOP는 부가 기능을 핵심 로직과 완전히 분리하여 “공통 기능을 한 곳에 모아두고(Aspect), 어디에(Pointcut), 언제(Advice)적용할지” 선언적으로 정의하는 기술이다.**

#### Aspect : 횡단 관심사(공통 기능)를 한군데 모아놓은 독립된 클래스이다.

```jsx
@Aspect // 공통 기능을 모아둔 클래스임을 선언
@Component // 스프링 빈으로 등록되어야 AOP가 동작
public class LoggingAspect {
		// 공통 로직과 적용 규칙이 들어갑니다
}
```

공통 기능 분리

#### Pointcut : 공통 기능을 “어느 계층, 어느 메서드에 넣을 것인가” 범위를 지정하는 규칙이다.

```jsx
// Pointcut : "어디에 적용할것인가?" 공통 로직은 따로 만들어뒀는데 이걸 어디서부터 어디까지 적용?
// "com.example 패키지 밑에 있는 모든 Service클래스의 모든 메소드에 적용하겠다"
@Pointcut("execution(* com.example..service..*(..))")
public void serviceLayer() {

}
```

→스프링에게 “내가 앞으로 지시할 공통 기능은 저기 저 서비스 패키지 안에 있는 클래스에게만 적용해라”

→코드를 찾아다니는게 아니라 정규표현식처럼 선언적으로 범위를 지정

#### Advice : 타겟 메서드가 실행되는 과정 중 어느 타이밍에 부가 기능을 꽂을 것인지를 결정하는 실제 로직

- `@Before`: 메서드 실행 전
- `@After`: 메서드 실행 후 (성공/실패 무관)
- `@AfterReturning`: 메서드가 정상적으로 결과를 반환했을 때
- `@AfterThrowing`: 메서드에서 예외가 터졌을 때
- `@Around` : 메서드 **실행 전과 후 모두를 감싸서** 제어 (`joinPoint.proceed()`로 실제 비즈니스 메서드를 호출할 타이밍을 직접 결정)

```jsx
		// advice 언제? 무엇을 할것인가?
		// @Around -> 메소드 실행 전후 모두를 감싸서 통제
		// 아까 지정한 serviceLayer() 규칙에 맞는 곳에 이 행동을 적용하겠다
    @Around("serviceLayer()")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {

        System.out.println("[LOG] 메서드 실행 시작: " + joinPoint.getSignature().getName());
        long start = System.currentTimeMillis();

        try {
		        // 여기가 핵심. 
		        // 실제 비즈니스 메소드를 호출하고 제어권을 넘깁니다.
            return joinPoint.proceed();

        } finally {
            long end = System.currentTimeMillis();
            System.out.println("[LOG] 메서드 실행 종료. 걸린 시간: " + (end - start) + "ms");
        }
    }
```

joinPoint.proceed();라는 코드를 기점으로 위쪽이 메소드 실행전, 아래쪽이 메소드 실행 후의 로직

→ 이제 이 상태로 스프링 구동하면 스프링 컨테이너 이  Aspect와 Pointcut을 읽어들여서 판단한다.

→ “아, OrderService 실행될때는 무조건 시간 측정 로직으로 앞뒤를 감싸야겠구나”라고 판단하고 스프링이 자동으로 적용을 해준다.

## 스프링 AOP의 내부 동작 원리 : 프록시

내 OrderService 코드에는 로그 코드가 한 줄도 없는데 대체 어떻게 실행 전후로 로그가 찍이는 걸까? 자바 바이트코드가 실행 중에 변조되는 걸까?

→ 스프링은 원본 코드를 절대 건드리지 않고 속임수를 쓴다.(프록시 패턴)

1. 스프링 부트가 켜지면서 컨테이너가 @Service인 진짜 OrderService 객체를 메모리에 만든다.
2. 그냥 내놓으려다가 컨테이너 내부의 빈 후처리기가 @Aspect 설정들을 확인한다. “어? OrderService는 아까 그 LoggingAspect 규칙(Pointcut)에 걸리는 녀석이네?”라고 알아챈다.
3. 스프링은 진짜 OrderService 대신, 겉모습은 완벽히 똑같지만 내부에는 부가 기능이 발려 있는 가짜 대리 객체(Proxy)를 힙 메모리에 하나 더 찍어낸다.
4. OrderController가 의존성 주입(DI)을 요청할 때 스프링은 진짜 객체 대신 이 가짜 프록시 객체를 주입해준다.

```jsx
[OrderController]
       │
       ▼ (createOrder() 호출)
┌──────────────────────────────────────────────┐
│ [가짜 프록시 객체 (스프링이 만든 대리인)]             │
│                                              │
│ 1. [Advice] 로그 찍고 타이머 시작! ([LOG] 시작)    │
│                                              │
│ 2. joinPoint.proceed() 호출 ───────────────── ┼───┐
└──────────────────────────────────────────────┘   │
                                                   ▼
┌──────────────────────────────────────────────────┐
│ [진짜 OrderService 객체]                           │
│                                                  │
│ 3. 순수한 비즈니스 로직 실행 (db.save(order))          │
└──────────────────────────────────────────────────┘
                                                   │
┌──────────────────────────────────────────────┐   │
│ [가짜 프록시 객체 (스프링이 만든 대리인)]             │ <─┘
│                                              │
│ 4. [Advice] 타이머 종료 및 걸린 시간 로그 출력       │
│ 5. 최종 결과를 Controller에게 반환                │
└──────────────────────────────────────────────┘
```

이 프록시 매커니즘 덕분에 우리의 비즈니스 로직 코드는 단 한줄로 오염이 되지 않고 유지할 수 있다. 우리가 매일 쓰는 @Transactional(트랜잭션 열고 닫기), Spring Security(권한 체크)도 전부 이 프록시 AOP위에서 돌아간다.

## OOP vs AOP의 관계 : 대립이 아닌 상호보완

- OOP(Object-Oriented Programmming) : 비즈니스 요구사항을 기반으로 시스템을 모듈화하고 세로(수직)로 기둥을 세워 역할 나누는 방식(도메인 중심 설계
- AOP(Aspect-Oriented Programming) : OOP로 기둥을 세워놨더니 그 기둥들을 가로(횡단)로 뚫고 지나가는 공통 부가 기능들이 생겼을 때 이를 깔끔하게 해결해 주는 방식

→ AOP는 OOP를 대체하는 기술이 아니라 OOP가 횡단 관심사 때문에 코드가 중복되고 지저분해지는 약점을 완벽하게 보완해주는 파트너이다.

## 정리

- AOP는 비즈니스 로직과 로깅, 트랜잭션 같은 횡단 관심사를 분리하여 부가 기능을 모듈화하는 프로그래밍 패러다임이다. 이를 통해 핵심 비즈니스 로직의 가독성을 높이고 코드의 중복을 줄일 수 있다.
- 스프링 AOP는 원본 자바 코드를 변형하지 않고 런타임 프록시 메커니즘을 사용해 동작한다.
    
    애플리케이션 구동시 빈 후처리기가 AOP 적용 대상인 객체를 감싸는 가짜 프록시 객체를 생성하고 이를 DI 컨테이너에 등록한다.
    
    클라이언트의 요청을 프록시가 먼저 받아 Advice로 정의된 부가 기능을 실행한 뒤, 진짜 객체의 비즈니스 메서드를 호출하는 방식으로 동작한다.
