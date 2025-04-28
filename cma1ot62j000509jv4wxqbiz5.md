---
title: "SQLAlchemy Session과 SELECT 쿼리: identity_map과 get() 메서드 활용법"
seoTitle: "Optimizing SQLAlchemy Queries with Identity Map"
seoDescription: "Explore utilizing SQLAlchemy Session's identity_map and get() method for optimized SELECT queries and efficient database interactions"
datePublished: Mon Apr 28 2025 23:06:11 GMT+0000 (Coordinated Universal Time)
cuid: cma1ot62j000509jv4wxqbiz5
slug: sqlalchemy-session-select-identitymap-get
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/ylveRpZ8L1s/upload/20c32be7563ed9c16cc0308ef46f3f88.jpeg
tags: python, orm, session, sqlalchemy

---

SQLAlchemy는 파이썬 개발자들에게 강력한 ORM(Object-Relational Mapper) 기능을 제공하여 데이터베이스 상호작용을 용이하게 합니다. 이 과정에서 `Session` 객체는 핵심적인 역할을 수행합니다. 이번 포스트에서는 SQLAlchemy `Session`의 개념을 다시 한번 살펴보고, 특히 SELECT 쿼리를 수행할 때 `identity_map`과 `Session.get()` 메서드가 어떻게 동작하는지 자세히 알아보겠습니다.

### 프로젝트 준비: 테이블 생성

먼저 실습을 위해 SQLAlchemy 엔진과 ORM 모델을 설정합니다. SQLite 메모리 데이터베이스를 사용하고, `User`와 `Address` 두 개의 테이블을 정의합니다.

Python

```python
from sqlalchemy import create_engine

engine = create_engine("sqlite+pysqlite:///:memory:", echo=True)
```

Python

```python
from typing import List
from typing import Optional
from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    """
    DeclarativeBase 를 상속 받은 Base 라는 하위 클래스를 만들고 시작.
    이 Base 에 Mapping 된 클래스들은 database 에서 단일 테이블임.

    `__tablename__` 을 클래스 레벨의 속성으로 지녀야 함.
    """
    pass

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(30))
    fullname: Mapped[Optional[str]]
    addresses: Mapped[List["Address"]] = relationship(
        back_populates="user", cascade="all, delete-orphan"
    )
    def __repr__(self) -> str:
        return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"

class Address(Base):
    __tablename__ = "address"
    id: Mapped[int] = mapped_column(primary_key=True)
    email_address: Mapped[str]
    user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    user: Mapped["User"] = relationship(back_populates="addresses")
    def __repr__(self) -> str:
        return f"Address(id={self.id!r}, email_address={self.email_address!r})"
```

정의된 모델을 기반으로 데이터베이스에 테이블을 생성합니다.

```python
Base.metadata.create_all(engine)
```

위 코드를 실행하면 `echo=True` 설정에 따라 테이블 생성 SQL 쿼리가 로그로 출력됩니다.

### SQLAlchemy Session이란?

SQLAlchemy ORM에서 `Session`은 애플리케이션과 데이터베이스 간의 모든 영속성 작업(persistence operations)을 관리하는 주요 인터페이스입니다. 간단히 말해, 파이썬 객체(ORM 매핑된 인스턴스)와 데이터베이스 테이블 간의 상호작용을 책임지는 "작업 단위(Unit of Work)"이자 "컨텍스트(Context)"라고 생각할 수 있습니다.

### Session 생성 및 데이터 삽입

가장 기본적인 방법으로 `Session` 객체를 생성하고 데이터를 삽입해보겠습니다.

```python
from sqlalchemy.orm import Session

with Session(engine) as sess:
    roach = User(
        name="roach",
        fullname="dev roach",
        addresses=[Address(email_address="roach@sqlalchemy.org")]
    )

    john = User(
        name="john",
        fullname="dev john",
        addresses=[Address(email_address="john@sqlalchemy.org")]
    )

    sess.add_all([roach, john])
    sess.commit()
```

`with` 구문을 사용하여 세션을 관리하면 블록 종료 시 자동으로 리소스가 정리됩니다. `add_all()` 메서드로 여러 객체를 한 번에 추가하고, `commit()` 메서드로 변경 사항을 데이터베이스에 반영합니다. 로그를 통해 `INSERT` 쿼리가 실행되었음을 확인할 수 있습니다.

### SELECT 조회와 identity\_map

이제 저장된 데이터를 조회해 보겠습니다. 먼저 `select` 구문과 `Session.execute()` 메서드를 사용합니다.

```python
from sqlalchemy.orm import Session
from sqlalchemy import select

with Session(engine) as sess:
    stmt = select(User).where(User.name == "roach")
    result = sess.execute(stmt).scalar_one_or_none()
    print(result)
```

로그를 보면 `SELECT` 쿼리가 실행되어 `User` 객체를 성공적으로 조회했음을 알 수 있습니다.

