---
title: 'JPA N+1 문제와 해결 방법'
description: 'N+1 문제가 생기는 원인과 해결 방법을 정리하고, 게시판 댓글 목록과 게시글 목록에 적용한 결과를 쿼리 수로 비교했습니다.'
pubDate: '2026-11-20'
category: 'study'
tags: ['JPA', 'N+1', 'EntityGraph', '3주 차']
author: 'hyunmin-choo'
draft: false
---

3주 차 과제로 게시판의 게시글과 댓글 API를 만들면서, N+1 문제가 무엇이고 왜 생기는지 정리한 뒤, 해결 방법을 비교하고 실제 게시판 코드에 적용한 결과를 쿼리 수로 확인합니다.

## N+1 문제란

N+1 문제는 목록을 조회하는 쿼리 1번으로 결과 N개를 가져온 뒤, 결과 하나하나에 대해 쿼리가 1번씩 더 실행되어 총 1+N번의 쿼리가 나가는 문제입니다.

| 상황 | 의도한 쿼리 | 실제 실행된 쿼리 |
|---|---|---|
| 댓글 10개와 각 작성자 닉네임 조회 | 1번 | 1 + 10번 |
| 댓글 100개와 각 작성자 닉네임 조회 | 1번 | 1 + 100번 |

데이터가 적을 때는 응답이 느려지지 않아서 알아채기 어렵습니다. 하지만 쿼리 수가 목록 크기에 비례해서 늘어나기 때문에, 데이터가 쌓이거나 한 페이지에 보여 주는 개수가 늘면 그만큼 DB 왕복이 늘어납니다.

## 왜 생기나

### 연관관계와 지연 로딩

