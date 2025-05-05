---
title: "SQLAlchemy ORM 매핑: 파이썬 클래스와 데이터베이스 테이블 연결하기"
seoTitle: "SQLAlchemy ORM: Map Python Classes to Tables"
seoDescription: "SQLAlchemy ORM을 통해 파이썬 클래스와 데이터베이스 테이블 간의 매핑을 설정하는 방법을 알아보세요. ORM 매핑 스타일과 Mapper 객체 활용법 소개"
datePublished: Mon May 05 2025 18:29:09 GMT+0000 (Coordinated Universal Time)
cuid: cmabezvkh000f09l74ghn4rqf
slug: sqlalchemy-orm
tags: orm, mapping, sqlalchemy

---

SQLAlchemy는 파이썬 애플리케이션에서 데이터베이스를 효과적으로 사용하기 위한 강력한 도구입니다. SQLAlchemy는 크게 **Object Relational Mapper (ORM)**와 **Core** 두 가지 구성 요소로 나뉩니다. Core는 SQL Expression Language와 데이터베이스 상호작용의 기반을 제공하며, ORM은 이 Core 기능을 활용하여 파이썬 객체와 데이터베이스 테이블 간의 매핑(mapping)을 정의하고 객체 지향적인 방식으로 데이터를 조작할 수 있도록 지원합니다.

이번 포스트에서는 SQLAlchemy ORM의 핵심 개념 중 하나인 매핑 방식에 대해 자세히 알아보겠습니다.

### ORM 매핑의 목적과 기본 구성 요소

SQLAlchemy ORM의 주된 목표는 개발자가 정의한 파이썬 클래스와 데이터베이스의 테이블 스키마를 연결하는 것입니다. 이를 통해 데이터베이스의 각 행(row)을 파이썬 객체 인스턴스로 표현하고, 객체 조작을 통해 데이터베이스와 상호작용할 수 있습니다. 즉, ORM 매핑은 파이썬 객체 모델과 관계형 데이터베이스 모델 사이의 다리 역할을 수행합니다.

매핑을 정의하기 위해서는 기본적으로 다음 두 가지 요소가 필요합니다.

1. **매핑될 파이썬 클래스:** 데이터베이스 테이블의 행을 나타낼 사용자 정의 클래스입니다.
    
2. **테이블(Table) 또는 기타 FROM 절 객체:** 매핑 대상이 되는 데이터베이스 테이블이나 서브쿼리 등 SELECT 문의 FROM 절에 올 수 있는 객체입니다.
    

이 두 요소를 연결하는 과정을 매핑이라고 하며, SQLAlchemy는 이를 위한 다양한 매핑 스타일을 제공합니다. 이 과정의 중심에는 클래스 속성과 테이블 컬럼 또는 관계(relationship)를 연결하는 `Mapper` 객체가 있습니다.

#### Metadata와 Table 객체

SQLAlchemy에서 데이터베이스의 스키마 구조는 `MetaData`와 `Table` 객체를 통해 파이썬 코드로 표현됩니다. `MetaData` 객체는 여러 `Table` 객체와 기타 스키마 구성 요소(인덱스, 제약 조건 등)를 담는 컨테이너 역할을 합니다. 예를 들어, 선언적 매핑에서 정의된 클래스들의 메타데이터 정보는 다음과 같이 접근할 수 있습니다.

```python
# Base 클래스 (declarative_base() 등으로 생성되었다고 가정)
>>> Base.metadata.tables
>>> FacadeDict({'user_account': Table('user_account', MetaData(), Column('id', Integer(), table=<user_account>, primary_key=True, nullable=False), Column('name', String(length=30), table=<user_account>, nullable=False), Column('fullname', String(), table=<user_account>), schema=None), 'address': Table('address', MetaData(), Column('id', Integer(), table=<address>, primary_key=True, nullable=False), Column('email_address', String(), table=<address>, nullable=False), Column('user_id', Integer(), ForeignKey('user_account.id'), table=<address>, nullable=False), schema=None)})
```

위 출력 결과에서 볼 수 있듯이, `MetaData` 객체는 `user_account`와 `address`라는 두 `Table` 객체 정보를 포함하고 있습니다. 이 `Table` 객체들은 매핑 과정에서 파이썬 클래스와 연결될 대상이 됩니다.

### 명령형 매핑 스타일 (Imperative Mapping Style)

SQLAlchemy ORM 매핑의 내부 동작, 특히 `Mapper` 객체의 역할을 이해하기 위해서는 명령형 매핑 스타일을 먼저 살펴보는 것이 도움이 될 수 있습니다. 이 방식은 매핑 과정을 명시적으로 제어합니다.

