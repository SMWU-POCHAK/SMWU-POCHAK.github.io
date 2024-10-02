---
title: 차단 로직을 Querydsl로 리팩토링한 과정
date: 2024-09-28 10:45:00 +0900
categories: [SpringBoot, Querydsl]
tags: [backend, springboot, querydsl]     # TAG names should always be lowercase
author: lucy-oh
---

# 기존 코드

## 구현해내고자 했던 기능

포착은 소셜 네트워킹 서비스로 앱스토어와 플레이스토어의 규정상 "차단" 기능이 필수적으로 필요하였습니다.

팀원들과 상의하며 규정한 차단 로직은 다음과 같습니다.

1. 사용자A가 사용자B를 차단한다.
2. 사용자A는 사용자B를 조회할수도, 사용자B가 포함된 게시물을 확인할 수 없다.
    - 여기서 사용자B가 포함된 게시물이란 사용자B가 포착하였거나, 포착된 게시물 전부
3. 사용자B도 마찬가지로 사용자A 조회가 불가능하며, 사용자A가 포함된 게시물을 확인할 수 없다.
4. 사용자A는 사용자B가 포함된 게시물이 아니더라도, 다른 게시물에 사용자B가 남긴 댓글을 확인할 수 없다. (사용자B도 마찬가지)
5. 차단한 순간 사용자A와 사용자B 사이의 팔로우 상태는 전부 끊긴다.
6. 차단한 순간 사용자A와 사용자B가 함께 포함된 게시물은 전부 INACTIVE한 상태가 된다.

"태깅" 기능이 있는 포착의 특성상 굉장히 어렵게.. 구현이 완료되었습니다.

## 리팩토링을 진행한 기존 쿼리

위의 차단 로직 중에서 2, 3번에 해당하는 "차단된 혹은 차단한 사용자가 포함된 게시물을 확인할 수 없다"는 부분을 구현한 기존 쿼리입니다.

- 기존 쿼리
```java
@Query("""
        select p from Post p
        join fetch p.owner
        where p.id = :postId and p.status = 'ACTIVE'
            and p.owner not in (select b.blockedMember from Block b where b.blocker = :loginMember)
            and :loginMember not in (select b.blockedMember from Block b where b.blocker = p.owner)
            and not exists (select t.member from Tag t where t.post = p intersect select b.blockedMember from Block b where b.blocker = :loginMember)
            and :loginMember not in (select b.blockedMember from Block b where b.blocker in (select t.member from Tag t where t.post = p))
        """)
Optional<Post> findById(
        @Param("postId") final Long postId,
        @Param("loginMember") final Member loginMember
);
```

### 이전 차단 기능 구현 방식

일단 각 조건인 and에 걸려있는 서브 쿼리인 `in (select ~ )` 절이 굉장히 많았습니다.

확인해야 할 부분이 총 4가지였습니다.
1. 현재 로그인한 사람이 조회하고자 하는 게시물의 포차커를 차단하였는가?
2. 현재 로그인한 사람이 조회하고자 하는 게시물의 포차키 중 한명이라도 차단하였는가?
3. 조회하고자 하는 게시물의 포차커가 현재 로그인한 사람을 차단하였는가?
4. 조회하고자 하는 게시물의 포차키 중 한 명이라도 현재 로그인한 사람을 차단하였는가?

> 여기서 포차커는 "포착해준 사람" 포차키는 "포착당한 사람"으로 정의됩니다.
{: .prompt-info }

### 문제점

문제는 이 조건을 모두 in (select 서브 쿼리) 절, 3번과 4번 조건의 경우엔 2번의 서브 쿼리가 필요했습니다.
- 총 6번의 서브 쿼리가 구현에 사용되었습니다.

불필요한 서브 쿼리를 남발하고 있었기에 (특히나 위 쿼리는 중복되는 쿼리 역시도 존재했기에) 개선이 굉장히 시급했습니다.

