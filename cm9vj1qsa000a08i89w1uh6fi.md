---
title: "SQLAlchemy 시리즈 1 - Connection 과 Result"
seoTitle: "SQLAlchemy Basics: Connection & Result"
seoDescription: "Python에서 SQLAlchemy를 사용한 데이터베이스 연결 및 결과 처리 방법과 트랜잭션 관리 기술을 소개합니다"
datePublished: Thu Apr 24 2025 15:38:16 GMT+0000 (Coordinated Universal Time)
cuid: cm9vj1qsa000a08i89w1uh6fi
slug: sqlalchemy-1-connection-result
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1745509491052/df66f978-9f62-42b1-a18e-1b769ed83fe1.png
tags: python, orm, session, sqlalchemy, fastapi

---

Python 애플리케이션에서 데이터베이스 작업을 수행할 때, SQLAlchemy는 강력하고 유연한 ORM(Object Relational Mapper)이자 SQL 툴입니다. 데이터베이스와의 상호작용에서 가장 기본적이면서도 중요한 부분은 **연결(Connection)을 관리하고 쿼리 결과를 처리하는 것**입니다. 이번 글에서는 SQLAlchemy의 `Connection` 객체를 활용한 트랜잭션 관리 방법과 쿼리 결과를 담는 `Result` 및 `Row` 객체의 효과적인 사용법에 대해 자세히 알아보겠습니다.

### SQLAlchemy Engine 설정

모든 SQLAlchemy 애플리케이션은 `Engine` 객체로부터 시작합니다. `Engine`은 데이터베이스와의 연결을 위한 입구 역할을 하며, 실제 DBAPI 연결을 관리하는 `Connection` 객체를 생성하는 **팩토리(Factory)**입니다. 가벼운 예제이므로 SQLite 인메모리(in-memory) 데이터베이스를 위한 `Engine`을 생성하여 진행해 보겠습니다.

(\*\*`echo=True` 인자는 실행되는 SQL 쿼리를 로깅하여 디버깅에 유용합니다.)

```python
from sqlalchemy import create_engine

engine = create_engine("sqlite+pysqlite:///:memory:", echo=True)
```

### Connection을 이용한 기본적인 작업과 트랜잭션

데이터베이스와 직접 상호작용하기 위해서는 `Engine`으로부터 `Connection` 객체를 얻어야 합니다. `Connection` 객체는 데이터베이스 작업을 수행하는 데 필요한 리소스를 제공하며, 작업 범위를 정의하고 해당 범위 내에서 리소스 관리를 돕습니다. 파이썬의 `with` 문과 함께 사용하면 `Connection` 객체가 컨텍스트 관리자(`contextmanager`)로 동작하여, 블록을 벗어날 때 자동으로 연결 리소스를 정리해 줍니다.

```python
from sqlalchemy import text

with engine.connect() as conn:
    result = conn.execute(text("select 'Hello World'"))
    print(result.all())
```

위 코드의 실행 결과를 보면, `with` 블록이 시작될 때 **암시적(implicit)으로 트랜잭션이 시작**(`BEGIN (implicit)`)되고, 쿼리가 실행된 후 블록이 종료될 때 명시적인 `commit()` 호출이 없었기 때문에 자동으로 `ROLLBACK`이 수행됩니다.

```python
2025-04-25 00:03:30,561 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2025-04-25 00:03:30,562 INFO sqlalchemy.engine.Engine select 'Hello World'
2025-04-25 00:03:30,563 INFO sqlalchemy.engine.Engine [generated in 0.00221s] ()
[('Hello World',)]
2025-04-25 00:03:30,564 INFO sqlalchemy.engine.Engine ROLLBACK
```

데이터베이스 상태를 변경하는 작업(예: `INSERT`, `UPDATE`, `DELETE`)을 영구적으로 반영하려면, 작업 완료 후 `conn.commit()`을 명시적으로 호출해야 합니다.

