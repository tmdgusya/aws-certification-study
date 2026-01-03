---
title: "이미지 중복 분류하기"
seoTitle: "이미지 중복 감지 방법"
seoDescription: "이미지 중복을 비교하는 다양한 알고리즘, 그레이스케일, 블록 해시, pHash 등을 설명하고 구현하여 최적의 방법을 탐구합니다"
datePublished: Tue Dec 23 2025 05:44:11 GMT+0000 (Coordinated Universal Time)
cuid: cmji5tqji000902l9bgds0s3p
slug: 7j206647keaioykkeuztsdrtotrpzjtlzjqula
tags: phash-block-hash

---

이미지를 다루는 작업을 많이 하다보면 빠르게 이미지의 중복을 비교해야 할 일들이 생긴다. 우리는 크롤러를 헤비하게 사용하다 보니, 이미지 간의 중복을 픽셀 수준의 완전일치의 중복을 넘어 유사도(**같은 이미지가 아니지만 거의 비슷한 혹은 같은 이미지이지만 리사이징 된**)에 따라 비교를 해야한다.

오늘은 최근 적용한 이미지 중복 처리과정에서 여러가지 방법들을 실험하고 최선의 알고리즘을 골랐던 과정을 적어보려고 한다.

## 이미지는 어떤 정보일까?

우리가 가지고 있는 이미지는 R,G,B 3개의 색깔 채널로 구성되어 있다. 각 채널은 보통 255(2^8) 가지의 색상을 가지고, 이 R,G,B 를 이용해 만들어 내는 색상의 조합은 각 픽셀이 가질수 있는 정보의 가짓수를 이야기 한다.

이미지를 다룰때 중요한 점은 우리가 정말 보고자 하는 데이터가 무엇인지다. 만약 옷의 맵시 혹은 인물의 형태와 같은 구조적인 부분이 중요하다면, 사실 R,G,B 데이터보다 윤곽이 더 중요할 수 있다. 따라서 처리하는 과정속에서 필요없는 부분들을 제거하여 봐야하는 정보의 양을 줄이는 것이 중요하다.

## Grayscale

grayscale 은 회색으로 만드는 것을 의미한다. R,G,B 축을 한가지의 색상만을 가지게 해 24비트의 조합에서 8비트의 조합으로 정보의 양을 줄여준다. 가장 간단하게는 해당 픽셀에서 R,G,B 값을 구하여 **3**으로 나누는 방법이 있다. (사람의 눈에 맞게 Grayscale 을 하는 연구결과도 있는데, 예시에서는 간단하게 3으로만 테스트 해보겠다)

`Go` 코드로 작성하면 `image` package 를 사용하면 쉽게 적용가능하다.

```go
func (p *Pipeline) ToGrayScale() {
	originalImage := p.Stages[p.current-1].Image
	bound := original.Bounds()
	gray := image.NewGray(bound)

	var totalBrightness float64
	minBrightness := 255.0
	maxBrightness := 0.0

	for y := bound.Min.Y; y < bound.Max.Y; y++ {
		for x := bound.Min.X; x < bound.Max.X; x++ {
			r, g, b, _ := original.At(x, y).RGBA()

			grayValue := (float64(r>>8) + float64(g>>8) + float64(b>>8)) / 3.0
			gray.Set(x, y, color.Gray{Y: uint8(grayValue)})

			totalBrightness += grayValue
			if grayValue < minBrightness {
				minBrightness = grayValue
			}
			if grayValue > maxBrightness {
				maxBrightness = grayValue
			}
		}
	}
}
```

`stage` 는 이 개념을 연습하는데는 중요하지 않고 GUI 용이니 무시하고, for 문 안만 봐도 충분하다. for 문 안을 보면 각 픽셀에서 `RGBA` 값을 추출해서 grayScale 을 하기 위해 비트 쉬프트 연산후에 3으로 나눠준다. (여기서 8비트 오른쪽으로 미는 이유는 RGBA 에서는 16비트가 나와 8비트를 오른쪽으로 밀어줘야 한다. 색상은 8비트 이므로)

