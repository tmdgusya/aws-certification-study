---
title: "Postgresql 커버링 인덱스"
seoTitle: "PostgreSQL Covering Index Explained"
seoDescription: "Learn PostgreSQL: indexes, importance, implementation, optimization, performance impact, practical examples, and database efficiency considerations"
datePublished: Sun Jan 18 2026 06:03:14 GMT+0000 (Coordinated Universal Time)
cuid: cmkjbydui000a02l26styahjl
slug: postgresql
tags: postgresql

---

## Index?

Index 란 무엇이고 왜 중요할까? Index 는 데이터베이스에서 File System 에 일어나는 **Random I/O** 를 줄이기 위해 존재한다. 랜덤 I/O 란, 해당 데이터가 저장된 Array 에 우리가 특정 주소값 `i` 를 넣어 조회하듯 일어나는 이벤트를 뜻한다.

그럼 느리지 않을거 같은데 왜 랜덤 I/O 를 줄여야 하지? 라는 고민이 충분히 들수도 있다. 이러한 이유는 File System 까지 가는 계층에서 드는 비용과 전통적인 HDD 의 경우 디스크헤드 비용, Kernel 에서 블락단위로 읽어오는 비용 등등 여러가지 부가적인 요소들이 들어가게 된다. 이는 B-Tree 같은 것들이 왜 `leaf node`에서 range 로 한번의 I/O 로 많은 것들을 읽을 수 있도록 설계해놨는지를 알 수 있게 해준다.

## B-Tree

```bash

                [Internal Node]
                /      |      \
               /       |       \
    [Internal Node] [Internal] [Internal]
        /    \         |  \        /  \
       /      \        |   \      /    \
   [Leaf]  [Leaf]  [Leaf] [Leaf] [Leaf] [Leaf]
```

B-Tree 의 자료구조이다. 정렬된 형태로 존재하며 빠르게 원하는 key 값으로 데이터를 찾을 수 있게 `Internal Node` 들은 키값만을 저장한다. `Leaf Node` 는 실제로 찾아야 하는 데이터 혹은 그 데이터를 가르키는 어떠한 값을 가르키도록 보통 설계되어 있다. (MySQL InnoDB 엔진의 경우 이 값이 클러스터링 인덱스또는 실제 값을 가르키게끔 되어 있거나, Postgresql 의 경우 TID 값을 통해 Heap 을 가르키도록 되어 있다)

```sql
CREATE TABLE logs (
    id BIGSERIAL PRIMARY KEY,
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    level VARCHAR(10) NOT NULL,
    service VARCHAR(50) NOT NULL,
    message TEXT NOT NULL,
    metadata JSONB
); 

CREATE UNIQUE INDEX logs_pkey ON public.logs USING btree (id)
CREATE INDEX idx_test ON public.logs USING btree (level, service)
```

만약 위와 같은 테이블이 있다고 해보자. 해당 테이블이 있을때 아래와 같은 쿼리를 날리면 어떻게 될까?

```sql
SELECT level, service, message FROM logs WHERE level='ERROR'
```

인덱스가 있어 빠를것 같지만 아래와 같은 실행 계획을 가지게 되며 실행시간도 `11021.287 ms` 로 꽤나 긴것을 확인할 수 있습니다.

| QUERY PLAN |
| --- |
| Bitmap Heap Scan on logs (cost=68034.25..822774.99 rows=7832699 width=37) (actual time=430.442..10830.196 rows=7774197 loops=1) |
| Recheck Cond: ((level)::text = 'ERROR'::text) |
| Heap Blocks: exact=656830 |
| Buffers: shared hit=6599 read=656800 dirtied=56788 written=18 |
| \-&gt; Bitmap Index Scan on idx\_test (cost=0.00..66076.08 rows=7832699 width=0) (actual time=325.108..325.108 rows=7774197 loops=1) |
| Index Cond: ((level)::text = 'ERROR'::text) |
| Buffers: shared hit=6569 |
| Planning Time: 0.054 ms |
| JIT: |
| Functions: 4 |
| Options: Inlining true, Optimization true, Expressions true, Deforming true |
| Timing: Generation 0.168 ms, Inlining 3.306 ms, Optimization 5.606 ms, Emission 3.804 ms, Total 12.884 ms |
| Execution Time: 11021.287 ms |

