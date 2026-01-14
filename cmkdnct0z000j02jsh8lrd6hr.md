---
title: "Postgresql MVCC"
seoTitle: "Understanding PostgreSQL's MVCC System"
seoDescription: "Explore PostgreSQL's MVCC, a concurrency control method allowing non-blocking reads and writes using xmin, xmax, and tuple visibility"
datePublished: Wed Jan 14 2026 06:35:45 GMT+0000 (Coordinated Universal Time)
cuid: cmkdnct0z000j02jsh8lrd6hr
slug: postgresql-mvcc
tags: postgresql

---

postgresql 의 MVCC 를 살펴보면 아주 재미있다. **MVCC(Multi-Version Concurrency Control)** 은 동시성을 처리하는 핵심아이디어로 기본적인 전제로 **“읽기는 쓰기를 블록해선 안되고, 쓰기도 읽기를 블록하지 않는다”** 라는 아이디어 에서 시작한다.

이 개념을 적용하기 위해서는 Postgresql 에서는 `xmin` 과 `xmax` 를 활용한다. Postgresql 에서 테이블안에 있는 데이터는 **튜플(Tuple)** 형태로 저장된다. 예를 들면, 아래와 같이 테이블이 있다고 해보자.

## 테이블 구조

| id | name | balance | created\_at |
| --- | --- | --- | --- |
| 1 | Alice | 1000 | 2026-01-14 01:14:08.515896 |
| 2 | Bob | 2000 | 2026-01-14 01:14:08.515896 |
| 3 | Charlie | 3000 | 2026-01-14 01:14:08.515896 |

위 테이블에는 `id`,`name`,`balance`,`created_at` 등의 column 이 존재한다. 여기서 튜플은 `(1,Alice,1000,2026-01-14)`를 의미한다. 즉, 하나의 레코드가 튜플로 저장된다.

## 튜플

튜플은 아래와 같은 정보를 가지고 있다.

* **ctid**: 페이지에서 저장된 위치를 나타내는 값
    
* **xmin**: 이 튜플을 INSERT 한 트랜잭션의 ID
    
* **xmax**: 이 튜플을 DELETE 한 트랜잭션의 ID
    

각 처리된 트랜잭션의 ID 정보를 가지고 있는 이유는 해당 Tuple 의 **가시성(Visibility)** 를 계산하기 위함이다. 쿼리를 날려보고 결과를 확인해보며 이해해 보자.

```pgsql
--- 세션 A 시작
BEGIN;

SELECT txid_current(); -- 810

SELECT xmin, xmax, ctid, id, name, balance
FROM accounts
ORDER BY id;
```

세션 A 는 `ximn` **txid(트랜잭션 ID)** 를 810 로 가지고 있고, `xmax` 는 0 으로 가지고 있다. `xmax` 는 삭제될때만 txid 를 남기므로 0 이라는 것은 아직 지워지지 않았음을 의미한다.

| xmin | xmax | ctid | id | name | balance |
| --- | --- | --- | --- | --- | --- |
| 734 | 0 | (0,1) | 1 | Alice | 1000 |
| 734 | 0 | (0,2) | 2 | Bob | 2000 |
| 734 | 0 | (0,3) | 3 | Charlie | 3000 |

즉, 이 세개의 데이터는 현재 **Transaction** 전에 생긴 데이터임을 알 수 있다. 이 트랜잭션을 닫지 않고, 다른 **트랜잭션(세션 B)** 를 열어보자.

```pgsql
BEGIN;

SELECT pg_current_snapshot();
SELECT txid_current(); -- 811

INSERT INTO accounts (name, balance)
VALUES ('New User (uncommitted)', 9999)
RETURNING xmin, xmax, ctid, id, name, balance;
```

여기서는 `811` 로 나온다. 여기서 INSERT 한게 세션 A 에 보일까? xmin 의 문제는 아니지만 세션 A 에는 보이지 않는다. 그 이유는 기본 격리수준인 `READ COMMITED` 를 이용하기 때문이다. B 세션을 커밋해보자. 그리고 A 세션은 아직 커밋하지 않았지만 다시 SELECT 를 해보자.

