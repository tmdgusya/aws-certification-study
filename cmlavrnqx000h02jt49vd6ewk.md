---
title: "Python 톺아보기 1화 - 토큰화(Tokenization)"
seoTitle: "Introduction to Python Tokenization"
seoDescription: "Python에서 토큰화를 이해하고 `tokenize` 모듈을 사용해 코드 구조를 분석하는 방법을 소개합니다"
datePublished: Fri Feb 06 2026 12:47:39 GMT+0000 (Coordinated Universal Time)
cuid: cmlavrnqx000h02jt49vd6ewk
slug: python-1-tokenization

---

Python 에서는 코드를 의미 있는 단위로 나누기 위한 **토큰화** 작업을 거친다. 이 작업을 거치면 코드는 토큰으로 분해된다. 오늘은 `tokenize` 모듈을 사용해서 이를 한번 눈으로 보고 확인해보도록 하자.

```python
import tokenize
import io

# 토큰 타입 이름 확인
print("주요 토큰 타입:")
print(f"  NAME: {tokenize.NAME} - 변수명, 함수명 등")
print(f"  NUMBER: {tokenize.NUMBER} - 숫자 리터럴")
print(f"  STRING: {tokenize.STRING} - 문자열 리터럴")
print(f"  OP: {tokenize.OP} - 연산자")
print(f"  NEWLINE: {tokenize.NEWLINE} - 줄바꿈")
print(f"  INDENT: {tokenize.INDENT} - 들여쓰기 시작")
print(f"  DEDENT: {tokenize.DEDENT} - 들여쓰기 종료")
print(f"  ENDMARKER: {tokenize.ENDMARKER} - 파일 끝")
```

이건 python 에서 주로 쓰이는 **토큰 타입**들이다. 이를 출력해보면 아래와 같이 출력된다.

```python
주요 토큰 타입:
  NAME: 1 - 변수명, 함수명 등
  NUMBER: 2 - 숫자 리터럴
  STRING: 3 - 문자열 리터럴
  OP: 55 - 연산자
  NEWLINE: 4 - 줄바꿈
  INDENT: 5 - 들여쓰기 시작
  DEDENT: 6 - 들여쓰기 종료
  ENDMARKER: 0 - 파일 끝
```

### 토큰화 함수

기본적으로 `tokenize.generate_tokens(readline)` 함수를 사용합니다.

* **readline**: 한 줄씩 읽어오는 함수
    
* **반환**: 토큰 네임드튜플 (type, string, start, end, line)
    

```python
# 간단한 코드 토큰화 예시
code = "x = 1 + 2"
print(f"코드: {code!r}")
print("\n토큰 목록:")
print("-" * 60)

tokens = tokenize.generate_tokens(io.StringIO(code).readline)
for tok in tokens:
    tok_name = tokenize.tok_name[tok.type]
    print(f"{tok.type:3} {tok_name:12} {tok.string!r:15} 위치: {tok.start}-{tok.end}")
```

```python
코드: 'x = 1 + 2'

토큰 목록:
------------------------------------------------------------
  1 NAME         'x'             위치: (1, 0)-(1, 1)
 55 OP           '='             위치: (1, 2)-(1, 3)
  2 NUMBER       '1'             위치: (1, 4)-(1, 5)
 55 OP           '+'             위치: (1, 6)-(1, 7)
  2 NUMBER       '2'             위치: (1, 8)-(1, 9)
  4 NEWLINE      ''              위치: (1, 9)-(1, 10)
  0 ENDMARKER    ''              위치: (2, 0)-(2, 0)
```

실제로 실행시켜보면 해당 토큰의 **타입과 이름과 위치정보 등등이 표기** 된다. 이를 통해 토큰이 파일내에서 어떤 위치에 있는지 등등을 판단할 수 있다.

### INDENT, DEDENT

