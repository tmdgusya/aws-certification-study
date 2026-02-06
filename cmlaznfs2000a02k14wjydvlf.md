---
title: "파이썬 톺아보기 2화 - Ast 와 바이트코드"
seoTitle: "AST와 바이트코드: 파이썬의 이면 세계"
seoDescription: "Discover Python's AST, bytecode, and differences between expressions and statements to deepen your understanding of Python's execution process"
datePublished: Fri Feb 06 2026 14:36:21 GMT+0000 (Coordinated Universal Time)
cuid: cmlaznfs2000a02k14wjydvlf
slug: 2-ast
tags: python, pvm

---

## 식(Expression) 과 문장(Statement)

프로그래밍을 공부하다보면 위 두 단어를 반드시 마주하게 된다. 가끔 헷갈려하는 경우가 많은데 오늘은 python 에서 기본 모듈인 `ast` 모듈을 공부하며 이를 알아보도록 하자.

## 식(Expression)

기본적으로 **식(Expression)** 이란 평가되면 값이 나오는 코드 조각을 뜻한다. 파이썬에서는 어떠한 부분들이 있을까?

| 노드 타입 | 설명 | 예시 |
| --- | --- | --- |
| `BinOp` | 이항 연산 | `a + b`, `x * y` |
| `UnaryOp` | 단항 연산 | `-x`, `not flag` |
| `BoolOp` | 논리 연산 | `a and b`, `x or y` |
| `Compare` | 비교 연산 | `x > 0`, `a == b` |
| `Call` | 함수 호출 | `print("hi")` |
| `Name` | 변수 이름 | `x`, `foo` |
| `Constant` | 상수 | `42`, `"hello"` |
| `Attribute` | 속성 접근 | `obj.method` |
| `Subscript` | 첨자 접근 | `lst[0]`, `dict["key"]` |

바로 위와 같은 코드 조각들이 존재한다. 특징 들을 보면 `1 + 2` 를 실행시키면 바로 `3` 이라는 값이 나오듯. 코드 조각들이 평가되는 순간에 바로 \*\*값(valude)\*\*이 나오게 된다. 이를 한번 `ast` 모듈을 통하여 파싱해보자.

> ast 모듈의 parse 함수에는 mode 라는 값이 존재하는데, eval 로 하게 되면 단일 표현식만 파싱이 가능하다

```python
expressions = {
    'BinOp': '1 + 2',
    'UnaryOp': '-x',
    'BoolOp': 'a and b',
    'Compare': 'x > 0',
    'Call': 'print("hello")',
    'Name': 'x',
    'Constant': '42',
    'Attribute': 'obj.method',
    'Subscript': 'lst[0]',
}

for expr_type, code in expressions.items():
    print(f"\n{'='*40}")
    print(f"{expr_type}: {code}")
    print(f"{'='*40}")
    tree = ast.parse(code, mode='eval')  # 표현식 모드로 파싱
    print(ast.dump(tree, indent=2))
```

이를 파싱하면 아래와 같은 출력값이 나온다.

```python
========================================
BinOp: 1 + 2
========================================
Expression(
  body=BinOp(
    left=Constant(value=1),
    op=Add(),
    right=Constant(value=2)))

========================================
UnaryOp: -x
========================================
Expression(
  body=UnaryOp(
    op=USub(),
    operand=Name(id='x', ctx=Load())))

(생략...)
```

보면 전부 `Expression` 이라는 큰 그룹으로 묶여 있음을 알 수 있다. 즉, AST 가 이 코드 조각들을 식으로 인식하고 있음을 알 수 있다. 이제 대략적으로 식(Expression) 에 대한 감은 왔을 것이다. 그렇다면 문장은 또 어떤 것이 있을까? 한번 알아보도록 하자.

## 문장(Statement)

