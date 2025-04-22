---
title: "파이썬 데이터 모델과 던더(Dunder) 메소드 이야기"
seoTitle: "파이썬 던더 메소드와 데이터 모델 이해하기"
seoDescription: "파이썬 데이터 모델과 던더 메소드의 활용을 이해하여 클래스 디자인을 향상시키는 방법을 소개합니다"
datePublished: Tue Apr 22 2025 14:34:58 GMT+0000 (Coordinated Universal Time)
cuid: cm9slwmtx000009jx0wz832v4
slug: dunder
tags: python, dunder-method, 7yym7j207i2s

---

오늘은 파이썬의 중요한 개념인 **데이터 모델(Data Model)**에 대해 정리해보겠습니다. "Fluent Python"이라는 책을 읽으면서 배운 내용인데, 개인적으로는 파이썬이 내부적으로 어떻게 동작하는지 이해하는 데 정말 도움이 되었습니다.

### 파이썬 데이터 모델? 그것이 무엇인가?

**파이썬 데이터 모델**은 쉽게 말해서 파이썬이라는 언어가 **'동작하는 규칙'** 같은 것입니다. 우리가 리스트 길이를 잴 때 `len(my_list)`를 쓰거나, 특정 항목에 접근할 때 `my_list[0]` 처럼 쓰는 것이 매우 상식적으로 여겨집니다. 이런 기본적인 동작들이 가능하게끔 하는 약속, 일종의 **프레임워크**가 바로 데이터 모델입니다.