```python
code2 = '''def greet(name):
    print(f"Hello, {name}!")
    return True
'''

print("=== 실습 2: 함수 정의 토큰화 ===")
print(f"\n원본 코드:")
print(code2)
print("=" * 70)
print(f"{'타입':<15} {'값':<20} {'줄':<5} {'열':<5}")
print("=" * 70)

tokens = list(tokenize.generate_tokens(io.StringIO(code2).readline))
for tok in tokens:
    tok_name = tokenize.tok_name[tok.type]
    line, col = tok.start
    value = tok.string[:18] + '...' if len(tok.string) > 20 else tok.string
    print(f"{tok_name:<15} {value!r:<20} {line:<5} {col:<5}")

print("\n주목할 점:")
print("- INDENT: 함수 본문의 들여쓰기가 시작됨을 표시")
print("- DEDENT: 들여쓰기가 종료됨을 표시 (return 문 이후)")
print("- f-string의 경우 FSTRING_START, FSTRING_MIDDLE, FSTRING_END로 분리됨")
```

```python
=== 실습 2: 함수 정의 토큰화 ===

원본 코드:
def greet(name):
    print(f"Hello, {name}!")
    return True

======================================================================
타입              값                    줄     열    
======================================================================
NAME            'def'                1     0    
NAME            'greet'              1     4    
OP              '('                  1     9    
NAME            'name'               1     10   
OP              ')'                  1     14   
OP              ':'                  1     15   
NEWLINE         '\n'                 1     16   
INDENT          '    '               2     0    
NAME            'print'              2     4    
OP              '('                  2     9    
FSTRING_START   'f"'                 2     10   
FSTRING_MIDDLE  'Hello, '            2     12   
OP              '{'                  2     19   
NAME            'name'               2     20   
OP              '}'                  2     24   
FSTRING_MIDDLE  '!'                  2     25   
FSTRING_END     '"'                  2     26   
OP              ')'                  2     27   
NEWLINE         '\n'                 2     28   
NAME            'return'             3     4    
NAME            'True'               3     11   
NEWLINE         '\n'                 3     15   
DEDENT          ''                   4     0    
ENDMARKER       ''                   4     0    

주목할 점:
- INDENT: 함수 본문의 들여쓰기가 시작됨을 표시
- DEDENT: 들여쓰기가 종료됨을 표시 (return 문 이후)
- f-string의 경우 FSTRING_START, FSTRING_MIDDLE, FSTRING_END로 분리됨
```

실제로 실행시켜보면 **“INDENT”** 와 **“DEDENT”** 등이 표기되는 것을 알 수 있다. `FSTRING_START` 등 신기한 토큰들도 많이보인다. Python 은 들여쓰기 수준이 증가하거나 감소할때 잘 알고 있듯이 `INDENT` 와 `DEDENT` 가 아래 처럼 발생한다.

```python
=== 중첩 함수의 INDENT/DEDENT ===
def outer():
    x = 1
    def inner():
        y = 2
        return y
    return x

============================================================
NAME: 'def'
NAME: 'outer'
OP: '('
OP: ')'
OP: ':'
INDENT → 레벨 1
  NAME: 'x'
  OP: '='
  NUMBER: '1'
  NAME: 'def'
  NAME: 'inner'
  OP: '('
  OP: ')'
  OP: ':'
  INDENT → 레벨 2
    NAME: 'y'
    OP: '='
    NUMBER: '2'
    NAME: 'return'
    NAME: 'y'
  DEDENT ← 레벨 2
  NAME: 'return'
  NAME: 'x'
DEDENT ← 레벨 1
```

### List comprehension 토큰화

```python
=== 연습 2: 리스트 컴프리헨션 토큰화 ===
코드: squares = [x**2 for x in range(10) if x % 2 == 0]

토큰 목록:
--------------------------------------------------
  NAME            'squares'
  OP              '='
  OP              '['
  NAME            'x'
  OP              '**'
  NUMBER          '2'
  NAME            'for'
  NAME            'x'
  NAME            'in'
  NAME            'range'
  OP              '('
  NUMBER          '10'
  OP              ')'
  NAME            'if'
  NAME            'x'
  OP              '%'
  NUMBER          '2'
  OP              '=='
  NUMBER          '0'
  OP              ']'
```

모든 코드가 토큰화 된다.

## 마치며

양질의 글은 아니지만 복리 효과를 믿으며 적어보는 글. 토큰화에 대한 개념을 알고 있으면 나중에 재밌는 것들을 해볼 수 있을 것 같다.