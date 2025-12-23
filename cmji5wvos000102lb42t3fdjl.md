---
title: "[밑바닥 부터 구현하는 데이터베이스] 1 - 운영체제에 파일을 어떻게 읽고 쓸까?"
datePublished: Tue Dec 23 2025 05:46:37 GMT+0000 (Coordinated Universal Time)
cuid: cmji5wvos000102lb42t3fdjl
slug: 1

---

우리가 저장하는 모든 데이터는 컴퓨터에 `바이트(byte)` 로 저장된다. 그래서 우리는 고 수준의 자료형을 **직렬화(Serialize)** 하여 저장하여야 한다. 예를 들어, 우리가 `int[]` 형을 직렬화 한다고 해보자.

```python
[1, 2, 3] => [0x00, 0x00, 0x00, 0x01, 0x00, 0x00, 0x00, 0x02, 0x00, 0x00, 0x00, 0x03]
```

각 정수를 4 Byte 로 이어 붙이는걸 생각해볼 수 있다. 다만 여기서 byte 로 변환한 것을 어떻게 적어야 할지 고민해볼 수 있다. 예를 들어 `[1] => [0x00, 0x00, 0x00, 0x01]` 으로 적는 사람도 있을 것이고, `[1] => [0x01, 0x00, 0x00, 0x00]` 으로 적는 사람도 있을 것이다.

실제로 이러한 사유때문에 **매핑(mapping)** 방식이 별도로 존재하게 된다. `[0x00, 0x00, 0x00, 0x01]` 에서 `0x00` 을 **가장 상위 바이트(MSB)** 라고 표현하고, `0x01` 을 **가장 하위 바이트(LSB)** 라고 표현한다. 메모리는 일반적으로 `0x00`, `0x01`, `0x02`, … 와 같이 증가하므로 이 상위 바이트에서 하위 바이트의 흐름을 어떻게 메모리 주소상에 맵핑할지를 나타내는 방식을 **엔디안(Endian)** 이라고 한다.

## BigEndian

네트워크나 포맷에서 자주 쓰는`BigEndian` 은 상위 바이트(MSB) 를 메모리에 낮은 주소(앞쪽)에 쓰는 방식이다. 즉, `[0x00, 0x00, 0x00, 0x01]` 이 된다.

## LittleEndian

그와 반대로 `LittleEndian` 은 하위 바이트(LSB) 를 메모리의 낮은 주소(앞쪽)에 쓰는 방식이다. 따라서, `[0x01, 0x00, 0x00, 0x00]` 이 된다.

왜 이렇게 세분화 되어 있는걸까? 그 이유는 네트워크 프로토콜, CPU 아키텍쳐 마다 이와 같이 맵핑하는 방식이 다르기 때문이다. 따라서 이를 잘 인지하는 것이 중요하다. 다른 맵핑 방식으로 읽게 되면 아예 다른 값으로 해석될 수 있기 때문이다.

## Go lang 에서는?

Golang 에서는 이를 어떻게 처리할까? `binary.BigEndian.PutUint32` 를 사용하면 쉽게 인코딩이 가능하다. 예시를 위해 정수값을 4 바이트로 변환하여 byte 로 변환한다고 해보자. 첫번째로 해야할 일은 무엇일까? 바로 **len(nums) \* 4** 만큼의 byte 배열을 확보해주는 것이다.

```go
buffer := make([]byte, 4*len(nums))
```

이후에는 `binary.BigEndian.PutUint32` 을 이용하면 쉽게 4byte 배열에 맞게 들어가도록 변환 가능하다.

```go
// PutUint32 stores v into b[0:4].
func (bigEndian) PutUint32(b []byte, v uint32) {
	_ = b[3] // early bounds check to guarantee safety of writes below
	b[0] = byte(v >> 24)
	b[1] = byte(v >> 16)
	b[2] = byte(v >> 8)
	b[3] = byte(v)
}
```

여기서 **“비트연산”** 이 생소하면 이 부분이 잘 이해가 안갈 수 있는데, 이 연산(right shift) 은 간단하게 비트를 오른쪽으로 미는 역할을 한다. 이 부분을 잘 이해하지 못하면 앞으로 시리즈가 어려우므로 예시를 들고 넘어가 보겠다.

### 비트연산(부가 설명)

예를 들어 어떤 수를 **이진수**로 변환했는데 `00010010 00110100 01010110 01111000` 과 같이 나왔다고 해보자. 우리가 이걸 4byte 에 하나하나 담으려면 어떻게 해야할까? 이럴때 비트 연산을 사용하면 쉽다. 첫번째로 `00010010` 을 담으려면 총 24번 오른쪽으로 밀어야 `00000000 00000000 00000000 00010010` 형태가 된다. (정확히는 shift 연산이지만 이해를 위해 민다는 표현을 차용했다)

