---
title: 'Spring Boot 예외 처리와 로그, 테스트 코드'
description: '게시판 API의 에러 응답을 통일한 방법, 처리하지 못한 예외를 로그로 남긴 기준, 예외 없이 잘못된 값이 나간 문제를 테스트 코드로 확인한 과정을 정리했습니다.'
pubDate: '2026-11-27'
category: 'study'
tags: ['Spring Boot', '예외 처리', '로그', 'JUnit', '4주 차']
author: 'hyunmin-choo'
draft: false
---

4주 차 과제로 게시판 API의 예외 처리를 정리했습니다. 에러 응답을 하나의 형식으로 맞춘 이유와 방법, 처리하지 못한 예외를 로그로 남긴 기준, 그리고 예외가 나지 않는 문제를 테스트 코드로 확인한 과정을 순서대로 정리합니다.

## 공통 에러 응답

### 응답 형식을 통일한 이유

3주 차까지는 모든 에러코드가 통일되지 않고 서비스에서 걸러지는 에러는`{code, message}`로 나가고 그외에는 Spring Boot 기본 응답으로 나갔습니다.

| 실패 | 응답 |
|---|---|
| 아이디 중복 (서비스에서 던진 예외) | `{"code": "DUPLICATE_ID", "message": "이미 사용중인 아이디 입니다."}` |
| 페이지 번호 0, 글 번호에 문자, 깨진 JSON, 없는 주소 | Spring Boot 기본 오류 응답 (`timestamp`, `status`, `error`, `path`) |

기본 오류 응답에는 `code`가 없어서 프론트가 실패 종류를 구분할 수 없습니다. 그래서 모든 실패 응답을 `{code, message}` 형식으로 맞췄습니다.

```java
public record ErrorResponse(String code, String message) { }
```

### ErrorCode

서비스에서만 사용되던 상태 코드와 메시지를 모든 경우의 에러도 `ErrorCode` enum에 모았습니다. 
서비스뿐만 아니라 모든 에러 메시지나 상태 코드를 바꿀 때 이 파일만 수정하면 됩니다.

```java
public enum ErrorCode {
    ...

    INVALID_INPUT(HttpStatus.BAD_REQUEST, "입력값이 올바르지 않습니다." ),
    
    //작성자 불일치 에러코드
    WRITER_MISMATCH(HttpStatus.FORBIDDEN, "작성자가 일치하지 않습니다."),
    
    //요청 에러코드
    INVALID_JSON(HttpStatus.BAD_REQUEST, "형식이 올바르지 않습니다."),
    API_NOT_FOUND(HttpStatus.NOT_FOUND, "존재하지 않는 요청입니다."),
    METHOD_NOT_ALLOWED(HttpStatus.METHOD_NOT_ALLOWED, "지원하지 않는 메서드입니다."),
    UNSUPPORTED_MEDIA_TYPE(HttpStatus.UNSUPPORTED_MEDIA_TYPE, "JSON 형식으로 요청해주세요."),

    //인증 에러코드 (로그인 기능 전까지 X-Member-Id 헤더로 확인)
    LOGIN_REQUIRED(HttpStatus.UNAUTHORIZED, "로그인이 필요합니다."),

    //DB제약조건 에러코드 (원인은 로그로 남기고 사용자에게는 일반 문구)
    DUPLICATE_VALUE(HttpStatus.CONFLICT, "이미 사용 중인 값입니다."),

    //처리하지 못한 에러코드
    INTERNAL_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "서버 오류가 발생했습니다.");

    ...
}
```


### 요청 단계별 예외 처리

요청은 주소 찾기, 파라미터와 본문 변환, 검증, 컨트롤러, 서비스 순서로 처리되고 단계마다 다른 예외가 발생합니다. 서비스의 `CustomException`만 처리하면 컨트롤러에 도착하기 전에 나는 예외는 기본 오류 응답으로 나갑니다.

| 요청 | 발생하는 예외 | 응답 |
|---|---|---|
| `GET /nope` | `NoResourceFoundException` | 404 `API_NOT_FOUND` |
| `GET /members/signup` (POST만 있음) | `HttpRequestMethodNotSupportedException` | 405 `METHOD_NOT_ALLOWED` |
| `GET /posts/abc/detail` | `MethodArgumentTypeMismatchException` | 400 `INVALID_INPUT` |
| 깨진 JSON, 본문 없음 | `HttpMessageNotReadableException` | 400 `INVALID_JSON` |
| 제목 101자 (`@Valid` 검증 실패) | `MethodArgumentNotValidException` | 400 + 검증 메시지 |
| `?page=0` (`@Min(1)` 검증 실패) | `HandlerMethodValidationException` | 400 + 검증 메시지 |
| 없는 글에 댓글 작성 | `CustomException` | 404 `POST_NOT_EXIST` |