> 물론 POCHAK은 최신 버전의 MySQL(8.0.31)을 사용하고 있기 때문에, 서브쿼리의 성능이 JOIN에 비해 크게 떨어지지 않습니다. (이후에 나올 성능 측정에서 역시도 큰 차이를 보이고 있지 않습니다.) <br>
> 하지만, String으로 작성해야하는 JPQL의 특성상 재사용성이 굉장히 떨어지고, 직관적이지 못해 유지보수성 역시도 좋다고 느껴지지 않았습니다. <br>
> 따라서 앞으로 포착 애플리케이션의 유지보수를 위해서라도 복잡한 쿼리는 QueryDSL을 통해 개선하고자한 의도도 있습니다!
{: .prompt-tip }

# 환경

## 개발 환경 정보

- SpringBoot 3.2.1 (Java 17)
- MySQL (8.0.31)
- Querydsl 5.0.0

# 리팩토링 결과

## 개선된 쿼리

- 개선한 쿼리

```java
public Optional<Post> findByIdWithoutBlockPost(
        final Long postId,
        final Long loginMemberId
) {
    return Optional.ofNullable(
            query.selectFrom(post)
                    .join(post.owner).fetchJoin()
                    .join(tag).on(tag.post.eq(post))
                    .leftJoin(block).on(
                            checkOwnerOrTaggedMemberBlockLoginMember(loginMemberId)
                                    .or(checkLoginMemberBlockOwnerOrTaggedMember(loginMemberId))
                    )
                    .groupBy(post)
                    .having(block.id.count().eq(0L))
                    .where(
                            post.id.eq(postId),
                            post.status.eq(BaseEntityStatus.ACTIVE)
                    )
                    .fetchOne()
    );
}

private BooleanExpression checkOwnerOrTaggedMemberBlockLoginMember(final Long loginMemberId) {
    return (block.blocker.eq(tag.member).or(block.blocker.eq(post.owner)))
            .and(block.blockedMember.id.eq(loginMemberId));
}

private BooleanExpression checkLoginMemberBlockOwnerOrTaggedMember(final Long loginMemberId) {
    return (block.blocker.id.eq(loginMemberId))
            .and(block.blockedMember.eq(tag.member).or(block.blockedMember.eq(post.owner)));
}
```
> 메소드 명은 저도 유감입니다.. 하지만 최선이었어요🙃

### 구현 방식

서브쿼리를 줄일 수 있는 방법에 대해 고민하였고, JOIN의 형태로 해결하는 방법에 대해 고민했습니다.
- 그림에서 post.owner 페치조인의 경우는 제외하였습니다

![img](/assets/img/2024-09-28-block-querydsl/img0.jpeg)

1. 먼저, 기준테이블인 post 테이블에 tag 테이블을 inner join을 합니다. 여기서 만약 게시글에 여러명을 태그했을 경우, 위 그림과 같이 데이터 뻥튀기가 발생할 수 있지만 이는 나중에 groupby 절을 통해 해결해줄 예정입니다.
2. 그렇게 산출된 결과테이블을 기준테이블로 block을 LEFT JOIN 해줍니다. 조건은 다음과 같습니다.
    1. 태그된(촬영된) 유저 혹은 게시물을 업로드한 유저가 현재 로그인한 사람이 차단했을 경우
    2. 또는, 현재 로그인한 유저가 태그된 유저 혹은 게시물을 업로드한 유저를 차단했을 경우
    - ⇒ 해당 조건에 충족한다면 block 데이터가, 아니라면 null로 행에 추가됩니다.
3. `where` 절을 통해 현재 상태가 `ACTIVE`하고, 사용자가 요청한 `게시물 id`값이 반환될 수 있도록 필터링 해줍니다.
4. 그 다음 1. 2. 과정에서 일어날 수 있는 데이터 뻥튀기를 방지하기 위하여 `.groupBy(post)` 를 통해 중복 데이터를 막아줍니다.
5. 또한, `.having(block.id.count().eq(0L))`을 통해 만약 2. 과정의 조건에 걸려 null이 아닌 하나의 block id라도 행에 표기되어 있다면 필터링을 통해 결과에 반영되지 않도록 설정하였습니다.

### 결과

일단 자바 코드로 작성한 쿼리이기에 가독성이 좋아졌습니다. `checkOwnerOrTaggedMemberBlockLoginMember()` 메소드 등의 사용을 통해 조건절에 대한 의도 전달이 명확해졌습니다.