즉, 이 상태에서 1 byte = 8bit 이므로 byte (v &gt;&gt; 24) 를 하게 되면 byte 는 00010010 을 가지게 된다. 나머지도 똑같다. 즉 담고 싶은 부분을 마지막 8bit 로 만들기 위해 shift 연산을 하는 것이다. 이해가 갔다면 스스로 **Little-endian** 방식도 한번 구현해보길 바란다.

```go
import "encoding/binary"

func IntSliceToBytes(nums []uint32) []byte {
    buffer := make([]byte, 4*len(nums))
    for i, n := range nums {
        binary.BigEndian.PutUint32(buffer[i*4:], n)
    }
    return buffer
}
```

본문으로 돌아와서 다시 코드를 보면 이제 이 코드가 어떤 동작을 하는지 명확하게 이해갔을 것이다. 그렇다면 이제 이 byte 들을 파일에 쓰는 작업을 진행해보자.

## 파일(os.File)

일단 운영체제는 우리가 적어놨던 파일 또는 데이터들을 어떻게 읽고 가져올까? 우리가 파일을 읽을때 파일 전체를 로드해서 가져올까? 아니면 부분만 읽어서 가져올까? OS 를 공부해봤다면 들어봤겠지만 운영체제는 블럭단위로 파일을 읽게 된다.

그렇다면 운영체제는 이를 어떻게 나눠서 읽는걸까? 예를 들어 우리가 10KB 짜리 파일을 읽는데 1KB 블럭단위로 이 페이지를 읽어온다고 해보자. 첫번째로 1KB 를 읽고, 다음 부터는 마지막으로 읽은(1024~2048) 까지를 읽어야 할 것이다.

### 오프셋(Offset)

이를 위해 파일에서 어디까지 읽었는지를 알려주는 **오프셋(Offset)** 을 필요로 하게 되었고, 이는 파일 포인터에 값으로 저장되어 있다. 코드로 보면 조금 더 수월하게 이해할 수 있으니 코드로 한번 작성해보면서 알아보자.

```go
	f, err := os.OpenFile("test.txt", os.O_RDWR|os.O_CREATE, 0666) // 파일을 생성

	if err != nil {
		panic(err)
	}

	arr := make([]uint32, 12) // 정수형 배열(우리가 쓸값)
	buf := make([]byte, 12*4) // 정수형 배열을 byte 로 변환할때 사용할 buffer (len(nums) * 4)

	for i := 0; i < 12; i++ {
		arr[i] = i
	}

	n, err := f.Write(IntSliceToBytes(arr))
```

파일을 쓰기 위해 파일을 생성하고, 우리가 원하는 정수 배열을 byte 배열로 전환한 다음에 `f.Write(b byte[])` 를 통하여 해당 파일에 값을 쓰게 된다. 값을 쓸때 우리는 **0번째 부터 48번째까지 offset 을 옮겨가며 파일에 값을 기록**하게 된다.

```go
	_, err = f.Read(buf)

	if err != nil {
		panic(err)
	}

	fmt.Printf("%v\n", BytesToIntSlice(buf))

// error	
panic: EOF

goroutine 1 [running]:
main.main()
        /home/roach/btree/file/main.go:51 +0x246
exit status 2
```

만약에, 우리가 이를 생각하지 않고 여기서 바로 `Read` 메소드를 호출하게 되면 어떻게 될까? 우리는 당연하게도 0번부터 읽을 것 같지만 `offset` 부터 읽게 된다. 즉, 48번째 부터 읽게 되므로 우리는 EOF 라는 에러를 마주하게 된다.

### Seek 메소드

```go
// Seek sets the offset for the next Read or Write on file to offset, interpreted
// according to whence: 0 means relative to the origin of the file, 1 means
// relative to the current offset, and 2 means relative to the end.
// It returns the new offset and an error, if any.
// The behavior of Seek on a file opened with O_APPEND is not specified.
func (f *File) Seek(offset int64, whence int) (ret int64, err error) {
	if err := f.checkValid("seek"); err != nil {
		return 0, err
	}
	r, e := f.seek(offset, whence)
	if e == nil && f.dirinfo.Load() != nil && r != 0 {
		e = syscall.EISDIR
	}
	if e != nil {
		return 0, f.wrapErr("seek", e)
	}
	return r, nil
}
```