```python
from sqlalchemy import Table, Column, Integer, String, ForeignKey
from sqlalchemy.orm import registry, relationship

# 1. 매핑될 파이썬 클래스 정의
class User:
    pass

class Address:
    pass

# 2. Registry 및 Metadata 생성
mapper_registry = registry()

# 3. Table 객체 정의
user_table = Table(
    "user",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(30)),
    Column("fullname", String(30)),
)

address_table = Table(
    "address",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("user_id", Integer, ForeignKey("user.id")),
    Column("email_address", String(50)),
)

# 4. map_imperatively를 사용하여 클래스와 테이블 명시적 매핑
address_mapper = mapper_registry.map_imperatively(Address, address_table)
user_mapper = mapper_registry.map_imperatively(
    User,
    user_table,
    properties={
        # relationship 정의
        "addresses": relationship(Address, backref="user", order_by=address_table.c.id)
    },
)

# 매핑된 테이블 정보 확인
mapper_registry.metadata.tables
```

명령형 매핑은 선언적 매핑과 달리, 클래스 정의와 테이블 정의, 그리고 이 둘을 연결하는 과정을 분리하여 명시적으로 수행합니다. `Table` 객체를 먼저 정의한 후, `registry.map_imperatively()` 함수를 호출하여 어떤 클래스를 어떤 `Table` 객체에 매핑할지, 그리고 추가적인 속성(예: `relationship`)은 무엇인지 직접 지정합니다. 이는 마치 명령형 프로그래밍처럼 절차를 하나하나 명시하는 방식입니다. (이 개념이 생소하다면, 명령형 프로그래밍과 선언형 프로그래밍의 차이점을 다룬 자료를 참고하시는 것도 좋습니다.)

여기서 핵심적인 역할을 하는 것이 `map_imperatively` 함수 호출을 통해 생성되는 `Mapper` 객체입니다.

### Mapper 객체: 매핑 구성의 핵심

SQLAlchemy 공식 문서에 따르면, 모든 ORM 매핑 구성(선언적이든 명령형이든)은 최종적으로 `Mapper` 객체를 생성하고 구성하는 과정을 포함합니다 \`\`. 즉, `map_imperatively`에 전달된 클래스, 테이블 정보, `properties` 사전 등은 모두 `Mapper` 객체를 설정하는 데 사용됩니다.

생성된 `Mapper` 객체는 매핑과 관련된 다양한 정보를 담고 있습니다.

```python
# user_mapper의 속성 확인
>>> user_mapper.attrs.items()
>>> [('addresses', <_RelationshipDeclared at 0x7f3484e04f50; addresses>),
 ('id', <ColumnProperty at 0x7f34701f1740; id>),
 ('name', <ColumnProperty at 0x7f34701f1840; name>),
 ('fullname', <ColumnProperty at 0x7f34701f1940; fullname>)]

# user_mapper에 연결된 기본 테이블 확인
>>> user_mapper.local_table
>>> Table('user', MetaData(), Column('id', Integer(), table=<user>, primary_key=True, nullable=False), Column('name', String(length=30), table=<user>), Column('fullname', String(length=30), table=<user>), schema=None)
```

`Mapper` 객체는 주로 다음과 같은 네 가지 핵심 구성 요소를 관리합니다.

#### 1\. 매핑될 클래스 (The class to be mapped)

하나의 파이썬 클래스는 원칙적으로 단 하나의 **주 매퍼(primary mapper)** 와 연결됩니다. 즉, 동일한 클래스(`User`)에 대해 `map_imperatively` (또는 다른 매핑 방식)를 두 번 이상 호출하여 주 매퍼를 중복 정의하려고 하면 오류가 발생합니다.

```python
from sqlalchemy import Table, Column, Integer, String, ForeignKey
from sqlalchemy.orm import registry, relationship

class User: pass
class Address: pass

mapper_registry = registry()
user_table = Table(
    "user", mapper_registry.metadata,
    Column("id", Integer, primary_key=True), Column("name", String(30)), Column("fullname", String(30))
)
address_table = Table(
    "address", mapper_registry.metadata,
    Column("id", Integer, primary_key=True), Column("user_id", Integer, ForeignKey("user.id")), Column("email_address", String(50))
)

address_mapper = mapper_registry.map_imperatively(Address, address_table)
# 첫 번째 User 매퍼 정의 (성공)
user_mapper = mapper_registry.map_imperatively(
    User, user_table,
    properties={"addresses": relationship(Address, backref="user", order_by=address_table.c.id)}
)