```python
with engine.connect() as conn:
    """데이터를 생성하고 커밋하는 예제"""
    conn.execute(text("CREATE TABLE some_table (x int, y int)"))
    conn.execute(text("INSERT INTO some_table (x, y) VALUES (:x, :y)"), [{"x": 1, "y": 2}, {"x": 3, "y": 4}])
    conn.commit() # 변경 사항을 커밋합니다.
```

실행 결과를 보면 `COMMIT`이 명시적으로 수행되어 데이터베이스 변경 사항이 저장됩니다.

```python
2025-04-25 00:03:30,571 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2025-04-25 00:03:30,572 INFO sqlalchemy.engine.Engine CREATE TABLE some_table (x int, y int)
2025-04-25 00:03:30,573 INFO sqlalchemy.engine.Engine [generated in 0.00173s] ()
2025-04-25 00:03:30,574 INFO sqlalchemy.engine.Engine INSERT INTO some_table (x, y) VALUES (?, ?)
2025-04-25 00:03:30,574 INFO sqlalchemy.engine.Engine [generated in 0.00048s] [(1, 2), (3, 4)]
2025-04-25 00:03:30,575 INFO sqlalchemy.engine.Engine COMMIT
```

이처럼 각 작업 단위가 끝날 때마다 `commit()`을 호출하는 방식을 SQLAlchemy에서는 **"Commit as you go"** 패턴이라고 부릅니다. 이 패턴은 개발자가 트랜잭션의 범위를 명확하게 제어할 수 있게 해주지만, 커밋 시점을 직접 관리해야 하는 부담이 있습니다.

### `begin()` 메소드를 활용한 자동 트랜잭션 관리 ("Begin once")

"Commit as you go" 패턴의 번거로움을 줄이고 트랜잭션 관리를 자동화하기 위해 SQLAlchemy는 `engine.begin()` **메소드를 제공**합니다. 이 메소드 역시 `with` 문과 함께 사용되며, 컨텍스트 블록에 진입할 때 자동으로 `Connection`을 얻고 트랜잭션을 시작합니다. 블록 내의 모든 작업이 성공적으로 완료되면 자동으로 `COMMIT`을 수행하고, 예외가 발생하면 자동으로 `ROLLBACK`을 수행합니다. 이를 **"Begin once"** 패턴이라고 합니다.

```python
with engine.begin() as conn:
    conn.execute(text("INSERT INTO some_table (x, y) VALUES (:x, :y)"), [{"x": 6, "y": 7}, {"x": 8, "y": 9}])
    raise Exception("test") # 예외 발생 시뮬레이션
```

위 코드에서는 `INSERT` 작업 후 의도적으로 예외를 발생시켰습니다. 실행 결과를 보면 트랜잭션이 시작되고 `INSERT` 쿼리가 실행되었지만, 예외 발생으로 인해 `ROLLBACK`이 수행된 것을 확인할 수 있습니다.

```python
2025-04-25 00:03:30,582 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2025-04-25 00:03:30,583 INFO sqlalchemy.engine.Engine INSERT INTO some_table (x, y) VALUES (?, ?)
2025-04-25 00:03:30,583 INFO sqlalchemy.engine.Engine [cached since 0.009627s ago] [(6, 7), (8, 9)]
2025-04-25 00:03:30,584 INFO sqlalchemy.engine.Engine ROLLBACK
```

```python
# 예외 없이 정상적으로 완료되는 경우
with engine.begin() as conn:
    conn.execute(text("INSERT INTO some_table (x, y) VALUES (:x, :y)"), [{"x": 10, "y": 20}, {"x": 30, "y": 40}])
    # conn.commit() # begin() 사용 시 명시적 커밋 불필요
```

성공적으로 완료된 경우, `COMMIT`이 자동으로 수행됩니다.