| xmin | xmax | ctid | id | name | balance |
| --- | --- | --- | --- | --- | --- |
| 734 | 0 | (0,1) | 1 | Alice | 1000 |
| 734 | 0 | (0,2) | 2 | Bob | 2000 |
| 734 | 0 | (0,3) | 3 | Charlie | 3000 |
| 811 | 0 | (0,7) | 11 | New User (uncommitted) | 9999 |

세션 B 에서 저장된 값의 `xmin` 을 보니 세션 B 의 트랜잭션 ID 가 남아있는 것을 확인할 수 있다. 이제 세션 C 를 열고 삭제해보자.

```pgsql
BEGIN;

SELECT pg_current_snapshot();
SELECT txid_current(); -- 813

DELETE FROM accounts where name = 'New User (uncommitted)';
```

**세션 C(813)** 에서 해당 튜플을 삭제했다. 이 튜플은 xmax 값이 아마 813 으로 남아있을 것이다. Postgresql 은 이처럼 트랜잭션이 열린동안에도 읽기와 쓰기에 대한 Lock 을 하지 최소화 하거나 하지 않기 위해 Tuple 을 계속해서 생성해내고, xmin, xmax 값을 바꾼다. 이제 후에 Vacuum 으로 정리될 죽은 튜플에서 xmax 값을 확인해보자.

| item | xmin | xmax | ctid | status | xmin\_committed |
| --- | --- | --- | --- | --- | --- |
| 1 | 734 | 0 | (0,1) | 🟢 LIVE | COMMITTED |
| 2 | 734 | 0 | (0,2) | 🟢 LIVE | COMMITTED |
| 3 | 734 | 0 | (0,3) | 🟢 LIVE | COMMITTED |
| 7 | 812 | 813 | (0,7) | 💀 DEAD (or being deleted) | COMMITTED |

상태가 죽음(DEAD) 로 표시한걸 확인할 수 있다. 이는 다음에 **AUTO VACUUM** 이 돌때 제거된다. 수동으로 호출도 가능하다.

```pgsql
VACUUM accounts;
```

이제 대략적으로 **쓰기/삭제(업데이트는 삭제와 쓰기가 일어남) 일어날 때 마다 튜플이 생성**되는 걸 확인할 수 있었다. 그리고 이를 후에 주기적으로 정리하는 VACUUM 이라는 것도 있다는 것을 알게 되었다. 이제 스냅샷에 대해 알아보자. 스냅샷은 직관적으로 내 트랜잭션에 무엇이 보여야 하는지를 관리해준다.

## 스냅샷

스냅샷이 중요한 개념인데 트랜잭션 격리 레벨에 따라 다르다. 기본 격리 수준인 `READ COMMITED` 에서는 각 쿼리가 실행될때마다 스냅샷이 기록됩니다. 예시와 함께 보시죠

```pgsql
BEGIN;

SELECT pg_current_snapshot(); --- 815:815

SELECT xmin, xmax, ctid, id, name, balance
FROM accounts
ORDER BY id;

SELECT pg_current_snapshot(); --- ???:???
```

위와 같은 쿼리가 있을때 두번째로 `snapshot()` 을 찍으면 어떻게 될까요? 만약 아무런 변경이 없어 `xmin` 값이 올라가지 않았다면 동일하게 `815:815` 가 나왔을 것입니다. 하지만 만약, 다른 세션에서 새로운 컬럼을 아래와 같이 추가한다면 어떻게될까요?

```pgsql
BEGIN;

SELECT pg_current_snapshot();
SELECT txid_current(); --- 816

INSERT INTO accounts (name, balance)
VALUES ('New User (uncommitted)', 9999)
RETURNING xmin, xmax, ctid, id, name, balance;

COMMIT;
```

위와 같이 추가하게 되면 이제 `xmin` 의 경계가 816까지 올라가게 됩니다. 이 상태에서 닫지않은 815 세션에서 동일하게 쿼리를 수행하면 어떻게 될까요?

```pgsql
BEGIN;

SELECT pg_current_snapshot(); --- 815:815

SELECT xmin, xmax, ctid, id, name, balance
FROM accounts
ORDER BY id;

SELECT pg_current_snapshot(); --- 816:816
```