이 데이터 모델의 핵심에는 **특별 메소드(Special Methods)**가 있습니다. 이름 앞뒤에 더블 언더스코어(`__`)가 붙어서 흔히 [**던더(dunder) 메소드**](https://docs.python.org/3/reference/lexical_analysis.html#reserved-classes-of-identifiers)라고 불리는 것들입니다. 예를 들면 `__len__`, `__getitem__` 같은 것들입니다. 우리가 직접 이 메소드들을 호출하는 경우는 거의 없지만, 파이썬 인터프리터는 우리가 `len()` 함수를 쓰거나 `[]` 문법을 사용할 때 내부적으로 이 던더 메소드들을 찾아서 호출합니다.

### 객체를 파이썬스럽게 만들기

말로만 하면 이해가 어려울 수 있으니, 카드 덱(Card Deck)을 만드는 예시를 한번 살펴보겠습니다.

```python
import collections

# 카드를 표현하는 간단한 방법: namedtuple 사용
Card = collections.namedtuple('Card', ['rank', 'suit'])

class FrenchDeck:
    ranks = [str(n) for n in range(2, 11)] + list('JQKA')
    suits = 'spades diamonds clubs hearts'.split()

    def __init__(self) -> None:
        # 카드 덱 초기화: 모든 카드 조합 만들기
        self._cards = [Card(rank, suit) for suit in self.suits for rank in self.ranks]

    def __len__(self):
        """
        덱에 있는 카드 개수를 반환 (len() 함수가 이것을 호출!)
        """
        return len(self._cards)

    def __getitem__(self, position):
        """
        주어진 위치(position)의 카드를 반환 (deck[i] 같은 인덱싱과 슬라이싱이 이것을 호출!)
        """
        return self._cards[position]

```

자, 이렇게 `FrenchDeck` 클래스를 만들었습니다. 특별한 것은 없으며 그냥 카드들을 리스트(`self._cards`)에 저장하고, `__len__`과 `__getitem__` 두 개의 던더 메소드를 정의했습니다.

```python
deck = FrenchDeck()

# 1. 길이 확인 가능! (내부적으로 deck.__len__() 호출)
print(f"카드 덱 길이: {len(deck)}")

# 2. 인덱스로 카드 접근 가능! (내부적으로 deck.__getitem__(0) 호출)
print(f"첫 번째 카드: {deck[0]}")

# 3. 슬라이싱도 가능! (내부적으로 deck.__getitem__(slice(0, 3, None)) 비슷한 것을 호출)
print(f"처음 3개 카드: {deck[:3]}")

# 4. 심지어 반복문도 동작한다! (__iter__가 없는데도 가능)
print("처음 3개 카드 (반복문):")
for card in deck[:3]:
    print(card)

# 5. 무작위 선택도 가능! (random.choice는 시퀀스를 기대하는데, 우리 덱이 시퀀스처럼 동작!)
from random import choice
print(f"무작위 카드: {choice(deck)}")
```

신기하게도 그냥 `__len__`과 `__getitem__` 두 개만 정의했을 뿐인데도, `FrenchDeck` 객체가 파이썬의 기본 리스트나 튜플처럼 길이도 잴 수 있고, 인덱싱/슬라이싱도 되고, 심지어 `for` 루프나 `random.choice` 같은 기능까지 그냥 쓸 수 있게 되었습니다.

이것이 바로 파이썬 데이터 모델의 힘입니다. **"어떤 객체가 특정 메소드를 가지고 있으면, 그 객체는 특정 프로토콜(규칙)을 지원하는 것으로 간주한다"** 이것이 파이썬의 중요한 철학 중 하나인 **덕 타이핑(Duck Typing)**과 연결되는 지점입니다. `__len__`과 `__getitem__`을 구현했더니 파이썬이 **"어, 너 시퀀스(sequence)처럼 행동할 수 있네?"** 하고 시퀀스 관련 기능을 쓸 수 있게 해주는 것입니다. 굳이 복잡한 상속 없이도 가능합니다.

### 반복의 미학: `__iter__` vs `__getitem__`

위에서 `__iter__` 메소드를 정의하지 않았는데도 `for` 루프가 동작했습니다. 그것은 파이썬이 `__iter__`가 없으면 차선책으로 `__getitem__`을 0부터 시작해서 1씩 증가시키면서 호출하기 때문입니다 (IndexError가 발생할 때까지).

그럼 만약 `__iter__`를 직접 정의하면 어떻게 될까요? 어떤 것이 우선순위를 가질까요?

```python
class FrenchDeck1:
    # ... (ranks, suits, __init__, __len__은 위와 동일) ...

    def __getitem__(self, position):
        print('called __getitem__')
        return self._cards[position]

    def __iter__(self):
        print('called __iter__') # 언제 호출되는지 확인용 로그
        # 간단한 예시를 위해 __init__에서 초기화했다고 가정.
        # Fluent Python 원서에서는 제너레이터로 구현하여 이 문제를 해결.
        # 여기서는 간단히 cards 를 순회하는 제너레이터를 반환한다고 가정
        for card in self._cards:
             yield card # yield를 쓰면 이 함수는 제너레이터가 됩니다!

# FrenchDeck1 인스턴스 생성 (간단하게 카드 10개만)
deck1 = FrenchDeck1()
# deck1._cards = deck1._cards[:10] # __init__ 에서 처리했다고 가정

print("\n--- FrenchDeck1 반복 테스트 ---")
for card in deck1: # deck1 전체를 반복
    pass # 카드 출력은 생략

print("\n--- FrenchDeck1 슬라이싱 후 반복 테스트 ---")
for card in deck1[:3]: # 슬라이싱 후 반복
    pass # 카드 출력은 생략
```

실행 결과를 보면 재밌는 것을 알 수 있습니다.

1. `for card in deck1:` 처럼 객체 전체를 직접 반복하면 `__iter__`가 **딱 한 번** 호출됩니다. 그리고 그 뒤로는 `__getitem__`은 전혀 호출되지 않습니다. 즉, `__iter__`가 있으면 파이썬은 반복을 위해 이것을 우선 사용한다는 것입니다. (`called __iter__`가 한 번만 출력되는 이유는, `yield`가 있는 함수는 호출 시 **제너레이터 객체를 반환**하고, `for` 루프가 **이 객체의** `__next__`**를 반복 호출**하기 때문입니다. `__iter__` 함수 자체가 계속 호출되는 것이 아니라는 점입니다!)
    
2. `for card in deck1[:3]:` 처럼 슬라이싱을 한 후에 반복하면, **이전처럼** `__getitem__`**이 여러 번 호출**됩니다. 이것은 슬라이싱(`deck1[:3]`) 자체가 `__getitem__`을 이용해서 처리되기 때문에, 반복 메커니즘 이전에 슬라이싱 과정에서 `__getitem__`이 호출되는 것입니다.
    

결론: **직접 반복** 시에는 `__iter__`가 우선입니다! 하지만 `__iter__`가 없거나, 슬라이싱 같은 다른 작업이 먼저 일어나면 `__getitem__`이 활용될 수 있습니다!

### 수치형 데이터 처럼 행동하기

이번엔 우리가 직접 만든 객체가 숫자처럼 덧셈, 곱셈 같은 연산을 할 수 있게 만들어 보겠습니다. 아까 잠깐 언급했던 `Vector` 클래스를 예시로 들어보겠습니다.

```python
import math
import doctest # doctest: 간단한 테스트를 주석에 넣을 수 있게 해줌

class Vector:
    """
    간단한 2차원 벡터 클래스

    # doctest 예시 시작 (아래 코드가 실제로 실행되고 결과가 맞는지 확인)
    >>> v1 = Vector(1, 2)
    >>> v2 = Vector(3, 4)
    >>> v1 + v2  # __add__ 테스트
    Vector(4, 6)

    >>> v = Vector(3, 4)
    >>> abs(v)  # __abs__ 테스트 (벡터 크기)
    5.0

    >>> v * 3  # __mul__ 테스트 (스칼라 곱)
    Vector(9, 12)

    >>> abs(v * 3)
    15.0
    # doctest 예시 끝
    """

    def __init__(self, x: int = 0, y: int = 0) -> None:
        self.x = x
        self.y = y

    def __repr__(self):
        """
        객체를 '개발자 친화적인' 문자열로 표현 (repr() 함수나 콘솔 출력이 이것을 사용)
        이것이 잘 되어 있으면 디버깅이 편리해집니다!
        >>> v = Vector(3, 4)
        >>> v
        Vector(3, 4)
        """
        # f-string을 사용하면 깔끔하게 표현 가능
        return f'Vector({self.x}, {self.y})'

    def __abs__(self):
        """
        벡터의 크기(magnitude)를 반환 (abs() 함수가 이것을 호출)
        math.hypot(x, y)는 sqrt(x*x + y*y)와 같음
        >>> v = Vector(3, 4)
        >>> abs(v)
        5.0
        """
        return math.hypot(self.x, self.y) # 유클리드 노름(norm)

    def __add__(self, other: "Vector"):
        """
        벡터 덧셈 (v1 + v2 연산이 이것을 호출)
        주의: 타입 힌트에서 클래스 자신을 참조할 때는 문자열로 감싸줌 ("Vector")
        >>> v1 = Vector(1, 2)
        >>> v2 = Vector(3, 4)
        >>> v1 + v2
        Vector(4, 6)
        """
        # 타입 체크를 추가하면 더 견고해짐 (여기선 생략)
        return Vector(self.x + other.x, self.y + other.y)

    def __mul__(self, scalar):
        """
        벡터와 스칼라(숫자)의 곱셈 (v * 3 연산이 이것을 호출)
        >>> v = Vector(3, 4)
        >>> v * 3
        Vector(9, 12)
        """
        # scalar가 숫자인지 확인하는 로직을 추가하면 좋음 (여기선 생략)
        return Vector(self.x * scalar, self.y * scalar)

# doctest 실행: 클래스나 함수 주석에 있는 테스트 코드를 실행
doctest.testmod()
```

위 `Vector` 클래스를 보면 `__repr__`, `__abs__`, `__add__`, `__mul__` 같은 **던더 메소드**를 정의했습니다.

* `__repr__`: `repr()` 함수나 인터프리터에서 **객체를 그냥 출력했을 때 보여줄 문자열을 정의**합니다. 디버깅할 때 객체 내용을 명확하게 보여줘서 아주 유용합니다. (`__str__`도 있는데, 이것은 `print()` 함수처럼 사용자에게 보여줄 때 사용하고, `__repr__`은 개발자를 위한 표현에 가깝습니다. 둘 다 정의할 수 있고, `__str__`이 없으면 `__repr__`이 대신 사용되기도 합니다.)
    
* `__abs__`: `abs()` 내장 함수가 `Vector` 객체에 사용될 수 있게 해줍니다. 여기서는 벡터의 크기(norm)를 계산하도록 했습니다.
    
* `__add__`, `__mul__`: 각각 `+` 와 `*` 연산자가 `Vector` 객체에 사용될 수 있게 해줍니다.
    

이렇게 던더 메소드를 정의하니까 우리가 만든 `Vector` 객체가 마치 파이썬의 기본 숫자 타입처럼 자연스럽게 연산이 가능해졌습니다!

(잠깐! `doctest` 것을 처음본 분들을 위한 설명: 주석 안에 `>>>` 로 예제 코드를 쓰고 바로 밑에 예상 결과를 적으면, `doctest.testmod()`가 이것을 진짜 실행해서 결과가 맞는지 테스트해줍니다. 복잡한 테스트는 `pytest` 같은 전문 도구를 쓰는 것이 좋지만, 간단한 예제나 문서에서는 정말 편리한 기능입니다.)

### 중위 연산자와 불변성(Immutability)

위 `Vector` 예시에서 `__add__`와 `__mul__`을 보면 중요한 특징이 하나 있습니다. 바로 연산 결과로 **새로운** `Vector` **객체를 생성해서 반환**한다는 것입니다. 기존의 `v1`이나 `v2` 객체를 직접 수정(inplace)하는 것이 아닙니다.

Python

```plaintext
v1 = Vector(1, 2)
v2 = Vector(3, 4)
v3 = v1 + v2 # v3는 Vector(4, 6) 이라는 '새로운' 객체

print(v1) # 출력: Vector(1, 2)  <- v1은 그대로!
print(v2) # 출력: Vector(3, 4)  <- v2도 그대로!
print(v3) # 출력: Vector(4, 6)
```

이것은 파이썬에서 일반적인 **연산자 오버로딩(operator overloading) 방식**입니다. `+`, `*` 같은 **중위 연산자(infix operator)** 들은 보통 **피연산자(operand)를 변경하지 않고 새로운 값을 반환**합니다. 우리가 `a = b + c` 라고 쓸 때 `b`나 `c`의 값이 바뀌길 기대하지 않는 것과 같습니다. 이런 방식은 코드를 예측 가능하게 만들고 의도치 않은 **부수 효과(side effect)**를 줄여줍니다. 리스트의 `sort()` 메소드처럼 객체 자체를 변경하는 **in-place** 연산과는 대비됩니다.

### 마치며

오늘은 "Fluent Python" 책을 길잡이 삼아 파이썬 데이터 모델과 던더 메소드의 세계를 살짝 엿보았습니다. 던더 메소드를 잘 활용하면 우리가 직접 만든 클래스도 파이썬 내장 타입처럼 아주 자연스럽고 '파이썬스럽게' 동작하게 만들 수 있다는 것을 알았습니다.

처음엔 `__init__` 말고는 던더 메소드를 쓸 일이 별로 없을 것으로 생각될 수 있지만, 내 객체가 `len()` 함수와 잘 동작하게 하고 싶거나, `+` 연산을 지원하게 하고 싶거나, `for` 루프에서 자연스럽게 사용되게 하고 싶을 때, 이 던더 메소드들을 떠올리면 됩니다. 파이썬의 강력함과 유연함이 바로 이런 데이터 모델 설계에서 나온다고 생각합니다.