## ReetrantLoc정의

“동일한 스레드가 여러 번 안전하게 진입할 수 있는 락”

ReetrantLoc은 같은 스레드가 이미 획득한 락을 다시 획득할 수 있도록 허용하는 락 구현체이다.

하나의 스레드가 메서드 A에서 락을 획득한 상태에서 내부적으로 메서드 B를 호출했고, 메서드 B도 같은 락을 요구하더라도 해당 스레드는 락을 다시 획득할 수 있다.

이러한 특성이 없다면 같은 스레드가 자기 자신에게 막혀 데드락이 발생할 수 있다.

## 사용 이유

실제로 개발하다보면 클래스 내부의 메소드들이 서로 호출하는 경우가 있다.

게시글 수정 메소드 안에서 권한 확인 메소드를 호출 할 수도 있는데 두 메소드 모두 동시성 제어가 필요해서 같은 락을 걸어두었을 때 재진입 기능이 없으면 코드를 분리하거나 락을 여러개 만들어야 된다.

→ ReetrantLock을 사용하면 셀프 데드락 없이 안전하고 직관적으로 코드를 짤 수 있다.

ReentrantLock은 synchronized 보다 더 **정밀한 락 제어**가 필요한 경우 사용된다.

특히, **중첩된 락 획득이 필요한 상황**이나, tryLock(), lockInterruptibly() 같은 **유연한 락 전략**이 필요할 때 유용하다.

동일한 스레드가 이미 획득한 락을 다시 요청할 경우 **예외 없이 재진입을 허용한다.**

또한 명시적으로 lock()과 unlock()을 제어할 수 있어 **락 해제 시점을 유연하게 관리**할 수 있다.

복잡한 동기화 시나리오가 요구되는 고성능 멀티스레드 환경에서 자주 사용된다.

## 사용 방법

ReentrantLock 객체를 생성한 뒤, lock() 메서드로 임계 구역 진입을 시도한다.

락을 획득한 스레드만 임계 영역 코드를 실행할 수 있으며, **작업이 끝난 후에는 반드시 unlock()을 호출**해야 한다.

try-finally 구문을 활용해 **예외 발생 시에도 락 해제를 보장**하는 것이 일반적인 사용 방식이다.

또한 tryLock()은 **대기 없이 락을 시도**하거나, lockInterruptibly()는 **인터럽트 대응이 필요한 경우** 사용된다.

### tryLock() - 비차단 락(즉시 시도 후 실패 가능)  “ 줄 안 서고 눈치 싸움하기”

대기를 하지 않고 갔는데 자리가 차 있으면 그 찰나의 순간에 바로 포기하고 뒤돌아서 다른 식당으로 간다.

(기다리는 시간 = 0초)

```jsx
public class Restaurant {
    private final ReentrantLock 가게문 = new ReentrantLock();

    public void enter(String 손님) {
        // 자리가 있으면 true를 얻고 들어가고, 이미 만석이면 기다리지 않고 즉시 false를 뱉습니다.
        if (가게문.tryLock()) { 
            try {
                // [성공 코스]
                System.out.println(손님 + " -> 마침 자리가 비었네! 입장해서 밥 먹는 중...");
                Thread.sleep(1000); // 1초 동안 식사
            } catch (InterruptedException e) {
            } finally {
                System.out.println(손님 + " -> 잘 먹었습니다. 퇴장! (문 열어둠)");
                가게문.unlock(); // 다 먹었으니 자물쇠 풀기
            }
        } else {
            // [실패 코스] 락을 못 얻었을 때 대기하지 않고 즉시 이리로 튕겨 나옵니다.
            System.out.println(손님 + " -> 아, 만석이네? 줄 서기 싫으니까 그냥 편의점 가야지. (포기)");
        }
    }
}
```

### lockInterruptibly() - 인터럽트 대응 락 “줄 서서 기다리다가 친구가 부르면 탈출하기”

줄을 서서 기다리긴 하지만 기존의 lock()과 다른 점은 기다리는 도중에 외부에서 나오라고 신호(Interrupt)를 보내면 기다리던 줄에서 탈출할 수 있다.

lock()은 문이 열릴 때 까지 문 앞에서 꼼짝 못 하고 기다려야 하는 락이라면 IockInterruptibly()는 도중에 도망칠 수 있는 락이다.

