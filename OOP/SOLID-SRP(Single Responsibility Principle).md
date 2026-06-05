## SRP(Single Responsibility Principle, 단일 책임 원칙)

하나의 클래스는 하나의 역할과 하나의 변경 이유만을 가지고 설계해야 한다는 객체지향 설계 원칙

책임 = 코드가 변경되어야 하는 이유 : SRP에서 말하는 ‘책임’은 단순히 기능의 개수가 아니라 “코드가 변경되어야 하는 이유가 무엇인가”를 의미한다.

하나의 클래스에 여러 책임이 몰려 있을 때 발생하는 연쇄적인 문제와 유지보수의 어려움을 인지해야 한다.

**단일 책임이니까 클래스 안에 메서드가 딱 하나만 있어야 하나? → 아니다**

리팩토링할 때 명확한 근거 없이 “이 클래스 좀 큰 거 같은데?”, “메서드가 너무 많은 것 같은데” 라는 주관적인 생각으로 쪼개는 것은 잘못된 방식이다.

책임은 해야 할 기능의 개수가 아니라 ‘변경이 발생하는 이유 단위’이다.

## SRP 위반 사례

**사내 급여 관리 시스템(Employee클래스)**

- calculatePay() : 직원들의 이번 달 급여 계산 로직 → 재무팀(비즈니스 정책)관할
- saveToDatabase() : 급여 내역을 오라클 DB에 저장하는 로직 → 인프라팀(기술 아키텍처)관할
- reportHours() : 근무 시간을 양식에 맞춰 화면에 출력하는 로직 → 인사팀(UX/UI)관할

→하나의 클래스를 수정해야 하는 이유가 3가지나 존재

→야간 수당 법이 바뀌어도, DB가 MySQL로 바뀌어도, 출력 양식에 부서명이 추가되어도 모두 Employee클래스를 열어서 수정해야 한다. 

→여러 부서의 요구사항 변경이 한곳에 집중되어 리스크가 커진다.

#### 나쁜 코드 예시 : 3개의 책임을 가진 클래스

CarReport라는 이름과 달리 보고서 생성, 파일 저장, 이메일 전송까지 서로 다른 이유로 변경되는 기능들이 뭉쳐있다.

```jsx
public class CarReport {

    // 1번 변경 이유: 보고서의 포맷이나 내용이 바뀔 때 (ex: 엑셀 -> PDF)
    public void generateReport() {
        System.out.println("자동차 점검 보고서를 엑셀 포맷으로 생성합니다.");
    }

    // 2번 변경 이유: 저장소 위치나 인프라가 변경될 때 (ex: 로컬 -> 클라우드)
    public void saveToFile() {
        System.out.println("생성된 보고서를 로컬 서버 D드라이브에 저장합니다.");
    }

    // 3번 변경 이유: 발송 방식이나 이메일 서버 설정이 바뀔 때
    public void sendToEmail() {
        System.out.println("저장된 보고서를 사장님 이메일로 전송합니다.");
    }
}
```

#### 좋은 코드 예시 : 책임을 분리하고 조합하기

변경 이유에 따라 클래스를 각각 분리하고, 상위 서비스에서 이를 조합(DI)하여 사용한다.

```jsx
// 1. 오직 보고서 생성 방식이 바뀔 때만 주로 수정되는 클래스
public class ReportGenerator {
    public String generateReport() {
        System.out.println("자동차 점검 보고서를 엑셀 포맷으로 생성합니다.");
        return "자동차 점검 보고서 내용";
    }
}

// 2. 오직 파일 저장 위치나 인프라가 바뀔 때만 수정되는 클래스
public class FileSaver {
    public String saveToFile(String content) {
        System.out.println("생성된 보고서를 로컬 서버 D드라이브에 저장합니다.");
        return "D:/reports/car-report.xlsx";
    }
}

// 3. 이메일 발송 시스템이 바뀔 때만 수정되는 클래스
public class EmailSender {
    public void sendToEmail(String filePath) {
        System.out.println(filePath + " 파일을 담당자 이메일로 전송합니다.");
    }
}

// 4. 분리된 객체들을 조율해서 전체 흐름을 처리하는 상위 서비스 클래스
public class ReportService {
    private final ReportGenerator generator;
    private final FileSaver saver;
    private final EmailSender sender;

    // 의존성 주입(DI)을 통해 분리된 책임들을 조합
    public ReportService(ReportGenerator generator, FileSaver saver, EmailSender sender) {
        this.generator = generator;
        this.saver = saver;
        this.sender = sender;
    }
    
    // 각 객체에게 메시지를 보내서 책임을 위임 (단일 책임: 비즈니스 흐름 제어)
    public void processReport() {
        String content = generator.generateReport();
        String filePath = saver.saveToFile(content);
        sender.sendToEmail(filePath);
    }
}
```

ReportService는 세부적인 인프라 기술(엑셀인지, D드라이브인지, 이메일 엔진이 무엇인지)은 전혀 모른채 “보고서를 생성하고 저장하고 보낸다”라는 전체 비즈니스 흐름만 제어하는 단일 책임을 갖게 된다.

## 정리

- SRP는 하나의 클래스가 오직 하나의 변경 이유를 가져야한다는 원칙
- 책임을 제대로 분리하지 않으면 하나의 요구사항을 수정할 때 다른 기능까지 문제가 생기는 상황이 생길 수 있다.
- 원칙을 맹목적으로 따르기보다, 현재 마주한 요구사항과 비즈니스 맥락에 따라 적절한 타이밍에 클래스를 분리하는 것이 필요하다.
