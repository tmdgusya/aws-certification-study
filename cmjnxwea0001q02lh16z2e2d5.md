---
title: "select 톺아보기"
seoTitle: "Exploring Select Statement Usage"
seoDescription: "Discover the role of 'select' in system programming, exploring nonblocking I/O and efficient file descriptor management with C code examples"
datePublished: Sat Dec 27 2025 06:48:55 GMT+0000 (Coordinated Universal Time)
cuid: cmjnxwea0001q02lh16z2e2d5
slug: select

---

## 배경

시스템 프로그래밍을 공부하거나 비동기 프로그래밍 아키텍쳐 부분을 공부하다보면 심심치 않게 `selector`, `epoll`, `kqueue` 등의 키워드를 마주하게 된다. 이를 개념적으로 이해하는 것도 좋으나 항상 코드와 함께 무엇이 문제여서 발전했는지 보다보면 조금 더 이해가 잘간다. 함께 `C`로 작성된 코드로 함께 알아보자. (대부분이 pseudo 느낌이라 사실 C 를 몰라도 이해하기 쉬울 것이다)

이 포스트에서는 `sleep` 상태로의 전환 그리고 읽기가 가능할때 다시 깨어나서 값을 출력하는 등을 보여주는데 집중할 것이므로 기타 부수적인 지식이나 사항은 따로 설명하지 않을 예정이다.

## Busy waiting

```c
    while (len != 0 && (ret = read(fd, ptr, len)) != 0) {
        if (ret == -1) {
            if (errno == EINTR) {
                continue;
            }
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                fprintf(stderr, "File is not ready for reading\n");
                sleep(1);
                break;
            }
        }
        if (ret == -1) {
            perror("read");
            break;
        }
        len -= ret;
        ptr += ret;
    }
```

일단 `Blocking` 은 대부분 알고 있으니 `NonBlocking` 부터 시작해보겠다. 기존 Blocking 시스템에서는 file 이 사용가능 할때까지 blocking 이 걸렸으나, nonblocking 은 **특수한 에러(EAGAIN or EWOULDBLOCK)** 과 같은 에러를 리턴해서 현재 파일 읽기 또는 쓰기가 불가능함을 알려준다. (또는 블락킹을 당할 상황)

따라서 `Busy waiting` 방식으로 while 문 안에서 계속해서 파일이 사용가능한지를 체크하게 되면 한가지 문제점이 발생한다. 바로 프로세스가 **계속해서 Sleep 상태로 들어가지 못하고 일해야 한다는 사실**이다. 즉, CPU 를 계속해서 사용하게 된다. (signal 로 구현도 가능하나 복잡하여 이 본문에서는 다루지 않겠다)

이 부분을 해결하기 위해서는 현실적으로 많이 사용하는 **“Blocking”** 으로 전환하고, 멀티 스레드 혹은 멀티 프로세스 모드로 전환하여 하나의 스레드(또는 프로세스)가 blocking 되더라도 다른 일을 처리 가능하므로 하나의 스레드의 블락킹이 전체 프로그램에 크게 영향을 미치지 않아 보이게 할 수 있다. 다만 이러한 방식은 그 유명한 C10K 문제를 유발하게 된다.

## select

그렇다면 프로세스는 원하는 fd 들이 준비될때까지 sleep 상태에 들어가고, **fd** 들이 쓸수 있는 상태가 되었을때 알림을 받고 일할 수 있다면 어떨까? select 는 이러한 문제를 해결하기 위해 개발되었다.

```c
int  select(int n, fd_set *read_fds, fd_set *write_fds, fd_set *except_fds,
         struct timeval *timeout);
```