또한, `in 서브쿼리` 방식에서 `join` 사용으로 변경함으로써 성능 향상을 기대할 수 있습니다.

## 성능 비교 및 마무리

결과적으로 개선이 되긴 되었지만, 드라마틱하게 실행 시간이 줄어들진 않았습니다. [이 곳](https://smwu-pochak.github.io/posts/test-query-performance/)에서 자세한 내용을 확인할 수 있습니다!

또한 Querydsl 도입 이후, CustomRepository를 테스팅하는 방법을 [이 곳](ttps://smwu-pochak.github.io/posts/testing-repository/)에서 정리하였습니다!

# [10/2 내용 추가] 리팩토링 V2

## V2 리팩토링 결과

### 개선된 쿼리

```java
private Optional<Post> findByIdWithoutBlockPost(
        final Long postId,
        final Long loginMemberId
) {
return Optional.ofNullable(
        query.selectFrom(post)
                .join(post.owner).fetchJoin()
                .join(tag).on(tag.post.eq(post).and(post.id.eq(postId)))
                .leftJoin(block).on(
                        checkOwnerOrTaggedMemberBlockLoginMember(loginMemberId)
                                .or(checkLoginMemberBlockOwnerOrTaggedMember(loginMemberId))
                )
                .groupBy(post)
                .having(block.id.count().eq(0L))
                .where(post.status.eq(BaseEntityStatus.ACTIVE))
                .fetchOne()
);
}
```

### 개선한 부분

기존에 where 절에 포함되어 있던 `post.id.eq(postId)` 조건을 초반에 tag를 inner join할 때 on절의 조건으로 이동시켰습니다.

[기존 쿼리](https://smwu-pochak.github.io/posts/block-querydsl/#%EA%B5%AC%ED%98%84-%EB%B0%A9%EC%8B%9D)의 구현방식에서 첫번째 `기준테이블인 post 테이블에 tag 테이블을 inner join` 해주는 과정에 on 조건절에 `post.id.eq(postId)` 를 추가하였습니다.

원래 `where(post.id.eq(postId))`를 통하여 Tag와의 JOIN, Block과의 LEFT JOIN이 끝난 뒤 where문을 통해 post id를 필터링하는 방식에서 미리 join 전에 필터링을 거는 방식으로 변경하였습니다.


## JOIN - ON 과 WHERE의 성능 차이

### 주된 차이점

- `ON` : join 전에 조건을 필터링
- `WHERE` : join 후에 조건을 필터링

inner join의 경우, 조건을 `on`과 `where` 어디에 위치시켜도 성능은 동일합니다. 
하지만 outer join일 경우 달라지게 됩니다.

무엇보다도 기존 테이블의 행을 줄여서 left join 전에 필요한 데이터 양을 미리 줄이는 것입니다.

### 개선 결과

이전 방식은 다음 그림과 같습니다. Post 테이블과 Tag 테이블을 JOIN 하기에 다음과 같은 결과가 산출됩니다.

![img](/assets/img/2024-09-28-block-querydsl/img1.jpeg)

따라서 산출된 결과 테이블에서 Block을 LEFT JOIN 해주기에, <br>
정말 찾으려는 id를 가진 Post 뿐만 아니라 다른 게시물 데이터까지 Block과 Left JOIN을 해주어야 한다는 한계가 있었습니다.

<br>

따라서 Tag와 inner join을 할 때 on 조건절에 `post.id.eq(postId)` 를 추가하여 찾고자하는 id를 가진 게시물만 결과값으로 나올 수 있도록 수정하였습니다.

![img](/assets/img/2024-09-28-block-querydsl/img2.jpeg)


## 성능 비교

기존, 개선한 V1, V2 쿼리 각각의 성능 측정은 [이 글](https://smwu-pochak.github.io/posts/test-query-performance/#2%EC%B0%A8-%ED%85%8C%EC%8A%A4%ED%8A%B8%EB%A5%BC-%ED%95%B4%EB%B4%85%EC%8B%9C%EB%8B%A4)에서 자세히 측정하였습니다.

근소하지만 성능 향상의 결과를 얻을 수 있었습니다.
