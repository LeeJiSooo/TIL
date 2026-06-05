기획자가 “이거 아주 간단한 기능인데 하나만 추가해주세요”라고 요청하는 일은 흔하다.

기획자는 간단한 추가라고 하지만, 코드를 열어보면 이미 if-else 분기문이 잔뜩 쌓여잇는 경우가 많다. 이걸 건드렸다가 기존에 잘 돌아가던 기능까지 망가질까봐 불안해진다.

OCP의 목적 : 새로운 요구사항이 들어왔을 때, 기존 코드를 건드리지 않고 어떻게 기능을 확장할 것인가에 대한 해답이다.

## OCP(Open-Closed Principle, 개방 폐쇄 원칙)

- 확장에 열려 있다 : 비즈니스 요구사항이 변경되거나 새로운 기능이 필요할 때, 시스템이 그것을 쉽게 수용하고 확장할 수 있는 구조를 갖추어야 한다.
- 변경에 닫혀 있다 : 새로운 기능을 시스템에 추가할 때, 이미 잘 돌아가고 검증이 끝난 기존 코드를 열어서 수정하지 않아야 한다.

## OCP 위반 사례

#### OCP를 위반한 설계(if-else문 많음)

차량 종류를 String 문자열로 받아 조건문으로 주행 비용을 계산하는 서비스이다.

```jsx
public class DrivingService {

    // 차량의 '문자열 타입'에 의존해서 100km 주행 비용을 계산하는 메서드
    public int getCostPer100Km(String type) {
        if (type.equals("Electric")) {
            return 2200; // 전기차 100km당 예상 충전 비용
            
        } else if (type.equals("Gasoline")) {
            return 12500;
            
        // 요구사항이 추가될 때마다 아래에 계속 else if가 붙어야 한다.
        } else if (type.equals("Hybrid")) {
            return 8500;
            
        } else if (type.equals("Diesel")) {
            return 9800;
            
        } else if (type.equals("Hydrogen")) {
            return 11000;
            
        } else {
            throw new IllegalArgumentException("지원하지 않는 차량 타입입니다: " + type);
        }
    }
}
```

- 확장에 닫혀 있고 변경에 열려 있음 : 새로운 차량이 추가될 때마다 DrivingService클래스 메서드 내부를 직접 고쳐야 하므로 OCP에 위배된다.
- 자주 변하는 계산 정책들이 하나의 메서드에 계속 누적되어, 나중에 코드가 길어지면 수정하다가 기존 로직을 건드릴 실수의 확률이 높아진다.
- 문자열에 강한 의존성(타입 안정성 부족) : 누가 실수로 소문자 electric으로 호출하거나 스펠링 오타를 내면 컴파일 에러가 잡지 못하고 런타임 오류가 발생한다.

**리팩토링 판단 기준 : 변하는 것과 변하지 않는 것 분리**

- 변하지 않는 것 : “100km 주행 비용을 계산한다”는 사실 그 자체
- 변하는 것 : “어떤 방식으로 비용을 계산할 것인가?(전기, 가솔린,디젤…)
- 해결책 : 자주 변하는 주행방식(비용계산)부분을 인터페이스로 추출

#### OCP를 적용한 구조(인터페이스와 다형성)

개선된 코드 : 클래스 추가를 통한 기능 확장

```jsx
// 1. 인터페이스 정의 -> 공통 규격 
public interface DrivingMode {
    int getCostPer100Km(); // 100km 주행 비용 계산 역할만 선언
}

// 2. 변하는 각각의 방식을 별도의 구현 클래스로 분리
public class ElectricDriving implements DrivingMode {
    @Override
    public int getCostPer100Km() {
        return 2200;
    }
}

public class GasolineDriving implements DrivingMode {
    @Override
    public int getCostPer100Km() {
        return 12500;
    }
}

// 3. [확장] 만약 태양광 자동차(Solar)를 추가해달라고 하면? 기존 코드는 안 건드리고 새 클래스만 만든다.
public class SolarDriving implements DrivingMode {
    @Override
    public int getCostPer100Km() {
        return 0; 
    }
}

// 4. 인터페이스에만 의존하여 변경에 닫혀있는 서비스 클래스
public class DrivingService {

    // 문자열(String)이 아니라 규격인 인터페이스(DrivingMode)로 파라미터를 받는다.
    public int calculateCost(DrivingMode mode) {
        return mode.getCostPer100Km(); // 다형성활용
    }
}
```

- 기능 확장은 새로운 클래스를 만드는 것으로 해결되며, 이미 검증된 기존 로직은 절대 건드리지 않는다.
- 핵심 서비스 코드(DrivingService)는 쉽게 흔들리지 않고 닫혀있으면서도, 새로운 구현체(SolarDriving 등)를 통해 언제든 확장할 수 있도록 열려있게된다.

## 정리

- 기능 확장은 열어두되, 기존 코드 변경은 닫아야 한다.
- 조건문이 계속 늘어나는 코드는 새 기능 추가 시 기존 코드에 문제를 일으킬 확률이 매우 높으므로, 인터페이스와 다형성을 활용해 분리해야 한다.
- OCP를 잘 지키면 실무에서 요구사항이 추가될 때 기존 코드를 ‘수정’하는게 아니라 새로운 클래스를 ‘추가’하는 방향으로 개발하게 된다.