```python
2025-04-25 00:00:13,047 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2025-04-25 00:00:13,049 INFO sqlalchemy.engine.Engine INSERT INTO some_table (x, y) VALUES (?, ?)
2025-04-25 00:00:13,050 INFO sqlalchemy.engine.Engine [cached since 9.264s ago] [(10, 20), (30, 40)]
2025-04-25 00:00:13,051 INFO sqlalchemy.engine.Engine COMMIT
```

흥미로운 점은 `begin()` 블록 시작 시 로그에 `BEGIN (implicit)`이라고 표시되는 부분입니다. 이는 실제 데이터베이스에 `BEGIN` 명령을 보내는 것이 아니라, SQLAlchemy **내부적으로 트랜잭션이 시작되었음을 논리적으로 표시**하는 것입니다. 실제 DB 트랜잭션은 첫 번째 SQL 명령이 실행될 때 시작됩니다. `engine.begin()`은 개발자의 실수를 줄이고 보다 안정적인 트랜잭션 관리를 가능하게 합니다.

다음 코드로 롤백된 데이터 `(6, 7)`, `(8, 9)`는 테이블에 존재하지 않고, 커밋된 데이터만 존재하는 것을 확인할 수 있습니다.

```python
"""
롤백된 데이터가 안들어간것 확인 (6,7), (8,9) 데이터가 롤백된것 확인
"""
with engine.begin() as conn:
    result = conn.execute(text("SELECT x, y FROM some_table"))
    for row in result:
        print(f"x: {row.x}, y: {row.y}")
```

```python
2025-04-25 00:02:22,486 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2025-04-25 00:02:22,488 INFO sqlalchemy.engine.Engine SELECT x, y FROM some_table
2025-04-25 00:02:22,489 INFO sqlalchemy.engine.Engine [cached since 126.4s ago] ()
x: 1, y: 2
x: 3, y: 4
x: 1, y: 2 # 이전에 conn.commit()으로 추가된 데이터
x: 3, y: 4 # 이전에 conn.commit()으로 추가된 데이터
x: 10, y: 20 # engine.begin()으로 추가된 데이터
x: 30, y: 40 # engine.begin()으로 추가된 데이터
2025-04-25 00:02:22,490 INFO sqlalchemy.engine.Engine COMMIT
```

### 쿼리 결과 처리: Result와 Row 객체

`conn.execute()` 메소드는 `Result` 객체를 반환합니다. 이 객체는 데이터베이스 쿼리 결과를 나타내며, **순회 가능(iterable)**한 특징을 가집니다. 따라서 `for` 루프 등을 사용하여 결과 행(row)들을 하나씩 처리하거나, `all()`, `first()`, `scalar()` 등 다양한 메소드를 활용하여 결과를 원하는 형태로 **변환(transform)**할 수 있습니다.

```python
# Result 객체 타입 확인 및 순회 예시
with engine.begin() as conn:
    result = conn.execute(text("SELECT x, y FROM some_table WHERE x > 5")) # 조건 추가
    print(type(result))
    for row in result:
        print(f"x: {row.x}, y: {row.y}")
```

```python
# (이전 실행 로그 생략)
2025-04-25 00:08:21,655 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2025-04-25 00:08:21,656 INFO sqlalchemy.engine.Engine SELECT x, y FROM some_table WHERE x > 5 # WHERE 조건 추가됨
2025-04-25 00:08:21,657 INFO sqlalchemy.engine.Engine [generated in 0.00043s] ()
<class 'sqlalchemy.engine.cursor.CursorResult'>
x: 10, y: 20
x: 30, y: 40
2025-04-25 00:08:21,658 INFO sqlalchemy.engine.Engine COMMIT
```

`Result` 객체를 순회할 때 얻어지는 각 행은 `Row` 객체입니다. `Row` 객체는 파이썬의 `collections.namedtuple`과 유사하게 동작하는 불변(immutable) 자료구조입니다. 각 컬럼의 값에 접근할 때 속성(attribute) 방식(`row.x`, `row.y`)이나 인덱스 방식(`row[0]`, `row[1]`)을 사용할 수 있습니다.