그렇다면 동일한 세션 내에서 동일한 기본 키(Primary Key)로 객체를 다시 조회하면 어떻게 될까요? SQLAlchemy `Session`은 `identity_map`이라는 메커니즘을 사용하여 로드된 객체를 관리합니다. 이론적으로 `identity_map`에 객체가 이미 존재한다면 데이터베이스에 다시 접근할 필요가 없을 것입니다. 이를 확인하기 위해 기본 키를 사용하여 조회해 보겠습니다.

```python
from sqlalchemy.orm import Session
from sqlalchemy import select

with Session(engine) as sess:
    stmt = select(User).where(User.id == 1)
    result = sess.execute(stmt).scalar_one_or_none()
    
    stmt = select(User).where(User.id == 1)
    result = sess.execute(stmt).scalar_one_or_none()

    print(result)
```

로그를 확인하면 예상과 달리 `SELECT` 쿼리가 두 번 실행된 것을 볼 수 있습니다. `identity_map`을 확인해보면 첫 번째 `SELECT` 이후 객체가 맵에 존재함에도 불구하고 두 번째 `SELECT` 쿼리가 다시 데이터베이스로 전송되었습니다.

```python
from sqlalchemy.orm import Session
from sqlalchemy import select

with Session(engine) as sess:
    stmt = select(User).where(User.id == 1)
    result = sess.execute(stmt).scalar_one_or_none()

    # identity_map 내용 확인 (내부 구조 확인용)
    print(sess.identity_map.all_states()[0].__dict__) 
    
    stmt = select(User).where(User.id == 1)
    result = sess.execute(stmt).scalar_one_or_none()

    print(result)
```

