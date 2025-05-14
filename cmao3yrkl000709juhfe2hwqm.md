---
title: "파인튜닝 hands-on (개발자를 위한 실습 위주의 아티클)"
seoTitle: "파인튜닝 실습: 개발자 가이드"
seoDescription: "파인튜닝과 Sentence Transformer를 사용한 문장 임베딩 실습 가이드, Triplet Loss를 통한 모델 성능 향상 방법 소개"
datePublished: Wed May 14 2025 15:41:22 GMT+0000 (Coordinated Universal Time)
cuid: cmao3yrkl000709juhfe2hwqm
slug: hands-on
tags: handson, finetuning, 7yym7j247yqc64ud

---

요새 파인튜닝에 관심이 많은 개발자들이 많은데 실제로 자료는 넘치나 어떻게 시작해야할지 모르시는 분들이 조금은 있는 거 같아 실습 위주로 일단 해보는 아티클을 작성해보았습니다. 가장 쉽게할수 있는 방법인 파이썬과 [Sentence Transformer](https://sbert.net/index.html) 를 활용하여 문장 임베딩 모델을 특정 작업에 맞게 **미세조정(fine-tuning)**하는 방법에 대해 자세히 알아보겠습니다. 특히 문장 간의 "유사도"를 모델이 더 잘 이해하도록 학습시키는 과정을 핸즈온 실습과 함께 살펴보겠습니다.

### Sentence Transformer 와 문장 임베딩이란 무엇일까요?

우선 **Sentence Transformer**가 무엇인지 짚고 넘어가겠습니다. Sentence Transformer는 텍스트나 이미지 같은 입력 정보를 **고정된 크기의 벡터(vector)로 변환하는 라이브러리**입니다. 이 과정을 **임베딩(embedding)**이라고 부릅니다. 이렇게 생성된 벡터는 문장의 의미론적인 정보를 담고 있으며, 벡터 공간에서 서로 가까운 벡터들은 의미적으로 유사한 문장을 나타냅니다.

예를 들어, 다음 세 문장을 Sentence Transformer를 이용해 임베딩해 보겠습니다.

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("all-MiniLM-L6-v2")

sentences = [
    "The weather is lovely today.",
    "It's so sunny outside!",
    "He drove to the stadium"
]

embeddings = model.encode(sentences)
```

이렇게 얻어진 `embeddings`의 형태(shape)를 확인해 보면, 각 문장이 384차원의 벡터로 변환된 것을 알 수 있습니다.

```python
embeddings.shape
```

```python
(3, 384)
```

이 임베딩 벡터들을 이용하면 문장 간 유사도를 측정할 수 있습니다. `model.similarity()` 함수는 두 임베딩 벡터 간의 유사도 점수를 계산합니다.

```python
model.similarity(embeddings, embeddings)
```

```python
tensor([[1.0000, 0.6660, 0.1058],
        [0.6660, 1.0000, 0.1471],
        [0.1058, 0.1471, 1.0000]])
```

결과를 보면, "The weather is lovely today."와 "It's so sunny outside!"는 0.6660으로 비교적 높은 유사도를 보이지만, "He drove to the stadium"과는 각각 0.1058, 0.1471로 낮은 유사도를 나타냅니다. 이처럼 임베딩은 문장의 의미론적 유사성을 파악하는 데 효과적입니다.

### 유사도 계산

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1747236939083/e786ee76-9687-4dee-ae50-2408fb22e390.png align="center")

Sentence Transformer는 다양한 **유사도 계산 방식**을 지원합니다. 유사도 계산 방식은 어렸을때 사분면에서 두점사이의 거리를 구한 공식을 생각하셔도 좋습니다. 기본값은 코사인 유사도(Cosine Similarity)이지만, 필요에 따라 내적(Dot Product), 유클리드 거리(Euclidean Distance), 맨해튼 거리(Manhattan Distance) 등을 사용할 수 있습니다.

```python
from sentence_transformers import SentenceTransformer, SimilarityFunction

model = SentenceTransformer("all-MiniLM-L6-v2", similarity_fn_name=SimilarityFunction.COSINE)
```

### "유사함"을 모델에게 가르치기: 미세조정(Fine-tuning)의 필요성

기본적으로 제공되는 사전 학습된 Sentence Transformer 모델도 훌륭하지만, 특정 도메인이나 작업에 대해서는 기대만큼의 성능을 내지 못할 수 있습니다. 예를 들어, '패션', '경제', '스포츠' 관련 문장들을 분류하거나 유사한 문장을 찾는 모델을 만든다고 가정해 봅시다.

모델이 **"이 스웨터는 최신 유행이야"**라는 문장과 **"파리 패션위크가 곧 열려"**라는 문장이 서로 '비슷하다'고 판단하고, "주식 시장이 하락했어"라는 문장과는 '다르다'고 판단하도록 만들려면 어떻게 해야 할까요? 바로 **미세조정(fine-tuning)**을 통해 모델에게 이러한 '비슷함'과 '다름'의 기준을 가르쳐야 합니다.

미세조정의 핵심은 우리가 원하는 목적에 맞는 데이터셋을 구성하고, 적절한 **손실 함수(Loss Function)**를 선택하는 것입니다. 손실 함수는 모델이 예측한 결과와 실제 정답 간의 차이를 계산하여, 모델이 얼마나 "잘못"하고 있는지를 알려주는 역할을 합니다. 모델은 이 손실 값을 최소화하는 방향으로 학습하면서 성능이 개선됩니다.

### Triplet Loss: 관계 기반 학습으로 유사도 정교화하기

문장 유사도 학습에 효과적인 손실 함수 중 하나는 **Triplet Loss**입니다. Triplet Loss는 세 개의 데이터 샘플, 즉 '세 쌍둥이'를 이용해 학습합니다.

* **Anchor (앵커)**: 기준이 되는 문장입니다. (예: "이 스웨터는 최신 유행이야" \[패션\])
    
* **Positive (포지티브)**: 앵커와 의미적으로 유사한 문장입니다. (예: "파리 패션위크가 곧 열려" \[패션\])
    
* **Negative (네거티브)**: 앵커와 의미적으로 다른 문장입니다. (예: "주식 시장이 하락했어" \[경제\])
    

Triplet Loss의 기본 아이디어는 다음과 같습니다: **"앵커와 포지티브 사이의 거리"는 "앵커와 네거티브 사이의 거리"보다 작아야 한다.**

더 나아가, 단순히 작은 것을 넘어 최소한 특정 **margin(여유 공간)**만큼 더 작아야 한다는 조건을 추가합니다. 즉, `거리(앵커, 포지티브) + margin < 거리(앵커, 네거티브)`가 되도록 학습합니다. 만약 이 조건이 만족되면 손실(loss)은 0에 가까워지고, 만족되지 않으면 모델에게 벌점(loss)을 부여하여 앵커와 포지티브는 더 가깝게, 앵커와 네거티브는 더 멀게 임베딩하도록 유도합니다.

### BatchAllTripletLoss를 이용한 실습

이제 `BatchAllTripletLoss`를 사용하여 실제로 모델을 **미세조정**해 보겠습니다. `BatchAllTripletLoss`는 배치 내의 모든 가능한 triplet을 고려하여 손실을 계산합니다. 공식 문서에 따르면, Triplet Loss를 사용하기 때문에 최소 3개 이상의 클래스(라벨)가 필요합니다.

#### 1\. 데이터셋 준비

먼저 '패션', '경제', '스포츠' 세 가지 카테고리에 해당하는 문장 데이터셋을 준비합니다. 각 문장에는 해당 카테고리를 나타내는 라벨(0: 패션, 1: 경제, 2: 스포츠)을 부여합니다.

Python

```python
label_map = {
    0: 'fashion',
    1: 'economy',
    2: 'sport',
}

# 예시 데이터 (실제로는 더 많은 데이터가 필요합니다)
existing_sentences = [
    # Fashion (패션)
    "This new collection features vibrant colors and bold patterns.",
    "The fashion show in Paris showcased the latest trends for spring/summer.",
    # ... (더 많은 패션 문장)

    # Economy (경제)
    "The central bank announced an increase in interest rates.",
    "Stock market volatility has been high in recent weeks.",
    # ... (더 많은 경제 문장)

    # Sport (스포츠)
    "The home team secured a dramatic victory in the final minutes.",
    "She won the gold medal in the 100-meter dash.",
    # ... (더 많은 스포츠 문장)
]

existing_labels = [
    0, 0, # 패션 라벨
    1, 1, # 경제 라벨
    2, 2  # 스포츠 라벨
]

# 데이터를 늘리기 위해 추가 문장 및 라벨 생성 (Jupyter Notebook의 전체 코드 참조)
# ... (new_fashion_sentences, new_economy_sentences, new_sport_sentences 등)

final_sentences = existing_sentences + new_fashion_sentences + new_economy_sentences + new_sport_sentences
final_labels = existing_labels + new_fashion_labels + new_economy_labels + new_sport_labels

# 각 클래스별 데이터 개수 확인
fashion_count = final_labels.count(0)
economy_count = final_labels.count(1)
sport_count = final_labels.count(2)

print(f"Total sentences: {len(final_sentences)}")
print(f"Total labels: {len(final_labels)}")
print(f"Fashion sentences: {fashion_count}")
print(f"Economy sentences: {economy_count}")
print(f"Sport sentences: {sport_count}")
```

글에는 생략되었지만 총 305개의 문장이 있으며, 패션 103개, 경제 101개, 스포츠 101개로 구성되어 있습니다.

```python
Total sentences: 305
Total labels: 305
Fashion sentences: 103
Economy sentences: 101
Sport sentences: 101
```

#### 2\. 미세조정 전 유사도 확인

**학습 전**, 동일 카테고리 내 문장들의 유사도를 확인해 보겠습니다. 예를 들어, 패션 카테고리의 처음 10개 문장과 그다음 10개 문장 간의 유사도를 측정해 봅니다.

```python
from sentence_transformers import SentenceTransformer
model = SentenceTransformer("all-MiniLM-L6-v2") # 사전 학습된 모델 로드

fashion_sentences_group1 = final_sentences[:10] # 패션 문장 그룹 1 (처음 10개)
fashion_sentences_group2 = final_sentences[10:20] # 패션 문장 그룹 2 (다음 10개)

embeddings1 = model.encode(fashion_sentences_group1)
embeddings2 = model.encode(fashion_sentences_group2)

similarity_scores_before_tuning = model.similarity(embeddings1, embeddings2)
print(similarity_scores_before_tuning)
```

출력된 텐서 값을 보면, 학습 전 모델은 같은 패션 카테고리 내 문장들임에도 불구하고 **유사도 점수가 0.2~0.4 (최대치가 1.0 인걸 고려하면)** 정도로 그리 높지 않게 나오는 경우가 많습니다. 이는 모델이 아직 특정 도메인의 미묘한 의미 차이를 충분히 학습하지 못했기 때문입니다.

```python
tensor([[0.2667, 0.2121, 0.2471, 0.2434, 0.2809, 0.1967, 0.2159, 0.2609, 0.1829,
         0.3568],
        [0.3857, 0.4224, 0.2886, 0.3052, 0.4230, 0.2564, 0.2260, 0.4032, 0.3134,
         0.3338],
        # ... (이하 생략)
       ])
```

#### 3\. 모델 학습

이제 `BatchAllTripletLoss`를 사용하여 모델을 학습시킵니다. `datasets` 라이브러리를 사용하여 학습 데이터를 준비하고, `SentenceTransformerTrainer`를 통해 학습을 진행합니다.

```python
from sentence_transformers import losses, SentenceTransformerTrainer
from datasets import Dataset

# 학습 데이터셋 생성
train_dataset = Dataset.from_dict({
    "sentence": final_sentences,
    "label": final_labels
})

# 손실 함수 정의
loss = losses.BatchAllTripletLoss(model)

# 트레이너 설정
trainer = SentenceTransformerTrainer(
    model=model,
    train_dataset=train_dataset,
    loss=loss,
)

# 학습 시작
trainer.train()
```

학습이 진행되면서 손실 값이 점차 줄어드는 것을 확인할 수 있습니다. (실제 학습 시에는 epoch, batch size 등 하이퍼파라미터 조정이 필요할 수 있습니다.)

#### 4\. 미세조정 후 유사도 확인

학습이 완료된 모델을 사용하여 다시 한번 패션 카테고리 내 문장들의 유사도를 측정해 보겠습니다.

```python
embeddings1_after_tuning = model.encode(fashion_sentences_group1)
embeddings2_after_tuning = model.encode(fashion_sentences_group2)

similarity_scores_after_tuning = model.similarity(embeddings1_after_tuning, embeddings2_after_tuning)
print(similarity_scores_after_tuning)
```

놀랍게도, 미세조정 후에는 동일 카테고리 내 문장들 간의 유사도 점수가 대부분 0.9 이상으로 매우 높게 나타나는 것을 확인할 수 있습니다! 다만, 이 예시에는 테스트에 사용한 문장을 평가에서도 사용하기 때문에 높은 경향을 보입니다. 실제로 평가할때는 하면 안되지만, 이번 예시는 hands-on 인 만큼 그냥 테스트에 사용한 부분을 이용하도록 하겠습니다.

```python
tensor([[0.9869, 0.9859, 0.9908, 0.9708, 0.9898, 0.9675, 0.9813, 0.9875, 0.9889,
         0.9878],
        [0.9959, 0.9961, 0.9909, 0.9859, 0.9970, 0.9633, 0.9880, 0.9965, 0.9945,
         0.9927],
        # ... (이하 생략)
       ])
```

이는 미세조정을 통해 모델이 '패션'이라는 특정 주제와 관련된 문장들의 의미적 유사성을 훨씬 더 잘 파악하게 되었음을 의미합니다. 즉, 같은 카테고리의 문장들은 임베딩 공간에서 서로 가까이 모이게 되고, 다른 카테고리의 문장들은 멀리 떨어지도록 학습된 것입니다.

### 미세조정된 모델의 활용 방안

이렇게 미세조정된 모델은 다양한 다운스트림 작업(downstream task)의 성능을 향상시키는 데 기여할 수 있습니다. 실제로 저는 최근 쇼핑몰의 검색을 벡터 검색으로만 진행하기 위해 여러 모델을 파인튜닝을 시도해보고 있습니다.

* **유사 문장 검색**: "이 패션 기사와 비슷한 다른 기사 찾아줘!"와 같은 요청에 대해 더욱 정확한 결과를 제공할 수 있습니다.
    
* **분류**: 학습된 임베딩 위에 간단한 분류기를 추가하면, '패션', '경제', '스포츠'와 같은 카테고리 분류 성능이 향상될 수 있습니다.
    
* **군집화**: 라벨이 없는 데이터에 대해서도 문장들을 의미에 따라 그룹으로 묶는 작업(군집화)에 유리합니다.
    

### 결론

오늘은 Sentence Transformer를 사용하여 문장 임베딩 모델을 특정 작업에 맞게 미세조정하는 과정을 살펴보았습니다. 특히 Triplet Loss와 같은 손실 함수를 활용하여 모델이 문장 간의 유사도를 더욱 정교하게 학습하도록 만들 수 있음을 확인했습니다. 데이터셋을 구축하고 적절한 손실 함수를 선택하여 모델을 미세조정하는 과정은 다소 노력이 필요할 수 있지만, 그 결과로 얻어지는 성능 향상은 매우 의미가 큽니다.

핸즈온 실습은 아래 github 에서 실제로 진행가능하니 따라해보셔도 좋을거 같습니다.

[https://github.com/tmdgusya/finetuning-course/blob/main/course/chapter02.ipynb](https://github.com/tmdgusya/finetuning-course/blob/main/course/chapter02.ipynb)