# SPRING ADVANCED

## Lv 0. 프로젝트 세팅 - 에러 분석

**1. Unable to start web server**

  스프링 부트 애플리케이션이 실행되지 못하고 종료됨

**2. Unable to start embedded Tomcat**

  내장 톰캣 서버가 시작되지 못함

**3. Error creating bean with name 'filterConfig' defined in file [/sparta/spring-advanced/build/classes/java/main/org/example/expert/config/FilterConfig.class]: Unsatisfied dependency expressed through constructor parameter 0:**

  `FilterConfig`에서 Bean 생성 실패

**4. Error creating bean with name 'jwtUtil': Injection of autowired dependencies failed**

  `jwtUtil` 객체 생성 중 의존성을 주입받지 못해 오류 발생

**5. Could not resolve placeholder 'jwt.secret.key' in value "${jwt.secret.key}"**

  `value "${jwt.secret.key}"`로 주입받으려는 값을 스프링이 설정 파일에서 찾지 못함
  
---

/resources/application.yml 경로의 파일 생성

```
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/expert
    username: root
    password: 12345678
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: create-drop
    properties:
      hibernate:
        format_sql: true
    defer-datasource-initialization: true

jwt:
  secret:
    key: xE/a27tV8qIFCSWFkbGJWLxKD1neSI+s9EHRxQbipm8=
```

`secret key`는 base64 기반의 코드를 사용

---

## Lv 1. ArgumentResolver

`WebMvcConfigurer`에 `AuthUserArgumentResolver`를 등록

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new AuthUserArgumentResolver());
    }
}
```

+ `AuthUserArgumentResolver`에 `@Component` 어노테이션 추가

---

## Lv 2. 코드 개선

### 1. Early Return

비밀번호를 encoding하기 전에 이메일을 검증하게끔 위치 변경

### 2. 불필요한 if-else 피하기

```java
if (!HttpStatus.OK.equals(responseEntity.getStatusCode())) {
    throw new ServerException("날씨 데이터를 가져오는데 실패했습니다. 상태 코드: " + responseEntity.getStatusCode());
}

if (weatherArray == null || weatherArray.length == 0) {
    throw new ServerException("날씨 데이터가 없습니다.");
}
```

불필요한 `if-else` 삭제

### 3. Validation

```java
@NotBlank
@Pattern(regexp = "^(?=.*[A-Z])(?=.*\\d)[A-Za-z\\d@$!%*?&]{8,}$",
        message = "새 비밀번호는 8자 이상이어야 하고, 숫자와 대문자를 포함해야 합니다.")
private String newPassword;
```

`UserChangePasswordRequestDto`를 수정

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<Map<String, Object>>
handleMethodArgumentNotValidException(MethodArgumentNotValidException ex) {
    String errorMessage = ex.getBindingResult().getFieldErrors().stream()
            .findFirst()
            .map(fieldError -> fieldError.getDefaultMessage())
            .orElse("입력 값이 올바르지 않습니다.");

    HttpStatus status = HttpStatus.BAD_REQUEST;

    return getErrorResponse(status, errorMessage);
}
```

`GlobalExceptionHandler`에 `MethodArgumentNotValidException` 예외 처리

---

## 3. N+1 문제

```java
@EntityGraph(attributePaths = {"user"})
Page<Todo> findAllByOrderByModifiedAtDesc(Pageable pageable);
```

`@Query`를 삭제하고 `@EntityGraph`를 추가

---

## 4. 테스트코드 연습

### 1. 예상대로 성공하는지에 대한 케이스

matches 메서드의 인자를 반대로 입력했으므로 수정

### 2-1. 예상대로 예외처리를 하는지에 대한 케이스

```java
@Test
public void manager_목록_조회_시_Todo가_없다면_InvalidRequestException_에러를_던진다() {
    // given
    long todoId = 1L;
    given(todoRepository.findById(todoId)).willReturn(Optional.empty());

    // when & then
    InvalidRequestException exception = assertThrows(InvalidRequestException.class,
            () -> managerService.getManagers(todoId)
    );
    assertEquals("Todo not found", exception.getMessage());
}
```

`NPE` 에러를  `InvalidRequestException`으로 수정

### 2-2.

```java
InvalidRequestException exception = assertThrows(
        InvalidRequestException.class, () -> {
    commentService.saveComment(authUser, todoId, request);
});
```

`ServerException`을 `InvalidRequestException`으로 수정

### 2-3.

```java
if (todo.getUser() == null || !ObjectUtils.nullSafeEquals(user.getId(), todo.getUser().getId())) {
    throw new InvalidRequestException("일정을 생성한 유저만 담당자를 지정할 수 있습니다.");
}
```

`todo.getUser() == null`일 때 `NPE`가 아닌 `InvalidRequestException`가 발생될 수 있도록 if문에 추가
