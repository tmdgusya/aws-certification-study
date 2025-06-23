---
title: "Python ContextVar"
seoTitle: "Understanding Python's ContextVar Feature"
seoDescription: "Use Python's ContextVar in async frameworks to maintain session context and manage changes with PEP 567 and asyncio create_task"
datePublished: Mon Jun 23 2025 16:35:23 GMT+0000 (Coordinated Universal Time)
cuid: cmc9bib83001y02ledhxlhmtp
slug: python-contextvar
tags: python, context, contextvar

---

## ContextVar 란?

FastAPI 와 같은 비동기 프레임워크를 사용하다보면 **하나의 세션안에서 동일한 컨텍스트**를 유지해야 하는 일들이 발생한다. 기존 멀티 스레드기반에서 주로 사용되는 **TLS(Thread Local Storage)** 기반의 방식을 비동기에 적용하게 되면 Task 가 다른 스레드에서 실행되어 Context 가 예기치 않게 다른 스레드에 노출될수도 있다.

파이썬 [**PEP567**](https://peps.python.org/pep-0567/) 에서는 이러한 문제점을 해결하기 위해 **ContextVar** 라는 방식을 제안하였다. 제안서에 따르면 **ContextVar** 에 `get` 과 `set` 메소드를 이용하여 값을 수정또는 읽기가 가능하다고 합니다. 백문이 불여일견이라고 코드로 한번 보도록하겠습니다

```python
import contextvars

from uuid import uuid4

ctx = contextvars.ContextVar("test_context", default=uuid4())
```

위와 같이 컨텍스트 객체를 생성할수 있습니다. ContextVar 의 첫번째 인자는 `name` 으로 **주로 debug 의 목적**으로 이용됩니다. 이제 이렇게 생성한 컨텍스트 객체가 **async 함수들** 안에서 잘 동작하는지 확인해보도록 하겠습니다.

```python
async def nested_context():
    print(f"Nested context value: {ctx.get()}")
    
async def nested_context2():
    print(f"Nested context2 value: {ctx.get()}")

async def test_context():
    # Get the current context value
    current_value = ctx.get()
    
    if current_value is None:
        ctx.set(uuid4())
        current_value = ctx.get()

    print(f"Current context value: {current_value}")
    
    await nested_context()
    await nested_context2()

await test_context()
```

코드는 아주 심플합니다. main async 함수인 `test_context` 에서는 ctx 에 값이 없다면 새롭게 값을 생성하여 넣습니다. nested 함수들은 `test_context` 에서 생성된 값과 동일한 값을 가지는지 확인하기 위해 로깅을 통해 ctx 값을 출력합니다.

```python
Current context value: c1df51e7-4524-4a6a-bc1a-c91500b33d1e
Nested context value: c1df51e7-4524-4a6a-bc1a-c91500b33d1e
Nested context2 value: c1df51e7-4524-4a6a-bc1a-c91500b33d1e
```

결과는 하나의 async 세션안에서 동일한 ctx 값이 이용되는 것을 확인할 수 있습니다.

### 값이 도중에 바뀌는 경우

만약 두번째 nested function 에서 ctx 의 값을 바꾼다면 어떻게 될까요?

```python
async def nested_context():
    ctx.set(uuid4()) # 값 변경 일어남
    print(f"Nested context value: {ctx.get()}")
    
async def nested_context2():
    print(f"Nested context2 value: {ctx.get()}")

async def test_context():
    # Get the current context value
    current_value = ctx.get()
    
    if current_value is None:
        ctx.set(uuid4())
        current_value = ctx.get()

    print(f"Current context value: {current_value}")
    
    await nested_context()
    await nested_context2()

await test_context()
```

```python
Current context value: da4f260c-337d-4e4b-901d-ed091c0c4474
Nested context value: b0459d83-cb3a-4bb9-a579-ce99d6125cf8
Nested context2 value: b0459d83-cb3a-4bb9-a579-ce99d6125cf8
```

결과를 보면 두번째 함수 이후에 값이 변경된것을 확인할 수 있습니다. 그렇다면 만약 도중에 Context 값을 바꾼채로 실행하고 싶다면 어떻게 해야할까요? 예를들어 같은 `test_context` 안에서 실행하지만 하나는 다른 context 에서 실행하고 싶을 경우에 말입니다.

```python
import contextvars
from uuid import uuid4

ctx = contextvars.ContextVar('test_context')

def nested_context():
    print(f"Inside nested_context, before set: {ctx.get()}")
    ctx.set(uuid4())
    print(f"Inside nested_context, after set: {ctx.get()}")
    
def nested_context2():
    print(f"nested_context2 value: {ctx.get()}")

def test_context():
    initial_value = uuid4()
    ctx.set(initial_value)
    print(f"Initial context value: {initial_value}")
    
    copy_ctx = contextvars.copy_context()
    
    print(f"Value in copied context: {copy_ctx[ctx]}")
    
    copy_ctx.run(nested_context)
    
    print(f"Context value after copy.run(): {ctx.get()}")
    
    nested_context2()
```

일단 동기적인 코드로 먼져 살펴보면, `test_context` 에서 먼져 context 를 복사한 뒤에 복사한 context 를 통해서 nested\_context 를 실행시키는 것을 확인할 수 있습니다. 이렇게 시작된 nested\_context 는 내부에서 context 값을 새롭게 uuid 함수를 실행시켜 바꿉니다.

```python
Initial context value: bdf1fbd7-d454-43fa-b3d9-b37d428f38b6
Value in copied context: bdf1fbd7-d454-43fa-b3d9-b37d428f38b6
Inside nested_context, before set: bdf1fbd7-d454-43fa-b3d9-b37d428f38b6
Inside nested_context, after set: 7157e2f4-57bc-430a-9a21-d55236649a60 # nested 안에서만 바뀜!!
Context value after copy.run(): bdf1fbd7-d454-43fa-b3d9-b37d428f38b6
nested_context2 value: bdf1fbd7-d454-43fa-b3d9-b37d428f38b6
```

예상대로 `nested` 안에서만 바뀌는 것을 확인할 수 있습니다. 즉 하나의 ctx 안이지만, 내부에서는 다른 상태를 가지게끔 할수 있는 것이죠. 그렇다면 이를 비동기로 전환만 해서 실행해볼까요?

```python
async def nested_context():
    print(f"Copy context value: {ctx.get(ctx)}")
    ctx.set(uuid4())
    print(f"Nested context value: {ctx.get()}")
    
async def nested_context2():
    print(f"Nested context2 value: {ctx.get()}")

async def test_context():
    # Get the current context value
    current_value = ctx.get()
    
    if current_value is None:
        ctx.set(uuid4())
        current_value = ctx.get()

    print(f"Current context value: {current_value}")
    
    original_ctx = contextvars.copy_context()
    copy_ctx = contextvars.copy_context()
    
    
    print(f"Original context value: {original_ctx.get(ctx)}")
    await copy_ctx.run(nested_context)
    
    print(f"Context value after copy: {ctx.get()}")
    
    await original_ctx.run(nested_context2)

await test_context()
```

코드는 동일하고 async 로 붙여 실행하는 함수입니다. test\_context 를 실행해보면 아까와는 다르게 **첫번째 중첩함수에서 바꾼 컨텍스트 값이 다른 컨텍스트에도 영향을 주는 것을 확인할 수 있습니다**.

```python
Current context value: bdf1fbd7-d454-43fa-b3d9-b37d428f38b6
Original context value: bdf1fbd7-d454-43fa-b3d9-b37d428f38b6
Copy context value: bdf1fbd7-d454-43fa-b3d9-b37d428f38b6
Nested context value: ffce1062-060e-4c03-aa2f-3ccf8eedaeaa
Context value after copy: ffce1062-060e-4c03-aa2f-3ccf8eedaeaa
Nested context2 value: ffce1062-060e-4c03-aa2f-3ccf8eedaeaa 
```

아마 파이썬을 많이 다루시는 분들은 눈치 채셨겠지만 가장 큰 이유는 `async def` 함수는 기본적으로 coroutine 을 반환하게끔 되어있습니다.

```python
coro = copy_ctx.run(nested_context)

coro: <coroutine object nested_context at 0x7f1c2ebda670>
```

즉 `await copy_ctx.run(nested_context)` 를 실행해도 `copy_ctx.run(nested_context)` 의 결과가 coroutine 이기 때문에 실행되는 구역은 결국 test\_context 함수안에서 실행되는 것입니다. 그렇기 때문에 원본 콘텍스트 값이 바뀔수 밖에 없는것이죠. 그렇다면 이를 어떻게 해결해야 할까요?

### Asyncio.create\_task

```python
def create_task(coro, *, name=None, context=None):
    """Schedule the execution of a coroutine object in a spawn task.

    Return a Task object.
    """
    loop = events.get_running_loop()
    if context is None:
        # Use legacy API if context is not needed
        task = loop.create_task(coro, name=name)
    else:
        task = loop.create_task(coro, name=name, context=context)

    return task
```

Asyncio 의 **create\_task** 를 이용하면 됩니다. 기본적으로 asyncio 는 context 를 받기때문에 `copy_context` 함수를 통해 OS 스레드가 복사해준 context 를 넘겨주기만 하면 우리가 예상한대로 실행됩니다.

```python
import asyncio

async def nested_context():
    print(f"Copy context value: {ctx.get(ctx)}")
    ctx.set(uuid4())
    print(f"Nested context value: {ctx.get()}")
    
async def nested_context2():
    print(f"Nested context2 value: {ctx.get()}")

async def test_context():
    # Get the current context value
    current_value = ctx.get()
    
    if current_value is None:
        ctx.set(uuid4())
        current_value = ctx.get()

    print(f"Current context value: {current_value}")
    
    original_ctx = contextvars.copy_context()
    copy_ctx = contextvars.copy_context()
    
    
    print(f"Original context value: {original_ctx.get(ctx)}")
    await asyncio.create_task(nested_context(), context=copy_ctx)
    print(f"Context value after copy: {ctx.get()}")
    
    await nested_context2()

await test_context()
```

중간에 `copy_ctx.run` 부분만 `create_task` 로 바뀐것을 확인할 수 있습니다. 이렇게 실행하게 되면 새롭게 실행되는 중첩함수는 복사된 context 에서 실행되게 되어 우리가 원하는 결과값을 아래와 같이 얻을 수 있습니다.

```python
Current context value: bdf1fbd7-d454-43fa-b3d9-b37d428f38b6
Original context value: bdf1fbd7-d454-43fa-b3d9-b37d428f38b6
Copy context value: bdf1fbd7-d454-43fa-b3d9-b37d428f38b6
Nested context value: 5d7a1ed1-316b-41ee-b289-ccd08976d869 # Nested 에서만 바뀐것 확인 가능
Context value after copy: bdf1fbd7-d454-43fa-b3d9-b37d428f38b6
Nested context2 value: bdf1fbd7-d454-43fa-b3d9-b37d428f38b6
```