# 두 번째 User 매퍼 정의 시도 (오류 발생)
try:
    duplicated_user_mapper = mapper_registry.map_imperatively(
        User, user_table,
        properties={"addresses": relationship(Address, backref="user", order_by=address_table.c.id)}
    )
except Exception as e:
    print(f"Error: {e}") # ArgumentError: Class '<class '__main__.User'>' already has a primary mapper defined.
```

위 코드 실행 시 `ArgumentError: Class '<class '__main__.User'>' already has a primary mapper defined.` 와 같은 오류 메시지를 볼 수 있습니다. 이는 `User` 클래스에 대한 주 매퍼가 이미 존재함을 의미합니다.

#### 2\. 테이블 또는 기타 FROM 절 객체 (The table, or other from clause)

`Mapper`는 반드시 데이터베이스 테이블(`Table` 객체)에만 연결될 필요는 없습니다. `SELECT` 문의 `FROM` 절에 올 수 있는 다른 객체, 예를 들어 서브쿼리(`Subquery` 객체) 결과에도 매핑될 수 있습니다. 이는 특정 조건을 만족하는 데이터만 조회하여 객체로 다루고 싶을 때 유용하게 사용될 수 있습니다.

```python
import sqlalchemy as sa
from sqlalchemy import Table, Column, Integer, String, MetaData, select
from sqlalchemy.orm import registry, Mapped, relationship

mapper_registry = registry()
metadata = mapper_registry.metadata

# 원본 테이블 정의
user_table = Table(
    "user_account", metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("priority", Integer), # 우선순위 컬럼
)

# 원본 테이블에 매핑될 클래스
class User:
    def __repr__(self):
        return f"User(id={self.id}, name={self.name}, priority={self.priority})"

# User 클래스를 user_account 테이블에 매핑
mapper_registry.map_imperatively(User, user_table)
print(f"User class mapped to: {User.__mapper__.local_table}")

# 특정 조건(priority > 5)을 만족하는 서브쿼리 생성
high_priority_select = select(user_table).where(user_table.c.priority > 5)
high_priority_subquery = high_priority_select.subquery("high_priority_users")
print(f"\nSubquery created:\n{high_priority_subquery}")

# 서브쿼리 결과에 매핑될 별도 클래스 정의
class HighPriorityUser:
    def __repr__(self):
        # 서브쿼리의 컬럼 이름을 사용하여 속성 접근
        return f"HighPriorityUser(id={self.id}, name={self.name}, priority={self.priority})"

# HighPriorityUser 클래스를 서브쿼리에 매핑
# local_table 인자에 Subquery 객체를 전달
mapper_registry.map_imperatively(class_=HighPriorityUser, local_table=high_priority_subquery)

# (데이터베이스 세션 및 데이터 삽입/조회 코드는 생략)
# 예시 결과:
# session.query(HighPriorityUser).all()
# [HighPriorityUser(id=2, name=Bob, priority=7), HighPriorityUser(id=3, name=Charlie, priority=8)]
```

위 예시처럼 `HighPriorityUser` 클래스는 `user_account` 테이블 자체가 아닌, `priority > 5` 조건을 가진 행들만 선택하는 서브쿼리(`high_priority_subquery`)에 매핑됩니다. 이렇게 매핑된 클래스를 조회하면 서브쿼리의 결과만을 객체 형태로 얻을 수 있습니다.

#### 3\. 속성 매핑을 위한 `properties` 사전

`registry.map_imperatively()` 함수의 `properties` 인자에 파이썬 사전을 전달하여 클래스 속성과 테이블 컬럼 또는 관계(relationship)를 명시적으로 매핑할 수 있습니다. 이 사전의 키는 클래스 속성 이름이 되고, 값은 `Column`, `relationship`, `column_property` 등 SQLAlchemy ORM이 제공하는 매핑 구성 요소가 됩니다.

```python
from sqlalchemy.orm import column_property # column_property 추가

# ... (User, Address 클래스 및 Table 객체 정의는 위와 동일) ...

# 명시적으로 매핑할 속성들을 사전으로 정의
user_properties = {
    "addresses": relationship(Address, backref="user", order_by=address_table.c.id),
    # fullname 컬럼을 full_name_attr 속성으로 매핑
    "full_name_attr": column_property(user_table.c.fullname),
    # 'id', 'name' 등 Table에 정의된 컬럼은 자동으로 매핑되므로 생략 가능
}
print(f"\nExplicit User Properties Dict: {user_properties.keys()}")

