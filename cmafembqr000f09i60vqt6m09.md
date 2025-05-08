---
title: "Pydantic을 활용한 파이썬 타입 정의 및 직렬화 배워보기"
seoTitle: "파이썬 Pydantic 타입 정의 및 직렬화"
seoDescription: "Pydantic을 사용한 파이썬 타입 정의, 데이터 유효성 검사 및 직렬화 방법을 배우는 가이드입니다"
datePublished: Thu May 08 2025 13:29:42 GMT+0000 (Coordinated Universal Time)
cuid: cmafembqr000f09i60vqt6m09
slug: pydantic
tags: python, types, pydantic

---

오늘은 파이썬에서 타입을 다루는 방법과 Pydantic 라이브러리를 활용한 데이터 유효성 검사 및 직렬화에 대해 알아보겠습니다.

### 파이썬 타입 시스템의 발전

파이썬은 3.5 버전부터 `typing` 모듈을 도입하여 복잡한 타입을 명시적으로 정의할 수 있는 기능을 제공하기 시작했습니다. 이를 통해 코드의 가독성과 유지보수성을 향상시킬 수 있게 되었습니다.

```python
class Fruit():
    name: str
    color: str
    weight: float
    bazam: dict[str, list[tuple[int, bool, float]]]

    def __init__(self, name: str, color: str, weight: float, bazam: dict[str, list[tuple[int, bool, float]]]):
        self.name = name
        self.color = color
        self.weight = weight
        self.bazam = bazam
```

위 코드와 같이 클래스 멤버 변수에 대한 타입을 선언할 수 있습니다. `name`은 문자열(`str`), `color`도 문자열(`str`), `weight`는 실수(`float`), 그리고 `bazam`은 키가 문자열이고 값이 (정수, 불리언, 실수) 튜플의 리스트인 딕셔너리 타입으로 정의되었습니다.

하지만 파이썬은 기본적으로 동적 타입 언어(dynamically-typed language)입니다. 이는 `typing` 모듈을 사용한 **타입 힌트(type hint)가 런타임(runtime) 시점에 타입 검사를 강제하지는 않는다는 것을 의미**합니다.

따라서 아래와 같이 타입 힌트와 다른 타입의 값을 전달해도 코드는 문제없이 실행됩니다.

```python
Fruit(name=1, color=1, weight=1, bazam="bazam")
```

실행 결과는 다음과 같습니다.

```python
<__main__.Fruit at 0x7f724dc6bb60>
```

이러한 유연성은 때로는 개발 과정에서 예기치 않은 오류를 발생시킬 수 있습니다. 동적언어의 장점이자 단점이라고 할 수 있죠. 바로 이러한 부분으로 인해 조금은 견고함이 필요한 부분에 Pydantic 라이브러리를 도입하면 좋습니다.

### Pydantic을 이용한 데이터 유효성 검사

Pydantic은 타입 힌트를 사용하여 데이터 유효성 검사 및 설정을 관리하는 라이브러리입니다. Pydantic을 사용하는 가장 일반적인 방법은 `BaseModel`을 상속하는 클래스를 정의하는 것입니다.

다음은 Pydantic을 사용하여 `Fruit` 클래스를 정의한 예제입니다.

```python
#!pip install pydantic
from typing import Annotated, Literal
from annotated_types import Gt # greater than x
from pydantic import BaseModel, Field

class Fruit(BaseModel):
    name: str
    color: Literal["red", "green"] # 색상은 "red" 또는 "green"만 허용
    weight: Annotated[float, Gt(0)] # 무게는 0보다 커야 함
    bazam: dict[str, list[tuple[int, bool, float]]]


Fruit(name="apple", color="red", weight=100, bazam={"a": [(1, True, 1.0)]})
```

위 코드에서 `color` 필드는 `Literal` 타입을 사용하여 "red" 또는 "green" 값만 허용하도록 제한했습니다. 또한 `weight` 필드는 `Annotated`와 `Gt(0)`를 사용하여 0보다 큰 값만 유효하도록 설정했습니다.

Pydantic을 사용하면 타입 힌트에 맞지 않는 데이터가 입력될 경우, 런타임에 `ValidationError`가 발생하여 잘못된 데이터를 사전에 방지할 수 있습니다.

