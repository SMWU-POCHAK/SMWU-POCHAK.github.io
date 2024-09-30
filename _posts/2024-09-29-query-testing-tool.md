---
title: 쿼리 테스팅 도구에 대해 알아보자
date: 2024-09-29 13:45:00 +0900
categories: [SpringBoot, Test]
tags: [backend, springboot, test]     # TAG names should always be lowercase
author: lucy-oh
---

# 🧐 들어가며

최근 차단 쿼리를 `in (서브쿼리)` 방식에서 `left join`을 사용한 방식으로 개선하고 있습니다. 물론 쿼리 가독성을 위해 querydsl도 도입하며 다양한 방식을 배울 수도 있었는데요! 과연 제가 개선한 쿼리가 이전 쿼리보다 얼마나 나아졌을까에 대한 궁금증이 생겼습니다.😆

# ⚒️ 테스팅 시나리오 정리

## 환경
클라우드 환경에서 테스팅도 좋겠지만 일단 비용이 굉장히 걱정되고, 순수한 "쿼리" 테스팅을 위해 로컬에서 테스트를 진행할 예정입니다!
- SpringBoot 3.2.1 (Java 17)
- H2 Database 2.3.232

## 데이터 세팅
각각 테스트 코드로 작성해서 H2 데이터베이스에 더미데이터를 넣어주었습니다
- 게시물 더미 데이터 40000개
- 멤버 더미 데이터 30000개
- 태그 더미 데이터 20000개
- 차단 더미 데이터 120000개

![img](/assets/img/2024-09-29-query-testing-tool/img0.png)
_설정 완료!_

## 테스팅할 메소드