[공식문서](https://pubs.opengroup.org/onlinepubs/009695199/basedefs/sys/select.h.html)를 보면 대략적으로 위와 같이 정의되어 있다. (가독성을 위해 인자 이름을 변경하였다)

간단하게 설명하면 우리가 확인하고 싶은 fd 들의 집합(e.g. 1,2,3,4,..) 를 넘기면 커널이 준비된 fd 들의 집합을 리턴해준다. 즉, **읽기가 가능한지 확인하고 싶은 집합, 쓰기가 가능한지 확인하고 싶은 집합** 등을 보내면 된다.

```c
read_fds = [1,2,3,4] ======Send======> readable: [1] ======return========> [1] 
```

대략적으로 위와 같이 읽기 가능한 목록을 리턴해준다고 생각하면 된다. 근데 이렇게 배열로 관리하지 않고 `fd_set` 이라는 특별한 자료구조로 관리하기 때문에 매크로를 이용해서 내가 원하는 fd 가 읽기 가능한지 파악해야 된다. select 에서는 `FD_ISSET` 이라는 매크로를 지원하는데 이를 통해서 내가 원하는 fd 가 읽기 가능한지 확인 가능하다.

이런식으로 계속해서 읽기 가능한 set 을 알려주므로 select 는 레벨 트리거 라고 불린다.

커널은 이를 어떻게 확인할까? `kernel` 은 우리가 제공한 fd\_set 을 N 번 순회하면서 이를 확인한다. 그래서 select 의 첫번째 인자는 `n` 을 받고 있는데 이는 **우리가 제공한 fd 들 중 가장 높은 fd 에 +1 을 해준 값**이다. 즉, kernel 이 내부적으로 **0~n 까지 순회하며 이를 확인한다는 것**이다. 따라서 기존 방식보다는 효율적이나 계속해서 커널이 N 번 확인이 반복된다는 단점이 존재한다.

## 실습

이정도만 알면 대략적으로 코드를 작성해볼수 있다. 코드에서 우리 프로세스의 STDIN 이 읽기 가능할때 해당 부분에서 1024 바이트만큼 읽어오는 코드를 작성해보겠다.

```c
#include <stdio.h>
#include <sys/select.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

#define TIMEOUT 5
#define BUF_LEN 1024

int main(void) {
    struct timeval tv;
    fd_set readfds;
    int ret;

    FD_ZERO(&readfds);
    FD_SET(STDIN_FILENO, &readfds);

    tv.tv_sec = TIMEOUT;
    tv.tv_usec = 0;

    ret = select(STDIN_FILENO + 1, &readfds, NULL, NULL, &tv); // (0 ~ nfds - 1)

    if (ret == -1) {
        perror("select");
        return 1;
    } else if (!ret) {
        printf("%d seconds elapsed.\n", TIMEOUT);
        return 0;
    }

    /**
     * File descriptor 에서 즉시 읽기가 가능함.
     */
    if (FD_ISSET(STDIN_FILENO, &readfds)) {
        char buf[BUF_LEN + 1];
        int len;

        len = read(STDIN_FILENO, buf, BUF_LEN);
        if (len == -1) {
            perror("read");
            return 1;
        }

        if (len) {
            buf[len] = '\0'; // null character
            printf("Read %d bytes: %s\n", len, buf);
        }

        return 0;
    }

    fprintf(stderr, "No input available.\n");
    return 1;
}
```

코드자체는 간단하다. 부분적으로 나눠서 보면 select 부분에서 `STDIN_FILENO` 가 읽기 가능할때까지 프로세스는 sleep 상태가 될것이고, 커널에서 읽기 가능하다 (`FD_ISSET(STDIN_FILENO, &readfds)`) 라고 알려주면 byte 를 읽고 정상적으로 종료할 것이다.

바이너리 파일로 만들어서 한번 테스트 해보자.

```c
gcc -o ./bin/test_select 10.c
./bin/test_select < test_pipe
```

이제 실행했으니 한번 프로세스의 상태를 알아보자. (pipe 는 go 의 channel 같은 느낌으로 이해하면 된다)

```c
ps aux | grep test_select
```

상태를 출력해보면 아래와 같이 S+ 상태로 나온다.

```c
roach      35347  0.0  0.0   2784  1436 pts/1    S+   14:50   0:00 ./bin/test_select
```

즉, fore ground 에서 sleep 상태로 있다는 뜻이다. 이제 해당 프로세스에 “hello” 를 입력해서 깨워보자.

```c
echo "hello" > my_input_pipe
```

이렇게 입력하고 나면 아래와 같이 `sleep` 상태에서 일어나서 들어온 값을 출력시키고 정상적으로 종료한다.

```c
readable!!
Read 6 bytes: hello
```

## 마치며

오늘은 간단하게 리눅스 커맨드들과 함께 프로세스의 상태를 확인하며 `select` 를 알아보았다. 다음시간에는 `poll` 과 `epoll` 등을 알아보려고 한다.

* ---
    
    **레벨 트리거(level trigger)**: LLM 의 설명으로는 전압이 1로 올라가게되면 그 상태를 유지하는 구간이 생기게 되는데 높은 레벨의 상태를 유지하는 동안에는 계속 `1` 을 리턴한다고 해서 특정 상태에 머무르면 트리거 된다고 해서 레벨방식이라고 한다.