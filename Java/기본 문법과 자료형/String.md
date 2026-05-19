## String의 정의

String은 문자의 연속을 표현하는 불변객체인 문자열 자료형이다.

불변 객체란 생성된 이후 내부 상태나 값이 변경되지 않는 객체를 의미한다.

- String greeting = “hello”; → hello라고 하는 문자열 객체가 만들어졌다.
- greeting = greeting + “jisoo”; → hello 뒤에 “jisoo”가 붙어서 기존 객체가 수정되는게 아니라 실제로 새로운 “hello jisoo”라는 객체가 생성된다.

자바에서 문자열은 기본 자료형이 아닌 객체로 취급되며, String클래스는 java.lang패키지에 포함된 대표적인 참조 타입이다.

![image.png](attachment:274589d8-003b-44e2-bf22-95a2151801a9:image.png)

문자열은 “ “를 사용해 표현하며, 문자열 리터럴을 사용하면 JVM 내부의 String Constant Pool이라는 영역에 저장된다.

## 문자열 비교

==이 아니라 equals()로 사용해야한다.

==은 두 변수가 같은 객체를 참조하는지 확인하는 것이다.

equals()는 문자열의 내용이 같은지를 비교한다.

```jsx
String a = "Adapterz";
String b = "Adapterz";
String c = new String("Adapterz");

System.out.println(a == b);         // true (같은 리터럴, 같은 객체)
System.out.println(a == c);         // false (다른 객체)
System.out.println(a.equals(c));    // true (값이 같음)
```

a, b는 같은 문자열 리터럴을 써서 String pool의 같은 객체를 참조할 수 있다.

c는 내용은 같아도 참조가 다르다.

### ==이랑 equlas()의 차이는?

==는 참조비교 equals는 객체가 정의한 동등성 비교!!