`@RestControllerAdvice` 클래스에 예외별 처리기를 두고, 응답을 만드는 부분은 `toResponse` 메서드로 모았습니다. 대부분은 `ErrorCode`의 기본 메시지를 그대로 쓰고, `@Valid` 검증 실패처럼 메시지를 바꿔야 할 때만 메시지를 직접 넘기도록 두 가지로 나눴습니다.

```java
//타입 변환 에러처리
@ExceptionHandler(MethodArgumentTypeMismatchException.class)
public ResponseEntity<ErrorResponse> handleTypeMismatch(MethodArgumentTypeMismatchException e) {
    return toResponse(ErrorCode.INVALID_INPUT);
}

//ErrorCode의 기본 메시지로 응답
private ResponseEntity<ErrorResponse> toResponse(ErrorCode errorCode) {
    return toResponse(errorCode, errorCode.getMessage());
}

//메시지를 직접 지정해서 응답 (검증 메시지 등)
private ResponseEntity<ErrorResponse> toResponse(ErrorCode errorCode, String message) {
    return ResponseEntity.status(errorCode.getHttpStatus())
            .body(new ErrorResponse(errorCode.name(), message));
}
```

## 처리하지 못한 예외와 로그

### 전체 예외 처리기

모든 예외를 미리 알 수는 없으므로, 마지막에 `Exception`을 받는 처리기를 두었습니다.

```java
//그 밖의 모든 에러처리 (원인은 로그로만 남김)
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleException(Exception e) {
    log.error("처리하지 못한 예외", e);
    return toResponse(ErrorCode.INTERNAL_ERROR);
}
```

응답에는 "서버 오류가 발생했습니다."만 보냅니다. 예외 메시지를 그대로 보내면 SQL, 테이블 이름, 클래스 경로 같은 내부 정보가 노출될 수 있기 때문입니다. 원인은 서버 로그에 남깁니다. `log.error("...", e)`처럼 예외 객체를 마지막 인자로 넘기면 예외 종류와 스택 트레이스까지 기록됩니다.
로그를 통해 처리하지 못한 예외를 하나하나 추가합니다.

### 로그 레벨 기준

모든 실패를 로그로 남기면 확인해야 할 로그를 찾기 어렵습니다.

| 경우 | 로그 | 이유 |
|---|---|---|
| 입력 형식 오류, 중복, 권한 없음 (400대) | 남기지 않음 | 사용자 입력 문제이고, 응답으로 이미 안내함 |
| DB 제약 위반 | `warn` | 서비스에서 먼저 중복을 확인하므로 동시에 같은 값으로 가입할 때만 발생. 드물지만 원인 확인이 필요함 |
| 처리하지 못한 예외 | `error` | 예상하지 못한 경우로, 수정이 필요함 |

이 기준에서는 `ERROR` 로그가 남은 경우가 곧 처리기가 없는 경우입니다.

### 로그로 확인한 처리 누락

처리기를 정리한 뒤 잘못된 요청을 보내고 `ERROR` 로그를 확인했습니다. 세 가지 경우가 남아 있었습니다.

```text
ERROR c.e.m.exception.GlobalExceptionHandler : 처리하지 못한 예외
org.springframework.web.bind.MissingRequestHeaderException: Required request header 'X-Member-Id' for method parameter type Long is not present

ERROR c.e.m.exception.GlobalExceptionHandler : 처리하지 못한 예외
org.springframework.web.HttpMediaTypeNotSupportedException: Content-Type 'text/plain;charset=UTF-8' is not supported

ERROR c.e.m.exception.GlobalExceptionHandler : 처리하지 못한 예외
org.springframework.web.HttpMediaTypeNotAcceptableException: No acceptable representation
```

| 요청 | 응답 | 수정 후 응답 |
|---|---|---|
| 로그인 헤더 `X-Member-Id` 없이 글 작성 | 500 `INTERNAL_ERROR` | 401 `LOGIN_REQUIRED` |
| `Content-Type: text/plain`으로 회원가입 | 500 `INTERNAL_ERROR` | 415 `UNSUPPORTED_MEDIA_TYPE` |
| `Accept: application/xml`로 목록 조회 | 406 + `ERROR` 로그 | 406, 로그 없음 |

세 예외는 원래 Spring이 400, 415, 406으로 응답하던 예외입니다. `Exception` 처리기를 추가하면서 별도 처리기가 없던 이 예외들이 모두 이 처리기로 들어왔고, 결과는 두 가지로 나뉘었습니다.