자세한 리팩토링 과정은 [이곳](https://smwu-pochak.github.io/posts/block-querydsl/)에서 확인할 수 있습니다.

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
```

## 테스팅 방법
1. 최대한 많은 요청을 각 v1, v2 api에 전송한다.
2. 동일한 환경에서 전송한다. (로컬 환경에서 테스팅할 예정!)
3. 전송 후 응답 시간 등 결과를 확인한다.

# 🛠️ 테스트
## Apache Bench

Apache AB (Apache Benchmark)는 Apache HTTP Server의 성능을 테스트하고 측정하기 위한 명령줄 도구입니다.

저는 맥북 프로를 사용하고 있기 때문에, 따로 세팅은 필요하지 않았고, 터미널을 통해 명령어를 보내서 바로 테스팅할 수 있었습니다!

### 세팅

아래의 명령어를 사용하여 서버에 "게시물 조회" 요청을 보내보겠습니다!

```
ab -n 12000 -c 100 -H 'Authorization: Bearer ey~~' http://localhost:8080/api/v2/posts/{postId}
```

- `-n` : 요청의 개수
- `-c` : 동시 접속자
- `-H` : 헤더 추가 -H 'Authorization: Bearer ey~~' 형태로 전송!

저는 요청은 12000개, 동시 접속자는 100명으로 설정을 해주었고, 서버 자체의 인증때문에 access token을 뽑아서 함께 전송해주었습니다.

> 사실 요청의 개수나 접속자는 각 쿼리마다 동일하게 세팅을 해주는 게 중요하지, 숫자 자체가 크게 유의미한 것 같지는 않아서 적당히 큰 숫자로 설정을 해주었습니다!
{: .prompt-info }

### 이전 쿼리
먼저, 이전에 작성한 쿼리의 결과를 확인해보겠습니다.

![img](/assets/img/2024-09-29-query-testing-tool/img1.png)
_이전 쿼리 결과_

- `1초당 처리한 응답의 개수` Requests per second: 1068.08 [#/sec] (mean)
- `요청 1건당 처리된 시간` Time per request: 93.626 [ms] (mean)

다만 여러번 보낼때마다 처리 시간이 변동되기에 5번 정보 보내보고 평균적인 응답 개수 및 처리 시간은 다음과 같습니다.

- `1초당 처리한 응답의 개수` Requests per second: 1185.2 [#/sec] (mean)
- `요청 1건당 처리된 시간` Time per request: 89.2736 [ms] (mean)


### 현재 쿼리
그리고 개선된 쿼리의 결과를 확인해보겠습니다.

![img](/assets/img/2024-09-29-query-testing-tool/img2.png)
_개선된 쿼리 결과_

- `1초당 처리한 응답의 개수` Requests per second: 1118.62 [#/sec] (mean)
- `요청 1건당 처리된 시간` Time per request: 89.396 [ms] (mean)

이 역시도 5번 정도 보낸 뒤 계산된 평균적인 응답 개수 및 처리 시간은 다음과 같습니다.

- `1초당 처리한 응답의 개수` Requests per second: 1047.412 [#/sec] (mean)
- `요청 1건당 처리된 시간` Time per request: 76.906 [ms] (mean)

## JMeter

Apache JMeter는 서버가 제공하는 성능 및 부하를 측정할 수 있는 테스트 도구입니다.

따로 다운을 받아야 해서 Apache보다는 조금 귀찮았지만, 세팅과정의 UI가 직관적이고, 제공하는 리포트 등을 통해 결과 역시도 상세히 확인할 수 있었기에 굉장히 편했습니다!

### JMeter 세팅

먼저 스레드 그룹을 설정해주었습니다.

![img](/assets/img/2024-09-29-query-testing-tool/img3.png)

Number of Threads는 유저 수, Loop Count는 각 유저가 보낼 요청 수를 의미합니다.

- Number of Threads : 100
- Ramp-up period : 10
- Loop Count : 100

저는 100명의 유저가 100번의 요청을 10초 동안 보내도록 세팅을 해주었습니다.

<br>

HTTP Request 정보를 다음과 같이 세팅해주었습니다.
![img](/assets/img/2024-09-29-query-testing-tool/img4.png)

- Server Name or IP : localhost
- Port Number : 8080
- Http Request : [GET] /api/v2/posts/12

<br>

추가적인 헤더 정보는 Header Manager를 추가해주었습니다.
![img](/assets/img/2024-09-29-query-testing-tool/img5.png)


### 기존 쿼리

기존 쿼리로 세팅해준 뒤 생성된 Report를 확인해주면 다음과 같습니다.
![img](/assets/img/2024-09-29-query-testing-tool/img6.png)

- Average : 4
- Median : 4
- 90% Line : 5
- 95% Line : 6
- 99% Line : 9

### 개선한 쿼리

기존 쿼리로 세팅해준 뒤 생성된 Report를 확인해주면 다음과 같습니다.
![img](/assets/img/2024-09-29-query-testing-tool/img7.png)

- Average : 4
- Median : 4
- 90% Line : 5
- 95% Line : 5
- 99% Line : 7

개선된 쿼리와 기존 쿼리가 평균 시간이나 중간값에서는 유사함을 보여주었지만, 기존 쿼리에서 95%, 99% Line 값이 높게 나온 것을 확인할 수 있었습니다.

# 결론
## 드라마틱한 개선은 없었다!

기존에 통상적으로 알려져있는 `in (서브쿼리)` 보다 `join` 사용이 더 유리하다고 알고 있었는데, 생각보다 극적으로 쿼리가 개선되진 않았습니다. 또한, 툴 각각의 소요 시간 역시도 천차만별이었습니다...!

그 이유 중 하나가 테스트 데이터 생성의 한계라고 생각하는데, 무작위가 아닌 반복문으로 생성된 데이터이다보니 한정적인 케이스로 쌓인 데이터에 대해서만 테스트가 가능했다는 점이 이번 테스팅의 한계점인 것 같습니다.😓 데이터를 "쌓는 것" 자체에만 너무 집중한 것 같네요.

> post에 tag는 2개씩, member는 3명씩 차단하고 있는 상태였습니다. 각 member가 3명 이상의 더 많은 유저들을 차단하고 있었다면, 혹은 아예 차단하고 있지 않았다면 등등의 모든 케이스를 아우르는 무작위 데이터였다면 또 다른 결과를 확인했을 수도 있을 것 같습니다.
{: .prompt-warning }

하지만 근소하더라도 결과적으로 향상된 결과를 얻을 수 있었고, 앞으로 서버 테스팅에서 사용하면서 더 효율적이고 유의미한 작업을 해낼 수 있도록 사용할 수 있을 것 같아 만족합니다☺️