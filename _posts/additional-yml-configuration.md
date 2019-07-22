# code-example-to-springboot
Code Example to spring boot.


# configuration (설정) 
1. Additional configuration files.\
기본 application.yml 에 추가로 xxx.yml 을 작성해서 사용하는...

예시)
프로그램 경로 하위 foo/ 디렉토리 아래에
__./config/bar.yml (새로 추가한 설정파일)__
```yml
foo:
  bar:
    - 
      name: Foo
      age: 29
    -
      name: Foo1
      age: 31
```
__Bar.java: 데이터 모델__
```java
package my.model;

public class Bar {
  private String name;
  private int age = -1;
  
  public Bar(){
  }
  
  public int getAge(){
    return age;
  }
  
  public String getName(){
    return name;
  }
  
  public void setAge(int age){
    this.age = age;
  }
  
  public void setName(String name) {
    this.name = name;
  }
```
__MyConfigration.java: 설정 클래스__
```java
package my.configuration;

import java.util.List;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

@Configuration
public class MyConfiguration{

  public static final String BEAN_QUALIFIER_BAR = "my.configuration.MyConfiguration#Bar";

  @Bean(BEAN_QUALIFIER_BAR)
  @ConfigurationProperteis("foo.bar")
  public List<Bar> getBar(){
    return new ArrayList<Bar>();
  }
}
```

__MyController.java: 사용예시 테스트 클래스.__
```java
package my.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

import my.config.MyConfiguration;

@RestController
@RequestMapping("/my")
public class MyController {

  private List<Bar> bar;
  
  public MyController(@Qualifier(MyConfiguration.BEAN_QUALIFIER_BAR) List<Bar> bar){
    this.bar = bar;
  }
  
  @RequestMapping(path="/bar", method = RequestMethod.GET, produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
  public ResponseEntity<Object> getBar(){
    return ResponseEntity.ok(bar);
  }
}

```

__프로그램 구동명령__
```
  java -jar myproject.jar --spring.config.additional-locatoin=file:./config/bar.yml
```

__데이터 조회__
```bash
  curl -X GET http://localhost:1234/my/bar
```


