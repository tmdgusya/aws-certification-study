---
title: "SQLAlchemy Select 심화: Result와 Row 객체 파헤치기"
seoTitle: "Deep Dive into SQLAlchemy: Results and Row Objects"
seoDescription: "SQLAlchemy의 `Result`와 `Row` 객체를 심층 분석하여 쿼리 결과 처리 및 데이터 접근 방식을 탐구합니다"
datePublished: Thu May 01 2025 15:50:33 GMT+0000 (Coordinated Universal Time)
cuid: cma5jkhtz000d09in66u5fbou
slug: sqlalchemy-select-result-row
tags: sqlalchemy, row, row0, result-object

---

오늘은 SQLAlchemy ORM의 `select` 문을 기본적인 사용법을 넘어 조금 더 깊이 있게 다뤄보겠습니다. 특히 `Session.execute()` 메서드의 반환값과 그 내부 구조에 초점을 맞추어, 쿼리 결과를 효과적으로 다루는 방법을 탐구할 것입니다.

### 기본 설정 및 예제 데이터 생성

먼저 예제 실행을 위한 기본적인 SQLAlchemy 설정을 살펴보겠습니다. `DeclarativeBase`를 상속받는 `Base` 클래스를 정의하고, `User`와 `Address` 모델을 매핑합니다. 데이터베이스는 메모리상의 SQLite를 사용합니다.

```python
from sqlalchemy import create_engine

# 메모리 내 SQLite 데이터베이스 엔진 생성 (echo=True로 SQL 쿼리 로깅 활성화)
engine = create_engine("sqlite+pysqlite:///:memory:", echo=True)

from typing import List
from typing import Optional
from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship
from sqlalchemy.orm import Session

class Base(DeclarativeBase):
    """
    DeclarativeBase를 상속받아 ORM 매핑의 기본 클래스로 사용합니다.
    이 Base를 상속하는 클래스는 데이터베이스 테이블과 매핑됩니다.
    __tablename__ 속성을 반드시 정의해야 합니다.
    """
    pass

class User(Base):
    __tablename__ = "user_account" # 테이블명 정의

    # 컬럼 정의 (Mapped와 mapped_column 사용)
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(30))
    fullname: Mapped[Optional[str]] # Optional 타입 매핑
    # Address 모델과의 관계 설정 (일대다)
    addresses: Mapped[List["Address"]] = relationship(
        back_populates="user", cascade="all, delete-orphan"
    )
    def __repr__(self) -> str:
        # 객체 표현 방식 정의
        return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"

class Address(Base):
    __tablename__ = "address" # 테이블명 정의

    id: Mapped[int] = mapped_column(primary_key=True)
    email_address: Mapped[str]
    # 외래키 설정 (user_account 테이블의 id 참조)
    user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    # User 모델과의 관계 설정 (다대일)
    user: Mapped["User"] = relationship(back_populates="addresses")
    def __repr__(self) -> str:
        return f"Address(id={self.id!r}, email_address={self.email_address!r})"

# 정의된 모델을 기반으로 데이터베이스 테이블 생성
Base.metadata.create_all(engine)
```

이제 예제 데이터를 생성하는 함수를 정의하고 실행하여 데이터베이스에 100명의 사용자 정보를 추가합니다.

```python
from sqlalchemy.orm import Session

def generate_100_users():
    """100명의 User와 연관된 Address 데이터를 생성하여 데이터베이스에 추가합니다."""
    with Session(engine) as session:
        for i in range(100):
            session.add(User(name=f"user{i}", fullname=f"User {i}", addresses=[Address(email_address=f"user{i}@example.com")]))
        session.flush() # 변경사항을 데이터베이스 세션에 반영 (커밋 전)
        session.commit() # 트랜잭션 커밋

generate_100_users()
```

### Result 와 Row 객체: 쿼리 결과의 핵심

SQLAlchemy 2.0 버전부터 ORM에서 쿼리를 실행하는 주요 방법은 `Session.execute()` 메서드를 사용하는 것입니다. 공식 문서에 따르면 이 메서드는 `Result` 객체를 반환한다고 명시되어 있습니다. 그렇다면 이 `Result` 객체는 정확히 무엇일까요?