이 실행계획을 간단히 설명해보면 첫번째로 `Bitmap Index Scan` 을 통해 해당 행들이 페이지내의 어느 블록에 있는지를 알아냅니다. 이렇게 하는 이유는 그냥 Index Scan 을 통해서 Random I/O 를 발생시키면 너무 오래걸리기 때문에 Bitmap 에서 정렬시켜 순차적으로 한번에 많이 긁어오기 위함입니다.

그리고 해당 Bitmap 을 이용하여 Heap Scan 을 실시합니다. 이때 정렬된 순서로 긁기 때문에 그냥 Random I/O 보다는 더 효율적으로 데이터를 가져오게 됩니다.

```sql
Buffers: shared hit=6599 read=656800 dirtied=56788 written=18
```

그래서 결과를 보면 실제로 Memory 에서 가져온것은 6599 건, 그리고 파일시스템을 통해서 656800 건 만큼 블록단위로 데이터를 읽어 가져오게 됩니다. 즉, 대부분을 파일 시스템을 통해 가져오는 것을 확인할 수 있습니다. 여기서 최적화를 해야 한다면 어떻게 해야할까요?

## 커버링 인덱스

여기서 커버링 Index 를 이용해 볼 수 있습니다. 커버링 Index 는 세컨더리 인덱스에도 값을 저장하게끔 하여 실제 Heap page 까지는 도달되는 I/O 를 줄이는 방법입니다. 인덱스에서 대부분의 질의가 해결되기 때문에 커버링 인덱스라고 불립니다.

만드는 가장 간단한 방법은 복합 인덱스를 아래와 같이 만들어보는 것입니다.

```sql
CREATE INDEX idx_composite ON logs(level, service, message);

VACUUM ANALYZE logs;
```

위와 같이 인덱스를 생성하고 쿼리를 날리게 되면 아래와 같은 결과를 얻을 수 있습니다.

| QUERY PLAN |
| --- |
| Index Only Scan using idx\_composite on logs (cost=0.56..143045.73 rows=7738598 width=37) (actual time=0.029..310.204 rows=7774197 loops=1) |
| Index Cond: (level = 'ERROR'::text) |
| Heap Fetches: 0 |
| Buffers: shared hit=1008 read=6889 written=12 |
| Planning: |
| Buffers: shared hit=35 |
| Planning Time: 0.154 ms |
| JIT: |
| Functions: 1 |
| Options: Inlining false, Optimization false, Expressions true, Deforming true |
| Timing: Generation 0.086 ms, Inlining 0.000 ms, Optimization 0.000 ms, Emission 0.000 ms, Total 0.086 ms |
| Execution Time: 475.544 ms |

일단 실행계획을 분석해보면 `Heap Fetches` 가 0 인것을 확인할 수 있습니다. **즉, 인덱스에서 모든 조회가 이루어졌음을 확인할 수 있습니다.** read 또한 `6889` 로 급격히 감소했음을 확인할 수 있습니다. 다만, 이 부분은 실제 데이터가 최신인지 확인할 수 없을때는 증가할 수 있습니다. (실험을 위해 VACUUM ANALYZE 를 미리 실행시킨 이유가 그 이유입니다)

즉, Index Scan 만을 통해 데이터를 모두 가져오고 Heap 을 하나도 Fetch 하지 않았음을 확인할 수 있습니다. 이전에 비해 비약적으로 빨라진 것을 확인할 수 있습니다. 이것이 인덱스에서 데이터를 Fetch 해오는 것이 커버가 되는 커버링 인덱스라고 할 수 있습니다.

다만, 여기서 의문은 방금의 인덱스는 과연 좋은 인덱스 였을까요? 보통 Tree 자료구조에서 key 로 무언갈 선정한다는건 이 데이터를 정렬의 축으로 잡겠다는 의미와 같습니다. 즉, level 과 service 는 ENUM 과 같은 제약된 다양성을 가지는 곳에는 괜찮을 수 있지만, message 와 같이 정렬이 필요없는 부분또한 key 에 속하게 되어 종단 노드의 크기가 커지게 됩니다.

```sql

      [Internal Node (level, service, message)]
                /      |      \
               /       |       \
    [Internal Node] [Internal] [Internal] => level, service, message
        /    \         |  \        /  \
       /      \        |   \      /    \
   [Leaf]  [Leaf]  [Leaf] [Leaf] [Leaf] [Leaf]
```

