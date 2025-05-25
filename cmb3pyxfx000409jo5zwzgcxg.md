---
title: "독특한 Go lang 의 String"
seoTitle: "Unique Features of Go Lang Strings"
seoDescription: "Go lang string length differs from Python due to UTF-8 encoding. Learn why Korean characters take 3 bytes in Go string"
datePublished: Sun May 25 2025 13:53:54 GMT+0000 (Coordinated Universal Time)
cuid: cmb3pyxfx000409jo5zwzgcxg
slug: go-lang-string
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1748181203503/d8e8ea6f-273d-416a-9bb1-cf4423e5f880.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1748181199523/8397fa74-f05f-4299-ba92-df9778869c02.png
tags: go, utf8, string

---

`len("Hello, 월드")` 의 출력값이 몇이 나올거라고 생각하는가? 파이썬과 같은 언어에서는 10이 나오기를 기대한다. 하지만 **Go lang** 에서는 14가 나온다. 그 이유는 무엇일까?

```go
package main

import (
    "fmt"
)

func main() {
    hello := "Hello, 월드!"

	fmt.Println("문자열 길이: ", len(hello))

    for i := 0; i < len(hello); i++ {
        fmt.Printf("타입: %T 값:%d 문자값: %c\n", hello[i], hello[i], hello[i])
    }
}
```

Go lang 의 `len()` 함수의 결과값이 14가 나오는 이유는 Go lang 에서는 문자열을 기본적으로 **UTF-8 로 인코딩된 바이트 시퀀스**로 다루기 때문이다. 즉, 14 바이트는 영어와 기호는 각 1byte, 한글이 각 3 byte 를 차지하기 때문에 `(1*8 + 3 * 2) = 14 byte` 가 나오게 된다. 여기서 **"왜 한글은 3 byte 일까?"** 라는 의문이 들 수 있다. 오늘은 그 의문을 한번 풀어보려고 한다.

## UTF-8 이란?

UTF-8 은 문자를 인코딩하는 체계화된 방식중 하나이다. 기본적으로 code point 를 1~4 byte 사이로 인코딩하도록 되어 있다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1748178195244/265d2fdb-f4a4-479e-8756-96584a55071e.png align="center")

**WikiPedia** 의 정의된 방식을 따라보면 code point 값으로 u~z 까지의 값을 치환해주면 된다고 한다. 즉, 한글은 3 Byte 가 소모되므로 **“월”** 이라는 문자의 유니코드를 **wwww xxxxyy yyzzzz** 값으로 치환해주면 될 것이다.

그렇다면 월이라는 단어의 유니코드를 확인해보자. 검색 해보니 “월” 의 유니코드는 **“U+C6D4”** 이다. 이걸 한번 16진수로 변환해보자. Go lang 에서는 16 진수를 나타내기 위해서는 `U+` 대신 `0x` 를 이용해야 한다.

```go
	codePoint := rune(0xC6D4)
	binary16BitStr := fmt.Sprintf("%016b", codePoint)

	fmt.Println("16비트 이진수 표현:", binary16BitStr)
```

위와 같이 코드를 작성하고 실행시키면 `1100 0110 1101 0100` 이 나온다. 보기 좋게 하기 위해 4글자 마다 개행을 하나씩 넣었다. 우리가 아까 위에서 나온 3바이트 기준으로 w~z 까지의 값을 대치하기 위해서는 4+6+6 으로 대입되어야 한다.

즉, `11101100 10011011 10010100` 이 나오게 될 것이다. **“월”** 이라는 단어는 3byte 로 표현가능하며 UTF-8에서는 이렇게 표기가 된다는 것을 계산해 볼수 있다. 그렇다면 한번 검증해보자. Go lang [String 공식문서](https://pkg.go.dev/builtin#string)에 따르면 String 은 UTF-8 로 인코딩된 8bit 의 집합이라 적혀있으므로 `%08b` 를 통해서 출력 가능할 것이다.

```go
	r := "월"
	c := string(r)
	for i := 0; i < len(c); i++ {
		fmt.Printf("%08b ", c[i]) // 11101100 10011011 10010100
	}
```

출력값을 보면 `11101100 10011011 10010100` 으로 우리가 계산한 값과 같은 것을 확인할 수 있다. 더 정확하게 하고 싶다면 **비트 연산(bit operators)** 를 이용해서도 가능하다. 첫번째 4비트는 16비트에서 12개를 right shift 하고 1111 로 비트 mask 를 하는 것으로 추출 가능하다.

```go
	standard3Bytes := "1110wwww 10xxxxyy 10yyzzzz"
	fmt.Println("표준 3바이트 인코딩:", standard3Bytes) // "1110wwww 10xxxxyy 10yyzzzz"

	wwww := (r >> 12) & 0b1111
	xxxxyy := (r >> 6) & 0b111111
	yyzzzz := r & 0b111111
	fmt.Println("첫 4비트:", fmt.Sprintf("%04b", wwww))     // 1100
	fmt.Println("다음 6비트:", fmt.Sprintf("%06b", xxxxyy))  // 011011
	fmt.Println("마지막 6비트:", fmt.Sprintf("%06b", yyzzzz)) // 010100

	standard3Bytes = strings.Replace(standard3Bytes, "wwww", fmt.Sprintf("%04b", wwww), 1)
	standard3Bytes = strings.Replace(standard3Bytes, "xxxxyy", fmt.Sprintf("%06b", xxxxyy), 1)
	standard3Bytes = strings.Replace(standard3Bytes, "yyzzzz", fmt.Sprintf("%06b", yyzzzz), 1)
	fmt.Println("최종 3바이트 인코딩:", standard3Bytes) // 11101100 10011011 10010100
}
```

결과가 동일하게 나오는 것을 확인할 수 있다.

## 마치며

Go 에서는 이러한 부분을 `rune` 을 쓰면 해결 가능하다. 다만 이 UTF-8 에 대한 이해가 선행되어야 rune 과 같은 타입에 대한 이해가 이루어진다고 보기에 먼져 정리해보았다.