`Row` 객체의 독특한 특징 중 하나는 **언패킹(unpacking)**이 가능하다는 점입니다. 이를 활용하면 조회된 데이터를 사용자 정의 객체나 다른 자료구조로 쉽게 변환할 수 있습니다.

**1\. 위치 인자 언패킹 (**`*row`): `Row` 객체를 `*` 연산자와 함께 사용하면, 각 컬럼 값이 순서대로 함수의 인자로 전달됩니다.

```python
class Data():
    def __init__(self, x, y) -> None:
        self.x = x
        self.y = y

    def __repr__(self) -> str:
        return f"Data ({self.x}, {self.y})"

with engine.begin() as conn:
    result = conn.execute(text("SELECT x, y FROM some_table WHERE x < 5"))
    """
    위치 인자 언패킹을 활용한 데이터 변환
    """
    data = [Data(*row) for row in result]
    print(data)
```

```python
# (이전 실행 로그 생략)
2025-04-25 00:21:09,976 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2025-04-25 00:21:09,978 INFO sqlalchemy.engine.Engine SELECT x, y FROM some_table WHERE x < 5
2025-04-25 00:21:09,979 INFO sqlalchemy.engine.Engine [cached since 768.3s ago] ()
[Data (1, 2), Data (3, 4), Data (1, 2), Data (3, 4)]
2025-04-25 00:21:09,980 INFO sqlalchemy.engine.Engine COMMIT
```

**2\. 키워드 인자 언패킹 (**`**row._mapping`): `Row` 객체는 내부적으로 컬럼 이름을 키(key)로, 값을 값(value)으로 가지는 매핑(mapping) 인터페이스를 제공합니다 (`_mapping` 속성). `**` 연산자와 `_mapping` 속성을 함께 사용하면, 컬럼 이름과 값을 키워드 인자로 전달할 수 있습니다. 이는 생성자나 함수가 키워드 인자를 받을 때 유용합니다.

```python
class Data():
    def __init__(self, **kwargs) -> None: # 키워드 인자를 받도록 수정
        self.x = kwargs["x"]
        self.y = kwargs["y"]

    def __repr__(self) -> str:
        return f"Data ({self.x}, {self.y})"

with engine.begin() as conn:
    result = conn.execute(text("SELECT x, y FROM some_table WHERE x < 5"))
    """
    키워드 인자 언패킹을 활용한 데이터 변환
    """
    data = [Data(**row._mapping) for row in result]
    print(data)
```

```python
# (이전 실행 로그 생략)
2025-04-25 00:21:13,280 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2025-04-25 00:21:13,282 INFO sqlalchemy.engine.Engine SELECT x, y FROM some_table WHERE x < 5
2025-04-25 00:21:13,283 INFO sqlalchemy.engine.Engine [cached since 771.6s ago] ()
[Data (1, 2), Data (3, 4), Data (1, 2), Data (3, 4)]
2025-04-25 00:21:13,284 INFO sqlalchemy.engine.Engine COMMIT
```

이러한 언패킹 기능은 데이터베이스 조회 결과를 파이썬 객체로 변환하는 과정을 매우 간결하고 직관적으로 만들어 줍니다.

### 결론

SQLAlchemy를 사용하여 데이터베이스와 상호작용할 때, `Connection` 객체를 통한 명시적인 트랜잭션 관리("Commit as you go")와 `engine.begin()`을 활용한 자동 트랜잭션 관리("Begin once") 방법을 이해하는 것은 중요합니다. 또한, 쿼리 결과를 담는 `Result` 객체의 순회 가능한 특성과 `Row` 객체의 불변성 및 언패킹 기능을 활용하면 데이터를 효율적으로 처리하고 원하는 형태로 변환하는 데 큰 도움이 됩니다. 이러한 SQLAlchemy의 핵심 기능들을 잘 이해하고 활용한다면 더욱 견고하고 효율적인 데이터베이스 애플리케이션을 구축할 수 있을 것입니다.