사실 message 를 빠르게 가져오기 위해 key 에 넣게 된다면 그 만큼의 INSERT 와 UPDATE 시에 오버헤드가 걸리게 되므로 trade-off 를 계산해야 되는 상황에 빠지게 됩니다. 그렇다면 message 를 탐색 key 로 잡지 않고 종단 노드에만 위치하게 하는 방법은 없을까요? Postgresql 에서는 이를 `INCLUDE` 라는 키워드를 통해 해결하게 해줍니다.

## Include

```sql
CREATE INDEX idx_include ON logs(level, service) INCLUDE (message);
```

위와 같이 INCLUDE 를 통해 `message` 를 추가하게 되면 종단 노드에만 잡혀서 아주 컴팩트한 인덱스가 되고, INSERT 와 UPDATE 시 비용이 덜 들게 된다고 생각할 수 있습니다. 실제 테스트를 위해 한번 추가하고 난 뒤에 Index 사이즈를 보도록 합시다.

```sql
SELECT
    indexrelname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE relname = 'logs';
```

| indexrelname | index\_size |
| --- | --- |
| logs\_pkey | 666 MB |
| idx\_test | 207 MB |
| idx\_composite | 217 MB |
| idx\_include | 1822 MB |

실제로 사이즈를 보니 예상 한 것보다 너무 비대한 것을 확인할 수 있습니다 . 이유는 무엇일까요? 이를 알아보기 위해서는 일단 데이터를 확인해보도록 하겠습니다.

```sql
SELECT COUNT(*) / COUNT(DISTINCT (level, service, message)) as duplication_ratio
  FROM logs;
```

로그 테이블에서 (level, service, message) 가 유니크한 수를 전체 개수로 나누어 얼마나 유니크한지를 보도록 하겠습니다. 1.0 에 근사한 수치가 나올수록 카디널리티가 높아 인덱싱에 유리한 구조임을 알수 있겠죠.

| duplication\_ratio |
| --- |
| 3048 |

실제로 해보면 약 3048 이 나오게 됩니다. 즉, 동일한 조합이 평균적으로 약 3048 개가 중복되어 있음을 알 수 있습니다. 그런데 왜 `inx_composite` 은 217MB 뿐인데, `idx_include` 는 1822MB 나 될까요? 이는 Postgresql 이 인덱스를 생성하는 방식이 대략적으로 아래와 같기 때문입니다. (**이는 설명하기 위한 수도코드로 살짝 동작이 다릅니다!!**)

```c
btree_insert(index, key=(level, service, message), tid) {
    existing_entry = search_btree(key);

    if (existing_entry != NULL) {
        add_tid_to_posting_list(existing_entry, tid);
    } else {
        create_new_entry_with_posting_list(key, tid);
    }
}

btree_insert_with_include(index, key=(level, service), tid, include=message) {
    create_new_index_tuple(key=(level, service), tid, include_data=message);
}
```

의사 코드를 보면 복합 인덱스는 중복 제거를 하지만, `include` 의 경우 중복제거를 하지 않습니다. 즉, 복합 인덱스에서는 (level, service, message) 가 합쳐저 하나의 posting\_list 로 관리되지만, include 에서는 (level, service) 가 같더라도 message 가 다르면 별도의 엔트리가 생성되게 됩니다. (저도 궁금해서 공식 문서를 읽어봤는데 [“`INCLUDE` indexes can never use deduplication“](https://www.postgresql.org/docs/current/btree.html) 라고 적혀있더군요)

| indexrelname | size | total\_pages | estimated\_tuples | tuples\_per\_page |
| --- | --- | --- | --- | --- |
| idx\_composite | 217 MB | 27817 | 31075028 | 1117.1236294352375 |
| idx\_include | 1822 MB | 233251 | 31075028 | 133.2257010688057 |

실제로 페이지에 저장된 튜플의 밀도도 훨씬 복합인덱스가 높은 것을 확인할 수 있습니다.

## 마치며

`INCLUDE` 관련 부하 테스트를 하게 되다가 알게 된 사실이라 적어봅니다. 언제 사용해야 할지 지금 까지 감은 카디널리티가 엄청나게 높고, key 로 잡아야 하는 컬럼의 데이터가 크다면 써볼 수 있을것 같습니다. 다만 이렇게 까다로운 경우 대부분 성능향상을 하기 위해서는 많은 테스팅이 필요하므로 테스트를 많이 해보고 도입해볼 것 같습니다.