- 헤더 누락과 `Content-Type` 오류는 500 `INTERNAL_ERROR`로 바뀌었습니다. 헤더 누락은 처리기를 추가하기 전에는 400이었습니다.
- `Accept` 오류는 응답은 406 그대로였지만 `ERROR` 로그가 남았습니다.

`Accept` 오류만 406으로 남은 이유는 응답 본문을 쓸 수 없었기 때문입니다. `Exception` 처리기는 `ErrorResponse`를 반환하지만, 클라이언트가 XML만 받겠다고 요청했기 때문에 JSON 본문을 쓸 수 없습니다. 처리기의 응답이 실패하면 Spring이 원래 예외를 다시 처리해서 406으로 응답합니다. 상태 코드는 맞게 나가지만 처리기에서 `log.error`가 이미 실행된 뒤라 `ERROR` 로그가 남습니다.

사용자 입력 문제가 500으로 나가면 프론트는 서버 오류로 안내하게 되고, `ERROR` 로그에 서버 문제가 아닌 기록이 섞입니다. 세 예외에 처리기를 추가했습니다.

```java
//로그인 헤더(X-Member-Id) 누락 에러처리
@ExceptionHandler(MissingRequestHeaderException.class)
public ResponseEntity<ErrorResponse> handleMissingHeader(MissingRequestHeaderException e) {
    return toResponse(ErrorCode.LOGIN_REQUIRED);
}
```

406 처리기는 다른 처리기와 다르게 본문 없이 응답합니다. 클라이언트가 받을 수 있는 형식이 JSON이 아니므로 `ErrorResponse`를 본문에 넣으면 같은 이유로 다시 실패합니다. 그래서 `ErrorCode`를 쓰지 않고 상태 코드만 보냅니다.

```java
//응답 형식(Accept) 오류 처리 (JSON 본문을 쓸 수 없으므로 본문 없이 응답)
@ExceptionHandler(HttpMediaTypeNotAcceptableException.class)
public ResponseEntity<Void> handleNotAcceptable(HttpMediaTypeNotAcceptableException e) {
    return ResponseEntity.status(HttpStatus.NOT_ACCEPTABLE).build();
}
```

## 예외 없이 잘못된 값이 나가는 경우

예외 처리기와 로그는 예외가 발생해야 동작합니다. 예외 없이 200 OK로 잘못된 값을 응답하면 둘 다 확인할 수 없습니다.

게시글 상세 조회의 조회수가 그런 경우였습니다. 같은 회원이 같은 글을 여러 번 봐도 조회수가 한 번만 오르도록 조회 기록을 저장하는데, 처음 코드는 다음과 같았습니다.

```java
public PostDetailResponse detail(Long postId, Long memberId) {
    Post post = findPost(postId);                       //1. 게시글 조회 (조회수 5)
    ...
    if (memberId == null || postViewRepository.insertIfAbsent(postId, memberId) == 1) {
        postRepository.plusViewCount(postId);           //2. DB에서 조회수 +1
    }
    return PostDetailResponse.from(post);               //3. 응답에서 getViewCount() + 1
}
```

1번에서 조회한 엔티티에는 증가 전 값이 들어 있어서 응답에서 `+1`을 했습니다. 처음 보는 회원은 맞는 값이 나오지만, 이미 본 회원은 2번을 건너뛰고도 3번에서 `+1`이 됩니다. 

이 경우 예외와 로그가 없고 상태 코드도 200이라, 같은 글을 두 번 이상 열어 숫자를 비교하지 않으면 확인할 수 없습니다.

## 테스트 코드로 확인하기

### 조회수 응답 테스트

테스트 코드는 특정 상황에서 나와야 하는 결과를 코드로 적어 둔 것입니다. 예외가 나지 않아도 기대값과 실제 값이 다르면 실패합니다. 위 문제는 다음 테스트로 확인했습니다.

```java
@Test
@DisplayName("상세 조회 성공: 이미 본 회원이면 조회수를 올리지 않고, 응답 조회수도 그대로")
void detail_secondView() {
    given(postRepository.findById(10L)).willReturn(Optional.of(post));
    given(postViewRepository.insertIfAbsent(10L, 2L)).willReturn(0);   //이미 본 회원

    PostDetailResponse response = postService.detail(10L, 2L);

    verify(postRepository, never()).plusViewCount(anyLong());
    assertThat(response.viewCount()).isEqualTo(0);
}
```

처음에는 조회수 증가 메서드가 호출되지 않는지만 확인했고 테스트는 통과했습니다. 응답 값을 확인하는 마지막 줄을 추가하자 실패했습니다.