이렇게 하면 아래와 같이 이미지가 회색으로 변하는 것을 확인할 수 있다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767419343887/2d127865-3fd2-417b-8155-32eb6e62d05f.png align="center")

## 이미지를 압축해보기

`BlockHash` 알고리즘을 보면 이후에 또 정보를 압축하는 과정이 있다. 정보를 어떻게 압축할 수 있을까? 예를 들어, 시력이 조금 안좋은 사람을 생각해보자. 시력이 안좋은 사람이 이 이미지를 멀리서 바라보면 아마도 뚜렷하게 보이지 않고 색깔이 번진 상태로 보일 것 이다.

이걸 아주 쉽게 구현할 생각으로 떠올려 보면 이미지가 `100*100` 의 크기 일때 `10*10` 칸씩 이미지의 픽셀의 평균값을 구해 하나의 블럭으로 만들어 버리면 `10*10` 의 크기로 변할 것 이다. 아마 이미지의 정보가 많이 훼손되겠지만 훼손되어도 시력이 안좋은 사람이 이미지를 알아볼 수 있듯, 이미지의 구조는 남아 있을 수 있다.

한번 코드로 작성해보자.

```go
func (p *Pipeline) DivideIntoBlock(blocksize int) [][]float64 {
	grayImg := p.Stages[p.current-1].Image
	bounds := grayImg.Bounds()
	width, height := bounds.Dx(), bounds.Dy()

	// 이미지를 blocksize x blocksize 그리드로 나눔 (고정 크기)
	blockWidth := width / blocksize
	blockHeight := height / blocksize

	blocks := make([][]float64, blocksize)

	visualImg := image.NewGray(image.Rect(0, 0, width, height))

	for y := range blocksize {
		blocks[y] = make([]float64, blocksize)
		for x := range blocksize {
			var sum float64
			count := 0

			// 각 블록 영역의 평균 밝기 계산
			for py := y * blockHeight; py < (y+1)*blockHeight && py < height; py++ {
				for px := x * blockWidth; px < (x+1)*blockWidth && px < width; px++ {
					grayColor := grayImg.At(px, py).(color.Gray)
					sum += float64(grayColor.Y)
					count++
				}
			}

			blockAvg := sum / float64(count)
			blocks[y][x] = blockAvg
	}
}
```

코드 자체는 어려운점이 없다. blocksize 만큼 `y` 축과 `x` 축으로 움직이며 해당 블럭내에서 색상값을 다 더하고, 모든 개수를 센다음 평균값을 내어 평균값으로 압축한다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767419372523/e8f17501-9fb4-463e-b284-c40cf877fc33.png align="center")

CNN 을 해봤다면 아마 익숙할 것이다. 이런 방식으로 압축하게 되면 이미지가 아래와 같은 형태가 된다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767419368169/a4e5edaf-d8a2-41f0-b76f-2c1abcba09ee.png align="center")

시력이 좀 안좋아진거 같긴 하지만, 그래도 흐릿흐릿하게 Gopher 의 실루엣 구조자체는 보이는 것을 확인할 수 있다.

## 이진화(Binarization)

이렇게 하고 나면 이미지가 구조적인지 파악하기 위해서 여러가지 방법들이 있지만 수평에 대한 구조적인 위치가 비슷한지를 파악해볼수 있다. 예를 들면, row 에서 median 을 측정해서 해당 median 값보다 클 경우에만 초록색으로 칠해보는 것이다. (작을 경우에는 빨간색)

이렇게 하게 되면 결국 `0000100010101` 과 같이 이진화가 가능하다. 한번 수평을 기준으로 이진화를 진행해보자.

```go
func (p *Pipeline) CalculateBandMedians(blocks [][]float64) []float64 {
	blockSize := len(blocks)
	numBands := 4
	bandHeight := blockSize / numBands

	if bandHeight == 0 {
		bandHeight = 1
		numBands = blockSize
	}

	for bandIdx := range numBands {
		startY := bandIdx * bandHeight
		endY := startY + bandHeight

		if bandIdx == numBands-1 {
			endY = blockSize
		}

		var bandValues []float64
		for y := startY; y < endY; y++ {
			for x := range blockSize {
				bandValues = append(bandValues, blocks[y][x])
			}
		}

		median := calculateMedian(bandValues)

		for y := startY; y < endY; y++ {
			for x := range blockSize {
				var col color.RGBA
				if blocks[y][x] > median {
					col = color.RGBA{0, 200, 0, 255}
				} else {
					col = color.RGBA{200, 0, 0, 255}
				}
			}
		}
	}
}
```