```jsx
public class Restaurant {
    private final ReentrantLock 가게문 = new ReentrantLock();

    public void enterWithEscape(String 손님) {
        try {
            System.out.println(손님 + " -> 줄 서서 기다리는 중...");
            
            // 줄을 서긴 서는데, 만약 외부에서 인터럽트(방해) 알림을 주면 
            // 대기를 중단하고 에러(InterruptedException)를 터뜨리며 탈출합니다!
            가게문.lockInterruptibly(); 
            
            try {
                // [성공 코스] 무사히 줄 서서 입장 성공한 경우
                System.out.println(손님 + " -> 드디어 입장! 밥 먹는 중...");
                Thread.sleep(1000);
            } finally {
                가게문.unlock();
            }
            
        } catch (InterruptedException e) {
            // [탈출 코스] 기다리다가 중간에 외부 자극에 의해 튕겨 나갔을 때 실행되는 곳!
            System.out.println(손님 + " -> 앗, 기다리다가 급한 전화(인터럽트)가 왔네! 줄 서기 취소하고 갑니다!");
        }
    }
}
```

- lock() : 문이 열릴 때까지 기다린다.(중간에 취소나 탈출 안됨)
- tryLock() : 지금 당장 자리가 없으면 1초도 기다리지 않고 나간다.
- lockInterruptibly() : 일단 자리가 날 때까지 줄을 서지만 중간에 인터럽트가 오면 대기를 취소하고 이탈한다.

## 다른 기술과의 비교

synchronized와 ReentrantLock은 **동기화를 위해 사용되는 도구**지만, **제어 수준과 유연성**에서 차이가 있다.

| **항목** | synchronized | ReentrantLock |
| --- | --- | --- |
| 락 획득 방법 | 자동 (블록 진입 시 획득) | 명시적 (lock(), unlock() 사용) |
| 락 해제 방법 | 자동 (블록 종료 시 해제) | 명시적 (unlock() 호출 필요) |
| 락 획득 대기 중 인터럽트 처리 | 불가능 (인터럽트 무시) | 가능 (lockInterruptibly() 사용 가능) |
| 락 획득 시도 및 실패 처리 | 불가능 | 가능 (tryLock()으로 대기 없이 시도 가능) |
| 조건 변수 (Condition) 사용 | 불가능 (wait/notify만 사용 가능) | 가능 (newCondition()으로 세분화 가능) |
| 공정성 (FIFO 보장) 설정 | 불가능 | 가능 (new ReentrantLock(true)) |
| 재진입 가능 여부 | 가능 | 가능 |

### **왜 ReentrantLock이 더 정밀한가?**

1. **명시적 제어 가능**
    - synchronized는 진입과 동시에 락을 획득하고, 블록을 빠져나올 때 자동으로 해제된다.
    - ReentrantLock은 락 획득과 해제를 **코드로 직접 제어**할 수 있어, 더 복잡한 조건에서의 락 처리나 분기 처리가 가능하다.
2. **락 획득 실패 처리**
    - tryLock()을 통해 락을 획득하지 못한 경우 **대기하지 않고 우회 처리**할 수 있다.
    - synchronized는 락을 획득할 때까지 **무조건 대기한다.**
3. **인터럽트 가능 제어**
    - lockInterruptibly()는 락 대기 중에도 **스레드 인터럽트 처리를 할 수 있는 유일한 락 방식이다.**
    - synchronized는 락 대기 중 인터럽트를 무시한다.
4. **다중 조건 변수 지원**
    - ReentrantLock은 Condition 객체를 통해 **여러 조건을 개별적으로 제어**할 수 있다.
    - synchronized는 wait(), notify(), notifyAll()만 제공되며 **구분 없이 브로드캐스트된다.**

## 정리

- Reetrant - 동일한 스레드가 이미 획득한 락을 다시 요청할 때 이를 허용하고 셀프 데드락을 방지한다.
- tryLock()을 이용해서 비차단 동기화를 구현할 수 있고 lockInterruptibly()를 통해서 대기중인 스레드에 중단 신호를 보낼 수 있다.
- 단순한 동기화는 synchronized로 충분하지만 대규모 트래픽에서 다임아웃을 다루거나 공정성, 다중 조건 제어가 필요하면 ReetrantLock을 사용해야한다.