| 노드 타입 | 설명 | 예시 |
| --- | --- | --- |
| `FunctionDef` | 함수 정의 | `def foo(): ...` |
| `ClassDef` | 클래스 정의 | `class Foo: ...` |
| `If` | 조건문 | `if x > 0: ...` |
| `For` | for 루프 | `for i in range(10): ...` |
| `While` | while 루프 | `while x < 10: ...` |
| `Return` | 반환문 | `return x + 1` |
| `Assign` | 할당문 | `x = 1` |
| `AugAssign` | 복합 할당 | `x += 1` |
| `Import` | 임포트 | `import os` |
| `ImportFrom` | from 임포트 | `from os import path` |

**문장(Statement)** 는 위와 같이 “**무언가를 한다/흐름을 만든다**” 에 가까운 하나의 실행 단위이다. 뭐 분기 흐름을 만든다, 클래스를 정의한다 등등과 같은 무언가 특정 행위를 만들거나 정의하는 코드 조각의 모음이다. 이 코드 조각들 또한 `ast` 를 이용해서 `parsing` 하는 것이 가능하다.

```python
statements = {
    'FunctionDef': '''
def greet(name):
    return f"Hello, {name}!"
''',
    'If': '''
if x > 0:
    print("positive")
else:
    print("non-positive")
''',
    'For': '''
for i in range(5):
    print(i)
''',
    'Return': '''
return x + y
''',
}

for stmt_type, code in statements.items():
    print(f"\n{'='*50}")
    print(f"{stmt_type} 예제:")
    print(f"{'='*50}")
    tree = ast.parse(code)
    # 첫 번째 문장의 타입 확인
    first_stmt = tree.body[0]
    print(f"첫 번째 문장 타입: {type(first_stmt).__name__}")
    print(f"\nAST 구조:")
    print(ast.dump(first_stmt, indent=2))
```

```python
==================================================
FunctionDef 예제:
==================================================
첫 번째 문장 타입: FunctionDef

AST 구조:
FunctionDef(
  name='greet',
  args=arguments(
    args=[
      arg(arg='name')]),
  body=[
    Return(
      value=JoinedStr(
        values=[
          Constant(value='Hello, '),
          FormattedValue(
            value=Name(id='name', ctx=Load()),
            conversion=-1),
          Constant(value='!')]))])

==================================================
If 예제:
==================================================
첫 번째 문장 타입: If

AST 구조:
If(
  test=Compare(
    left=Name(id='x', ctx=Load()),
    ops=[
      Gt()],
    comparators=[
      Constant(value=0)]),
  body=[
    Expr(
      value=Call(
        func=Name(id='print', ctx=Load()),
        args=[
          Constant(value='positive')]))],
  orelse=[
    Expr(
      value=Call(
        func=Name(id='print', ctx=Load()),
        args=[
          Constant(value='non-positive')]))])

(생략 ...)
```

도중에 생략하긴 했는데 위와 같이 나오게 된다. `If` 와 같은 문장들은 식(Expression) 과 다르게 `Statement`로 감싸져 있지 않음을 확인할 수 있다. 이는 자리가 중요하기 때문이다. **문장 자리(stmt position)** 에서는 Expression 이 들어갈 수 없기 때문에 ast.Expr 로 감싸게 된다.

```python
def f():
    return 1 + 2  # ← Return(value=BinOp(...)) (BinOp를 Expr로 감싸지 않음)
```

하지만, 만약 문장 자리가 아닌 **표현식 자리(expr position)** 이라면 위와 같이 Expr 로 감싼 상태로 나오지 않게 된다.

## 바이트코드

이렇게 AST 로 해석되고 나면 어떻게 될까? 바로 컴파일 되게 된다. 파이썬도 Java 처럼 플랫폼 독립적이기 위해 이를 **파이썬 가상 머신(PVM)** 이 해석할 수 있는 구조인 바이트코드로 해석한다. 이를 코드로 확인해보기 위해서는 `dis` 모듈을 사용해보면 된다.

```python
def add(a, b):
    return a + b

print("=== dis.dis() 출력 ===")
dis.dis(add)
```