이는 `Session.execute()` 메서드의 기본적인 동작 방식 때문입니다. 공식 문서([`Session.execute`](https://www.google.com/search?q=%5Bhttps://docs.sqlalchemy.org/en/20/orm/session_api.html%23sqlalchemy.orm.Session.execute))에\](https://www.google.com/search?q=https://docs.sqlalchemy.org/en/20/orm/session\_api.html%23sqlalchemy.orm.Session.execute))%EC%97%90) 따르면, 이 메서드는 주어진 SQL 표현식을 실행하고 그 결과를 `Result` 객체로 반환하는 역할을 합니다. 즉, `identity_map`을 우선적으로 확인하는 로직이 내장되어 있지 않습니다.

### Session.get()을 이용한 최적화된 조회

반복적인 `SELECT` 쿼리를 피하고 `identity_map`을 효과적으로 활용하기 위해서는 `Session.get()` 메서드를 사용해야 합니다.

```python
from sqlalchemy.orm import Session

with Session(engine) as sess:
    roach = sess.get(User, 1) # 기본 키 값 '1'로 User 객체 조회
    print(roach)

    roach = sess.get(User, 1) # 동일한 기본 키로 다시 조회
    print(roach)
```

이번에는 로그를 보면 첫 번째 `get()` 호출 시에만 `SELECT` 쿼리가 실행되고, 두 번째 호출 시에는 쿼리가 실행되지 않았음을 확인할 수 있습니다. `Session.get()`은 먼저 `identity_map`에서 해당 기본 키를 가진 객체를 찾고, 존재하면 데이터베이스 접근 없이 바로 반환합니다. 객체가 `identity_map`에 없거나 만료(expired)된 상태일 경우에만 `SELECT` 쿼리를 실행합니다.

SQLAlchemy 공식 문서 [`Session.get`](https://docs.sqlalchemy.org/en/20/orm/session_api.html#sqlalchemy.orm.Session.get)에서도 이 동작을 명확히 설명하고 있습니다.

> Session.get() is special in that it provides direct access to the identity map of the Session. If the given primary key identifier is present in the local identity map, the object is returned directly from this collection and no SQL is emitted, unless the object has been marked fully expired. If not present, a SELECT is performed in order to locate the object. Session.get() also will perform a check if the object is present in the identity map and marked as expired - a SELECT is emitted to refresh the object as well as to ensure that the row is still present. If not, ObjectDeletedError is raised.

따라서 ORM을 사용할 때 기본 키로 객체를 조회하는 경우에는 `Session.execute(select(...))` 대신 `Session.get(Entity, pk)`를 사용하는 것이 성능상 이점을 가집니다.

### 실제 시나리오에서의 중요성

현업 개발 환경에서는 `Session`의 시작 지점을 명확하게 통제하기 어려울 수 있습니다. 예를 들어, 어떤 함수가 외부에서 생성된 `Session` 객체를 전달받아 사용한다고 가정해 봅시다. 만약 이 함수 내부에서 `Session.execute()`를 사용하여 기본 키로 조회하는 코드가 있다면, 외부에서 이미 해당 객체를 로드했음에도 불구하고 불필요한 `SELECT` 쿼리가 발생할 수 있습니다.

```python
# Session.execute() 사용 시 (비효율적)
from sqlalchemy.orm import Session
from sqlalchemy import select

with Session(engine) as sess:
    # 외부에서 이미 로드되었다고 가정
    stmt = select(User).where(User.id == 1)
    result = sess.execute(stmt).scalar_one_or_none()

    print(sess.identity_map.all_states()[0].__dict__) # identity_map에 객체 존재 확인
    
    # 함수 내부에서 동일 객체 재조회
    with sess: # 동일 세션 사용
        stmt = select(User).where(User.id == 1)
        result = sess.execute(stmt).scalar_one_or_none() # SELECT 쿼리 또 실행됨
        print(result)
```

로그를 보면 `identity_map`에 객체가 있음에도 불구하고 내부 `with sess:` 블록에서 `Session.execute()`를 호출했을 때 다시 `SELECT` 쿼리가 실행됩니다. I/O 작업은 비용이 높은 작업이므로 이러한 중복은 피하는 것이 좋습니다.

반면, `Session.get()`을 사용했다면 어떻게 될까요?

```python
# Session.get() 사용 시 (효율적)
from sqlalchemy.orm import Session
from sqlalchemy import select # select는 여기서는 사용되지 않음

with Session(engine) as sess:
    # 외부에서 이미 로드되었다고 가정
    user = sess.get(User, 1) # SELECT 실행됨
    print(user)
    print(sess.identity_map.all_states()[0].__dict__) # identity_map 확인
    
    # 함수 내부에서 동일 객체 재조회
    with sess: # 동일 세션 사용
        user = sess.get(User, 1) # identity_map에서 바로 반환 (SELECT 실행 안됨)
        print(user)
```

로그를 통해 확인하면, 두 번째 `Session.get()` 호출 시에는 `SELECT` 쿼리가 발생하지 않고 `identity_map`에서 객체를 즉시 반환하는 것을 볼 수 있습니다. 이것이 `Session.get()` 사용을 권장하는 중요한 이유입니다.

### identity\_map의 한계: 캐시가 아니다?

`identity_map` 덕분에 중복 쿼리를 피할 수 있지만, 이를 모든 종류의 쿼리에 대한 범용 캐시(cache)로 오해해서는 안 됩니다. SQLAlchemy 공식 문서에서도 이 점을 지적합니다.

> Yeee…no. It’s somewhat used as a cache, in that it implements the identity map pattern, and stores objects keyed to their primary key. However, it doesn’t do any kind of query caching. This means, if you say session.query(Foo).filter\_by(name='bar'), even if Foo(name='bar') is right there, in the identity map, the session has no idea about that. It has to issue SQL to the database, get the rows back, and then when it sees the primary key in the row, then it can look in the local identity map and see that the object is already there. It’s only when you say query.get({some primary key}) that the Session doesn’t have to issue a query.

요약하자면, `identity_map`은 기본 키를 기반으로 객체를 저장하고 조회하는 **'아이덴티티 맵 패턴'**을 구현한 것이지, 쿼리 자체를 캐싱하는 기능은 아닙니다. `filter_by` 등 다른 조건으로 쿼리하면 `identity_map`에 해당 객체가 있더라도 SQL을 실행해야 합니다. 오직 `Session.get()`만이 `identity_map`을 직접 활용하여 SQL 실행을 건너뛸 수 있습니다.

또한, `identity_map`은 객체에 대한 약한 참조 ([weak reference](https://docs.python.org/ko/3.13/library/weakref.html))를 사용하여 객체를 관리하는 경우가 많습니다. 이는 파이썬의 가비지 컬렉션(Garbage Collection, GC)에 의해 세션 내에서 더 이상 강력하게 참조되지 않는 객체가 `identity_map`에서 제거될 수 있음을 의미합니다. 따라서 동일 세션 내라고 할지라도 특정 시점에는 `identity_map`에 객체가 존재하지 않을 수 있다는 점도 유념해야 합니다.

### 결론

SQLAlchemy `Session`은 데이터베이스와의 상호작용을 관리하는 핵심 요소입니다. 특히 객체를 조회할 때 `Session`의 동작 방식을 이해하는 것은 중요합니다.

* `Session.execute(select(...))`는 주어진 쿼리를 직접 실행하며, `identity_map`을 우선적으로 확인하지 않습니다.
    
* `Session.get(Entity, pk)`는 기본 키를 사용하여 객체를 조회할 때 `identity_map`을 먼저 확인하고, 객체가 존재하면 데이터베이스 접근 없이 반환하여 성능상 이점을 제공합니다.
    
* `identity_map`은 기본 키 기반 조회에 대한 캐시 역할을 하지만, 모든 쿼리에 대한 범용 캐시는 아니며 약한 참조로 인해 객체가 제거될 수도 있습니다.
    

따라서 효율적인 데이터베이스 상호작용을 위해, 특히 기본 키로 객체를 조회할 때는 `Session.get()` 메서드를 적극적으로 활용하는 것이 권장됩니다.