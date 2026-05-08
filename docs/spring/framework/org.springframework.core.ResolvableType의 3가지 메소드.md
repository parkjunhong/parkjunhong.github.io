# 질문: `toClass()` vs. `getClass()

```
`org.springframework.core.ResolvableType.toClass()` 와 `org.springframework.core.ResolvableType.getClass()` 는 어떻게 다른가요?

반환하는 데이터나 사용하는 곳에 대해서 설명해주세요.
```
# 답변

수석 아키텍트입니다.

두 메서드는 이름이 비슷해서 혼동하기 쉽지만, **아키텍처와 자바 리플렉션 관점에서 완전히 다른 의미와 역할**을 가집니다.

결론부터 말씀드리면, 스프링 프레임워크에서 객체의 타입을 판별하고 제어할 때 우리가 써야 하는 메서드는 `toClass()` (또는 `resolve()`)이며, `getClass()`는 비즈니스 로직에서 절대 사용하면 안 됩니다.

이 두 메서드의 차이를 명확히 비교해 드립니다.

---

### 1. `ResolvableType.toClass()`

* **의미:** `ResolvableType`이 '분석하고 있는 대상(Target)의 실제 클래스'를 반환합니다.
* **반환 데이터:** 제네릭 정보가 제거된 원시 클래스(Raw Class)를 반환합니다. 만약 제네릭 타입 등을 끝내 유추할 수 없는 경우 기본값인 `java.lang.Object.class`를 반환합니다.
* **사용하는 곳:** * 개발자님이 작성하신 `MessageConverter`의 `canRead`, `canWrite`처럼 "내가 지금 처리해야 할 데이터의 진짜 타입이 무엇인가?"를 판별할 때 사용합니다.
* 스프링 내부에서 Bean을 주입할 때 해당 Bean이 어떤 클래스인지, 어떤 인터페이스를 구현했는지 확인할 때 사용됩니다.

*(참고 💡: 스프링 실무에서는 `toClass()` 대신 `resolve()`**를 더 많이 사용합니다. `toClass()`는 타입을 모를 때 무조건 `Object.class`를 반환해버리지만, `resolve()`는 타입을 알 수 없으면 `null`을 반환하므로 Null Check를 통한 안전한 방어 로직을 짜기 좋기 때문입니다.)*

---

### 2. `ResolvableType.getClass()`

* **의미:** 자바의 최상위 클래스인 `java.lang.Object`로부터 상속받은 메서드로, '자기 자신(현재 메모리에 떠 있는 객체)의 클래스'를 반환합니다.
* **반환 데이터:** 분석 대상이 무엇이든 상관없이, **항상 `org.springframework.core.ResolvableType.class` 자체를 반환**합니다.
* **사용하는 곳:**
* 일반적인 비즈니스 로직이나 타입 검사에서는 **절대 사용하지 않습니다.**
* 단순히 JVM 레벨에서 로깅을 찍거나 디버깅을 할 때 "이 변수의 인스턴스 타입이 뭐지?"를 확인할 때만 사용됩니다.



---

### 💻 코드로 보는 직관적인 차이

만약 컨트롤러에서 `List<String>` 데이터를 반환했고, 스프링이 이를 `ResolvableType` 객체로 만들어서 컨버터에게 던져주었다고 가정해 보겠습니다.

```java
// 스프링이 List<String>을 분석하기 위해 ResolvableType 객체를 생성함
ResolvableType type = ResolvableType.forClassWithGenerics(List.class, String.class);

// 1. toClass() 호출: "네가 분석하고 있는 녀석의 껍데기가 뭐야?"
Class<?> targetClass = type.toClass(); 
System.out.println(targetClass.getName()); 
// 👉 결과: "java.util.List" (우리가 원했던 진짜 판별 대상)

// 2. getClass() 호출: "너(type) 자신의 정체가 뭐야?"
Class<?> myClass = type.getClass();
System.out.println(myClass.getName()); 
// 👉 결과: "org.springframework.core.ResolvableType" (아무짝에도 쓸모없는 정보)

```

### 🏛️ 아키텍트의 결론

이전 코드에서 `type.toClass()`를 사용하셨던 것은 **100% 정답**입니다.

만약 거기서 `type.getClass()`를 호출하셨다면, `valueClass` 변수에는 우리가 검사해야 할 `String`이나 `byte[]`가 들어가는 것이 아니라 `ResolvableType.class`가 들어가게 되어 모든 방어 로직(`isBypassType`)이 무용지물이 되었을 것입니다.

자바의 근간이 되는 리플렉션(Reflection)과 메타데이터(Metadata)의 차이를 정확하게 구분하고자 하신 아주 훌륭한 질문이었습니다!

<hr>

# 질문: `toClass()` vs. `resolve()`

```
설명한 내용에 따르면 'toClass()' 보다는 'resolve()'를 사용할까요?

그리고 `org.springframework.core.ResolvableType.getRawClass()`는 어떤 건가요?
```

# 답변

수석 아키텍트입니다.

개발자님의 탐구심이 정말 깊고 날카롭습니다! 프레임워크가 제공하는 API의 미묘한 차이를 파고드는 것은 시니어 아키텍트로 도약하기 위한 필수 과정입니다.

