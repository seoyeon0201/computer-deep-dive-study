# 입출력이 없는 컴퓨터가 있을까?

## 파일을 읽을 떄 프로그램에는 어떤 일이 발생할까?

### 메모리 관점에서 입출력

메모리 관점에서 입출력은 단순한 메모리의 복사(copy)일뿐이다.

- input : 데이터가 외부장치에서 메모리로 복사
- output : 데이터가 메모리에서 외부장치로 복사

### read 함수는 어떻게 파일을 읽는 걸까?

운영체제는 I/O가 진행중일 때 현재 프로세스의 실행을 일시 중지하고 입출력 블로킹 대기열에 넣는다.
작업이 완료된 친구는 준비완료 대기열에 넣는다.

일반적으로 입출력 데이터는 운영체제 내부로 복사되며 이후 운영 체제가 프로세스의 주소 공간으로 복사합니다.
그래서 실제로 운영체제를 통한 복사라는 한 계층이 또 존재

이런 운영체제를 우회해서 직접 복사하는 방법을 zero-copy 라고함.

## 높은 동시성의 비결: 입출력 다중화

파일은 N bytes의 수열이다.
디스크, 네트워크 데이터, 터미널, 프로세스간 통신 도구인 pipe 까지 파일로 취급된다.
(예전에 소켓도 파일이다! 라는 내용을 널널한 개발자님이 자주 말했는데 유사한 개념인듯?)

I/O는 전부 파일이란 개념으로 추상화되어있다. 좀 더 넓은 범위를 생각해보자.

### FD(File Descriptor)

파일을 열때 커널은 fd를 반환하며, 파일 작업을 실행할 땐 이 fd를 커널에 전달해야합니다.
일종의 file identifier의 역할을 합니다.

### 다중 입출력을 어떻게 효율적으로 처리하는 것일까?

높은 동시성이란 서버가 동시에 많은 사용자 요청을 처리할 수 있음을 의미한다.
3-way handshake에 성공하면, fd를 얻을 수 있다. 소켓도 I/O니까 파일이다!

일반적으로 read함수는 **블로킹** 입출력이다. 사용자가 어떤 데이터도 안 보내면 스레드 전체가 일시 중지한다.

그렇다면 멀티 스레딩을 하면 어떨까? 하지만 이런 경우 스레드 수가 너무 많아질 수도 있고 스레드 스케쥴링과 전환에 너무 많은 부담이 가해진다.

생각을 바꿔야 한다.

### 상대방이 아닌 내가 전화하게 만들기

많은 수의 fd를 처리하는 더 나은 방법은 커널이 응용프로그램에 통지하도록 하는 것.
요게 후에 기술할 입출력 다중화다.

### 입출력 다중화

1. fd를 획득한다.
2. 함수를 호출해 커널에 '이 fd를 감시하다 읽거나 쓸 수 있는 fd가 나타나면 반환해줘'라고 말한다.
3. 함수가 반환된다는 것은 읽고 쓸 수 있는 조건이 준비된 fd를 획득했다는 의미. 이를 통해 상응하는 처리를 할 수 있다.

리눅스에서 입출력 다중화 기술을 사용하는 방법은 select, poll, epoll이다.

### select, poll, epoll

위 함수는 모두 동기 입출력 다중화 기술이다.
이 세함수 모두 호출된 스레드가 일시 중지되고, fd가 이벤트를 생성할 떄까지 반환하지 않는다.

- select
    - select의 경우엔 감시할 fd를 fd_set 로 묶어서 넘겨받음
    - 커널은 전체 fd를 계속 순회하면서 fd가 준비되었는지 확인.
    - 조건에 맞는 fd가 있는 경우 이를 사용자 영역에 복사해서 알려줌.
    - 매번 fd_set을 다시 세팅해야 함
- poll
    - poll의 경우에는 감시할 fd를 pollfd로 묶어서 넘겨받음
    - 커널은 전체 fd를 계속 순회하면서 fd가 준비되었는지 확인.
    - 조건에 맞는 fd가 있는 경우 revents 필드에 마킹해서 유저 영역에 복사해서 알려줌.
- epoll
    - 커널은 epfd라는 이벤트 감시용 객체(커널 내부 큐) 생성
    - fd를 등록
    - fd가 준비된 경우 발생한 경우 이벤트를 fd를 ready 큐에 넣음
    - caller는 이 ready 큐에서 반환받아 처리

## mmap : 메모리 읽기와 쓰기 방식으로 파일 처리하기

메모리에 쓰는 것과 달리 파일에 쓰는 것은 어렵다.

이유는

- 디스크에서 특정 주소를 지정하는 것과 메모리에서 특정 주소를 지정하는 방법이 다르고
- cpu와 외부 장치 간 속도에 차이가 있기 때문입니다.

### 마술사 운영체제

mmap을 사용하면 파일의 내용을 프로세스의 가상 메모리 주소 공간에 매핑할 수 있습니다.  
이 매핑이 이루어지면, 파일 내용을 마치 메모리 배열처럼 접근할 수 있어 `read()`나 `write()` 없이도 읽기/쓰기 작업이 가능합니다.
실제로는 페이지 폴트가 발생할 때 커널이 필요한 부분만 디스크에서 메모리로 로딩하며, 이후 수정된 내용은 페이지 캐시를 통해 디스크로 반영됩니다.

