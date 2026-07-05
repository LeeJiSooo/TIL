## 가져가야 할 핵심

- Mockito는 가짜 객체 코드 몇줄로 만들어 주는 도구이다. → 이미 아는 개념이 어떻게 코드가 되는지를 본다.
- 검증에는 두 종류가 있다. 테스트 대상이 내놓은 결과값이 맞는지 확인하는 검증, 메서드를 제대로 호출했는지 확인하는 검증
- 어떤 Mockito코드를 보든 테스트 코드를 보든 문법을 먼저 보지 말고 “지금 테스트 하는 대상이 무엇인가?” “격리하는 의존성이 무엇인가” “무엇을 검증하려고 하는가”에 대해서 이해를 먼저 하는 게 필요하다.

## Mockito의 정의

- 자바 개발 생태계에서 가장 널리 쓰이는 테스트 더블(가짜 객체) 생성 및 검증 프레임워크이다.
- 가짜 객체를 자동으로 만들어주고, 그 객체가 어떤 상황에서 어떻게 답변할지 정하며, 테스트 대상이 그 객체와 똑바로 상호작용했는지를 검증한다.
- 도입 이유(개발자 편의성) : 테스트 더블을 일일이 손으로 코딩하는 귀찮은 작업을 자동화하여, 개발자가 오직 “핵심 비즈니스 로직과 예외 상황 검증”에만 집중할 수 있는 환경을 만들어 준다.

## 사용 방법

어떤 Mockito코드를 보든 복잡한 문법보다 “1) 테스트 대상이 무엇인가? 2) 격리할 의존성이 무엇인가? 3) 무엇을 검증(상태 vs 행위)하려는가?”를 짚어내는 게 중요하다.

#### 가짜 객체 생성(Mock 객체)

```java
@ExtendWith(MockitoExtension.class)
class CarServiceTest {
    @Mock // 테스트가 시작될때 mockito가 알아서 이 필드를 가짜객체로 자동으로 만들어준다.
    private CarRepository carRepository; // CarRepository의 Mock 객체 생성
    // ...
}
```

→ 의존 객체 위에 Mock 어노테이션을 붙일 수 있다.

```java
@Test
void myTest() {
    CarRepository carRepository = Mockito.mock(CarRepository.class);
    // ...
}
```

→ 테스트 메서드 안에 Mockito.mock을 직접 호출해서 처리하는 것도 가능하다.

→ 진짜 carRepository가 아니라 우리가 통제할 수 있는 carRepository가 된다.

#### 가짜 답변(Stubbing)

가짜객체를 만들었으니까 이게 어떻게 동작 할지를 결정해줘야 한다.

```java
// carRepository.findById("avante")가 호출되면 expectedCar 객체를 반환하도록 설정
when(carRepository.findById("avante")).thenReturn(Optional.of(expectedCar));
```

→ carRepository의 findById가 avante라는 인자를 호출되면 미리 준비한 자동차 객체를 담은 expectedCar를 돌려줘라.

→ 의존 객체의 답을 우리가 통제해서 테스트 대상을 안정적으로 검증이 가능하게 한다.

#### 행위 검증(Verification)

스터빙으로 상황 만들고 테스트 대상 실행한 다음에 마지막으로 검증한다.

→ SUT의 로직이 다 실행된 후, 외부 의존 객체를 정말 올바르게 호출했는지 행동 과정을 추적하는 단계이다.

```java
// carRepository의 findById 메서드가 "avante" 인자와 함께 1번 호출되었는지 검증
verify(carRepository, times(1)).findById("avante");
```

→ carRepo의 findById메서드가 Avante라는 인자와 함께 정확히 한 번 호출됐는지를 확인

→ 호출 행위 자체를 확인

**검증 대상 프로덕션 코드**

```java
public class CarService {
    private final CarRepository carRepository; // 격리해야 할 외부 의존성(DB)

    public CarService(CarRepository carRepository) {
        this.carRepository = carRepository;
    }

    public Car getCarDetails(String id) {
        return carRepository.findById(id)
                .orElseThrow(() -> new CarNotFoundException("차량을 찾을 수 없습니다."));
    }
}
```

Mockito 기반 단위 테스트 코드

