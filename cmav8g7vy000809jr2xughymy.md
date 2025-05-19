---
title: "Python GIL 병목 현상 시각적으로 분석하기"
seoTitle: "Visualizing Python GIL Bottleneck"
seoDescription: "Python의 GIL을 통해 CPU-Bound 작업의 병목 현상을 이해하고 시각적으로 분석하는 방법을 알아봅니다. GIL의 영향과 해결책 소개"
datePublished: Mon May 19 2025 15:21:18 GMT+0000 (Coordinated Universal Time)
cuid: cmav8g7vy000809jr2xughymy
slug: python-gil
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1747668221854/b0e52261-e000-47e3-a9fe-8f948bda4ec1.png
tags: python, gil, global-interpreter-lock

---

들어가기에 앞서 블로그를 구독해주고 계신 분들 감사드립니다 :)

## 들어가며

오늘은 Python 의 [**GIL(Global Interpreter Lock)**](https://wiki.python.org/moin/GlobalInterpreterLock) 에 대해서 알아보고 왜 GIL 로 인해 **CPU-Bound** 작업에서 영향을 받을 수 있는지를 알아보고, 이걸 직접 코드로 작성하여 시각적으로 분석해보는 시간까지 가져보도록 하겠습니다. 아마도 장문의 글이 예상되니 꼼꼼히 읽으면서 따라오시길 바랍니다

## Python 이 실행되는 방식

우리가 보통 별다른 구현를 사용하지 않으면 파이썬에서는 기본적으로 **CPython** 을 이용하고 계실겁니다. 위키피디아에 따르면 [CPython](https://en.wikipedia.org/wiki/CPython) 은 Interpreter 와 Compiler 역할을 동시에 하며 Python code 를 **바이트코드** 형태로 바꾼뒤에 Interpreting 을 진행합니다.

```python
roach@roach:~$ python3
Python 3.12.3 (main, Feb  4 2025, 14:48:35) [GCC 13.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import dis
>>> def hello():
...   print("hello, dis!")
...
>>> dis.dis(hello)
  1           0 RESUME                   0

  2           2 LOAD_GLOBAL              1 (NULL + print)
             12 LOAD_CONST               1 ('hello, dis!')
             14 CALL                     1
             22 POP_TOP
             24 RETURN_CONST             0 (None)
```

CPython 에 의해 해석된 바이트 코드를 실제로 보기 위해서는 위와 같이 파이썬의 공식문서에 적혀있는대로 [**dis 모듈**](https://docs.python.org/ko/3.8/library/dis.html)을 이용하면 확인할 수 있습니다. 이제 한번 dis 모듈에서 나온 코드를 실행시켜보도록 하겠습니다.

```python
>>> dis.dis(hello.__code__)
  1           0 RESUME                   0

  2           2 LOAD_GLOBAL              1 (NULL + print)
             12 LOAD_CONST               1 ('hello, dis!')
             14 CALL                     1
             22 POP_TOP
             24 RETURN_CONST             0 (None)
>>> exec(hello.__code__)
hello, dis!
```

Python 의 [**exec**](https://docs.python.org/ko/3.13/library/functions.html#exec) 를 이용하면 code object 를 실행시킬 수 있는데요. 공식문서에 따르면 `__code__` 를 통해서 코드 객체를 얻을 수 있습니다. 이는 파이썬에서 함수가 정의되면 CPython 인터프리터가 해당 함수의 소스코드를 컴파일하여 **소스코드 객체(code object)** 를 생성하기 때문입니다.

일단 지금까지의 코드와 설명을 통해 우리의 Python 이 어떻게 실행되는지 간략하게 알아보았습니다. 이제 **GIL(Global Interpreter Lock)** 에 대해서 알아볼 것인데, GIL 은 파이썬의 **메모리 구조**와 밀접하게 관련 있으므로 메모리 구조에 대한 설명을 먼져 진행하면서 GIL 에 대한 설명을 이어나가겠습니다.

### 파이썬 메모리 모델

python 의 메모리 관리는 [공식문서](https://docs.python.org/3/c-api/memory.html)에 따르면 기본적으로 **private heap** 에서 이루어 집니다. private heap 에는 파이썬과 관련된 객체들이 올라가게 되는데요. 이 메모리를 관리하기 위해 **GC(Garbage Collecter) 및 레퍼런스 카운팅(Reference Counting)** 해당 객체들은 자신이 얼마나 참조되고 있는지를 나타내는 **참조 횟수(reference count)** 값을 가지고 있습니다. 코드와 함께보면 편하니 코드로 이해해보도록 하겠습니다.

```python
py_obj_data = Data(id = "obj-1", name="obj-1-name")

actual_referrer_global = py_obj_data

def inner(obj_param: Data):
    local_ref_in_inner = obj_param
    print(f"[INNER] py_obj_data를 직접 참조하는 변수들: {get_direct_referring_names(obj_param, locals(), globals())}, 참조 횟수: {sys.getrefcount(obj_param)}")


def inner2(obj_param: Data): 
    local_ref_in_inner2 = obj_param
    print(f"[INNER2] py_obj_data를 직접 참조하는 변수들: {get_direct_referring_names(obj_param, locals(), globals())}, 참조 횟수: {sys.getrefcount(obj_param)}")


gc.collect()
print(f"GC 실행 횟수: {gc.get_count()}")
print(f"객체(py_obj_data)가 GC에 의해 트래킹되고 있는지 여부: {gc.is_tracked(py_obj_data)}")
print(f"[GLOBAL] py_obj_data를 직접 참조하는 변수들: {get_direct_referring_names(py_obj_data, locals(), globals())}, 참조 횟수: {sys.getrefcount(py_obj_data)}")

referrer1_name_val = py_obj_data
print(f"[GLOBAL] py_obj_data를 직접 참조하는 변수들 (referrer1_name_val='{referrer1_name_val}' 할당 후): {get_direct_referring_names(py_obj_data, locals(), globals())}, 참조 횟수: {sys.getrefcount(py_obj_data)}")

# inner 함수 호출 (내부에서 local_ref_in_inner 및 obj_param이 py_obj_data를 참조)
inner(obj_param=py_obj_data)

# inner2 함수 호출 (내부에서 local_ref_in_inner2 및 obj_param이 py_obj_data를 참조)
inner2(obj_param=py_obj_data)

print(f"GC 실행 횟수: {gc.get_count()}")
# 모든 함수 호출 후 전역 상태에서 py_obj_data 참조 확인
print(f"[GLOBAL] py_obj_data를 직접 참조하는 변수들 (모든 함수 호출 후): {get_direct_referring_names(py_obj_data, locals(), globals())}, 참조 횟수: {sys.getrefcount(py_obj_data)}")
```

코드는 다음과 같습니다. `py_obj_data` 를 만들고 이 객체를 다른 변수에 할당했을때 **reference count** 가 어떻게 변하는지 살펴봅니다. 파이썬 기본 모듈인 `sys` 의 `getrefcount` 를 통해 참조 횟수를 확인가능합니다. 그리고 함수 내부에서도 참조를 하는 경우를 살펴보며 함수가 끝나는 경우 참조횟수가 어떻게 변하는지도 살펴봅니다

```python
GC 실행 횟수: (0, 0, 0)
객체(py_obj_data)가 GC에 의해 트래킹되고 있는지 여부: True
[GLOBAL] py_obj_data를 직접 참조하는 변수들: ['actual_referrer_global (global)', ...], 참조 횟수: 3
[GLOBAL] py_obj_data를 직접 참조하는 변수들 [...] 참조 횟수: 4
[INNER] py_obj_data를 직접 참조하는 변수들: [...], 참조 횟수: 6
[INNER2] py_obj_data를 직접 참조하는 변수들: [...], 참조 횟수: 6
GC 실행 횟수: (12, 0, 0)
[GLOBAL] py_obj_data를 직접 참조하는 변수들 (모든 함수 호출 후): [...], 참조 횟수: 4
```

실제 결과를 살펴보면 처음 참조횟수 3으로 시작해서 두번의 함수 호출을 통해 6까지 증가하고, 함수가 끝난뒤 4까지 **감소하는 모습**을 살펴볼수 있습니다. 위와 같이 파이썬은 객체가 어떻게 참조되는지 카운트를 측정하고 있으며 이를 볼수 있다는 것을 확인해보았습니다.

```c
typedef struct _object {
    _PyObject_HEAD_EXTRA
    Py_ssize_t ob_refcnt;
    struct _typeobject *ob_type;
} PyObject;
```

위와 같이 CPython 객체에 참조횟수를 넣어 놓는다. 객체의 참조횟수를 멀티 스레드에서 thread-safe 하게 연산하기 위해서는 **참조횟수(reference count)에 접근해야 할때 일종의 Mutex 에 Lock** 을 걸고 제한해야 합니다. 다만 그렇게 되면 모든 객체마다 Mutex Object 가 필요하게 될 것이고, 성능의 저하가 있을 수 있기 때문에 **Python Interpreter 자체에 Lock 을 걸어버리는 방식을 채택**했습니다. 즉, 동일 시간대에 하나의 Thread 만이 **GIL** 을 실행시킬 수 밖에 없는 것이죠.

## GIL(Global Interpreter Lock)

**GIL** 은 언뜻 괜찮아 보이나 무엇이 문제일까요? 바로 하나의 Thread 가 GIL 을 해제해야만 다른 Thread 가 GIL 을 획득할 수 있으므로 **CPU-Bound** 작업에 취약합니다. 이를 한번 시각적으로 보기 위해 코드와 함께 보도록 하겠습니다.

```python
# CPU-bound 작업을 시뮬레이션하는 함수
def work_cpu(label="", iterations=2):
    """CPU를 많이 사용하는 작업을 시뮬레이션합니다."""
    # 매우 긴 리스트를 생성하고 최소값을 찾는 작업
    min_val = min([random.random() * 100 for _ in range(iterations)])

# I/O-bound 작업을 시뮬레이션하는 함수
def work_io(label="", sleep_duration=0.5):
    """I/O 대기 작업을 시뮬레이션합니다. time.sleep()은 GIL을 해제합니다."""
    time.sleep(sleep_duration)
```

위의 함수는 **CPU-Bound** 와 **I/O Bound** 를 테스트 하기 위한 함수입니다. (**Python** 에서는 sleep 을 하게되면 GIL 이 해제되므로 I/O Bound 작업을 시뮬레이션 하기 위해 코드를 위와 같이 작성했습니다)

```python
def run_single_thread_sequential_cpu(num_tasks=2, iterations_per_task=20_000_000):
    """단일 스레드에서 CPU 바운드 작업을 순차적으로 실행합니다."""
    for i in range(num_tasks):
        work_cpu(label=f"SingleCPU-{i+1}", iterations=iterations_per_task)

def run_single_thread_sequential_io(num_tasks=4, sleep_per_task=0.5):
    """단일 스레드에서 I/O 바운드 작업을 순차적으로 실행합니다."""
    for i in range(num_tasks):
        work_io(label=f"SingleIO-{i+1}", sleep_duration=sleep_per_task)
```

이 코드를 단일 스레드에서 실행시킨다면 어떠한 결과를 얻을 수 있을까요?

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1747666419745/1b06aea4-2785-4858-8155-5393979e625f.png align="center")

(**\*작은 초록색 막대들이 랜덤 함수의 호출 프레임입니다**)

위와 같이 순차적으로 랜덤함수가 호출되며 하나의 스레드에서 실행되는 것을 확인할 수 있습니다. I/O 또한 아래그림과 같아 비슷하게 수행됩니다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1747666633970/6ef93eff-2ead-49c6-a8df-85d35fd53445.png align="center")

그렇다면 이 **CPU-Bound** 와 **I/O Bound** 를 각각 멀티스레드로 실행한다면 어떻게 될까요?

```python
def run_multi_threaded_cpu(num_threads=2, iterations_per_task=20_000_000):
    """다중 스레드에서 CPU 바운드 작업을 실행합니다."""
    threads = []
    for i in range(num_threads):
        thread = Thread(target=work_cpu, args=(f"MultiCPU-Thread-{i+1}", iterations_per_task))
        threads.append(thread)
        thread.start()
    for thread in threads:
        thread.join()

def run_multi_threaded_io(num_threads=4, sleep_per_task=0.5):
    """다중 스레드에서 I/O 바운드 작업을 실행합니다."""
    threads = []
    for i in range(num_threads):
        thread = Thread(target=work_io, args=(f"MultiIO-Thread-{i+1}", sleep_per_task))
        threads.append(thread)
        thread.start()
    for thread in threads:
        thread.join()
```

### Multi-thread I/O

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1747666809570/a22dfec7-82bd-4960-8e62-d3dfaeee2cf9.png align="center")

일단 **I/O** 작업은 중간에 **GIL** 이 해제되므로 코드를 `time.sleep` 이 **병렬적으로 같은 시간 프레임안에서 수행**되는 것을 확인할 수 있습니다. 그렇다면 **CPU-Bound** 작업은 어떨까요?

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1747666968487/c4543c23-4f71-4fff-9753-9ca8028cdab6.png align="center")

보시면 **초록색 막대(랜덤함수 실행)**이 **병렬적으로 이루어 지지 않고(동시적으로 이루어짐)**, Thread 들간 번갈아가며 작업을 수행하는 것을 확인할 수 있습니다. 즉, **GIL** 에 의해 영향을 받아 우리의 예상과는 다르게 병렬적으로 실행되지 않음을 확인할 수 있습니다.

### 부록) yield 와 같은 효과로 조금 더 병렬적으로 운용되게 할 수 있을까?

보통 이렇게 **CPU-Bound** 작업으로 인해 하나의 스레드가 길게 실행시간을 잡게 되는 경우를 방지하기 위해 `yield` 방식으로 다른 스레드에게 실행기회를 양보하도록 프로그래밍 하는 경우도 있는데요. 오늘은 아까 배운 `time.sleep` 을 중간중간 넣으면 어떻게 되는지를 살펴보도록 하겠습니다.

```python
def work_cpu(label="", iterations=2):
    """CPU를 많이 사용하는 작업을 시뮬레이션합니다."""
    yield_interval = iterations // 100  # 전체 작업을 100번으로 나눔
    
    for i in range(0, iterations, yield_interval):
        # yield_interval 크기만큼의 작업 수행
        chunk = [random.random() * 100 for _ in range(min(yield_interval, iterations - i))]
        min_val = min(chunk)
        # 주기적으로 yield하여 다른 스레드에 실행 기회 제공
        if i + yield_interval < iterations:
            time.sleep(0)  # yield 효과를 내기 위한 짧은 sleep
```

코드는 아주 간단한데요. 작업 중간중간마다 `time.sleep(0)` 을 넣어 GIL 을 해제하게끔 만듭니다. 이렇게 하면 우리가 예상한대로 **주기적으로 스레드가 번갈아가며 작업하게 될것**임이 예상됩니다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1747667693756/d7668b1f-df7c-4a76-b629-42f0bd2f1341.png align="center")

예상한대로 **GIL 을 풀며 주기적으로 스레드가 번갈아가면서 작업하는걸** 확인해볼수 있습니다.

## 마치며

보통 서버용으로 프레임워크를 작성하게 되면 대부분 **I/O Bound** 작업이 많기 때문에 테스트 한것 만큼의 성능저하를 겪기는 어렵습니다. 다만, 이러한 사유로 인해 성능이 저하될 수도 있다는 사실을 아는 것이 중요하기 때문에 작성한 코드를 샅샅이 살펴보며 오늘 배운 내용으로 원인 탐구를 해보셔도 좋을거 같습니다.