게시판의 댓글 엔티티는 작성자를 `member_id` 숫자가 아니라 `Member` 객체로 가지고 있습니다. 댓글 여러 개가 회원 한 명을 가리키므로 `@ManyToOne`입니다.

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "member_id", nullable = false)
private Member member;
```

덕분에 `comment.getMember().getNickname()`처럼 객체를 따라가며 작성자 정보를 꺼낼 수 있습니다.

`fetch = FetchType.LAZY`는 지연 로딩입니다. 댓글을 조회할 때 작성자는 바로 가져오지 않고, 그 자리에 프록시라는 빈 껍데기 객체를 넣어 둡니다. 그리고 `getNickname()`처럼 실제 값을 꺼내는 순간 그 회원을 DB에서 조회합니다. 그래서 코드 한 줄마다 이런 SQL이 나갑니다.

| 코드 | 실행되는 SQL |
|---|---|
| `commentRepository.findByPostIdOrderByIdAsc(postId)` | `select ... from comment where post_id = ?` |
| `comments.get(0).getMember().getNickname()` | `select ... from member where id = ?` |
| `comments.get(1).getMember().getNickname()` | `select ... from member where id = ?` |
| ... | ... |

댓글 목록을 응답 DTO로 바꾸면서 댓글마다 작성자 닉네임을 꺼내니, 댓글 수만큼 회원 조회가 추가로 실행됩니다.

### EAGER로 바꾸면 되지 않나

`FetchType.EAGER`(즉시 로딩)로 바꾸면 댓글을 조회할 때 작성자도 함께 가져오니 해결될 것처럼 보입니다. 하지만 실제로 바꿔서 실행해 보면 쿼리 수는 그대로입니다.

쿼리 메서드나 JPQL로 목록을 조회하면 JPA는 먼저 `select ... from comment`만 실행합니다. 결과를 받은 뒤에야 "작성자는 즉시 로딩"이라는 설정을 보고 회원을 하나씩 조회합니다. 조회하는 시점이 `getNickname()`을 부를 때에서 목록을 받자마자로 바뀔 뿐, N+1은 그대로입니다. 게다가 작성자가 필요 없는 곳에서도 항상 회원을 조회하게 됩니다. 그래서 연관관계는 `LAZY`로 두고, 함께 필요한 곳에서만 한 번에 가져오는 것이 기본입니다.

### 영속성 컨텍스트와 같은 작성자

영속성 컨텍스트는 한 트랜잭션 안에서 조회한 엔티티를 기억해 두는 저장소입니다. 같은 id의 엔티티를 다시 찾으면 DB에 가지 않고 이미 가지고 있는 것을 돌려줍니다(1차 캐시).

이 때문에 N+1로 추가되는 쿼리 수는 정확히는 "댓글 수"가 아니라 "서로 다른 작성자 수"입니다. 댓글 10개를 한 사람이 썼다면 회원 조회는 1번이면 됩니다. 혼자 만든 테스트 데이터로 확인하면 쿼리가 몇 개 안 나와서 문제를 놓치기 쉽습니다.

## 해결 방법

해결 방법은 세 가지가 많이 쓰입니다. 공통 아이디어는 "회원을 하나씩 조회하지 말고 한 번에 가져오기"입니다.

| 방법 | 쓰는 법 | 실행되는 SQL | 특징 |
|---|---|---|---|
| fetch join | `@Query("SELECT c FROM Comment c JOIN FETCH c.member WHERE c.post.id = :postId ORDER BY c.id ASC")` | comment와 member를 조인해 한 번에 | JPQL을 직접 씀 |
| `@EntityGraph` | 쿼리 메서드 위에 `@EntityGraph(attributePaths = "member")` | fetch join과 같음 | 메서드 이름으로 만든 쿼리를 그대로 쓰면서 조인만 추가 |
| `default_batch_fetch_size` | `spring.jpa.properties.hibernate.default_batch_fetch_size=100` | 댓글 조회 후 `where id in (?, ?, ...)`로 회원을 한 번에 | 조인 없이 설정 한 줄로 전체에 적용. 쿼리가 1번 더 나감 |

고를 때 기준은 이렇게 정리할 수 있습니다.

- 이번 게시판처럼 `@ManyToOne`(댓글 → 작성자) 하나를 함께 가져오는 경우는 세 방법 모두 잘 동작합니다.
- `@OneToMany` 같은 컬렉션을 조인해서 가져오면서 페이징까지 하면, fetch join과 `@EntityGraph`는 페이징을 DB가 아니라 메모리에서 처리하게 되어 위험합니다. 이때는 `default_batch_fetch_size`가 많이 쓰입니다.
- fetch join으로 `Page`를 받을 때는 전체 개수를 세는 `countQuery`를 따로 지정해 주는 것이 안전합니다.

`@EntityGraph`는 left join으로 나간다는 설명을 자주 볼 수 있습니다. 그런데 이 게시판에서는 실제 로그에 `join`(inner join)으로 찍혔습니다. `Comment.member`가 `@JoinColumn(nullable = false)`라서 작성자가 없는 댓글은 있을 수 없으니, Hibernate가 inner join을 쓴 것으로 보입니다.

## 게시판에 적용하기

측정은 한 게시글에 댓글 14개(작성자 12명)를 넣은 뒤 요청 한 번에 실행된 SQL 수를 셌습니다. SQL 수는 `spring.jpa.properties.hibernate.generate_statistics=true`를 켜고 Hibernate 통계의 실행 횟수로 확인했고, `logging.level.org.hibernate.SQL=debug`로 찍은 로그 줄 수와도 맞춰 봤습니다.

### 댓글 목록

처음 댓글 목록 코드는 쿼리 메서드 하나였습니다.

```java
public interface CommentRepository extends JpaRepository<Comment, Long> {
    List<Comment> findByPostIdOrderByIdAsc(Long postId);
}
```

응답 DTO를 만들면서 댓글마다 작성자 닉네임을 꺼냅니다. 탈퇴한 회원은 "탈퇴한 회원"으로 표시하기 때문에 회원 상태도 확인합니다.

```java
public static CommentListResponse from(Comment comment) {
    return new CommentListResponse(
        comment.getId(),
        comment.getMember().getStatus() == MemberStatus.WITHDRAWN ? "탈퇴한 회원" : comment.getMember().getNickname(),
        comment.getCreatedAt(),
        comment.getContent()
    );
}
```

이 상태로 댓글 목록을 한 번 조회하면 이렇게 실행됩니다.

```sql
select count(*) from post p1_0 where p1_0.id=?                                   -- 게시글 존재 확인
select c1_0.id, ... from comment c1_0 where c1_0.post_id=? order by c1_0.id      -- 댓글 10개
select m1_0.id, ... from member m1_0 where m1_0.id=?                             -- 작성자 1
...                                                                              -- (같은 쿼리 8번)
select m1_0.id, ... from member m1_0 where m1_0.id=?                             -- 작성자 10
```


쿼리 메서드 위에 `@EntityGraph` 한 줄을 추가했습니다.

```java
@EntityGraph(attributePaths = "member")
List<Comment> findByPostIdOrderByIdAsc(Long postId);
```

이제 댓글과 작성자를 조인 한 번으로 가져옵니다.

```sql
select count(*) from post p1_0 where p1_0.id=?
select c1_0.id, ..., m1_0.id, m1_0.nickname, m1_0.status, ...
from comment c1_0 join member m1_0 on m1_0.id=c1_0.member_id
where c1_0.post_id=? order by c1_0.id
```


### 게시글 목록

게시글 목록은 N+1이 두 군데서 생깁니다. 게시글마다 작성자 닉네임을 꺼낼 때, 그리고 게시글마다 댓글 수를 셀 때입니다. 과제 요구사항에서 댓글 수는 컬럼으로 저장하지 않고 조회할 때 계산하도록 되어 있어서, 게시글마다 `count` 쿼리를 보내면 그것도 N번이 됩니다.

작성자는 댓글 목록과 같은 방법으로, 댓글 수는 게시글 id를 모아서 `group by` 쿼리 한 번으로 구했습니다.

```java
// 작성자: @EntityGraph
@EntityGraph(attributePaths = "member")
Page<Post> findByVisibleTrue(Pageable pageable);