```java
@ExtendWith(MockitoExtension.class) // 1. 나 Mockito 프레임워크 쓸게 선언
class CarServiceTest {

    @Mock
    private CarRepository carRepository;   // DB 연동 저장소를 가짜 대역으로 마킹

    @InjectMocks
    private CarService carService;         // 가짜 carRepository가 자동으로 내장된 진짜 테스트 대상 생성

    @Test
    @DisplayName("차량 ID로 조회 시, 차량 정보가 성공적으로 반환되어야 한다 (성공 케이스)")
    void getCarDetails_Success() {
        // [Given] 준비 단계
        Car expectedCar = new Car("Avante", 2022);
        // 스터빙(Stubbing): 가짜 객체의 대답을 내가 완벽히 통제한다.
        when(carRepository.findById("avante")).thenReturn(Optional.of(expectedCar));

        // [When] 실행 단계
        Car foundCar = carService.getCarDetails("avante");

        // [Then] 검증 단계
        assertNotNull(foundCar);                     // 1. 상태 검증: 결과물이 null이 아니어야 함
        assertEquals("Avante", foundCar.getName());  // 2. 상태 검증: 이름이 진짜 아반떼인지 확인
        verify(carRepository, times(1)).findById("avante"); // 3. 행위 검증: 메서드가 1번 실행되었는지 감시
    }

    @Test
    @DisplayName("존재하지 않는 차량 ID로 조회 시, 예외가 발생해야 한다 (예외 케이스)")
    void getCarDetails_NotFound_ThrowsException() {
        // [Given] 준비 단계
        // 차를 찾지 못해 빈 상자(Optional.empty())가 튀어나오는 고의적인 실패 상황을 스터빙함
        when(carRepository.findById("unknown")).thenReturn(Optional.empty());

        // [When & Then] 실행 및 검증 단계
        // SUT의 약속대로 CarNotFoundException이 정상 발생해서 방어막이 작동하는지 확인
        assertThrows(CarNotFoundException.class, () -> {
            carService.getCarDetails("unknown");
        });

        verify(carRepository, times(1)).findById("unknown"); // 예외가 터지더라도 조회가 수행되긴 했는지 행위 확인
    }
}
```

## 알아두면 좋은 정보

#### 스터빙으로 예외를 발생시키는 방법

```java
when(carRepository.findById("error-id")).thenThrow(new RuntimeException("DB 연결 오류"));
```

실제 테스트 환경에서 멀쩡한 DB를 끊을 수는 없으니까 thenThrow를 쓰면 “DB 커넥션이 끊긴 상황”을 코드로 재현할 수 있다. 

→ 시스템이 예외 상황을 먹통 안 되고 유연하게 복구하는지 검증할 때 필요하다.

#### 넓은 범위로 인자 매칭하기

```java
// 어떤 문자열 ID로 호출되든 항상 expectedCar를 반환하도록 설정
when(carRepository.findById(anyString())).thenReturn(Optional.of(expectedCar));

// Car 타입의 어떤 객체로 save 메서드가 호출되었는지 검증
verify(carRepository).save(any(Car.class));
```

스터빙할 때 굳이 정확히 ‘avante’라는 문자열을 안 따지고, “어떤 문자열 ID(anyString())가 파라미터로 들어오든 상관 없이 차를 리턴해라” 혹은 “어떤 Car객체(any(Car.class))든 상관없이 save메서드가 호출되면 패스시켜라”처럼 인자의 구체적인 값이 테스트의 핵심 관심사가 아닐 때 코드를 유연하고 간결하게 만들어 준다.

→ 과도하게 쓰는 건 의도가 불분명해지고 정확한 테스트가 안될 수도 있다.

#### 가독성을 극대화하는 BDD Mockito

```java
import static org.mockito.BDDMockito.*;

// given
given(carRepository.findById("avante")).willReturn(Optional.of(expectedCar));
```

기능은 원본 when-thenReturn과 똑같다. 

다만, 우리가 테스트 코드를 짤 때 준비 단계 내부에 when이라는 단어가 들어가서 영문 구조상 가독성이 살짝 어색하다.

이걸 해결하기 위해 가독성있게 만들어준다.

## 정리

- 테스트 더블을 손으로 만들지 않고 가짜 객체를 어노테이션으로 만들어준다.
- 흐름은 가짜 객체를 배치하고 → 스터빙(when-thenReturn)으로 대답을 통제한 뒤 → SUT의 비즈니스 판단 결과를 비교하거나 verify()로 행위를 감시한다.
- 어떤 테스트 코드를 보던간에 세가지를 먼저 파악하려고 해야 한다. 이 테스트 대상이 무엇이고, 격리할 의존성은 무엇이고, 지금 검증하려는 게 결과값인지 호출 행위인지 판단하면 그냥 문법을 코드로 옮기는 구조가 된다.
