## IoC(Inversion of Control : 제어의 역전)

객체의 생성, 생명주기 관리, 의존성 조립의 주체가 개발자에게서 외부 컨테이너(Spring Container)로 넘어간 설계원칙을 말한다.

- 전통적인 제어(Before) : 개발자가 코드 내에서 new 키워드를 사용해 필요한 객체를 직접 생성하고 언제 메모리에 올릴지(생명주기)를 직접 통제했다.
- 역전된 제어(After) : 개발자는 인터페이스를 통해 “나 이거 필요해”라고 역할만 선언하고 실제 객체의 생성과 조립은 스프링 컨테이너가 알아서 수행한다.

## IoC를 대규모 서비스에서 무조건 쓰는 이유

코드가 그냥 돌아가게 만드는 것은 new키워드로 충분하다. 하지만 대규모 서비스에서는 유지보수성과 확장성 때문에 IoC가 필수적이다.

1. 극단적인 결합도 낮춤
    
    OrderService 내부에서 new MySQLRepository()를 하는 순간 두 클래스는 강한 결합이 생성된다. DB를 PostgreSQL로 바꿀 때 데이터베이스와 아무 상관 없는 OrderService코드까지 줄줄이 고쳐야 한다.
    
    → IoC를 적용하면 OrderService는 추상적인 인터페이스(OrderRepository)만 바라본다. 외부(스프링)에서 구현체를 갈아 끼워주기 때문에 기술 스펙이 변경되어도 기존 비즈니스 로직 코드는 단 한 줄도 수정되지 않는다.
    
2. 단위 테스트의 용이성
    
    CarService 내부의 순수한 “주행 로직”만 검증하고 싶은데 내부에 new GasolineEngine()이 박혀있으면 테스트를 돌릴 때마다 실제 엔진 객체가 무겁게 돌아가야 한다.(DB의 경우 실제 네트워크를 타고 DB에 접속해야 하는 문제 발생)
    
    → 객체의 생성이 외부에서 이루어지므로 테스트 코드 작성 시 스프링 컨테이너 대신 가짜 객체를 생성자로 넣을 수 있다. 무거운 외부 환경(DB, 외부 API)없이 순수한 자바 로직만 테스트할 수 있다.
    

## IoC와 DI

강한 결합

```jsx
public class CarService {
    // CarService가 본연의 책임 외에 "가솔린 엔진을 쓰겠다"는 선택과 "생성"의 책임까지 다 가짐
    private final GasolineEngine engine = new GasolineEngine();

    public void drive() {
        engine.start();
    }
}
```

→ 전기차로 바꾸려면 CarService코드를 고쳐야하기 때문에 좋지 않다.

→ 이 클래스는 그냥 “자동차를 주행시킨다”라는 책임 외에도 “가솔린엔진을 사용할 것이다”라는 책임과 “가솔린 엔진을 지금 시점에 생성한다”라는 생성의 책임까지 갖고 있다.

```jsx
// 1. 역할을 인터페이스로 추상화
public interface Engine {
    void start();
}

// 2. 구현체를 스프링컨테이거 관리하도록 등록. @Component를 이용해서 등록한다. 
@Component
@Primary
public class GasolineEngine implements Engine {
    @Override
    public void start() {
        System.out.println("가솔린 엔진 시동");
    }
}

@Component
@Qulifier
public class ElectricEngine implements Engine {
    @Override
    public void start() {
        System.out.println("전기 모터 가동 조용함");
    }
}
```

→ 엔진이라는 공통의 역할을 인터페이스로 빼내고 가솔리이든 전기든 각각의 구현체를 만들고 그 위에 @Component 라는 어노테이션을 붙인다. → 이 어노테이션이 붙어있으면 스프링에게 “내가 어플리케이션 시작할 때 이 객체들을 메모리에 미리 다 띄워놓고 관리해줘”라고 컨테이너에 권한을 위임하는 선언이다.

```jsx
// 3. Service는 구현을 직접 만들지 않고 외부에서 주입받는다.
@Service
public class CarService {

		// 구체적인 전기, 가솔린? 모릅니다. 엔진이라는 역할만 알고 있다.
    private final Engine engine;

		// 생성자를 통해서 외부(스프링 컨테이너)로부터 진짜 객체를 전달받습니다.
    public CarService(Engine engine) {
        this.engine = engine;
    }

    public void drive() {
        engine.start();
        System.out.println("자동차가 주행합니다.");
    }
}
```

스프링 컨테이너의 작동 : 애플리케이션이 켜질 때 @Component가 붙은 클래스들을 스캔해 창고(Bean 컨테이너)에 보관한다. 이후 CarService를 만들려고 보니 Engine이 필요하다는 것을 깨닫고 창고에 있던 ElectricEngine을 생성자 파라미터에 넣어준다.

## IoC와 SOLID(DIP)의 관계

- DIP(의존 역전 원칙) : 고수준 모듈(비즈니스 로직을 담은 CarService)은 저수준 모듈(구체적인 기술이 들어간 GasolineEngine)에 의존하면 안되고, 둘 다 추상화에 의존해야 한다.
    
    → 인터페이스를 사이에 두고 설계해라
    
- IoC(제어의 역전) : DIP를 지키기 위해 인터페이스를 만들었더라도 누군가는 프로그램 실행 시점에 실제 구현체(new ElectricEngine())를 만들어서 연결해 줘야 코드가 돌아간다. 실제 객체를 생성하고 꽂아주는 작업을 개발자 대신 프레임워크가 전담하는 전략이다.

## 정리

내가 하던 객체 생성과 관리를 스프링 컨테이너가 대신 해준다. → 구체적인 클래스에 의존하는 강한결합을 끊어내어 요구사항 변경에 유연하게 대처하고 실제 환경 없이도 빠르게 검증(테스트용이)할 수 있다.

IoC는 제어권이 넘어갔다는 전체적인 ‘설계 원칙’이고 그 원칙을 달성하기 위해 외부에서 객체를 주입해주는 구체적인 ‘방법론’이 DI(의존성 주입)이다.
