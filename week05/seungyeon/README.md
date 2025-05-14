## Malloc

- `malloc`은 C 표준 라이브러리 함수
- 내부적으로 시스템콜을 사용하여 커널과 상호작용해서 메모리를 확보함
- `malloc` 자체는 라이브러리 함수


[glibc malloc implementation](https://github.com/bminor/glibc/blob/master/malloc/malloc.c)

```C
/* ----------- Routines dealing with system allocation -------------- */

/*
   sysmalloc handles malloc cases requiring more memory from the system.
   On entry, it is assumed that av->top does not have enough
   space to service request for nb bytes, thus requiring that av->top
   be extended or replaced.
 */

static void *
sysmalloc_mmap (INTERNAL_SIZE_T nb, size_t pagesize, int extra_flags, mstate av)
{
  long int size;

  /*
    Round up size to nearest page.  For mmapped chunks, the overhead is one
    SIZE_SZ unit larger than for normal chunks, because there is no
    following chunk whose prev_size field could be used.

    See the front_misalign handling below, for glibc there is no need for
    further alignments unless we have have high alignment.
   */
  if (MALLOC_ALIGNMENT == CHUNK_HDR_SZ)
    size = ALIGN_UP (nb + SIZE_SZ, pagesize);
  else
    size = ALIGN_UP (nb + SIZE_SZ + MALLOC_ALIGN_MASK, pagesize);

  /* Don't try if size wraps around 0.  */
  if ((unsigned long) (size) <= (unsigned long) (nb))
    return MAP_FAILED;

  char *mm = (char *) MMAP (NULL, size,
			    mtag_mmap_flags | PROT_READ | PROT_WRITE,
			    extra_flags);
  if (mm == MAP_FAILED)
    return mm;

#ifdef MAP_HUGETLB
  if (!(extra_flags & MAP_HUGETLB))
    madvise_thp (mm, size);
#endif

  __set_vma_name (mm, size, " glibc: malloc");

  /*
    The offset to the start of the mmapped region is stored in the prev_size
    field of the chunk.  This allows us to adjust returned start address to
    meet alignment requirements here and in memalign(), and still be able to
    compute proper address argument for later munmap in free() and realloc().
   */

  INTERNAL_SIZE_T front_misalign; /* unusable bytes at front of new space */

  if (MALLOC_ALIGNMENT == CHUNK_HDR_SZ)
    {
      /* For glibc, chunk2mem increases the address by CHUNK_HDR_SZ and
	 MALLOC_ALIGN_MASK is CHUNK_HDR_SZ-1.  Each mmap'ed area is page
	 aligned and therefore definitely MALLOC_ALIGN_MASK-aligned.  */
      assert (((INTERNAL_SIZE_T) chunk2mem (mm) & MALLOC_ALIGN_MASK) == 0);
      front_misalign = 0;
    }
  else
    front_misalign = (INTERNAL_SIZE_T) chunk2mem (mm) & MALLOC_ALIGN_MASK;

  mchunkptr p;                    /* the allocated/returned chunk */

  if (front_misalign > 0)
    {
      ptrdiff_t correction = MALLOC_ALIGNMENT - front_misalign;
      p = (mchunkptr) (mm + correction);
      set_prev_size (p, correction);
      set_head (p, (size - correction) | IS_MMAPPED);
    }
  else
    {
      p = (mchunkptr) mm;
      set_prev_size (p, 0);
      set_head (p, size | IS_MMAPPED);
    }

  /* update statistics */
  int new = atomic_fetch_add_relaxed (&mp_.n_mmaps, 1) + 1;
  atomic_max (&mp_.max_n_mmaps, new);

  unsigned long sum;
  sum = atomic_fetch_add_relaxed (&mp_.mmapped_mem, size) + size;
  atomic_max (&mp_.max_mmapped_mem, sum);

  check_chunk (av, p);

  return chunk2mem (p);
}
```


## 동적 메모리 할당과 관련된 함수들

- `malloc()`
    - size만큼 메모리 동적 할당
    - 매개변수: 할당할 메모리의 크기(size_t size)​
    - 반환값: 할당된 메모리의 시작 주소를 반환. 실패 시 NULL 반환.
```C
  int *arr;
  arr = (int *)malloc(sizeof(int) * 5); // 5개의 int 크기만큼 메모리 할당
```
 
- `calloc()`
    - malloc과 유사하지만, 할당된 메모리를 모두 0으로 초기화
    - 매개변수: 할당할 요소의 개수(count), 각 요소의 크기(size)
    - 반환값: 할당된 메모리의 시작 주소를 반환. 실패 시 NULL 반환.
```C
  int *arr;
  arr = (int *)calloc(5, sizeof(int)); // 5개의 int 크기만큼 메모리를 0으로 초기화하며 할당

```

- `realloc()`
    - 이미 할당된 메모리 블록의 크기를 변경
    - 매개변수: 기존에 할당된 메모리 포인터(ptr), 새로운 메모리 크기(newsize)
    - 반환값: 재할당된 메모리의 시작 주소를 반환. 실패 시 NULL 반환.
```C
  arr = (int *)realloc(arr, sizeof(int) * 10); // 기존 메모리를 10개의 int 크기로 재할당
```

- `free()`
    - 동적으로 할당된 메모리 해제
 

## 시스템콜 종류들

- brk 및 sbrk: 데이터 세그먼트의 크기를 변경하여 힙 영역을 확장하거나 축소
- mmap: 새로운 가상 메모리 영역을 요청하여 메모리를 매핑

### 데이터 세그먼트

프로세스는 세그먼트의 집합으로 구성됨
세그먼트란 여러 정보나 데이터를 기억하는 메모리 공간

- 코드 세그먼트
- 데이터 세그먼트
- 스택 세그먼트


## 참고한 기타 자료들

[BJ.16 인터럽트와 시스템 콜을 설명합니다! 유저 모드, 커널 모드도 설명합니다. 시스템 콜와 프로그래밍 언어의 상관 관계도 설명합니다!](https://youtu.be/v30ilCpITnY?feature=shared)
[[Malloc-lab] malloc, calloc, realloc, free, brk, sbrk, mmap 기본 개념 정리](https://bo5mi.tistory.com/162)
[한 장으로 정리하는 Malloc-Lab 기초 다지기](https://velog.io/@ngngs/%ED%95%9C-%EC%9E%A5%EC%9C%BC%EB%A1%9C-%EC%A0%95%EB%A6%AC%ED%95%98%EB%8A%94-Malloc-Lab-%EA%B8%B0%EC%B4%88-%EB%8B%A4%EC%A7%80%EA%B8%B0)