# properties 사전을 사용하여 매핑
mapper_registry.map_imperatively(
    User,
    user_table,
    properties=user_properties
)

# 최종 매퍼에 포함된 모든 속성 확인 (자동 매핑된 속성 포함)
print(f"User Mapper Properties (Includes explicit and implicit): {User.__mapper__.attrs.keys()}")
```

실행 결과는 다음과 같습니다.

```bash
Explicit User Properties Dict: dict_keys(['addresses', 'full_name_attr'])
User Mapper Properties (Includes explicit and implicit): ['addresses', 'id', 'name', 'full_name_attr']
```

`properties` 사전에 정의한 `addresses`와 `full_name_attr` 외에도, `user_table`에 정의된 `id`, `name` 컬럼이 자동으로 `User` 클래스의 동명 속성에 매핑되었음을 확인할 수 있습니다. `properties`는 이러한 자동 매핑 규칙을 재정의하거나 추가적인 매핑(예: `relationship`)을 정의할 때 사용됩니다. 더 자세한 내용은 `sqlalchemy.orm.registry.map_imperatively` 공식 문서를 참고하시기 바랍니다 \`\`.

#### 4\. 기타 Mapper 구성 파라미터

`Mapper` 객체는 위에서 설명한 요소 외에도 다양한 옵션을 통해 동작을 세밀하게 조정할 수 있습니다. 예를 들어, 선언적 매핑 스타일에서는 클래스 내부에 `__mapper_args__` 라는 특별한 속성을 사용하여 이러한 추가 옵션을 전달할 수 있습니다.

```python
from sqlalchemy.orm import declarative_base, mapped_column, Mapped
from sqlalchemy import String

Base = declarative_base()

# 복합 기본 키를 가진 테이블 예시
class GroupUsers(Base):
    __tablename__ = "group_users"

    # 컬럼 정의
    user_id: Mapped[str] = mapped_column(String(40), primary_key=True) # primary_key=True 사용
    group_id: Mapped[str] = mapped_column(String(40), primary_key=True) # primary_key=True 사용

    # __mapper_args__ 를 사용하여 복합 기본 키 명시 (대안적 방식)
    # mapped_column 에서 primary_key=True 를 사용하면 이 설정은 보통 불필요합니다.
    # __mapper_args__ = {"primary_key": [user_id, group_id]}
```

`__mapper_args__` 에는 복합 기본 키 지정 외에도 상속 제어, 버전 관리 설정 등 다양한 옵션을 지정할 수 있습니다. 어떤 옵션들이 가능한지는 SQLAlchemy 공식 문서의 [Mapper Configuration Options](https://www.google.com/search?q=https://docs.sqlalchemy.org/en/20/orm/declarative_config.html%23orm-declarative-mapper-options) \`\` 섹션에서 자세히 확인할 수 있습니다. 모든 옵션을 여기서 다루기는 어려우므로, 필요에 따라 해당 문서를 참조하여 학습하시는 것을 권장합니다.

### 선언적 매핑 스타일 (Declarative Mapping)

현대 SQLAlchemy 애플리케이션에서 가장 널리 사용되는 방식은 **선언적 매핑**입니다. 이 스타일은 파이썬 클래스 정의 안에 테이블 및 컬럼 매핑 정보를 함께 포함시켜 코드를 간결하고 직관적으로 만듭니다.

#### `DeclarativeBase` 사용

가장 기본적인 선언적 매핑은 `declarative_base()` (또는 최신 버전의 `DeclarativeBase`)를 사용하여 베이스 클래스를 만들고, 이 베이스 클래스를 상속받아 매핑할 클래스를 정의하는 방식입니다.

```python
from typing import List, Optional
from sqlalchemy import create_engine, Column, Integer, String, ForeignKey
from sqlalchemy.orm import declarative_base, relationship, Mapped, mapped_column, sessionmaker

# 1. 베이스 클래스 생성
Base = declarative_base()

# 2. 베이스 클래스를 상속받아 매핑 클래스 정의
class User(Base):
    # 테이블 이름 지정
    __tablename__ = "user_account"

    # 컬럼 및 타입 어노테이션을 사용한 매핑 (SQLAlchemy 1.4+ 스타일)
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(30))
    fullname: Mapped[Optional[str]] # 기본적으로 컬럼 이름은 속성 이름과 동일

    # 관계 정의
    addresses: Mapped[List["Address"]] = relationship(
        back_populates="user", cascade="all, delete-orphan"
    )

    def __repr__(self) -> str:
        return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"

