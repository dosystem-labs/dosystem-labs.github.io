---
title: '[멘토링 2주차] Spring Boot 기초'
description: 'Spring IoC/DI, Bean, 레이어드 아키텍처 학습 과정'
pubDate: 2026-11-12
author: hogyun-lee
category: 'study'
tags: ['Java', 'Spring', 'IoC', 'DI', 'Bean', '레이어드 아키텍처']
draft: false
---

안녕하세요 개발 5팀 이호균입니다. 이번 주에는 IoC, DI, Bean의 개념과 레이어드 아키텍처에 대해 정리했습니다.

## 학습 과정

멘토링 1주차에는 순수 Java로 도서 관리 프로그램을 구현하며 객체의 생성과 의존관계를 직접 관리했습니다. 이후 Spring Boot를 적용하면서 이러한 과정을 Spring이 어떻게 대신 관리하는지 알아봤습니다.

2주차에는 회원 API를 개발하며 레이어드 아키텍처를 적용했고, 이 과정에서 Spring의 객체 관리 방식에 대해 더욱 깊이 학습하게 되었습니다.

## 본문

### 1. IoC (Inversion of Control)란?

IoC는 제어의 역전을 의미합니다. Spring에서는 객체의 생성과 관리에 대한 제어권을 개발자가 직접 가지는 대신 스프링 컨테이너에 위임합니다.

기존에는 개발자가 new 키워드를 사용해 필요한 객체를 직접 생성하고 관리했지만, IoC를 적용하면 이러한 책임을 외부에 맡길 수 있습니다.

이를 통해 개발자는 객체 관리보다 비즈니스 로직에 집중할 수 있습니다.

### 2. 스프링 컨테이너(Spring Container)란?

스프링 컨테이너는 IoC를 실제로 구현하는 주체입니다. 

Spring Boot에서는 `SpringApplication.run()`을 호출하면 스프링 컨테이너(ApplicationContext)가 자동으로 생성됩니다. 따라서 개발자가 스프링 컨테이너를 직접 생성하거나 관리할 필요가 없습니다.

```java
@SpringBootApplication
public class MentoringApplication {

    public static void main(String[] args) {
        SpringApplication.run(MentoringApplication.class, args);
    }
}
```

생성된 스프링 컨테이너는 Bean을 관리하고 객체 간의 의존성을 주입(DI)하는 역할을 수행합니다.

의존성은 하나의 클래스가 자신의 기능을 수행하기 위해 다른 클래스나 인터페이스를 필요로 하는 것을 의미합니다.

### 3. DI (Dependency Injection)란?

DI는 **의존성 주입**을 의미합니다. 필요한 객체를 직접 생성하지 않고 외부에서 전달받는 방식입니다.

#### 기존 방식

```java
public class BookServiceImpl implements BookService {
    private final BookRepository repository = new MemoryBookRepository();
}
```

현재 Service에서는 `MemoryBookRepository` 구현체(클래스)를 직접 생성하여 사용하고 있습니다.

```java
public class BookServiceImpl implements BookService {
    private final BookRepository repository = new MySQLBookRepository();
}
```

만약 데이터 저장 방식을 메모리에서 MySQL로 변경하려면 위와 같이 Service 코드를 직접 수정해야 합니다.

즉, Service가 Repository의 구체적인 구현체에 의존하고 있어 사용하려는 구현체가 변경될 때마다 Service 코드도 수정해야 하는 문제가 발생합니다.

이러한 문제를 해결하기 위해 DI(의존성 주입)를 적용할 수 있습니다.


#### 생성자 주입 방식

```java
public class BookServiceImpl implements BookService {
    private final BookRepository repository;

    public BookServiceImpl(BookRepository repository) {
        this.repository = repository;
    }
}
```

기존에는 Service 내부에서 Repository 구현체를 직접 생성했지만, 이제는 생성자를 통해 외부에서 구현체를 전달받습니다.

#### AppConfig를 통한 의존성 주입

```java
public class AppConfig {

    public BookRepository bookRepository() {
        return new MemoryBookRepository();
        // return new MySQLBookRepository();
    }

    public BookServiceImpl bookService() {
        return new BookServiceImpl(bookRepository());
    }
}
```

이것이 가능한 이유는 AppConfig에서 사용할 Repository 구현체를 미리 결정해놓고, Service 객체를 생성할 때 해당 구현체를 전달하기 때문입니다.

따라서 데이터 저장 방식을 Memory에서 MySQL로 변경하더라도 Service 코드를 수정할 필요가 없고, AppConfig의 코드만 수정하면 됩니다.

이처럼 객체의 생성과 의존관계 설정을 외부에서 담당하도록 분리하면 유지보수성을 높일 수 있습니다.

```java
public class Main {

    public static void main(String[] args) {

        AppConfig appConfig = new AppConfig();
        BookService service = appConfig.bookService();
    }
}
```

