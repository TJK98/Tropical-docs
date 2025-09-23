# [Troubleshooting] JSON 순환 참조로 인한 Whitelabel Error Page 발생

> **날짜**: 2025-09-15  
> **담당자**: 신동준  
> **관련 기능**: Schedule, Diary API

---

## 1. 문제 상황 (Problem)

- API 테스트 시, 데이터베이스에 데이터가 없을 때는 정상적으로 빈 배열(`[]`)이 반환됨.
- 하지만, 더미 데이터를 삽입하고 API를 호출하면 **Whitelabel Error Page (500 Internal Server Error)** 가 발생함.
- 서버 로그에는 `StackOverflowError`가 기록됨.

## 2. 문제 원인 (Cause)

이 문제의 핵심 원인은 **JSON 직렬화 과정에서의 무한 순환 참조(Infinite Circular Reference)** 입니다.

1.  **양방향 연관관계**: `Schedule` ↔ `User` 그리고 `Diary` ↔ `User`는 양방향 연관관계로 설정되어 있습니다.
    -   `Schedule` 엔티티는 `User` 객체를 참조합니다.
    -   `User` 엔티티는 `List<Schedule>`을 참조합니다.

2.  **JSON 직렬화 과정**:
    - Spring의 `ResponseEntity`는 객체를 JSON으로 변환하기 위해 Jackson 라이브러리를 사용합니다.
    - Jackson이 `Schedule` 객체를 JSON으로 변환하려고 시도합니다.
    - `Schedule`의 `user` 필드를 변환하기 위해 `User` 객체로 이동합니다.
    - `User` 객체를 변환하던 중, `schedules` 리스트 필드를 발견하고 다시 `Schedule` 객체로 이동합니다.
    - 이 과정이 **`Schedule` → `User` → `Schedule` → `User` ...** 처럼 무한 반복되면서 `StackOverflowError`가 발생하고, 서버는 500 에러를 반환합니다.

```
// 순환 참조 구조 예시

Schedule {
  Long id;
  String title;
  User user; // -> User 객체 참조
}

User {
  Long id;
  String nickname;
  List<Schedule> schedules; // -> 다시 Schedule 리스트 참조 (무한 반복 시작)
}
```

## 3. 해결 방안 (Solution)

순환 참조의 고리를 끊기 위해, 한쪽 방향의 참조를 JSON 변환 과정에서 무시하도록 설정해야 합니다. 일반적으로 자식(N) 엔티티에서 부모(1) 엔티티를 참조하는 필드에 어노테이션을 추가합니다.

-   **`@JsonIgnore` 어노테이션 사용**: `Schedule`과 `Diary` 엔티티의 `user` 필드 위에 `@JsonIgnore`를 추가하여, JSON으로 변환될 때 해당 필드를 무시하도록 설정합니다.

### 적용 코드

**`Schedule.java`**
```java
// ...
import com.fasterxml.jackson.annotation.JsonIgnore;
// ...

public class Schedule {
    // ...
    @JsonIgnore // JSON 직렬화 시 순환 참조 방지
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
    // ...
}
```

**`Diary.java`**
```java
// ...
import com.fasterxml.jackson.annotation.JsonIgnore;
// ...

public class Diary {
    // ...
    @JsonIgnore // JSON 직렬화 시 순환 참조 방지
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
    // ...
}
```

## 4. 결과 (Result)

- `@JsonIgnore`를 적용한 후, `Schedule`이나 `Diary` 객체가 JSON으로 변환될 때 `user` 필드는 포함되지 않습니다.
- 순환 참조가 발생하지 않아 `StackOverflowError`가 해결되었습니다.
- API가 정상적으로 데이터를 담은 JSON 응답을 반환합니다.

---

