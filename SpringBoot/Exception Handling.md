## 예외 처리는 시스템의 흐름을 제어하기 위한 신호

- Bean Validation으로 유효성 검사에 실패했을 때, 혹은 비즈니스 로직 상에서 ‘조회하려는 회원이 없을 때’, ‘결제 잔액이 모자랄 때’ 등 수많은 에러 상황이 터진다.
- 예외는 프로그램이 완전히 잘못되었다는 뜻이 아니라 “지금 네가 시킨 일을 이런저런 이유로 더 이상 진행할 수 없으니, 하던 일 멈추고 다른 조치 취해라”하고 시스템에 보내는 구조 신호이다.

회원 조회 예시(Get /users/999)

데이터베이스에 999번 회원이 존재하지 않는 상황은 프로그램의 버나 결함이 아니라 현실에서 언제든지 자연스럽게 발생할 수 있는 비즈니스 상황이다.

이때 NotFoundException같은 예외로 신호를 주면 예외 처리기에서 이 신호를 받아 “아, 회원을 못 찾았구나. 그럼 프론트엔드에게 404 Not Found 상태 코드와 ‘회원이 존재하지 않습니다’라는 JSON메시지를 포장해서 보내야겠다”라고 비정상적인 상황을 정상적인 흐름으로 제어하게된다.

## 에러와 예외의 차이

#### 에러 → 재앙

- 시스템 레벨에서 수습할 수 없는 심각한 문제이다.
- 예 : JVM메모리가 터지는  OutOfMemoryError(OOME), 호출 스택이 넘치는 StackOverflowError
- 이는 코드 안에서 try-catch로 잡는 것이 불가능하며, 프로그램이 그 자리에서 뻗어버리는 것이 정상이다. 인프라 담당자가 붙어서 서버 스펙을 늘리거나, 개발자가 메모리 누수를 잡아 근본 원인을 제거해야 한다.

#### 예외 → 통제 가능한 일반적인 사고

- 개발자가 코드를 통해 미리 예측하고 다룰 수 있는 가벼운 사고이다.
- 사용자가 아이디/비밀번호를 틀렸거나, 첨부파일 용량이 제한을 초과했거나, DB에 데이터가 없는 상황

## 비즈니스 로직과 오류 처리 책임의 분리

[나쁜 예시] 비즈니스 로직에 에러 응답 코드가 섞여 있는 상태

```jsx
public ResponseEntity<?> getOrder(Long orderId) {
    Order order = orderRepository.findById(orderId).orElse(null);

    // 핵심 비즈니스 로직 사이에 HTTP 상태 코드와 응답 형식을 직접 짜 넣고 있다.
    if (order == null) {
        Map<String, String> errorResponse = new HashMap<>();
        errorResponse.put("error", "Not Found");
        return ResponseEntity.status(404).body(errorResponse);
    }

    return ResponseEntity.ok(order);
}
```

- **문제점:** 순수하게 조회만 담당해야 하는 메서드가 직접 에러 맵을 만들고 상태 코드를 결정하느라 코드가 지저분해진다.
- **해결:** 예외 처리를 분리하면, 비즈니스 영역에서는 단 한 줄(`throw new NotFoundException()`)로 예외 신호만 던지고, 실제 포장 및 응답 처리는 외부의 `@RestControllerAdvice`가 전담하게 만든다.

## try-catch vs throw

예외를 만났을 때 **그 자리에서 수습할 것인가(`try-catch`)**, 상위로 던져서 전파할 것인가(`throw`)를 결정하는 명확한 기준이 있다.

#### 메서드 내부에서 try-catch로 직접 처리하는 기준

"이 예외가 발생했을 때, 내가 **지금 이 위치에서 대안(Plan B)을 마련해 정상 흐름으로 복구**할 수 있는가?"

지금 내 메서드 안에서 문제를 수습하고 해결할 수 있다면, 상위 계층으로 예외를 던지지 않고 그 자리에서 `try-catch`로 삼켜서 해결한다.