예를 들어, 다음과 같이 의도적으로 잘못된 타입의 데이터를 `Fruit` 모델에 전달해 보겠습니다.

```python
Fruit(name=1, color=1, weight=1, bazam="bazam")
```

이 코드는 Pydantic 모델의 유효성 검사에 의해 다음과 같은 `ValidationError`를 발생시킵니다.

```python
ValidationError: 3 validation errors for Fruit
name
  Input should be a valid string [type=string_type, input_value=1, input_type=int]
    For further information visit https://errors.pydantic.dev/2.11/v/string_type
color
  Input should be 'red' or 'green' [type=literal_error, input_value=1, input_type=int]
    For further information visit https://errors.pydantic.dev/2.11/v/literal_error
bazam
  Input should be a valid dictionary [type=dict_type, input_value='bazam', input_type=str]
    For further information visit https://errors.pydantic.dev/2.11/v/dict_type
```

오류 메시지를 통해 `name`은 문자열이어야 하지만 정수형(int) `1`이 입력되었고, `color`는 "red" 또는 "green"이어야 하지만 정수형 `1`이 입력되었으며, `bazam`은 딕셔너리 타입이어야 하지만 문자열 "bazam"이 입력되었음을 명확히 알 수 있습니다.

만약 `color` 필드에 허용되지 않은 값(예: "yellow")을 입력하면 어떻게 될까요?

```python
Fruit(name="banana", color="yellow", weight=100, bazam={"a": [(1, True, 1.0)]})
```

이 경우에도 Pydantic은 다음과 같이 `ValidationError`를 발생시켜 데이터의 무결성을 보장합니다.

```bash
ValidationError: 1 validation error for Fruit
color
  Input should be 'red' or 'green' [type=literal_error, input_value='yellow', input_type=str]
    For further information visit https://errors.pydantic.dev/2.11/v/literal_error
```