```python
from sqlalchemy import select

with Session(engine) as session:
    stmt = select(User) # User 테이블 전체를 조회하는 select 문 생성
    users_result = session.execute(stmt) # select 문 실행
    print(users_result) # 실행 결과 객체 출력
    print(type(users_result)) # 결과 객체의 타입 출력
```

위 코드를 실행하면 실제로는 `ChunkedIteratorResult` 타입의 객체가 반환되는 것을 확인할 수 있습니다. SQLAlchemy의 내부 코드를 살펴보면 `Result` 클래스 자체는 여러 메서드가 `NotImplementedError`를 발생시키는, 일종의 인터페이스(Interface) 역할을 하고 있음을 알 수 있습니다.

```python
# SQLAlchemy 내부 코드 예시 (간략화)
class Result(_WithKeys, ResultInternal[Row[Unpack[_Ts]]]):
    """데이터베이스 결과 집합을 나타냅니다.
    ORM 사용 시에는 일반적으로 ChunkedIteratorResult 객체가 사용됩니다.
    """
    @property
    def _soft_closed(self) -> bool:
        raise NotImplementedError()

    @property
    def closed(self) -> bool:
        raise NotImplementedError()
    # ... 다른 메서드들 ...
```

즉, `Session.execute()`는 실제 구현체인 `ChunkedIteratorResult` 객체를 반환하며, 이 객체는 `Result` 인터페이스를 구현합니다. 따라서 우리는 `Result` 객체가 제공하는 다양한 메서드를 통해 쿼리 결과를 다룰 수 있습니다.

`Result` 객체는 다음과 같은 여러 유용한 메서드를 제공합니다.

```python
all(), close(), closed, columns(), fetchall(), fetchmany(), fetchone(), first(), freeze(), keys(), mappings(), merge(), one(), one_or_none(), partitions(), scalar(), scalar_one(), scalar_one_or_none(), scalars(), t, tuples(), unique(), yield_per()
```

이번 포스트에서는 이 중 몇 가지 주요 메서드, 특히 데이터를 가져오는 메서드들을 중심으로 살펴보겠습니다.

### Result.all() 메서드 심층 분석

가장 흔히 사용되는 메서드 중 하나인 `all()`은 쿼리 결과의 모든 행(row)을 리스트 형태로 반환합니다. 내부적으로 어떻게 동작하는지 SQLAlchemy 코드를 통해 간단히 살펴보겠습니다. (주석은 설명을 위해 추가되었습니다)

```python
# Result 클래스의 _allrows 메서드 (all() 내부에서 호출됨, 간략화)
def _allrows(self) -> List[_R]:
    # 후처리 필터: 결과를 최종 형태로 가공하는 내부 로직
    post_creational_filter = self._post_creational_filter

    # 행(row) 생성 함수: 가져온 데이터를 Row 객체 등으로 변환
    make_row = self._row_getter

    # 데이터베이스로부터 모든 데이터 가져오기
    rows = self._fetchall_impl()
    made_rows: List[_InterimRowType[_R]]

    # 가져온 데이터를 Row 객체 또는 지정된 타입으로 변환
    if make_row:
        made_rows = [make_row(row) for row in rows]
    else:
        made_rows = rows

    interim_rows: List[_R]

    # unique 필터 적용: strategy에 따라 중복 제거 (set 사용)
    if self._unique_filter_state:
        uniques, strategy = self._unique_strategy
        interim_rows = [
            # strategy 함수를 적용하여 고유 식별자를 얻고, uniques set에 없는 경우만 포함
            made_row
            for made_row, sig_row in [
                (made_row, strategy(made_row) if strategy else made_row)
                for made_row in made_rows
            ]
            if sig_row not in uniques and not uniques.add(sig_row)
        ]
    else:
        interim_rows = made_rows

    # 후처리 필터 적용
    if post_creational_filter:
        interim_rows = [post_creational_filter(row) for row in interim_rows]

    return interim_rows # 최종 결과 리스트 반환
```