// 댓글 수: 게시글 id를 모아 group by 한 번
@Query("SELECT c.post.id AS postId, COUNT(c) AS count FROM Comment c where c.post.id in :postIds GROUP BY c.post.id")
List<CommentCount> countByPostIds(@Param("postIds") List<Long> postIds);
```

`CommentCount`는 `getPostId()`, `getCount()`만 가진 인터페이스입니다. 쿼리의 별칭(`AS postId`, `AS count`)과 메서드 이름이 맞으면 Spring Data JPA가 결과를 채워 줍니다. 서비스에서는 이 결과를 `Map`으로 바꿔서 게시글마다 꺼내 씁니다. 댓글이 하나도 없는 게시글은 결과에 없으므로 `getOrDefault`로 0을 넣습니다.

```java
List<Long> postIds = posts.stream().map(Post::getId).toList();

Map<Long, Long> commentCountMap = commentRepository.countByPostIds(postIds).stream()
    .collect(Collectors.toMap(CommentCount::getPostId, CommentCount::getCount));

return posts.stream()
    .map(post -> PostListResponse.of(post, commentCountMap.getOrDefault(post.getId(), 0L))).toList();
```

목록 한 페이지에 실행되는 SQL은 세 개입니다.

```sql
select p1_0.id, ..., m1_0.nickname, ... from post p1_0 join member m1_0 on m1_0.id=p1_0.member_id
  where p1_0.visible=true order by p1_0.created_at desc, p1_0.id desc fetch first ? rows only   -- 게시글 + 작성자
select count(*) from post p1_0 where p1_0.visible=true                                          -- 전체 개수 (페이지 수 계산)
select c1_0.post_id, count(c1_0.id) from comment c1_0
  where c1_0.post_id in (?,?,?,?,?,?,?,?,?,?) group by c1_0.post_id                             -- 댓글 수
```

두 번째 `count(*)`는 N+1이 아니라 `Page`가 전체 페이지 수를 알려 주기 위해 실행하는 쿼리입니다.

### 결과

| 요청 | 적용 전 | 적용 후 |
|---|---|---|
| 댓글 목록(총 14개, 작성자 12명) | 13 | 2 |
| 게시글 목록 페이지(게시글 10개 작성자 10명) | 13 | 3 | 

같은 댓글 목록을 방법만 바꿔서 쟀습니다.

| 방법 | 쿼리 수 |
|---|---|
| 적용 전 (LAZY) | 13 |
| `FetchType.EAGER`로 변경 | 13 |
| fetch join | 2 |
| `@EntityGraph` | 2 |
| `default_batch_fetch_size=100` | 3 |

`EAGER`는 앞에서 설명한 대로 쿼리 수가 줄지 않았습니다. fetch join과 `@EntityGraph`는 실행된 SQL까지 같았습니다.

이 게시판에서는 `@EntityGraph`를 골랐습니다. 목록과 검색을 쿼리 메서드 이름으로 만들어 두었기 때문에, JPQL을 새로 쓰지 않고 어노테이션 한 줄로 해결할 수 있었습니다. `Page`와 함께 써도 전체 개수 쿼리를 따로 신경 쓰지 않아도 됐습니다.

## 정리

- N+1은 목록을 가져온 뒤 연관된 엔티티를 하나씩 조회해서 생깁니다. `LAZY`든 `EAGER`든 목록 조회에서는 생길 수 있습니다.
- 함께 필요한 연관 엔티티는 fetch join, `@EntityGraph`, `default_batch_fetch_size` 중 하나로 한 번에 가져옵니다.
- 같은 작성자가 많은 테스트 데이터에서는 N+1이 잘 보이지 않습니다. 목록 API를 만들면 작성자를 여러 명으로 섞은 데이터로 SQL 로그를 보고 쿼리 수를 확인하는 습관을 들이려고 합니다.
