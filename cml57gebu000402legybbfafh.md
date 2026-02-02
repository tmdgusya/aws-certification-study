---
title: "Redis ZSET"
seoTitle: "Redis Sorted Sets Explained"
seoDescription: "Explore Redis ZSET, a unique, ordered string data structure ideal for leaderboards and rate limiters, with various operations and use cases"
datePublished: Mon Feb 02 2026 13:28:12 GMT+0000 (Coordinated Universal Time)
cuid: cml57gebu000402legybbfafh
slug: redis-zset
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1770037043882/ce506fdc-f3ff-4d89-937e-ae3a4a4be2c3.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1770038878676/01b42704-2783-44bf-ab41-35a11756ee35.png

---

## ZSET 이란?

ZSET 은 Redis 에서 unique 한 string 들이 score 순서대로(in order) 정렬되어 있는 자료구조이다. 그래서 **Leader board** 나 **Rate limiter** 에 쓰일수 있습니다. 기본적으로 자료구조가 Hash Table + Skip List(스킵 리스트) 두가지 자료구조를 합쳐서 사용하기 때문에 접근에는 **O(1)**, 추가에는 **O(log N)** 이 소요됩니다.

## 추가(ZADD)

```bash
ZADD KEY [NX | XX] [GT | LT] [CH] [INCR] score member
```

```bash
localhost:6379> ZADD roach_set 1 "roach"
(integer) 1
localhost:6379> ZADD roach_set 2 "roach2"
(integer) 1
localhost:6379> ZADD roach_set 3 "roach3"
(integer) 1
localhost:6379> ZADD roach_set 4 "roach4" 5 "roach5"
(integer) 2
```

위와 같이 명령어를 입력하여 추가할 수 있습니다. `[` 친 부분은 Optional 하다고 생각해주시면 됩니다. 실제로 ZSET 에 추가해봅시다.

위와 같이 추가가 잘 되었고 제가 몇개를 넣었는지 리턴해주는 것을 확인할 수 있습니다. 삽입간 정렬이 일어나므로 공식문서에 적힌대로 **O(log N)** 시간이 소요되는 것을 알 수 있습니다.

## 범위 검색(ZRANGE)

```bash
ZRANGE key start stop [BYSCORE | BYLEX] [REV] [LIMIT offset count]
```

```bash
localhost:6379> ZRANGE roach_set 1 3
1) "roach2"
2) "roach3"
3) "roach4"
localhost:6379> ZRANGE roach_set 1 4
1) "roach2"
2) "roach3"
3) "roach4"
4) "roach5"
```

`ZRANGE` 는 score 가 아닌 인덱스 기반으로 조회가 가능한 메소드 입니다. `O(log(N)+M)` 의 시간복잡도를 가지고 있으며 N 은 sorted set 안의 멤버들의 개수이고, M 은 리턴되는 멤버들의 개수입니다.

첫번째 질의로는 `첫번째 인덱스 ~ 세번째 인덱스` 까지를 가져오도록 질의했고, 두번째 인덱스로는 `첫번째 인덱스 ~ 네번째 인덱스` 를 가져오도록 질의하였습니다. 참고로 마지막 인덱스를 `-1` 로 하면 lastIndex 와 동일한 의미를 지닙니다. 시간복잡도가 꽤 커질수 있으므로 redis 에서도 주의해서 사용하라는 `@slow` 어노테이션이 붙어있습니다.

## 스코어 기반 범위 검색(ZRANGEBYSCORE)

```bash
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]
```

```bash
localhost:6379> ZRANGEBYSCORE roach_set 1 3
1) "roach"
2) "roach2"
3) "roach3"
localhost:6379> ZRANGEBYSCORE roach_set 1 4
1) "roach"
2) "roach2"
3) "roach3"
4) "roach4"
```

`ZRANGEBYSCORE` 는 점수기반 범위로 검색하고 시간복잡도는 ZRANGE 와 동일하게 `O(log(N)+M)` 의 복잡도를 지닙니다.

특별하게 설명할 부분은 없고 점수 기반은 점수 사이에 얼마가 있을지 모르므로 꼭 LIMIT 과 OFFSET 을 잘 활용하여 검색해야 한다는 점만 알아두면 좋을거 같습니다.

## 삭제(ZREM)

```bash
ZREM key member [member ...]
```

```bash
localhost:6379> ZREM roach_set "roach"
(integer) 1
localhost:6379> ZRANGEBYSCORE roach_set 1 4
1) "roach2"
2) "roach3"
3) "roach4"
```

SET 에서 KEY 와 MEMBER 기반으로 삭제하는 메소드 입니다. 삭제하는 것도 정렬의 오버헤드가 드므로 시간 복잡도는 `O(log(N)+M)` 이 소요됩니다.