코드를 분석해보면 `all()` 메서드는 단순히 모든 데이터를 가져오는 것 외에도 다음과 같은 과정을 포함합니다.

1. **데이터 인출 (**`_fetchall_impl`): 데이터베이스로부터 결과 행들을 가져옵니다.
    
2. **행 객체 생성 (**`make_row`): 가져온 데이터를 SQLAlchemy의 `Row` 객체나 다른 지정된 형태로 변환합니다.
    
3. **고유값 처리 (**`_unique_filter_state`): `unique()` 메서드가 호출된 경우, 제공된 `strategy` 함수를 사용하여 중복된 행을 제거합니다. `strategy`는 각 행에서 고유성을 판단할 값을 반환하는 함수이며, 파이썬의 `set`을 이용하여 중복을 검사합니다.
    
4. **후처리 (**`post_creational_filter`): 내부적인 추가 처리를 통해 최종 반환 형태를 만듭니다.
    

이렇게 `all()` 메서드는 내부적으로 여러 단계를 거쳐 최종 결과 리스트를 반환합니다.

```python
from sqlalchemy import select

with Session(engine) as session:
    # User 테이블에서 name과 fullname 컬럼만 선택
    stmt = select(User.name, User.fullname)
    # 쿼리 실행 후 모든 결과를 리스트로 가져옴
    users = session.execute(stmt).all()
    print(type(users)) # <class 'list'>
    # 리스트의 첫 번째 요소 (Row 객체)에 접근하여 'name' 속성 출력
    print(users[0]["name"]) # Row 객체는 키(컬럼명)로 접근 가능
    print(users[0].name)   # Row 객체는 속성처럼 접근 가능
```