따라서 이러한 문제를 마주하지 않기 위해서는 offset 을 우리가 원하는 위치로 이동시켜야 한다. `Seek` 함수를 보면 `offset` 은 양수/음수 값에 따라 현재 `기준점(whence)` 에서 오프셋을 계산하게 된다.

주석의 설명을 보면 `0` 은 파일의 시작 지점을 의미하고, `1` 은 현재 offset 의 위치에서, `2` 는 마지막을 기준으로 계산된다. 그렇다면 현재 Offset 위치를 알아낼때는 [`f.Seek`](http://f.Seek)`(0, 1)` 을 이용하면 현재의 오프셋 위치를 알아낼 수 있을 것이다. 실제로 테스트를 한번 해보자.

```go
pos, err := f.Seek(0, 1) // 48 출력
```

따라서 파일을 읽기전에 [`f.Seek`](http://f.Seek)`(0, 0)` 으로 옮겨주자. 쓰기 이후 실행해보면 48이 잘 출력되는 걸 확인할 수 있다. 그렇다면 우리가 읽기 작업전에 해줘야 할 것은 무엇일까? 바로 파일의 offset 을 시작지점으로 옮겨줘야 한다.

```go
	f.Seek(0, 0) // 0 이점 시작지 이므로

	_, err = f.Read(buf)

	if err != nil {
		panic(err)
	}

	fmt.Printf("%v\n", BytesToIntSlice(buf)) // [0 1 2 3 4 5 6 7 8 9 10 11]
```

위와 같이 파일을 시작지점으로 옮기고 이후 읽기를 실행하면 값을 잘 읽어오는 것을 확인할 수 있다. 그렇다면 여기서 아까의 질문인 `운영 체제는 어떻게 파일을 나눠서 읽을 수 있을까?` 에 대답할 수 있게 된다. 즉, Offset 으로 일정 단위로 읽는다면 충분히 나눠 읽을 수 있다는 것이다.

### 왜 나눠 읽지?

그렇다면 왜 나눠 읽는 것일까? 바이트를 하나하나 가져오면 안되는 걸까? 우리가 일반적으로 특정 데이터를 어디서 부터 가져오는 행위는 항상 가벼운 행위는 아니다. 따라서, 배치로 묶어서 처리하는 이유는 보통 I/O 와 같은 무거운 행위를 덜 하기 위해서이다.

즉, 운영체제가 나눠 읽는 이유도 이러한 성능적인 부분에서 최적화의 목적에 있다. 따라서 나눠 읽는 부분을 구현해보고 생각해보면서 어떤 최적화가 되는지 생각해보자.

## Page

예를 들어 우리가 `16 byte` 씩 데이터를 읽고, 이를 특정 구조체에 저장한다고 해보자. 우리는 이러한 블럭단위를 `Page` 라고 부를 것이고, 이를 아래와 같이 구조체로 정의할 것이다.

```go
const PAGE_SIZE = 16 // Byte

type Page struct {
	Id   int32
	Data []byte
}
```

만약 **24개의 정수(96byte)** 를 Page 로 읽게 되면 몇개의 Page 가 생성되게 될까? `96 / PAGE_SIZE = 96 / 16 = 6` 개가 필요하게 될 것이다. 즉, 우리는 `6` 개의 Page 를 가지게 된다. 하지만 Page 마다 순서가 있으므로 이를 구분하기 위해 Id 라는 값을 둔다.

첫번째 페이지를 읽고, 두번째 페이지를 읽을 때 \[[`f.Seek`](http://f.Seek)`](`[`http://f.Seek)(첫번째`](http://f.Seek\)\(첫번째) `페이지 이후, 0)` 으로 만들어야 하기 때문에 Id 를 두는 것이다. 그렇다면 [`f.Seek`](http://f.Seek) 의 첫번째 인자로 들어갈 인자는 `PAGE_SIZE * Id` 가 됨을 알 수 있다.

### Page 로 읽기

그렇다면 위의 예시(총 크기 96Byte) 에서 Page 단위로 우리가 읽는 부분의 프로세스는 어떻게 될까? 이미 예측하고 있을 수 있겠지만 아래와 같이 진행된다.

**읽기 알고리즘**

1. 총 크기 / PageSize 만큼의 루프를 생성한다.
    
2. 루프 내부에서 PageSize 만큼의 Buffer(byte 배열) 을 생성한다.
    
3. 루프 내부에서 현재 루프의 `I(반복 횟수) * PAGE_SIZE` 로 offset 을 설정한다. (`i=0 일때 0 * 16, i = 1 일때 1 * 16, …`)
    
4. 파일 포인터의 offset 을 계산된 값으로 이동시킨다. [`f.seek`](http://f.seek)`(I(반복 횟수) * PAGE_SIZE, 0)`
    
5. 제공된 버퍼를 넣어 값을 읽는다.
    

여기서 고민이 한가지 생긴다. 어떻게 파일 단위로 이 페이지를 관리하지? 🤔 이를 위해 `PageManager` 라는 객체를 하나 만들어 볼수 있다.

```go
type PageManager struct {
	f     *os.File
	pages []*Page
}
```

이렇게 되면 우리가 읽은 페이지를 Id 를 인덱스 삼아 Page 를 관리할 수 있게 되고, 필요한 `전체 읽기(ReadAll)` 이라는 함수 또한 이 객체에서 관리하게 할 수 있다.

```go
func (p *PageManager) ReadAll() error {
	for i := 0; i < BYTE_LENGTH/PAGE_SIZE; i++ {
		buf := make([]byte, PAGE_SIZE)
		p.f.Seek(int64(i*PAGE_SIZE), 0) // 이건 사실 옮겨지기 때문에 불필요하나 명확한 예시를 위해 적음
		p.f.Read(buf)
		p.pages[i] = &Page{
			Id:   int32(i),
			Data: buf,
		}
	}
	return nil
}
```

이 ReadAll 이라는 함수를 이용하면 우리는 내부 SYSTEM\_CALL 을 통해 디스크에서 파일을 읽어오게 된다. 근데 만약 정수 배열 `0~4 번째` 에 Read 가 유독 많다면 어떻게 될까? 이 SYSTEM\_CALL 을 통해 계속 디스크로 부터 읽어오는 작업을 해야 할까?

이제는 그럴 필요가 없다. 우리가 이미 객체화 하여 메모리에 값을 올려뒀기 때문이다. `ReadAt(id int32)` 를 통해 메모리에서 값이 있다면 리턴하게 해보자.

```go
func (p *PageManager) ReadAt(id int32) []byte {
	return p.pages[id].Data
}
```

이제는 `SYSTEM_CALL` 이 아닌 단순 Memory Random Access 로 해당 부분에 적혀있는 값을 가져올 수 있게 됬다. 즉, 이전의 Disk 에서 읽어오는 것보다는 가벼운 행위가 되었다.

## 전체 코드

```go
package main

import (
	"encoding/binary"
	"fmt"
	"os"
)

const PAGE_SIZE = 16 // Byte
const INT_LENGTH = 24
const BYTE_LENGTH = INT_LENGTH * 4

type Page struct {
	Id   int32
	Data []byte
}

type PageManager struct {
	f     *os.File
	pages []*Page
}

func (p *PageManager) ReadAt(id int32) []byte {
	return p.pages[id].Data
}

func (p *PageManager) ReadAll() error {
	for i := 0; i < BYTE_LENGTH/PAGE_SIZE; i++ {
		buf := make([]byte, PAGE_SIZE)
		p.f.Seek(int64(i*PAGE_SIZE), 0)
		p.f.Read(buf)
		p.pages[i] = &Page{
			Id:   int32(i),
			Data: buf,
		}
	}
	return nil
}

func IntSliceToBytes(nums []uint32) []byte {
	buf := make([]byte, 4*len(nums))
	for i, n := range nums {
		binary.BigEndian.PutUint32(buf[i*4:], n)
	}
	return buf
}

func BytesToIntSlice(buf []byte) []int {
	n := len(buf) / 4
	out := make([]int, n)
	for i := 0; i < n; i++ {
		out[i] = int(binary.BigEndian.Uint32(buf[i*4:]))
	}
	return out
}

func main() {
	f, err := os.OpenFile("test.txt", os.O_RDWR|os.O_CREATE, 0666)
	if err != nil {
		panic(err)
	}

	pageManager := &PageManager{
		f:     f,
		pages: make([]*Page, BYTE_LENGTH/PAGE_SIZE),
	}

	arr := make([]uint32, INT_LENGTH)

	for i := 0; i < INT_LENGTH; i++ {
		arr[i] = uint32(i)
	}

	_, err = f.Write(IntSliceToBytes(arr))
	if err != nil {
		panic(err)
	}

	f.Seek(0, 0)

	pageManager.ReadAll()

	fmt.Printf("%v\n", BytesToIntSlice(pageManager.ReadAt(0)))
}
```

## 헷갈릴 만한 부분

* 우리가 Page 단위로 읽어야만 운영체제가 블럭 단위로 읽는 것은 아니다. 운영체제는 기본적으로 블럭단위로 읽고, 필요에 의해서는 더많은 데이터를 prefetch 하기도 한다.