질문하신 세 가지 메서드(`toClass()`, `resolve()`, `getRawClass()`)는 스프링에서 리플렉션과 제네릭(Generics)을 다룰 때 가장 헷갈리는 삼총사입니다. 이들의 차이점과 실무에서 어떤 것을 선택해야 하는지 명쾌하게 정리해 드립니다.

---

### 1. `toClass()` vs `resolve()` : 무엇을 쓸까요?

결론부터 말씀드리면, **현재 개발자님이 작성 중인 `MessageConverter`의 `canRead`/`canWrite` 내부에서는 `toClass()`를 쓰시는 것이 가장 완벽한 정답**입니다.

이유를 알기 위해서는 스프링 내부 구현 코드를 봐야 합니다. `toClass()`는 내부적으로 이렇게 구현되어 있습니다.

```java
// Spring 내부 구현체
public Class<?> toClass() {
    return this.resolve(Object.class);
}

```

* **`resolve()`:** 제네릭이나 복잡한 타입을 실제 `Class`로 추론합니다. 만약 제네릭 변수(`T`)라서 도저히 타입을 알 수 없다면 **`null`을 반환**합니다.
* **`toClass()`:** `resolve()`를 호출하되, 타입을 도저히 알 수 없어서 `null`이 나오려 하면 `Object.class`를 대신 반환(Fallback)합니다.

**💡 아키텍트의 선택 가이드:**
우리의 방어 로직(`shouldYieldToDefaultConverter`)은 인자로 받은 `valueClass`가 `String`, `byte[]` 등인지 `isAssignableFrom`으로 검사합니다.
만약 `resolve()`를 썼다가 `null`이 반환되면 `NullPointerException`이 발생할 위험이 있습니다. 
반면, `toClass()`를 쓰면 최악의 경우라도 `Object.class`가 반환되며, `Object.class`는 `String`이나 `byte[]`가 아니므로 안전하게 `false`로 넘어가서 에러 없이 동작합니다.
**따라서 현재 로직에서는 `toClass()`가 가장 안전하고 적절합니다.**

---

### 2. `getRawClass()` 란 무엇인가요?

`getRawClass()`는 `resolve()`와 비슷해 보이지만, '제네릭(Generic) 추론에 얼마나 깊이 개입하느냐'에서 결정적인 차이가 있습니다.

* **의미:** 꺾쇠 `< >` 안에 있는 제네릭 타입은 모두 무시하고, 겉으로 선언된 가장 껍데기(Raw) 클래스만 있는 그대로 가져옵니다.
* **차이점:** 만약 대상이 단순한 제네릭 변수 `T`일 경우, `getRawClass()`는 추론을 시도하지 않고 그냥 `null`을 뱉어버립니다.
하지만 `resolve()`는 그 `T`가 선언된 클래스의 상한선(Bounds, 예: `T extends Number`)까지 끝까지 쫓아가서 최대한 클래스(`Number.class`)를 찾아내려 노력합니다.

#### 💻 한눈에 보는 직관적인 비교

```java
// 상황 1: List<String> 
ResolvableType type1 = ResolvableType.forClassWithGenerics(List.class, String.class);
type1.getRawClass(); // 👉 List.class (제네릭 떼고 껍데기만)
type1.resolve();     // 👉 List.class (동일)
type1.toClass();     // 👉 List.class (동일)

// 상황 2: 제네릭 변수 <T extends Number> 인 경우
ResolvableType type2 = ... // (T를 가리키는 ResolvableType)
type2.getRawClass(); // 👉 null (T는 클래스가 아님! 추론 포기)
type2.resolve();     // 👉 Number.class (extends Number 니까 Number로 추론 성공!)
type2.toClass();     // 👉 Number.class (resolve와 동일)

// 상황 3: 그냥 완전 미지의 제네릭 변수 <T> 인 경우
ResolvableType type3 = ... 
type3.getRawClass(); // 👉 null (T는 클래스가 아님)
type3.resolve();     // 👉 null (상한선도 없어서 도저히 추론 불가)
type3.toClass();     // 👉 Object.class (null 대신 자바의 최상위 부모 반환)

```

---

### 🏛️ 최종 정리 및 요약

1. **`getRawClass()`:** 단순히 제네릭을 제거한 껍데기 클래스만 알고 싶을 때 사용. (제네릭 변수를 만나면 바로 포기하고 `null` 반환)
2. **`resolve()`:** 제네릭의 `extends` 상한선까지 끝까지 추적해서 실제 클래스를 알아내고 싶을 때 사용. (실패 시 `null` 반환)
3. **`toClass()`:** `resolve()`와 똑같이 똑똑하게 추적하지만, 실패해도 절대 `null`을 뱉지 않고 `Object.class`를 반환. (안전성 최고)

따라서 지금처럼 파라미터로 넘어온 미지의 타입(`ResolvableType`)을 `isAssignableFrom()` 같은 리플렉션 메서드와 결합해서 안전하게 검사해야 할 때는
**지금 작성하신 대로 `type.toClass()`를 유지하시는 것이 가장 견고한(Robust) 아키텍처**입니다.