### mmap 대 전통적인 read/write 함수

전통적인 read/write의 경우에 low level의 system call을 사용한다.

read를 쓸 땐 데이터를 커널에서 user로 복사해야 하고,
write를 쓸 땐 데이터를 user에서 커널로 복사해야 한다.

mmap은 이런 문제가 없지만, 완벽하진 않다.

프로세스 메모리 공간과 파일 간의 링크를 유지하기 위해 특정 데이터 구조를 사용해야 하며 이 또한 성능 부담이다.

그럼 결국에 중간 가상메모리라는 계층이 생기고 페이지 폴트가 발생하면 디스크에 쓰는 식처럼 이해?했다.

### mmap 직접 조작하기

```shell
execve("/usr/bin/ls", ["ls"], 0x7ffe9f5c1e80 /* 38 vars */) = 0
brk(NULL)                               = 0x5ac4d6334000
arch_prctl(0x3001 /* ARCH_??? */, 0x7ffed85a36c0) = -1 EINVAL (Invalid argument)
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f7d6ff25000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=68695, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 68695, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f7d6ff14000
close(3)                                = 0
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libselinux.so.1", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 832) = 832
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=166280, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 177672, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f7d6fee8000
mprotect(0x7f7d6feee000, 139264, PROT_NONE) = 0
mmap(0x7f7d6feee000, 106496, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x6000) = 0x7f7d6feee000
mmap(0x7f7d6ff08000, 28672, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x20000) = 0x7f7d6ff08000
mmap(0x7f7d6ff10000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x27000) = 0x7f7d6ff10000
mmap(0x7f7d6ff12000, 5640, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f7d6ff12000
close(3)                                = 0
# C 표준 라이브러리
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
# 생략
...
+++ exited with 0 +++
```

## 컴퓨터 시스템의 각 부분에서 얼만큼 지연이 일어날까?

디스크를 순차적으로 읽는건 오래걸리지 않는다.
그렇지만, 임의로 읽을 떈 오래걸린다. 팬이 회전하는 게 가장 큰 지연
그래서 많은 db가 append Only를 채택한다.


## 요약

CPU는 단독으로 동작할 수 있는 구성 요소가 아니다.
폰 노이만 구조에 따르면 적어도 CPU를 지휘하는 기계어와 기계어가 연산하는 데이터는 메모리에 적재되어야 한다.

CPU가 기계어를 실행하려면 데이터는 반드시 메모리에 있어야 하지만, 데이터가 늘 준비되어있지 않을 수 있다(I/O)

외부장치의 속도는 CPU에 비해 매우 느리며, 높은 효율의 입출력 처리와 컴퓨터 내에 이쓴 각양각색의 속도를 가진 하드웨어 리소스를 잘 활용하는가?는 흥미롭다.

# 질문

- 왜 입출력 데이터는 `운영체제를 통한 복사` 라는 layer를 거칠까? 안 거쳐도 되지 않나?
    - 아무튼 거치는 이유는 범용성/보안 등의 이유가 있다. 유저 버퍼에 올리는 경우가 일반적이고, 그렇지 않고 커널 버퍼에만 두고 직접적으로 접근해서 쓰는건 굉장히
      보안적으로는 위험하니까
    - 일반적인 경우 디스크에서 네트워크로 보내기
      [디스크] -> [커널 버퍼] -> [사용자 버퍼] -> [커널 버퍼] -> [네트워크]
    - zero-copy (예: sendfile)
      [디스크] -> [커널 버퍼] -> [네트워크]
    - zero copy라 해서 운영체제를 아예 우회하는 것은 아님. 커널 버퍼를 거치긴 함. 유저 버퍼를 안 거칠 뿐
- 생각보다 select, poll, epoll에 대해서 자세하게 설명해주진 않았음;; 별도로 찾아보고 정리
- 아니 결국엔 동기적으로 처리하면, 결국 멀티스레딩이 되는거 아닌가? 비동기로 하거나?
- 그래서 mmap 어케 쓰는데?

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

int main() {
    // 1. 파일 열기
    int fd = open("example.txt", O_RDONLY);
    if (fd == -1) {
        perror("open");
        exit(EXIT_FAILURE);
    }

    // 2. 파일 크기 얻기
    struct stat sb;
    if (fstat(fd, &sb) == -1) {
        perror("fstat");
        close(fd);
        exit(EXIT_FAILURE);
    }

    // 3. mmap: 파일을 메모리에 매핑
    char *addr = mmap(NULL, sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
    if (addr == MAP_FAILED) {
        perror("mmap");
        close(fd);
        exit(EXIT_FAILURE);
    }

    // 4. 배열처럼 접근하여 출력
    for (off_t i = 0; i < sb.st_size; i++) {
        putchar(addr[i]);  // or write(1, &addr[i], 1);
    }

    // 5. 정리
    munmap(addr, sb.st_size);
    close(fd);

    return 0;
}
```