class Address(Base):
    __tablename__ = "address"

    id: Mapped[int] = mapped_column(primary_key=True)
    email_address: Mapped[str]
    # 외래 키 및 관계 정의
    user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    user: Mapped["User"] = relationship(back_populates="addresses")

    def __repr__(self) -> str:
        return f"Address(id={self.id!r}, email_address={self.email_address!r})"

# (데이터베이스 연결 및 테이블 생성 코드)
# engine = create_engine("sqlite:///:memory:")
# Base.metadata.create_all(engine)
```

선언적 매핑에서는 클래스 속성 `__tablename__`을 사용하여 연결될 데이터베이스 테이블 이름을 명시합니다. 컬럼은 `mapped_column()` 함수와 파이썬 타입 어노테이션(`Mapped[...]`)을 함께 사용하여 정의하는 것이 권장되는 방식입니다. 관계는 `relationship()` 함수를 사용하여 설정합니다. SQLAlchemy는 이 클래스 정의를 분석하여 내부적으로 `Table` 객체와 `Mapper` 객체를 자동으로 생성하고 구성합니다.

#### 데코레이터(`@mapper_registry.mapped`) 사용

선언적 매핑의 또 다른 변형은 `registry` 객체와 `@mapper_registry.mapped` 데코레이터를 사용하는 방식입니다. 이 방식은 `DeclarativeBase`를 직접 상속하는 대신 데코레이터를 통해 클래스를 레지스트리에 등록합니다.

```python
from datetime import datetime
from typing import List, Optional
from sqlalchemy import ForeignKey, func, Integer, String
from sqlalchemy.orm import Mapped, mapped_column, registry, relationship

# 1. 레지스트리 생성
mapper_registry = registry()

# 2. @mapper_registry.mapped 데코레이터를 사용하여 클래스 정의 및 매핑
@mapper_registry.mapped
class User:
    # __tablename__은 여전히 필요
    __tablename__ = "user"

    # 컬럼 정의 (mapped_column 사용)
    id: Mapped[int] = mapped_column(Integer, primary_key=True) # 타입 명시도 가능
    name: Mapped[str]
    fullname: Mapped[Optional[str]]
    nickname: Mapped[Optional[str]] = mapped_column(String(64)) # 컬럼 타입 지정
    # 기본값 설정 (삽입 시)
    create_date: Mapped[datetime] = mapped_column(insert_default=func.now())

    # 관계 정의
    addresses: Mapped[List["Address"]] = relationship(back_populates="user")

    # __repr__ 등 필요한 메서드 추가 가능
    def __repr__(self) -> str:
        return f"<User(name={self.name})>"

@mapper_registry.mapped
class Address:
    __tablename__ = "address"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    # 외래 키 정의
    user_id: Mapped[int] = mapped_column(ForeignKey("user.id"))
    email_address: Mapped[str]

    # 관계 정의
    user: Mapped["User"] = relationship(back_populates="addresses")

    def __repr__(self) -> str:
        return f"<Address(email_address={self.email_address})>"

# (데이터베이스 연결 및 테이블 생성 코드)
# engine = create_engine("sqlite:///:memory:")
# mapper_registry.metadata.create_all(engine)
```

이 방식은 `DeclarativeBase` 상속 방식과 기능적으로 유사하지만, 메타데이터와 매핑 구성을 관리하는 `registry` 객체를 명시적으로 사용한다는 차이점이 있습니다.

### 결론

SQLAlchemy ORM은 파이썬 클래스와 데이터베이스 테이블 간의 매핑을 정의하는 다양한 방법을 제공합니다. 명령형 매핑은 매핑 과정을 명시적으로 제어하며 `Mapper` 객체의 역할을 이해하는 데 도움을 주지만, 실제 애플리케이션 개발에서는 클래스 정의와 매핑 정보를 함께 기술하는 선언적 매핑(Declarative Mapping) 방식이 더 널리 사용됩니다. 선언적 매핑은 `DeclarativeBase`를 상속하거나 `@registry.mapped` 데코레이터를 사용하여 구현할 수 있으며, 코드를 더 간결하고 유지보수하기 쉽게 만들어 줍니다.

어떤 방식을 사용하든 핵심은 파이썬 클래스와 데이터베이스 테이블(또는 서브쿼리 등) 간의 연결 정보를 `Mapper` 객체를 통해 설정하고 관리하는 것입니다. 이 매핑 구성을 통해 개발자는 객체 지향적인 방식으로 데이터베이스와 상호작용하며 생산성을 높일 수 있습니다.