```python
=== dis.dis() 출력 ===
  2           RESUME                   0

  3           LOAD_FAST_LOAD_FAST      1 (a, b)
              BINARY_OP                0 (+)
              RETURN_VALUE
```

위와 같이 첫번째로 `2` 와 `3` 같은 소스코드의 줄 번호가 나오고, **RESUME, LOAD\_FAST, BINARY\_OP, RETURN\_VALUE** 와 같은 `opcode(명령어)` 그리고 `0`,`1`,`0` 과 같은 피연산자 인덱스가 나오게 된다. 위와 같이 `dis` 모듈을 통해 코드의 바이트 코드를 출력할 수 있다는 사실을 알 수 있다.

## 바이트 코드 예시

몇가지 바이트 코드를 한번 알아보도록 하자.

* `LOAD_CONST` : 상수를 스택에 푸시
    
* `BINARY_OP` : 이항 연산 수행
    
* `STORE_FAST`: 스택에서 값을 꺼내 지역변수에 저장
    

```python
def simple_math():
    x = 1 + 2
    return x

print("=== x = 1 + 2 의 바이트코드 ===")
dis.dis(simple_math)

print("\n=== 상수 테이블 ===")
print(f"co_consts: {simple_math.__code__.co_consts}")
```

이 코드를 실행하면 어떻게 될까? 일단 결과를 보기보다 예측해보자.

* **LOAD\_CONST** 1 (1) → 스택 = \[1\]
    
* **LOAD\_CONST** 2 (2) → 스택 = \[1, 2\]
    
* **BINARY\_OP** 0 (+) → 스택 = \[3\] (1과 2를 팝하고 3을 푸시)
    
* **STORE\_FAST** 0 (x) → 스택 = \[\] (3을 팝하여 x에 저장)
    
* **LOAD\_FAST** 0 (x) → 스택 = \[3\] (x의 값을 푸시)
    
* **RETURN\_VALUE** → 스택 = \[\] (3을 반환)
    

위와 같이 생각해 볼수 있다. 가장 첫번째로 `1` 과 `2` 를 스택에 넣어두고 BINARY\_OP 를 통해 Pop 해서 3을 밀어넣고 이 값을 지역변수에 저장하는 것들을 생각해볼 수 있다. 실제로 실행하면 어떨까?

```python
=== x = 1 + 2 의 바이트코드 ===
  2           RESUME                   0

  3           LOAD_CONST               1 (3)
              STORE_FAST               0 (x)

  4           LOAD_FAST                0 (x)
              RETURN_VALUE

=== 상수 테이블 ===
co_consts: (None, 3)
```

실제로 실행하게 되면 위와 같은 결과를 얻게 된다. 그 이유는 Cpython 의 상수 폴딩(constant folding) 때문인데 `1+2` 같이 사실상 컴파일시점에 값을 알 수 있는 **식(Expression)** 들은 `3` 하나만 상수테이블에 넣고 바이트 코드는 **LOAD\_CONST 3** 만 남기게 된다.

## Bytecode tracer

위와 같이 다른 바이트코드들도 많지만 굳이 다뤄야 할 정도로 유익하진 않다고 생각해서 `bytecode_tracer` 라는 `tool` 을 소개하고 이글을 마치려고 한다. 만약 스택 상태를 추적하고 싶다거나, 강의 목적으로 스택이 변화하는걸 보여주고 싶다면 아래와 같이 bytecode\_tracer 를 이용하면 쉽게 시각화 할 수 있다.

```python
import sys
sys.path.insert(0, '/home/roach/python-debug')

from tools.bytecode_tracer import trace_execution

# 간단한 함수 추적
def add(a, b):
    return a + b

print("=== 스택 상태 추적: add(1, 2) ===")
trace_execution(add, (1, 2))
```