조회수 증가를 게시글 조회보다 먼저 하도록 순서를 바꾸고, 응답의 `+1`을 제거했습니다. 트랜잭션 안에서 게시글을 처음 조회하는 시점이 UPDATE 이후이므로 DB의 현재 값이 그대로 응답됩니다.

```java
public PostDetailResponse detail(Long postId, Long memberId) {
    if (memberId == null || postViewRepository.insertIfAbsent(postId, memberId) == 1) {
        postRepository.plusViewCount(postId);           //1. DB에서 조회수 +1
    }
    Post post = findPost(postId);                       //2. 게시글 조회 (증가된 값)
    if (!post.isVisible()) {
        throw new CustomException(ErrorCode.POST_NOT_EXIST);
    }
    return PostDetailResponse.from(post);               //3. 그대로 응답
}
```

이 테스트는 이전 코드에서 실패하고 수정한 코드에서 통과합니다. 처음 버전의 테스트가 통과했던 것처럼, 테스트는 기대값을 적은 부분만 확인합니다.

### 순서 변경 후 비노출 테스트

순서를 바꾼 뒤 "비노출 글이면 조회수를 올리지 않는다"를 확인하던 테스트가 실패했습니다. 조회수 증가가 비노출 확인보다 먼저 실행되기 때문입니다.

실제 실행에서는 비노출 확인에서 예외가 발생하면 `@Transactional`이 트랜잭션을 롤백하므로, 앞에서 실행한 조회 기록 저장과 조회수 증가도 취소됩니다. `CustomException`은 `RuntimeException`을 상속하므로 롤백 대상입니다. 이 내용은 서비스 코드에 주석으로 남겼습니다.

### 단위 테스트로 확인할 수 없는 것

위 테스트는 리포지토리를 Mock 객체로 대체한 서비스 단위 테스트입니다. 빠르고 원하는 상황을 쉽게 만들 수 있지만, 다음은 확인할 수 없습니다.

| 확인할 수 없는 것 | 이유 | 확인 방법 |
|---|---|---|
| 예외 발생 시 롤백 | Mock 객체에는 트랜잭션이 없음. 그래서 비노출 테스트는 예외만 확인하도록 수정함 | 실제 DB를 사용하는 `@SpringBootTest` 통합 테스트 |
| 조회 기록 중복 판단 | 테스트용 DB(H2)는 `ON DUPLICATE KEY UPDATE`에서 중복이어도 1을 반환하고, MySQL(`useAffectedRows=true`)은 0을 반환함 | 운영과 같은 MySQL로 테스트 (Testcontainers) |
| 동시 요청 | 테스트는 요청을 하나씩 실행함 | 동시성 테스트 |

조회 기록 중복 판단은 H2에서 확인할 수 없어서, 단위 테스트에서 `insertIfAbsent`의 반환값을 0 또는 1로 지정하고 그 이후 동작만 확인했습니다. 테스트 환경이 운영 환경과 다르면 그 차이는 테스트로 확인되지 않습니다.

### 테스트 대상 선정

모든 기능을 테스트하지 않고, 잘못되면 데이터가 잘못 바뀌거나 조건이 많아 직접 확인하기 어려운 기능 세 가지를 선택했습니다. 각 기능에 성공과 실패 경우를 함께 작성했습니다.

| 기능 | 선택 이유 | 테스트 |
|---|---|---|
| 회원가입 | 실패 경우가 가장 많고, 비밀번호 암호화는 응답에 나타나지 않음 | 서비스: 암호화 저장, 재확인 불일치, 아이디 중복 |
| 게시글 수정 권한 | 잘못되면 다른 회원의 글이 수정됨. 삭제와 노출 변경도 같은 작성자 확인을 사용 | 서비스: 작성자 수정, 다른 회원 수정 |
| 게시글 상세 조회 | 비회원, 비노출, 조회수 중복 방지가 한 메서드에 있음 | 서비스: 비회원, 처음 조회, 재조회, 비노출<br>MockMvc: 성공, 404, 400 검증, 401 헤더 없음, 415 JSON 아님 |

서비스 테스트는 비즈니스 규칙을 확인하고, MockMvc 컨트롤러 테스트는 요청이 예외 처리기를 거쳐 어떤 상태 코드와 `code`로 응답되는지 확인합니다. 

## 정리

| 방법 | 확인할 수 있는 것 | 확인할 수 없는 것 |
|---|---|---|
| 예외 처리 | 예상한 실패를 정해진 응답으로 변환 | 예상하지 못한 예외 |
| 로그 | 예상하지 못한 예외의 원인 | 예외 없이 잘못된 값 |
| 테스트 | 기대값과 다른 결과 | 기대값을 적지 않은 부분, 테스트 환경과 운영 환경의 차이 |