`all()` 메서드는 `Row` 객체들의 리스트를 반환합니다. `Row` 객체는 마치 이름 있는 튜플(named tuple)처럼 동작하여, 컬럼 이름([`user.name`](http://user.name))이나 인덱스(`user[0]`) 또는 키(`user["name"]`)로 값에 접근할 수 있습니다.

### 중복 데이터 처리: `unique()` 메서드 활용

쿼리 결과에서 중복을 제거해야 하는 경우가 있습니다. 예를 들어 특정 조건을 만족하는 사용자를 조회했는데, 이름이 같은 사용자가 여러 명 포함될 수 있습니다.

먼저, 이름이 'user1'인 사용자를 추가하여 중복 상황을 만들어 보겠습니다.

```python
# 예시: 중복 데이터 확인
with Session(engine) as session:
    # 기존 'user1' 조회
    stmt_before = select(User).where(User.name == "user1")
    users_before = session.execute(stmt_before).all()
    print(f"중복 추가 전 'user1' 수: {len(users_before)}")
    if users_before:
        print(f"기존 'user1' 정보: {users_before[0][0]}") # Row 객체 안의 User 객체 접근

# 이름이 'user1'인 사용자 추가 (ID는 다름)
with Session(engine) as session:
    duplicated_user = User(id=101, name="user1", fullname="user1_duplicated", addresses=[Address(email_address="duplicated_user@gmail.com")])
    session.add(duplicated_user)
    session.commit()

# 중복 추가 후 'user1' 조회
with Session(engine) as session:
    stmt_after = select(User).where(User.name == "user1")
    users_after = session.execute(stmt_after).all()
    print(f"중복 추가 후 'user1' 수: {len(users_after)}")
    print(f"조회된 'user1' 사용자 리스트: {users_after}")
```

이제 이름('user1')을 기준으로 중복을 제거해 보겠습니다. `Result` 객체의 `unique()` 메서드를 사용하고, `strategy` 인자에 어떤 기준으로 고유성을 판단할지 정의하는 함수(lambda)를 전달합니다.

```python
with Session(engine) as session:
    stmt = select(User).where(User.name == "user1")
    # unique() 메서드로 중복 제거. strategy는 User 객체의 name 속성을 기준으로 함.
    unique_users = session.execute(stmt).unique(strategy=lambda row: row[0].name).all()
    print(f"unique() 적용 후 'user1' 수: {len(unique_users)}") # 중복 제거 확인
```

`unique()` 메서드는 `all()`과 같은 다른 결과 추출 메서드( `first()`, `scalars()` 등) 전에 호출되어야 합니다. `strategy` 함수는 `Row` 객체를 인자로 받으므로, `Row` 객체의 구조를 이해하고 올바른 접근 방식을 사용해야 합니다 (이후 `Row` 섹션에서 자세히 설명).

### Row 객체와 접근 방식의 이해

앞선 `unique()` 예제에서 `lambda row: row[0].name` 와 같은 접근 방식을 사용했습니다. 어떤 경우에는 [`row.name`](http://row.name)처럼 바로 속성에 접근할 수 있는데, 왜 여기서는 `row[0]`을 거쳐야 했을까요? 이 차이를 이해하려면 `Row` 객체의 동작 방식과 `select()` 구문에 따른 변화를 알아야 합니다.

`Row` 객체는 기본적으로 파이썬의 `namedtuple`과 유사하게 설계되었습니다. `namedtuple`은 튜플처럼 불변(immutable)성을 가지면서도 각 요소에 이름을 부여하여 속성처럼 접근할 수 있게 해주는 편리한 자료구조입니다.

```python
from collections import namedtuple

# 클래스를 사용한 데이터 표현
class BookClass:
    def __init__(self, title, code):
        self.title = title
        self.code = code
book_obj = BookClass(title="roach", code="1")
print(f"클래스 객체: {book_obj.title}")

# 튜플을 사용한 데이터 표현 (인덱스로 접근)
book_tuple = ("roach", "1")
print(f"튜플: {book_tuple[0]}")

# namedtuple을 사용한 데이터 표현 (이름 또는 인덱스로 접근 가능)
BookNamedTuple = namedtuple('Book', ['title', 'code'])
book_namedtuple = BookNamedTuple("roach", "1")
print(f"namedtuple (속성 접근): {book_namedtuple.title}")
print(f"namedtuple (인덱스 접근): {book_namedtuple[0]}")
```

`Row` 객체도 이와 비슷하게, **인덱스(**`row[0]`), **키/컬럼명(**`row["name"]`), 그리고 **속성(**[`row.name`](http://row.name)) 방식으로 값에 접근할 수 있습니다.

**핵심은** `select()` 구문에 무엇을 전달했는지에 따라 `Row` 객체의 구조가 달라진다는 점입니다.

1. `select(`[`User.name`](http://User.name)`, User.fullname)`: 특정 컬럼들을 명시적으로 선택한 경우
    
    * `Result` 객체의 메타데이터(`ResultMetadata`)에는 선택된 컬럼 이름들(`['name', 'fullname']`)이 `keys`로 저장됩니다.
        
    * 반환된 `Row` 객체는 이 `keys`를 기반으로 구조화됩니다.
        
    * 따라서 [`row.name`](http://row.name) 또는 `row[0]` (첫 번째 선택 컬럼), `row.fullname` 또는 `row[1]` (두 번째 선택 컬럼) 방식으로 직접 접근할 수 있습니다.
        
    
    ```python
    from sqlalchemy import select
    
    with Session(engine) as session:
        # name, fullname 컬럼만 선택
        stmt = select(User.name, User.fullname).where(User.name == "user1")
        users = session.execute(stmt).all()
        print(f"\nselect(User.name, User.fullname) 결과 타입: {type(users)}")
        for user_row in users:
            print(f"Row 객체 타입: {type(user_row)}")
            # 속성 접근
            print(f"  속성 접근: user_row.name = {user_row.name}, user_row.fullname = {user_row.fullname}")
            # 인덱스 접근
            print(f"  인덱스 접근: user_row[0] = {user_row[0]}, user_row[1] = {user_row[1]}")
            # 키 접근
            print(f"  키 접근: user_row['name'] = {user_row['name']}, user_row['fullname'] = {user_row['fullname']}")
    ```
    
2. `select(User)`: ORM 모델 객체 자체를 선택한 경우
    
    * `Result` 객체의 메타데이터에는 모델 클래스 이름(`['User']`)이 `keys`로 저장됩니다.
        
    * 반환된 `Row` 객체의 첫 번째 요소(`row[0]`)가 바로 `User` **객체** 자체가 됩니다. `Row` 객체는 이 `User` 객체를 감싸고 있는 형태입니다.
        
    * 따라서 `User` 객체의 속성에 접근하려면 `row[0].name`, `row[0].fullname`과 같이 인덱스로 먼저 `User` 객체에 접근한 후 속성에 접근해야 합니다.
        
    * 또는 `Row` 객체가 키로도 접근을 지원하므로, [`row.User.name`](http://row.User.name) (키 `User`로 `User` 객체 접근 후 속성 접근) 방식으로도 가능합니다.
        
    
    ```python
    with Session(engine) as session:
        # User 객체 전체를 선택
        stmt = select(User).where(User.name == "user1")
        users_result = session.execute(stmt) # Result 객체
        print(f"\nselect(User) Result keys: {users_result.keys()}") # ['User']
    
        users = users_result.all()
        print(f"select(User) 결과 타입: {type(users)}")
        for user_row in users:
            print(f"Row 객체 타입: {type(user_row)}")
            # 인덱스 접근 후 속성 접근
            user_obj = user_row[0] # User 객체
            print(f"  인덱스 접근: type(user_row[0]) = {type(user_obj)}")
            print(f"  인덱스 접근 -> 속성: user_row[0].name = {user_obj.name}")
            # 키/속성 접근 후 속성 접근
            print(f"  키/속성 접근 -> 속성: user_row.User.name = {user_row.User.name}") # row.User 로 User 객체 접근 가능
    ```
    

이제 아까 `unique()` 메서드의 `strategy`에서 `lambda row: row[0].name` 또는 `lambda row:` [`row.User.name`](http://row.User.name)을 사용해야 했던 이유가 명확해졌습니다. `select(User)`로 조회했기 때문에 `Row` 객체에서 `User` 객체(`row[0]` 또는 `row.User`)를 먼저 얻어온 다음 `name` 속성에 접근해야 했던 것입니다.

```python
# select(User) 결과에 대해 .User 속성을 사용한 unique 처리
with Session(engine) as session:
    stmt = select(User).where(User.name == "user1")
    # strategy에서 row.User.name 사용
    users = session.execute(stmt).unique(strategy=lambda row: row.User.name).all()
    print(f"\nselect(User) + unique(strategy=lambda row: row.User.name) 결과 수: {len(users)}")
```

이처럼 `select()` 구문에 무엇을 전달하느냐에 따라 `Row` 객체의 구조와 접근 방식이 달라지므로, 코드를 작성할 때 이를 명확히 인지하는 것이 중요합니다.

### 결론

이번 포스트에서는 SQLAlchemy ORM의 `select` 쿼리 실행 결과인 `Result` 객체와 그 안의 `Row` 객체에 대해 심도 있게 알아보았습니다.

* `Session.execute()`는 `ChunkedIteratorResult` ( `Result` 인터페이스 구현체) 객체를 반환합니다.
    
* `Result` 객체는 `all()`, `unique()` 등 다양한 메서드를 통해 쿼리 결과를 가공하고 추출하는 기능을 제공합니다.
    
* `Row` 객체는 `namedtuple`과 유사하게 동작하며, 인덱스, 키(컬럼명), 속성 방식으로 데이터에 접근할 수 있습니다.
    
* `select()` 구문에 어떤 요소를 전달하는지에 따라 (`select(`[`User.name`](http://User.name)`)` vs `select(User)`) `Row` 객체의 내부 구조와 접근 방식이 달라진다는 점을 유의해야 합니다.
    

`Result`와 `Row` 객체의 동작 방식을 이해하면 SQLAlchemy를 더욱 효과적으로 활용하고, 다양한 상황에 맞는 최적의 데이터 접근 코드를 작성하는 데 큰 도움이 될 것입니다.