`Pydantic`이 어떻게 동작하는지에 대한 더 자세한 내용은 [Pydantic 공식 문서](https://www.youtube.com/watch?v=pWZw7hYoRVU)에서 확인하실 수 있습니다.

### 직렬화 (Serialization)

직렬화(Serialization)는 객체를 바이트 스트림(byte stream)과 같이 전송하거나 저장하기 쉬운 형태로 변환하는 과정을 의미합니다. API를 개발할 때 프로그램 내부의 객체를 외부로 전달해야 하는 경우가 많은데, 이때 직렬화가 필수적입니다.

Pydantic은 객체 직렬화를 위한 세 가지 편리한 방법을 제공합니다:

1. **파이썬 객체로 이루어진** `dict`**로 변환**
    
2. **JSON으로 변환 가능한 타입으로만 이루어진** `dict`**로 변환**
    
3. **JSON 문자열로 변환**
    

각각의 방법에 대해 예제를 통해 자세히 살펴보겠습니다. 먼저, 사용자 정보를 나타내는 `User` 모델을 정의합니다.

```python
from pydantic import BaseModel
from datetime import datetime
from typing import Optional, List

class User(BaseModel):
    id: int
    username: str
    signup_ts: Optional[datetime] = None
    friends: List[int] = []
    is_active: bool = True

user_data = {
    "id": 123,
    "username": "pydantic_lover",
    "signup_ts": datetime(2023, 4, 15, 10, 30, 0),
    "friends": [1, 2, 3]
}
user_instance = User(**user_data)
```

#### 1\. 파이썬 객체로 이루어진 `dict`로 변환 (`model_dump()`)

`model_dump()` 메소드는 Pydantic 모델 인스턴스를 **파이썬 딕셔너리로 변환**합니다. 이 딕셔너리는 원본 객체의 필드와 값을 그대로 유지합니다.

```python
# 방법 1: 파이썬 객체로 이루어진 dict로 변환
python_dict_objects = user_instance.model_dump()

print(f"타입: {type(python_dict_objects)}")
print(f"내용: {python_dict_objects}")

if 'signup_ts' in python_dict_objects and python_dict_objects['signup_ts'] is not None:
    print(f"signup_ts의 타입: {type(python_dict_objects['signup_ts'])}")
else:
    print("signup_ts 필드가 없거나 None입니다.")
print("-" * 30)
```

실행 결과:

```bash
방법 1: model_dump() 결과 (Python 객체 유지)
타입: <class 'dict'>
내용: {'id': 123, 'username': 'pydantic_lover', 'signup_ts': datetime.datetime(2023, 4, 15, 10, 30), 'friends': [1, 2, 3], 'is_active': True}
signup_ts의 타입: <class 'datetime.datetime'>
------------------------------
```

결과에서 볼 수 있듯이, `python_dict_objects`는 `dict` 타입이며, `signup_ts` 필드는 여전히 파이썬의 `datetime.datetime` 객체로 유지됩니다.

#### 2\. JSON으로 변환 가능한 타입으로만 이루어진 `dict`로 변환 (`model_dump(mode='json')`)

`model_dump(mode='json')` 메소드는 모델 인스턴스를 딕셔너리로 변환하되, 모든 필드 값을 **JSON으로 직접 변환 가능한 타입(Jsonable)**으로 만듭니다. 예를 들어, `datetime` 객체는 ISO 8601 형식의 문자열로 변환됩니다.

```python
jsonable_dict = user_instance.model_dump(mode='json')

print(f"타입: {type(jsonable_dict)}")
print(f"내용: {jsonable_dict}")

if 'signup_ts' in jsonable_dict and jsonable_dict['signup_ts'] is not None:
    print(f"signup_ts의 타입: {type(jsonable_dict['signup_ts'])}\n")
else:
    print("signup_ts 필드가 없거나 None입니다.\n")
print("-" * 30)
```

실행 결과:

```bash
방법 2: model_dump(mode='json') 결과 (JSON 호환 타입)
타입: <class 'dict'>
내용: {'id': 123, 'username': 'pydantic_lover', 'signup_ts': '2023-04-15T10:30:00', 'friends': [1, 2, 3], 'is_active': True}
signup_ts의 타입: <class 'str'>
------------------------------
```

`jsonable_dict`의 `signup_ts` 필드 값이 `'2023-04-15T10:30:00'` 와 같이 문자열로 변환된 것을 확인할 수 있습니다. 이는 JSON이 파이썬의 `datetime` 객체를 직접 이해하지 못하기 때문입니다. 이렇게 변환된 딕셔너리는 `json.dumps()`를 사용하여 쉽게 JSON 문자열로 변환할 수 있습니다.

실제로 `json.dumps()`를 사용하여 변환해 보겠습니다.

```python
import json

json_string = json.dumps(jsonable_dict)
print(f"JSON 문자열: {json_string}")
print("-" * 30)
```

실행 결과:

```bash
JSON 문자열: {"id": 123, "username": "pydantic_lover", "signup_ts": "2023-04-15T10:30:00", "friends": [1, 2, 3], "is_active": true}
------------------------------
```

`is_active`의 값이 파이썬의 `True`에서 JSON의 `true`로 올바르게 변환된 것도 주목할 만합니다.

#### 3\. JSON 문자열로 변환 (`model_dump_json()`)

`model_dump_json()` 메소드는 Pydantic 모델 인스턴스를 직접 JSON 문자열로 변환합니다. 이는 `model_dump(mode='json')` 호출 후 `json.dumps()`를 실행하는 것과 유사한 결과를 제공하지만, 한 번의 호출로 처리할 수 있어 편리합니다.

```bash
json_string = user_instance.model_dump_json()

print(f"타입: {type(json_string)}")
print(f"내용: {json_string}")
print("-" * 30)
```

위 코드를 실행하면 `json_string` 변수에는 `User` 인스턴스가 직렬화된 JSON 문자열이 저장되며, 그 타입은 `str`이 됩니다. 내용은 이전 단계에서 `json.dumps(jsonable_dict)`를 실행한 결과와 동일할 것입니다.

### 결론

오늘은 파이썬의 타입 힌트 기능과 Pydantic 라이브러리를 활용하여 데이터의 유효성을 검사하고, 다양한 방식으로 객체를 직렬화하는 방법에 대해 알아보았습니다. Pydantic을 사용하면 타입 안정성을 높이고, 외부 데이터와의 상호작용을 더욱 견고하게 만들 수 있습니다.