“roach” 라는 key 를 이용해서 해당 Set 에 제거하는 방식입니다.

## 랭크(ZRANK)

```bash
ZRANK key member [WITHSCORE]
```

```bash
localhost:6379> ZRANK roach_set "roach1"
(nil)
localhost:6379> ZRANK roach_set "roach2"
(integer) 0
localhost:6379> ZRANK roach_set "roach2" WITHSCORE
1) (integer) 0
2) "2"
localhost:6379> ZRANK roach_set "roach4" WITHSCORE
1) (integer) 2
2) "4"
```

ZRANK 는 key 와 member 기반으로 RANK 를 알려주는 메소드입니다. 기본적으로 **zero-based(0 부터 시작)** 이며 `WITHSCORE` 와 함께 조회할 시에는 스코어까지 함께 리턴받을 수 있습니다. 시간복잡도는 `O(log N)` 시간복잡도 안에 수행히 가능합니다.

## Rate Limiter 구현

유저가 **5초 동안 요청할 수 있는 허용된 요청의 수는 5개** 입니다. 요걸 어떻게 ZSET 으로 구현해볼 수 있을까요? 가볍게 생각해보면 1초를(1000ms) 로 잡고 계산하여 이를 score 화 하는 방법이 있습니다. ZSET 자체는 정렬된 자료구조이므로 RANGE 를 이용하여 쉽게 범위 검사가 가능합니다.

```bash
localhost:6379> ZADD user:1 1000 req_1
(integer) 0
localhost:6379> ZADD user:1 1100 req_2
(integer) 1
localhost:6379> ZADD user:1 1300 req_3
(integer) 1
localhost:6379> ZADD user:1 2000 req_4
(integer) 1
localhost:6379> ZADD user:1 4000 req_5
(integer) 1
localhost:6379> ZADD user:1 5000 req_6
```

현재 1초에서 5초사이에 `user:1` 이 총 6건의 요청을 보낸 것을 확인할 수 있습니다. 이를 확인하기 위해서는 `ZCARD` 메소드를 이용하면 됩니다. ZCARD 는 현재 SET 의 Cardinality 를 리턴해주므로 중복이 아닌 멤버의 갯수를 리턴해주게 됩니다.

```bash
localhost:6379> ZCARD user:1
(integer) 6
```

즉 `ZCARD user:1` 을 하게 되면 user:1 에 얼마나 많은 `member` 가 있는지 확인할 수 있습니다. 5초 동안 허용된 요청수 5를 넘었으므로 user:1 은 더이상 요청을 보내지 못하게 됩니다. 근데 만약 시간이 더 흘러서 `5-10초` 구간까지 갔다고 해봅시다.

```bash
localhost:6379> ZADD user:1 1000 req_1
(integer) 1
localhost:6379> ZADD user:1 1100 req_2
(integer) 1
localhost:6379> ZADD user:1 1300 req_3
(integer) 1
localhost:6379> ZADD user:1 2000 req_4
(integer) 1
localhost:6379> ZADD user:1 4000 req_5
(integer) 1
localhost:6379> ZADD user:1 5000 req_6
(integer) 1
localhost:6379> ZADD user:1 6000 req_7
(integer) 1
localhost:6379> ZADD user:1 10000 req_8
(integer) 1
```

사실 만약 지금이 10초 부근이라 했을때 5초 이전의 `req_1 ~ req_5` 요청들은 해당 구간의 sliding window 에 없어야 합니다. 그 경우에는 `ZREMRANGEBYSCORE` 로 0~5 초 구간의 요청데이터를 지워주면 됩니다.

```bash
localhost:6379> ZREMRANGEBYSCORE user:1 0 5000
(integer) 6
```

**ZREMRANGEBYSCORE** 는 KEY 기반으로 **점수가 min ~ max 구간에 있는 member 들을 제거**해줍니다. Return 값을 보면 `req_1 ~ req_6` 까지 총 6개가 잘 지워진것을 확인할 수 있습니다.

```bash
localhost:6379> ZCARD user:1
(integer) 2
```

이제 ZCARD 를 해보면 총 2 개로 `req_7` 과 `req_8` 만 남은것을 확인할 수 있습니다.

## 마치며

이 글은 복리 효과를 위해 매일 30분정도를 투자하여 작성되는 글입니다 :). 가볍게 읽어주시고 더 정확한 자료는 공식문서를 참고 바랍니다.

---

### References

* **Redis sorted set**: [https://redis.io/docs/latest/develop/data-types/sorted-sets/](https://redis.io/docs/latest/develop/data-types/sorted-sets/)