```python
┏━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ Offset ┃ Opcode                ┃ Arg            ┃ Stack Before                  ┃ Stack After                   ┃
┡━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┩
│      0 │ RESUME                │                │ []                            │ []                            │
│      2 │ LOAD_FAST_LOAD_FAST   │ a, b           │ []                            │ [1, 2]                        │
│      4 │ BINARY_OP             │ +              │ [1, 2]                        │ [3]                           │
│      8 │ RETURN_VALUE          │                │ [3]                           │ []                            │
└────────┴───────────────────────┴────────────────┴───────────────────────────────┴───────────────────────────────┘
```

## CFG

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1770387961072/653b36c9-a8d0-4e64-87e6-046cc8007993.png align="center")

```python
from tools.cfg_visualizer import visualize_cfg

def test_loop(n):
    total = 0
    for i in range(n):
        total += i
    return total
    
print("=== for 루프의 CFG 생성 ===")
output_path = visualize_cfg(test_loop, 'outputs/cfg/test_loop.png')
print(f"CFG 저장됨: {output_path}")
```

`cfg` 라는 tool 을 설치하면 위와 같이 바이트 코드의 흐름도 또한 확인해볼 수 있다.

## 연습 문제

```python
def loop_with_range(n):
    total = 0
    for i in range(n):
        total += i
    return total

def loop_with_while(n):
    total = 0
    i = 0
    while i < n:
        total += i
        i += 1
    return total
```

n 회 기준으로 `for-loop` 와 `while` 루프가 위 처럼 코드가 존재할때 과연 바이트 코드가 같을까? 아니면 누가 더 빠를까? 한번 바이트 코드를 보면 아래와 같이 컴파일된다 (python 3.13 기준이다)

```python
=== for + range ===
  2           RESUME                   0

  3           LOAD_CONST               1 (0)
              STORE_FAST               1 (total)

  4           LOAD_GLOBAL              1 (range + NULL)
              LOAD_FAST                0 (n)
              CALL                     1
              GET_ITER
      L1:     FOR_ITER                 7 (to L2)
              STORE_FAST               2 (i)

  5           LOAD_FAST_LOAD_FAST     18 (total, i)
              BINARY_OP               13 (+=)
              STORE_FAST               1 (total)
              JUMP_BACKWARD            9 (to L1)

  4   L2:     END_FOR
              POP_TOP

  6           LOAD_FAST                1 (total)
              RETURN_VALUE

=== while ===
  8           RESUME                   0

  9           LOAD_CONST               1 (0)
              STORE_FAST               1 (total)

 10           LOAD_CONST               1 (0)
              STORE_FAST               2 (i)

 11           LOAD_FAST_LOAD_FAST     32 (i, n)
              COMPARE_OP              18 (bool(<))
              POP_JUMP_IF_FALSE       16 (to L2)

 12   L1:     LOAD_FAST_LOAD_FAST     18 (total, i)
              BINARY_OP               13 (+=)
              STORE_FAST               1 (total)

 13           LOAD_FAST                2 (i)
              LOAD_CONST               2 (1)
              BINARY_OP               13 (+=)
              STORE_FAST               2 (i)

 11           LOAD_FAST_LOAD_FAST     32 (i, n)
              COMPARE_OP              18 (bool(<))
              POP_JUMP_IF_FALSE        2 (to L2)
              JUMP_BACKWARD           16 (to L1)

 14   L2:     LOAD_FAST                1 (total)
              RETURN_VALUE
```

바이트 코드의 양만 봐도 알 수 있듯이 `while` 문에 조금 더 많은 바이트 코드가 존재한다. 그 이유는 **아래 연산이 매 반복의 분기마다 이뤄지기 때문**이다.

* **비교(COMPARE\_OP) + 분기(POP\_JUMP\_IF\_FALSE)**
    
* **증가를 위한(LOAD\_CONST/BINARY\_OP/STORE\_FAST)**
    

실제 어느정도 크지 않다면 비슷하겠지만 바이트 코드를 보게 된다면 위와 같이 미세한 차이들도 발견해볼 수 있다. 이러한 지식은 언젠가 알아두면 도움이 되니 파이썬을 사용하고 있다면 한번정도는 공부해보면 좋은 것 같다.