`816`으로 나오게 됩니다. 그리고 저 SELECT 에서는 새롭게 추가한 `New User (uncommitted)` 가 보이게 됩니다. 당연한 **READ COMMITED** 의 동작이지만 여기에는 스냅샷 기반으로 가시성을 통제하는 뒷단의 마법같은 로직이 숨겨져있습니다.

## 스냅샷

snapshot 은 기본적으로 `xmin:xmax:[진행중인 txid]` 로 구성됩니다. 각 값은 아래와 같은 정의를 가집니다.

* **Snapshot xmin:** 아직 완료되지 않은(Active) 트랜잭션 중 가장 낮은 ID. (이보다 작은 ID는 모두 커밋됨이 보장됨)
    
* **Snapshot xmax:** 현재까지 할당된 TXID 중 가장 큰 값 + 1. (이보다 크거나 같은 ID는 스냅샷 생성 시점에 아직 시작도 안 한 "미래" 트랜잭션)
    
* **xip\_list (진행 중인 txid):** `xmin`과 `xmax` 사이에서 아직 진행 중인 트랜잭션들의 목록
    

그래서 특정한 `xmin` 을 높이는 작업이나 `xmax` 를 높이는 작업이 끝나면, 각 `xmin`과 `xmax` 의 값이 올라갔던 것입니다. 위에서는 아직 진행중인 transaction 을 제가 로깅하진 않았지만 아마 816 에서 세션을 열고, 815 에서 한번더 확인했다면 815 에서 816이 진행중인 세션으로 보였을 것입니다. 이 스냅샷의 범위와 튜플을 이용해 가시성을 통제합니다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1768371762951/8f7168f1-6139-4b58-87fe-99f3d851b29e.png align="center")

이해를 하기 위해 그림과 함께 보면 현재 snapshot 이 **807:814:\[807\]** 이라고 가정해봅시다. 각 튜플마다 지금 보여야 하는지 안보여야 하는지를 한번 설명해보도록 하겠습니다.

* A 튜플은 이미 734 에서 처리되었으므로 우리 세션에 보여야 합니다.
    
* B 튜플은 806 에서 커밋되었으나 xmax 가 커밋된 결과에 있으므로 삭제되었으므로 보이지 않습니다.
    
* C 튜플은 현재 트랜잭션이 커밋되지 않은 상태로 진행중이므로 현재에서 보이지 않습니다.
    
* D 튜플은 810 에서 커밋되었으니 **snapshot\_xmax(814) 보다 tuple\_xmin(810) 이 작으므로 현재 트랜잭션에 보입니다**.
    
* E 튜플 또한 삭제 되었으니 보이지 않습니다.
    
* F 튜플은 커밋되었으나 **snapshot\_xmax(814) 보다 tuple\_xmin(820) 이 더 크므로 미래에 일어난 일이므로 보이지 않습니다**.
    

즉 이런식으로 현재 snapshot 이 가지고 있는 범위에 따라 보이는 튜플들을 통제합니다. 만약 기본 격리수준인 `READ COMMITED` 의경우 쿼리를 실행한번 더 하게되면 xmax 가 820 까지 늘어나게 되면서 F 튜플이 보였을 수 있습니다.

### REPEATABLE READ 는?

그렇다면 REPEATBALE READ 는 어떨까요? 본질적으로 REPEATABLE READ 는 PHANTOM READ 를 방지하므로 820 은 제 트랜잭션에서 노출되서는 안됩니다. Postgresql 은 여기서 쉽게 REPETABLE READ 는 트랜잭션이 시작되고 첫번째 쿼리가 만든 스냅샷만 해당 트랜잭션에서 이용하게 합니다.

```python
BEGIN;

SELECT * FROM TABLE; --- 815:819

SELECT * FROM TABLE; --- 815:819

COMMIT;
```

즉, 같은 트랜잭션에서 스냅샷이 바뀌지 않으므로 여러번 SELECT 해도 결과가 바뀌지 않습니다. 다만, MySQL 과 다르게 동시에 해당 튜플에 값을 쓸때 처리하는 방식이 다르므로 Postgresql 에서는 **REPEATABLE READ** 를 쓸때 조심하셔야 됩니다.

## 마치며

Postgresql 에 Tuple 과 스냅샷에 대해 알아보았는데요. 다음시간에는 격리수준에 대한 조금 더 자세한 내용을 알아보려고 합니다