코드는 전체를 다 볼 필요는 없고 `for-loop` 안에만 보면 된다. y 축 기준으로 밴드의 크기 만큼 루프를 진행하며 해당 row 에 있는 값들을 모두 파악해서 bandValues 에 저장하고 **중앙 값보다 크다면** **Green(0, 200, 0, 255)** 를 아니라면 **빨강(200, 0, 0, 255)** 로 칠한다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767419403565/d9cd03a5-2467-497c-98ea-bebc386aad99.png align="center")

실제로 수행하고 나면 이미지가 위와 같이 변한다.

## Hasing

이제 이미지를 보면 이진형태의 데이터만 남게 되었다. 그렇다면 이 데이터는 쉽게 bit 로서 다룰수 있게 된 것이다. 즉 `32*32` 크기의 이진 데이터는 1024 비트로 표현이 가능하다. 따라서 바이트 배열로 이를 관리하게 되면 1024/8 = 128 개로 변하게 된다. 따라서 각 정보가 바이트 배열내에 어느 자리에 위치할지만 비트 연산을 통해 넣어주기만 하면된다.

```go
if blocks[y][x] > median {
    bit = 1
    onesCount++
    byteIndex := bitIndex / 8
    bitOffset := uint(7 - (bitIndex % 8))
    hash[byteIndex] |= 1 << bitOffset
}
```

코드를 전부 가져오지는 않겠다. 왜냐면 이해하는데는 위에 부분만 이해해도 충분하기 때문이다. loop 안에서는 위의 부분만 수행하는데 비트 연산에 익숙하지 않다면 어렵겠지만, 이해하면 코드는 생각보다 간단하다.

```go
byteIndex := bitIndex / 8

bitIndex=0  → byteIndex=0  (hash[0])
bitIndex=7  → byteIndex=0  (hash[0])
bitIndex=8  → byteIndex=1  (hash[1])
bitIndex=15 → byteIndex=1  (hash[1])
bitIndex=16 → byteIndex=2  (hash[2])
```

byteIndex 는 128 개의 바이트 배열에서 어느 인덱스에 들어가야 하는지를 정한다. byte = 8 bit 이므로 bitIndex 에 8 을 나눠주면 8개 비트마다 다음 인덱스로 이동하게 된다.

```go

bitIndex=0  → bitIndex%8=0 → bitOffset=7
bitIndex=1  → bitIndex%8=1 → bitOffset=6
bitIndex=2  → bitIndex%8=2 → bitOffset=5
...
bitIndex=7  → bitIndex%8=7 → bitOffset=0
bitIndex=8  → bitIndex%8=0 → bitOffset=7 (다시 시작)
```

bitOffset 은 그 바이트 배열내부에서 위치를 찾는 것이다. 8 bit 이므로 0-based 에서는 0~7 까지를 차지하게 된다. 따라서 bitIndex 를 8과 함께 modular 연산을 하게 되면 0~7 사이 이므로 원하는 위치를 찾을 수 있게 된다.

```go
bitIndex가 0, 3, 5, 8, 11, 15일 때만 bit=1이라고 가정:

bitIndex=0:
  hash[0] |= 1 << 7  →  hash[0] = 10000000

bitIndex=3:
  hash[0] |= 1 << 4  →  hash[0] = 10010000

bitIndex=5:
  hash[0] |= 1 << 2  →  hash[0] = 10010100

bitIndex=8:
  hash[1] |= 1 << 7  →  hash[1] = 10000000

bitIndex=11:
  hash[1] |= 1 << 4  →  hash[1] = 10010000

bitIndex=15:
  hash[1] |= 1 << 0  →  hash[1] = 10010001

최종:
hash[0] = 10010100 = 0x94
hash[1] = 10010001 = 0x91
```

