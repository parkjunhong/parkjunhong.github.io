Java의 `record`는 **“값(데이터) 묶음”을 간결하게 선언하기 위한 클래스 형태**예요. JDK 25에서도 `record` 자체의 문법/개념은 그대로이고, `record`와 특히 궁합이 좋은 **패턴 매칭(record pattern)** 쪽이 계속 확장되는 흐름입니다. ([Oracle Docs][1])

## 1) `record`가 무엇인가

`record`는 **고정된 구성요소(components)** 를 가지는 “투명한 데이터 운반체(transparent carrier)”로, 선언한 구성요소를 기반으로 컴파일러가 많은 것들을 자동 생성합니다. (JDK 25 API 문서 표현으로는 *shallowly immutable* 입니다.) ([Oracle Docs][2])

### 기본 선언

```java
record Point(int x, int y) {}
```

### 컴파일러가 “강제/자동”으로 만들어 주는 것들

* 각 컴포넌트에 대한 **private final 필드**
* 각 컴포넌트에 대한 **public 접근자 메서드** (`x()`, `y()`)
* 헤더와 동일 시그니처의 **canonical constructor**
* 컴포넌트들을 기준으로 한 `equals`, `hashCode`, `toString` ([Oracle Docs][2])

즉, 아래처럼 “DTO/값 객체”에서 늘 쓰던 보일러플레이트가 크게 줄어듭니다. ([Oracle Docs][1])

## 2) `record`를 쓸 때 알아두면 좋은 규칙/특성

### (1) 얕은 불변(shallow immutability)

컴포넌트 필드는 `final`이지만, **컴포넌트가 가변 객체면 내부는 변할 수 있어요.** (예: `List`, 배열, mutable POJO 등) ([Oracle Docs][2])

그래서 필요하면 생성자에서 방어적 복사를 합니다:

```java
import java.util.List;

record Team(String name, List<String> members) {
    Team {
        members = List.copyOf(members); // 방어적 복사(불변화)
    }
}
```

(배열도 마찬가지로 `clone()` 등으로 복사하는 게 흔합니다.) ([Oracle Docs][2])

### (2) 검증/정규화는 생성자에서

`record`는 “데이터가 그대로 드러나는 타입”이라서, **유효성 검사/정규화가 필요하면 canonical constructor(특히 compact constructor)** 를 명시합니다. ([Oracle Docs][2])

```java
record User(String name, int age) {
    User {
        if (name == null || name.isBlank()) throw new IllegalArgumentException("name");
        if (age < 0) throw new IllegalArgumentException("age");
    }
}
```

### (3) 메서드/인터페이스 구현은 가능

`record`도 클래스라서 메서드 추가, 정적 필드/메서드 추가, 인터페이스 구현이 가능합니다. (대신 “데이터 운반체” 컨셉을 해치지 않는 선에서 쓰는 게 보통이에요.) ([OpenJDK][3])

## 3) JDK 25에서 `record`와 같이 보면 좋은 것: Record Patterns(레코드 패턴)

`record`의 강점이 커지는 이유 중 하나가 **패턴 매칭으로 레코드를 “분해(deconstruct)”** 할 수 있기 때문입니다. 이 기능은 JDK 21에서 최종 확정된 “Record Patterns”로 들어왔고, JDK 25에서도 표준으로 사용합니다. ([OpenJDK][4])

### `instanceof`에서 record pattern

```java
record Point(double x, double y) {}

static void f(Object obj) {
    if (obj instanceof Point(double x, double y)) {
        System.out.println(x + ", " + y);
    }
}
```

“Point인지 확인 + 내부 컴포넌트 꺼내기”가 한 번에 됩니다. ([Oracle Docs][5])

### `switch`에서 record pattern (자주 쓰는 형태)

```java
record Point(int x, int y) {}

static String quadrant(Object obj) {
    return switch (obj) {
        case Point(int x, int y) when x > 0 && y > 0 -> "I";
        case Point(int x, int y) when x < 0 && y > 0 -> "II";
        case Point(int x, int y) when x < 0 && y < 0 -> "III";
        case Point(int x, int y) when x > 0 && y < 0 -> "IV";
        case Point(int x, int y) -> "Axis";
        default -> "Not a point";
    };
}
```

이 스타일이 “데이터를 데이터처럼 다룬다”는 record의 의도와 잘 맞습니다. ([OpenJDK][4])

## 4) (참고) JDK 25의 패턴 매칭 확장: 기본형(primitive)도 패턴에 포함 (Preview)

JDK 25에는 패턴 매칭을 **기본형까지 확장**하는 미리보기(Preview) 기능(JEP 507)이 포함돼 있어요. 이건 record 자체 기능은 아니지만, record pattern을 포함한 “패턴 매칭 세계”를 더 넓히는 방향입니다. ([OpenJDK][6])

예시(Preview):

```java
static String grade(Number n) {
    return switch (n) {
        case int i when i >= 90 -> "A";
        case int i when i >= 80 -> "B";
        default -> "C";
    };
}
```

Preview 기능이라면 빌드/실행에 `--enable-preview` 같은 옵션이 필요합니다. ([OpenJDK][6])

---

[1]: https://docs.oracle.com/en/java/javase/25/language/records.html "Record Classes"
[2]: https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/Record.html "Record (Java SE 25 & JDK 25)"
[3]: https://openjdk.org/jeps/395 "JEP 395: Records"
[4]: https://openjdk.org/jeps/440 "JEP 440: Record Patterns"
[5]: https://docs.oracle.com/en/java/javase/25/language/record-patterns.html?utm_source=chatgpt.com "Record Patterns"
[6]: https://openjdk.org/jeps/507?utm_source=chatgpt.com "JEP 507: Primitive Types in Patterns, instanceof, and ..."
