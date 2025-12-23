---
title: "크롤링 파이프라인 개선기 - 코드 구조화"
datePublished: Tue Dec 23 2025 05:44:59 GMT+0000 (Coordinated Universal Time)
cuid: cmji5us6z000002lbfajodij6
slug: 7ygs66gk66ebio2mjoydto2uhoudvoyducdqsjzshkdqulaglsdsvztrk5wg6rws7kgw7zmu

---

![generated_image_for_blog.png](https://storage.googleapis.com/roach-wiki/images/9b0b63df-e379-4690-b819-23bf85a29b70.webp align="left")

크롤링 기반 서비스를 만들다 보면 크롤링 결과를 그대로 운영에 넣을 수 있는 경우는 사실 거의 없다. 보통 실제 운영까지 가기 위해서는 아래와 같은 형태의 전처리 파이프라인을 무수히 많이 거치게 된다.

```plaintext
크롤링 → 데이터 정제(이미지 사이징, 중복 제거, ...) → 분류 → 기타 가공 → 검수 → 운영 배포
```

초기에는 이런 과정을 함수 호출로만 연결해도 큰 문제가 없었다. 하지만 서비스가 오래되고, 비즈니스 로직이 추가되고, 예외 케이스가 늘어나면서 어느 순간 코드가 점점 중간에서 끼어들고 비집고 들어오는 로직들이 생겨나다보면 읽기 힘들어지는 시점이 오게된다.

## 개선 동기

새롭게 작업하기 위해 코드를 보다보니 기존에 이 코드에 익숙한 사람이 아니라면, 코드 자체의 흐름을 파악하기 어렵다는 생각이 들었다. 그 이유는 코드에는 아래와 같은 몇가지 문제들이 존재했기 때문이다.

1. **코드의 실행 흐름을 함수를 하나하나 따라 읽어가며 파악해야 한다.**
    
2. **수십가지의 함수들의 입력/출력값을 단 한번에 보기 어렵다.**
    

위와 같은 문제는 익숙하지 않은 상태에서 코드를 이해할때 필요치 않은 코스트를 생성하고, 코드를 작성하는 시점에 실수가 일어나기 쉬운 상태라고 생각이 들었다.

그래서 유지보수를 위해 코드를 해체해서 구조적으로 작성하게 끔 만들지 않으면 유지보수 비용이 꾸준히 늘어날 것 이고, 최대한 작업자가 현재 비즈니스 로직에 집중하여 코드의 작성하게 끔 만드는 것이 중요했다.

따라서 모든 프로세스가 처리되는 구조와 실행 흐름을 한눈에 어떤 순서로 실행되는지 알 수 있게끔 만들수 있는 구조로 변경하기로 했다.

## Pipeline 설계 방향

핵심은 아래와 같이 매우 단순하게 만드는 것이다. 각 `Stage` 는 한가지의 책임만을 가질수 있도록 구조화하여 해당 부분에만 집중할 수 있게끔 만들고, `Pipeline` 은 이 연결된 단계들을 순차적으로 실행할 수 있게끔 한다.

1. Stage는 하나의 입력을 받아 하나의 출력을 만든다.
    
2. StageResult로 **성공/실패/부분성공을 일관되게 표현**한다.
    
3. Pipeline은 **Stage를 순차적으로 실행하면서 흐름을 제어**한다.
    

그래야 단계가 늘어나도 **“단순한 연결”** 처럼 읽히게 되고 파이프라인 도중에 `Stage`(새로운 비즈니스 로직) 을 추가하더라도 해당 부분에만 집중하고, 파이프라인에만 연결하면 되어 쉽다.

```python
TInput = TypeVar("TInput")
TOutput = TypeVar("TOutput")


# 모든 Stage가 공통으로 반환하는 표준 구조
class StageStatus(str, Enum):
    SUCCESS = "success"
    PARTIAL_SUCCESS = "partial"
    FAILURE = "failure"
    SKIPPED = "skipped"


# StageResult는 Stage가 반환하는 유일한 타입
@dataclass
class StageResult(Generic[TOutput]):
    status: StageStatus
    data: Optional[TOutput] = None
    errors: List[str] = field(default_factory=list)
    metrics: Dict[str, Any] = field(default_factory=dict)

    @property
    def is_successful(self) -> bool:
        return self.status in (StageStatus.SUCCESS, StageStatus.PARTIAL_SUCCESS)

    @property
    def should_continue(self) -> bool:
        return self.status in (
            StageStatus.SUCCESS,
            StageStatus.PARTIAL_SUCCESS,
            StageStatus.SKIPPED,
        )
```

`StageResult` 를 생성하고 각 함수에서 이를 반환하게끔 한 이유는 모든 Stage의 반환 형태가 같으니 예측 가능해지므로 Result 를 활용하여 `Stage` 의 결과를 통일된 구조로 확인해볼 수 있다. 또한 Stage 를 진행시키는 Pipeline은 should\_continue 하나만 보면 되므로 쉽다.

`metric` 은 모니터링을 위한 편의성 기능으로 각 Stage는 필요한 데이터만 metrics로 추가하면 된다. DB 기록, 모니터링, Slack 알림 등 확장에 유연하게 가져가기 위함이다.

```python
class PipelineStage(ABC, Generic[TInput, TOutput]):
    @property
    @abstractmethod
    def name(self) -> str:
        pass

    @abstractmethod
    async def execute(self, input_data: TInput) -> StageResult[TOutput]:
        pass

    async def on_success(self, result: StageResult[TOutput]) -> None:
        pass

    async def on_failure(self, result: StageResult[TOutput]) -> None:
        pass
```

`Pipeline` 에서 `Stage` 의 핵심은 다음 두 가지로 아래와 같다.

1. `execute` 는 입력을 받아서 출력만 만든다. **side-effect는 최소화하고, 필요하면 metrics에 기록하는 방식**이다. `execute` 를 작성하는 팀원은 이 `Stage` 에서 Input 으로 무엇을 할지에만 집중하면 된다.
    
2. `on_success / on_failure(옵션)` 와 같은 **Hook 형태의 함수**들로 각 `Stage` 가 성공하거나 실패했을때 알림을 발송하거나 특정 작업을 트리거하는 형태가 가능하다.
    

## Pipeline

Pipeline은 정말로 **“흐름만”** 관리하도록 만들었다. **Stage를 정의된 순서대로 하나하나씩 실행**시킨다.

1. 실패 시 즉시 중단
    
2. 부분 성공 또는 성공이면 다음 Stage로 진행
    
3. 마지막 Stage의 결과를 그대로 반환
    

Production 에는 멀티 프로세스 환경을 대비하기 위한 `Message Queue` 를 놓아 Input 과 Output 을 여러 `worker` 에서 실행될 수 있게 하는 부분과 실행 `metadata` 를 저장하는 DB 연결부 부분도 존재한다. 예제에서는 복잡할 수 있어 코드를 최대한 간소화하였다

```python
class Pipeline:
    def __init__(self, stages: List[PipelineStage]):
        if not stages:
            raise ValueError("Pipeline must have at least one stage")
        self.stages = stages

    async def run(self, initial_input: Any = None) -> StageResult:
        current_input = initial_input
        last_result: Optional[StageResult] = None

        for index, stage in enumerate(self.stages, 1):
            try:
                result = await stage.execute(current_input)

                # Stage Hook 호출
                if result.is_successful:
                    await stage.on_success(result)
                else:
                    await stage.on_failure(result)

                # 실패면 즉시 종료
                if not result.should_continue:
                    return result

                # 다음 Stage 에게 넘길 data
                current_input = result.data
                last_result = result

            except Exception as e:
                error_msg = f"Unexpected error in '{stage.name}': {e}"
                failure = StageResult(
                    status=StageStatus.FAILURE,
                    data=None,
                    errors=[error_msg],
                    metrics={"stage": stage.name},
                )
                await stage.on_failure(failure)
                return failure

        return last_result or StageResult(status=StageStatus.SUCCESS)
```

이 Pipeline은 매우 단순하게 정의된 `Stage` 를 처리할 수 있는 `Orchestrator` 이다. 단순하게 각 `Stage` 만을 순차적으로 처리해주는 역할을 진행한다.

## 간단한 실제 예제: 크롤링 → 이미지 정제 → 분류

이제 실제로 서비스에서 자주 쓰는 구조를 예제로 만들어보자.

1. 크롤링 Stage
    

```python
class CrawlStage(PipelineStage[None, List[dict]]):
    @property
    def name(self) -> str:
        return "crawl"

    async def execute(self, _):
        items = await crawl_products()
        return StageResult(status=StageStatus.SUCCESS, data=items)
```

2. 이미지 중복 제거 Stage
    

```python
class DeduplicateImageStage(PipelineStage[List[dict], List[dict]]):
    @property
    def name(self) -> str:
        return "dedupe_images"

    async def execute(self, items):
        cleaned, errors = [], []

        for item in items:
            try:
                item["images"] = remove_duplicates(item["images"])
                cleaned.append(item)
            except Exception as e:
                errors.append(str(e))

        status = StageStatus.PARTIAL_SUCCESS if errors else StageStatus.SUCCESS

        return StageResult(
            status=status,
            data=cleaned,
            errors=errors,
            metrics={"input": len(items), "output": len(cleaned)},
        )
```

3. 분류 Stage
    

```python
class ClassifyStage(PipelineStage[List[dict], List[dict]]):
    @property
    def name(self) -> str:
        return "classify"

    async def execute(self, items):
        classified = []
        for item in items:
            item["category"] = await classify(item)
            classified.append(item)
        return StageResult(status=StageStatus.SUCCESS, data=classified)
```

```python
pipeline = Pipeline(
    stages=[
        CrawlStage(),
        DeduplicateImageStage(),
        ClassifyStage(),
    ]
)

result = await pipeline.run()
```

이렇게 호출하면 읽는 사람으로 하여끔 전체 흐름이 아래처럼 읽힌다. `크롤링 → 중복제거 → 분류`. 보는 순간 어떻게 절차적으로 실행됨을 빠르게 알수 있으며, 내가 구현할 비즈니스 로직의 코드가 어디쯤 위치해야 하는지도 알기 쉽다.

결론적으로는 Stage마다 단일 책임 원칙이 지켜지고, Pipeline은 흐름을 제어할 뿐이며, 각 Stage 실행 결과는 StageResult로 표준화돼 있기 때문에 코드가 길어져도 읽기가 편하다.

## 모니터링

![image.png](https://storage.googleapis.com/roach-wiki/images/638f5f81-3c68-4d07-8d3f-ed0d1d2c1cbe.webp align="left")

구조를 바꾸며 얻게 된 결과 중 하나인데, 코드를 구조화 하여 결과물이 각 단계의 결과물을 저장하기 쉽게끔 변하여 각 단계에서 나오는 산출물을 통해 모니터링 대시보드를 구축하게 되었다. 각 단계에서의 결과를 확인하고, 해당 단계에서 추가적으로 모니터링 하고 싶은 부분들은 `StageResult` 에 넣기만 하면 자동으로 데이터베이스에 저장되고 모니터링 대시보드를 통해 출력되게 된다.

## 마치며

이번에 Pipeline 구조를 도입하면서 얻은 가장 큰 성과는 각 단계를 구조적으로 처리할 수 있게끔 만들었다는 점이다. 각 단계가 무엇을 책임지는지 명확해지고, 파이프라인 전체 흐름이 눈에 자연스럽게 들어오다 보니 새로운 비즈니스 로직을 추가하거나 기존 로직을 고치는 작업이 훨씬 편해졌다.

다만, 구현하며 살짝 아쉬운 포인트들도 있다. 개선하지 못한 부분을 정리해보면 아래와 같다.

**1) FastAPI DI 구조(FastAPI Depends)에 더 자연스럽게 녹아들도록 개선**

현재 구성은 우리 프로젝트 특성에 맞춰 살짝 바인딩되어 있어 DI 주입 방식이 Pipeline/Stage 바깥에 노출되는 경우가 몇 군데 있다. 이건 내부적으로는 큰 문제는 아니지만, Pipeline이 “독립적인 실행 단위"가 되려면 DI도 자연스럽게 숨겨져야 한다.

그래서 다음 단계에서는 Stage 내부에서 필요한 의존성들이 깔끔하게 캡슐화되는 구조로 리팩토링할 계획이다.

**2) Message Queue 추상화 레이어 분리**

프로덕션 환경에서 Pipeline은 여러 worker가 동시에 실행하므로 Message Queue 를 이용하게 되는데, 이 부분에서 자체 구현체인 QueueService 를 이용하다보니 `DI` 를 씀에도 쉽게 고치는것이 쉽지만은 않다. 따라서 이 부분을 Interface 로 사용하고, `DI` 를 통해 쉽게 끔 구현체를 갈아 낄 수 있게끔 구현해보려고 한다.

**3) DB Tracking / Execution Metadata 구조 독립화**

현재 Execution Metadata(DB에 저장되는 파이프라인 실행 기록)가 Pipeline 내부에서 직접 호출되는 형태로 되어 있다. 이건 편하긴 한데, 오픈소스 형태를 목표로 하면 **“DB를 쓰지 않는 환경에서도 쓸 수 있는 구조”** 로 만드는 게 맞을거 같다는 생각이 들었다.