그 이후에는 MSB First 방식으로 앞자리 부터 채워주면 되므로 비트 쉬프트 연산을 이용한다. 여하튼 이렇게 하면 긴 하나의 Bit 가 나오게 된다.

## Hamming distance

[Hamming distance](https://en.wikipedia.org/wiki/Hamming_distance) 는 간단하게 얼마나 flip 해야 유사해지는 지를 나타낸다. 따라서 둘이 다를때만 1이 나오면 되므로, 이를 구하는 방법은 아주 쉽게 `XOR` 연산을 이용하면 된다.

```go
func hammingDistance(a, b []byte) int {
	dist := 0
	for i := range a {
		xor := a[i] ^ b[i]
        dist += bits.OnesCount8(xor) // 1인개수 리턴(하드웨어 수준이라 빠르다고 함. Go 의 기본 라이브러리)
	}
	return dist
}
```

이제 이 distance 를 이용해서 이미지는 같지만 resizing 이 되거나, 뒤에 확장자가 다른 경우에 얼마나 유사하다고 판단하는지 확인해보자.

![image.png](https://storage.googleapis.com/roach-wiki/images/dea91ecd-cd0d-49eb-a485-8963f087aeab.webp align="left")

![image.png](https://storage.googleapis.com/roach-wiki/images/87729ff8-8fd0-4b80-9bef-dd34208d3f74.webp align="left")

![image.png](https://storage.googleapis.com/roach-wiki/images/0a100e92-a9c2-4ba3-b059-33c6f6b60405.webp align="left")

비교해보니 약 29 만큼 차이가 난다고 하고 97.2% 의 확률로 동일하다고 한다.

![image.png](https://storage.googleapis.com/roach-wiki/images/b1c2e060-f8c8-4a57-bd56-7299786e284d.webp align="left")

확실한 비교를 위해 다른 이미지랑 비교해보자.

![image.png](https://storage.googleapis.com/roach-wiki/images/b01ac5cb-70ce-468d-8594-7dc9f8be2eb0.webp align="left")

hamming distance 가 약 501 로 그냥 다른 이미지라고 나온다. 즉, 잘 비교되는 것을 확인해볼 수 있다. 여기까지가 BlockHash 의 구현이다. 다만 Band 를 정하는 부분에서 살짝 다른 사람들과 차이가 있을 수 있다. (보통은 전체의 median 값을 이용하는 것 같음)

나는 예시에서 블럭단위의 Band 로 조금 세분화 해보았다. 테스트 해볼거면 여러가지 방식으로 테스트 해보길 바란다.

## pHash

일단 pHash 란 **이미지의 내용을 기반으로 Hash 함수를 만들어내는 알고리즘**이다. **pHash** 의 특징으로는 아래와 같은 점이 있다.

1. **회전 저항성**: 약간 회전된 이미지도 유사한 해시 생성
    
2. **크기 저항성**: 크기가 다른 동일 이미지도 유사한 해시 생성
    
3. **노이즈 저항성**: 약간의 노이즈나 압축 아티팩트에도 강함
    
4. **색상 저항성**: 색상 변경에도 구조가 같으면 유사한 해시 생성
    

이건 `pHash` 가 `DCT` 라는 뭔 주파수를 기반으로 하기 때문이다. 지금 이해가는 BlockHash 기반으로 설명을 하면 한 10비트가 밀려서 비트들이 다른 경계로 가게되면 Band 드의 median 값이 변하게 될수 있지만, pHash 는 주파수를 이용하기 때문에 이런 작은 변화는 전체적인 Hash 값을 바꾸는데는 큰 영향을 주지 않는다고 한다.

> 사실 위와 같다고 하는데 이거 이해하려면 꽤나 큰 수학적 지식을 요구하는 것 같다. 일단 `주파수` 부터 머리가 띵해진다. BlockHash 까지는 충분히 구현 가능인데 여기서 부터는 라이브러리의 힘을쓰고 결과를 보는데 집중하겠다.

## 알고리즘

```plaintext
원본 이미지 (예: 1920x1080 컬러)
    ↓
[1단계] 그레이스케일 변환
    ↓ 1920x1080 흑백 이미지
[2단계] 32x32로 리사이즈 (Bilinear)
    ↓ 32x32 픽셀
[3단계] DCT(Discrete Cosine Transform) 변환
    ↓ 32x32 주파수 계수
[4단계] 저주파 8x8 영역 추출 (DC 제외)
    ↓ 8x8 = 64개 계수
[5단계] 중앙값 기준 비트화
    ↓ 64bit = 8byte 해시
최종 pHash (16진수 문자열)
```

## 리사이징

알고리즘은 아까 구현했던것과 같이 그레이스케일을 한뒤에 `32*32` 로 리사이징을 해준다. 여기서 중요한게 리사이징에서 다운샘플링을 하는데 크기 만큼 가중치를 줘서 조금 색깔 보정을 해준다.

```go
만약 srcX=3.25, srcY=2.75라면:
  wx=0.25, wy=0.75

100 ---- 200      각 픽셀의 기여도:
 |    X   |       좌상: 100 * 0.75 * 0.25 = 18.75
 |        |       우상: 200 * 0.25 * 0.25 = 12.5
150 ---- 180      좌하: 150 * 0.75 * 0.75 = 84.375
                  우하: 180 * 0.25 * 0.75 = 33.75
결과 = 18.75 + 12.5 + 84.375 + 33.75 = 149.375
```

보간 방법으로 샘플링 하는거라고 하는데 이렇게 정규화를 해서 그런지 정보 손실이 조금 더 적다고 한다. 내 방법은 아까 Average Pooling 이 였는데 이 방법이 더 좋나보다.

![image.png](https://storage.googleapis.com/roach-wiki/images/03affe5b-bdf4-4c21-9d3f-47de1cc93852.webp align="left")

내 눈엔 더 안좋아 보이긴 하는데.. 컴퓨터 눈엔 더 좋아보일수도 있다.

## DCT 변환후 저주파 영역 추출

DCT 의 목적은 공간 도메인(픽셀)을 주파수 도메인(계수)으로 변환하는 것이라고 한다. 나도 이해는 못했지만 수식을 보고 그냥 구현만 해봤다. (이 주파수를 알려니 퓨리에 머시기가 나와서 지금은 건드릴 지식은 아닌거 같다는 생각이 들었다. 나중에 더 파보겠다..)

```plaintext
공간 도메인 (원본 32x32 픽셀)
    ↓ DCT
주파수 도메인 (32x32 계수)
- (0,0): DC 성분 (전체 평균 밝기)
- (0,1), (1,0): 저주파 (큰 패턴)
- (31,31): 고주파 (세밀한 디테일)
```

#### DCT 수식

```plaintext
DCT[u,v] = 0.25 * C(u) * C(v) * Σ Σ pixels[x,y] *
           cos(π*u*(x+0.5)/N) * cos(π*v*(y+0.5)/N)

여기서:
- u, v: 주파수 좌표 (0~31)
- x, y: 픽셀 좌표 (0~31)
- C(u) = 1/√2 (u=0), 1 (u≠0)
```

**코드**:

```go
func (p *Pipeline) Dct2D(pixels [][]float64) [][]float64 {
    for u := range size {
        for v := range size {
            sum := 0.0
            for x := range size {
                for y := range size {
                    sum += pixels[y][x] *
                        math.Cos(math.Pi*float64(u)*(float64(x)+0.5)/float64(size)) *
                        math.Cos(math.Pi*float64(v)*(float64(y)+0.5)/float64(size))
                }
            }

            // 정규화 계수
            cu := 1.0
            if u == 0 { cu = 1.0 / math.Sqrt(2) }
            cv := 1.0
            if v == 0 { cv = 1.0 / math.Sqrt(2) }

            dct[u][v] = 0.25 * cu * cv * sum
        }
    }
    return dct
}
```

코드는 수식을 따라치기만 하면 되니 그다지 어려운 건 없다. 변환하고 나면 아래와 같이 저주파 영역만을 추출하여 이용한다.

![image.png](https://storage.googleapis.com/roach-wiki/images/0e7d19c0-960e-43ec-adff-ec051c5743e2.webp align="left")

여기서 저주파가 하늘/땅 경계, 큰 물체와 같은 이미지의 전체적인 구조나 밝기 변화를 인식한다고 한다. 우하단에 세밀한 디테일이 들어있어, 이미지의 전체적인 구조로 유사성을 판단하기에는 저주파를 이용하면 충분하다고 한다.

## 이진 변환

이제 부터는 익숙한 영역인데 바로 이진화이다. 아까 뽑은 `8*8` 영역을 이진화 하기위해서는 어떻게 해야할까? 바로 중앙값을 이용하여 중앙값보다는 클경우 1, 작을 경우 0 으로 지정하는 것이다.

```go
// 1. 모든 64개 값을 1차원 배열로
allValues := []float64{
    lowFreq[0][0], lowFreq[0][1], ..., lowFreq[7][7]
}

// 2. 정렬하여 중앙값 찾기
sorted := sort(allValues)
median := sorted[32]  // 64개 중 중간값

// 3. 각 값을 비트로 변환
for i := 0 to 63 {
    if lowFreq[i] > median {
        hash[i] = 1  // 중앙값보다 크면 1
    } else {
        hash[i] = 0  // 작거나 같으면 0
    }
}
```

![image.png](https://storage.googleapis.com/roach-wiki/images/b44f08c1-f71f-4aa3-b121-5025b3ccc5d5.webp align="left")

이제 pHash 로 계산했을때 결과값을 보자.

![image.png](https://storage.googleapis.com/roach-wiki/images/00b32a8a-4111-4756-8ee0-883833719bb7.webp align="left")

아까 BlockHash 가 97.2% 인것에 비해. 유사도가 낮은 것을 확인할 수 있다. 그렇다면 flip 할 경우에는 어떻게 될까?

## 테스트

![image.png](https://storage.googleapis.com/roach-wiki/images/9ca4fd7c-eeca-47e2-8b9b-0e9e6bfca697.webp align="left")

180도로 회전하는 경우에는 어떻게 될까?

![image.png](https://storage.googleapis.com/roach-wiki/images/81520fda-aebc-446b-aae8-621eddf62999.webp align="left")

![image.png](https://storage.googleapis.com/roach-wiki/images/3af2999b-3087-425a-a9bd-0ef8c3e5aec1.webp align="left")

180도로 회전하는 경우에는 pHash 의 경우 46.9% 이지만 BlockHash 의 경우 95.5% 이다. 하지만 우리앱의 경우 저정도로 flip 되거나 바뀐건 다른것으로 검출되어야 하므로 pHash 가 더 좋아보인다.

BlockHash 의 경우 추측해보자면 수직으로 봤을때 저 golang 이미지가 가운데 길쭉한거 하나있는 느낌이기 때문에 크게 안바뀐다고 느끼기 때문인거 같기도 하다.

![image.png](https://storage.googleapis.com/roach-wiki/images/846ad961-21b8-466c-af34-e1591235c566.webp align="left")

**우로 90**도 돌려보니 BlockHash 또한 확연하게 떨어진것을 확인할 수 있다. 다만 느껴지는건 BlockHash 자체는 뭔가 동일한 이미지를 flip 하고 했을때 pHash 에 비해 크게 값이 바뀌지 않는 것 같다는 느낌을 받았다.

## 마치며

`pHash` 는 약간 이해하기 어렵지만 계속 공부하다 보면 이해할수 있을거라고 생각한다. 다만 중요한 건 자신이 필요한 상황에 따라 여러 알고리즘이나 Pooling 기술등을 적용해보며 벤치마크를 내보고 적절한 알고리즘을 정하는 것이 중요한 것 같다.

다음에 pHash 가 완벽하게 이해되면 지금의 BlockHash 에서 Band 를 다르게 가져갔듯이 pHash 도 중간중간에 다른 방법들을 적용해보며 사용해봐야겠다는 생각이 들었다.

참고로 운영환경에서는 Python code 로 구현했는데 운영환경에서는 cv2 를 이용해서 쉽고 안전하게 라이브러리로 구현해두었다.

## 코드

[깃허브 코드](https://github.com/tmdgusya/check-dup)