```jsx
// 파일 업로드 시나리오 예시
public void upload(MultipartFile file) {
    try {
        // AWS S3 같은 외부 저장소에 파일 전송 시도
        storageClient.save(file);
    } catch (IOException e) {
        // 네트워크 단절 등으로 IOException 발생
			  
        // 1. 에러 로그를 남긴다.
        log.error("파일 저장 실패, 임시 디렉토리에 백업합니다.", e);

        // 2. 대안(Plan B): 로컬 디스크에라도 일단 저장해 둔다. (정상 흐름으로 복구)
        localBackupStorage.save(file);

        // 3. 만약 복구할 수 없는 상황이라면, 의미를 명확히 정제해서 다시 던진다.
        // throw new IllegalStateException("FILE_UPLOAD_FAILED", e);
    }
}
```

- 만약 여기서 `IOException`을 잡지 않고 그냥 밖으로 튕겨버리면 클라이언트에게 뜬금없는 `500 Server Error`가 나간다.
- 하지만 내 선에서 "S3에 못 올렸어? 그럼 일단 에러 내지 말고 로컬 임시 디렉토리에 백업해 두자"라는 대안을 실행함으로써 **정상 흐름으로 복구**해 내는 것이다.

#### 예외를 throw하고 상위 계층으로 전파하는 기준

"이 예외가 발생하면 **더 이상 여기서 로직을 진행하는 것이 무의미한가?** 내가 당장 여기서 해결할 수 없는 문제인가?"

```jsx
public Order findOrder(Long orderId) {
    // 만약 DB를 조회했는데 찾으려는 주문 데이터가 없다면?
    return orderRepository.findById(orderId)
            // orElseThrow: 값이 없으면 예외를 발생시키고 당장 로직을 중단해라
            .orElseThrow(() -> new IllegalArgumentException("order not found"));
}
```

- 데이터베이스에 주문 번호 자체가 없는데 `OrderService`가 여기서 할 수 있는 대안(Plan B)이 있을냐? **없다.** 없는 주문을 무에서 유로 만들어낼 수는 없다
- 주문 정보가 없기 때문에 뒤이어 수행할 주문 취소나 결제 확인 로직을 실행하는 것 자체가 완전히 무의미해진다.

## 스프링 프레임워크의 예외 처리 구조

1. `OrderService`에서 주문이 없어 예외를 던지면 실행이 중단되고 예외가 호출자인 `OrderController`로 넘어간다.
2. `OrderController` 내부에도 별도의 `try-catch`가 없으므로, 예외는 다시 콜 스택(Call Stack)을 타고 더 상위인 스프링 MVC의 핵심 중심점, `DispatcherServlet`까지 전파된다.
3. 이 예외가 최종적으로 스프링의 전역 예외 처리기인 `@RestControllerAdvice`에 도달한다.
4. 전역 예외 처리기는 들어온 예외 신호를 해독한다. *",`IllegalArgumentException("order not found")`이 올라왔군. 그럼 내가 클라이언트가 이해할 수 있는 `404 Not Found` 상태 코드와 깔끔한 포맷의 JSON 데이터로 변신시켜서 전송해 줄게"* 하고 최종 응답을 만들어 준다.

→ 이렇게 함으로써 **하위 로직은 오직 실패 신호를 던지는 것에만 집중**하고, **최상위의 단 한 곳(`@RestControllerAdvice`)에서 응답 포맷을 만드는 책임을 전담**하여 프로젝트 코드가 깔끔하게 유지된다.

## 정리

- 에러는 프로그램이 아예 뻗어버리는 것을 의미하고 예외는 통제할 수 있고 제어할 수 있는 영역이다.
- 예외는 문제가 아니라 비즈니스 로직이 더 이상 진행될 수 없음을 알리고 흐름을 관리하는 신호이다.
- throw를 통해서 로직 안에서 발생한 예외를 상위에 던지는 기준을 판단해야 한다. → 이를 통해서 스프링 전역 예외처리기에서 깔끔하게 포맷해서 처리할 수 있다.
