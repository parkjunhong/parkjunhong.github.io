# 질문: 일반적인 POJO 형태의 Setter/Getter 가 아닌 메소드로 인한 이슈 
> 첨부파일은 요청을 받는 DTO 객체입니다.
> 
> 이 객체의 'setter' 메소드는 아래와 같이 'field'와 일치하지 않는 것들이 있습니다.
> ```
> setCreateAndUpdateUser(String)
> setCreatedAndUpdated(Long)
> ```
> 
> 이로 인해서 `open.commons.spring.web.jackson.deserialization.AuthorizedFieldDeserializerModifier.updateBuilder(DeserializationConfig, Supplier, BeanDeserializerBuilder)`를 overriding 한 로직에서,
> 
> `tools.jackson.databind.deser.BeanDeserializerBuilder.getProperties()` 를 통해서 제공받는 `Iterator<SettableBeanProperty>` 정보에 있는 `tools.jackson.databind.deser.SettableBeanProperty`의 `SettableBeanProperty.getName()` 값과 `tools.jackson.databind.BeanDescription.findProperties()`의 `tools.jackson.databind.introspect.BeanPropertyDefinition.getName()` 값이 일치하지만,  `tools.jackson.databind.introspect.BeanPropertyDefinition.getField()` 값이 null 인 경우가 발생합니다.
> 
> 이에 대한 검증과 개선방안을 제안해 주세요.
> 

# 답변

첨부해주신 `EvalModelDTO.java` 코드와 질문 내용을 바탕으로 정확한 원인 분석 및 해결 방안을 제시해 드리겠습니다. Jackson 역직렬화(Deserialization) 과정에서 커스텀 DTO의 편의성 메서드가 리플렉션 기반의 필드 제어와 충돌하는 전형적인 케이스입니다.

### 🕵️‍♂️ 1. 원인 분석 및 검증

Jackson 라이브러리는 JSON을 객체로 역직렬화할 때 `BeanDeserializerBuilder`를 사용하여 객체의 프로퍼티(Property) 목록을 구성합니다. 이때 프로퍼티를 인식하는 기본 기준은 Getter/Setter 메서드의 이름(Naming Convention)입니다.

문제의 메서드들을 분석해 봅시다.

```java
public void setCreateAndUpdateUser(String userId) {
    this.createUser = userId;
    this.updateUser = userId;
}

public void setCreatedAndUpdated(Long timestamp) {
    this.updated = timestamp;
    this.created = timestamp;
}

```

* **Jackson의 인식:** Jackson의 `Introspector`는 메서드 이름이 `set`으로 시작하므로, 이를 프로퍼티 설정 메서드로 인식합니다.
* `setCreateAndUpdateUser` -> `createAndUpdateUser`라는 이름의 프로퍼티로 인식.
* `setCreatedAndUpdated` -> `createdAndUpdated`라는 이름의 프로퍼티로 인식.


* **충돌 발생 지점:** 개발자님이 작성하신 커스텀 `AuthorizedFieldDeserializerModifier` 내부에서는 권한 처리를 위해 해당 프로퍼티와 연결된 **실제 멤버 변수(Field)** 정보가 필요합니다. 하지만 DTO 클래스 내부에는 `createAndUpdateUser`나 `createdAndUpdated`라는 이름의 멤버 변수가 선언되어 있지 않습니다.
* **결과:** `BeanPropertyDefinition.getField()`를 호출하면 매칭되는 실제 필드가 없으므로 `null`을 반환하게 됩니다. 이 `null` 값을 가지고 리플렉션이나 어노테이션 검사를 수행하려 하면 예외가 발생하거나 로직이 비정상적으로 동작하게 됩니다.

---

### 🚀 2. 개선 방안

이 문제를 해결하기 위해 두 가지 접근 방식을 제안합니다.

#### 방안 A: Jackson에게 해당 메서드가 프로퍼티용 Setter가 아님을 명시 (권장)

가장 깔끔하고 확실한 방법입니다. DTO 내부의 편의성 메서드가 Jackson의 데이터 바인딩 과정에 개입하지 못하도록 차단합니다. Jackson 애노테이션을 사용하여 해당 메서드를 무시하도록 설정합니다.

1. **`@JsonIgnore` 사용:**
해당 편의성 메서드 위에 `@JsonIgnore` 애노테이션을 추가하여 Jackson이 역직렬화할 때 이 메서드를 프로퍼티로 취급하지 않도록 만듭니다.
``` java
import com.fasterxml.jackson.annotation.JsonIgnore; // (또는 사용하는 Jackson 버전에 맞는 패키지)

// ...

@JsonIgnore // Jackson 역직렬화 대상에서 제외
public void setCreateAndUpdateUser(String userId) {
    this.createUser = userId;
    this.updateUser = userId;
}

@JsonIgnore // Jackson 역직렬화 대상에서 제외
public void setCreatedAndUpdated(Long timestamp) {
    this.updated = timestamp;
    this.created = timestamp;
}

```


이 방식을 적용하면 `BeanDeserializerBuilder.getProperties()` 결과에 애초에 이 가짜 프로퍼티들이 포함되지 않으므로, Modifier 로직에서 `null` 필드를 만날 일이 사라집니다.

#### 방안 B: 커스텀 `DeserializerModifier` 내부에 방어 로직 추가

DTO 소스 코드를 수정할 수 없는 상황이거나, 범용적인 처리를 원할 경우 커스텀 Modifier 내부에서 방어 코드(`BeanPropertyDefinition.getField()`의 `null` 처리) 를 작성해야 합니다.


### 🏛️ 아키텍트의 권고

방안 A (`@JsonIgnore` 추가)를 적용하는 것을 강력히 권장합니다.

방안 B처럼 방어 코드를 추가하는 것도 필요하지만, 근본적인 원인은 직렬화/역직렬화에 참여하지 않아야 할 '비즈니스 로직성 편의 메서드'가 Jackson의 프로퍼티 인식 규칙에 걸려들었다는 점입니다.  
해당 메서드에 `@JsonIgnore`를 명시적으로 선언함으로써 클래스의 데이터 바인딩 의도를 명확하게 표현(Explicit is better than implicit)하는 것이 더 견고한 설계입니다.