하지만 이러한 방식도 여전히 Main에서 직접 AppConfig 객체를 생성하고 관리해야하는 불편함이 있습니다. Spring에서는 객체를 Bean으로 등록하여 이러한 객체 관리 작업을 스프링 컨테이너에 위임할 수 있습니다.

### 4. 스프링 빈(Spring Bean)이란?

스프링 빈은 **스프링 컨테이너가 관리하는 객체**입니다.

객체를 Bean으로 등록하면 스프링 컨테이너가 객체의 생성, 의존성 주입, 생명주기 등을 관리합니다.

Bean을 등록하는 방법은 크게 수동 등록과 자동 등록으로 나뉩니다.

#### 수동 등록: @Bean, @Configuration

기존 AppConfig에 @Configuration과 @Bean 어노테이션을 추가하여 Bean을 등록할 수 있습니다.

```java
@Configuration
public class AppConfig {

    @Bean
    public BookRepository bookRepository() {
        return new MemoryBookRepository();
    }

    @Bean
    public BookServiceImpl bookService() {
        return new BookServiceImpl(bookRepository());
    }
}
```

```java
public class Main {

    public static void main(String[] args) {

        ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

        BookService service = context.getBean(BookService.class);
    }
}
```

이제 Main에서 AppConfig 객체를 직접 생성하거나 메서드를 호출할 필요가 없기때문에 객체 관리에 대한 부담을 줄일 수 있습니다.

#### 자동 등록: @RestController, @Service, @Repository

수동 등록 방식은 여전히 AppConfig에서 객체 생성과 의존관계 설정 코드를 직접 작성해야 합니다.

Spring에서는 @RestController, @Service, @Repository 어노테이션을 사용하여 자동화할 수 있습니다.

```java
@Repository
public class MemoryBookRepository implements BookRepository {
    
}
```

```java
@Service
public class BookServiceImpl implements BookService {
    private final BookRepository repository;

    public BookServiceImpl(BookRepository repository) {
        this.repository = repository;
    }
}
```

자동 등록을 사용하면 AppConfig에 객체를 하나씩 등록할 필요가 없고, AppConfig 클래스도 제거할 수 있습니다.

### 5. 레이어드 아키텍처(Layered Architecture)란?

레이어드 아키텍처는 애플리케이션을 역할과 책임에 따라 여러 계층으로 분리하는 설계 방식입니다.

이를 적용함으로써 코드의 가독성과 유지보수성을 높일 수 있습니다.

#### 계층별 역할

| 계층 | 역할 | 어노테이션 |
|---|---|---|
| Controller | 클라이언트의 HTTP 요청 및 응답 처리 | `@RestController` |
| Service | 비즈니스 로직 처리 | `@Service` |
| Repository | 데이터베이스 접근 및 데이터 관리 | `@Repository` |

#### 프로젝트 구조

이번 멘토링 프로젝트에서는 다음과 같이 계층을 분리하여 회원 API를 구현했습니다.

```text
member/
├── controller/
│   ├── AuthController.java
│   └── MemberController.java
├── service/
│   ├── MemberService.java
│   └── MemberServiceImpl.java
├── repository/
│   └── MemberRepository.java
├── entity/
│   └── Member.java
└── dto/
    ├── SignUpRequest.java
    └── SignUpResponse.java
```

#### 계층 간 의존관계

**Controller**

```java
@RestController
public class MemberController {

    private final MemberService memberService;

    public MemberController(MemberService memberService) {
        this.memberService = memberService;
    }
}
```

Controller는 Service에 의존하며, 클라이언트의 요청을 받아 Service에 전달합니다.

**Service**

```java
@Service
public class MemberServiceImpl implements MemberService {

    private final MemberRepository memberRepository;

    public MemberServiceImpl(MemberRepository memberRepository) {
        this.memberRepository = memberRepository;
    }
}
```

Service는 Repository에 의존하며, 회원가입, 로그인 등의 비즈니스 로직을 처리합니다.

**Repository**

```java
public interface MemberRepository extends JpaRepository<Member, Long> {

    boolean existsByLoginId(String loginId);
}
```

Repository는 DB에 접근하여 데이터를 조회하거나 저장합니다.

#### Request 처리 흐름

```text
Client
    ↓ 
Controller
    ↓ 
Service
    ↓ 
Repository
    ↓ 
Database
```

예를 들어 회원가입 요청이 들어오면 다음과 같은 과정으로 처리됩니다.

1. Controller에서 회원가입 요청을 받습니다.
2. Service에서 중복 검사, 비밀번호 검증 및 해시 처리를 수행합니다.
3. Repository를 통해 회원 정보를 데이터베이스에 저장합니다.
4. 처리 결과를 Controller를 통해 클라이언트에 반환합니다.

#### 레이어드 아키텍처의 장점

- 계층별 역할과 책임 분리
- 코드의 가독성 및 유지보수성 향상
- 코드 변경에 따른 다른 계층의 영향 최소화