<br>

## ☁️ Overview

<br>

### 📂 File structure

```bash
malloc-lab
	├── Makefile # 테스트를 실행하는 makefile
	├── README.md
	├── clock.c
	├── clock.h
	├── config.h
	├── fcyc.c
	├── fcyc.h
	├── fsecs.c
	├── fsecs.h
	├── ftimer.c
	├── ftimer.h
	├── mdriver.c
	├── memlib.c # 메모리 시스템의 모델 함수들 정의
	├── memlib.h # memlib.c의 함수에 대한 헤더파일
	├── mm.c # malloc 함수 구현
	├── mm.h # malloc 함수 전역으로 선언
	├── short1-bal.rep
	├── short2-bal.rep
	└── traces
```

<br>

- `memlib.c` : 메모리 시스템의 모델
  - 기존의 시스템 수준의 `malloc` 과 상관없이 돌 수 있도록 하기 위함

<br>

<br>

- 최대 블록 크기 : $2^{32}$
- 32bit 설정 : `gcc -m32`
- 64bit 설정 : `gcc -m64`
  - 코드가 프로세스에서 수정 없이 돌려면 `-m64` 로 돌아야한다.

<br>

<br>

- 할당기의 블록 포맷
<img src="https://user-images.githubusercontent.com/67156494/206301734-a073d4b5-0bb1-4a11-b350-bf841b68f7fc.png" width=400>

![image](https://user-images.githubusercontent.com/67156494/206301795-b904ecb9-25b7-4fe3-9921-925b91ed6dac.png)


- 정렬 조건 : 더블 워드(8byte)
- 최소 블록 크기 : 16바이트

<br>

- 미사용 패딩 블록
- 프롤로그 블록
  - 헤더와 푸터로만 구성된 8바이트 할당 블록
- 에필로그 블록
  - 헤더로만 구성(크기 0)

<br>

- 프롤로그 블록과 에필로그 블록은 연결과정에서 edge 로 인해 발생하는 문제를 해결할 수 있다.

<br>

<br>

## ➕ PLUS

- `size_t`
  - `long unsigned int` 의 별칭
  - 이론상 가장 큰 사이즈를 담을 수 있는 unsigned 데이터 타입

<br>

<br>

---

## 0. _Basic Constants and Macros_

<br>

- source code

```c
/* single word (4) or double word (8) alignment */
#define ALIGNMENT 8

/* rounds up to the nearest multiple of ALIGNMENT */
#define ALIGN(size) (((size) + (ALIGNMENT-1)) & ~0x7)

#define SIZE_T_SIZE (ALIGN(sizeof(size_t)))

/* 기본 상수와 매크로 */
#define WSIZE 4 // 워드(4byte) 와 헤더, 푸터 사이즈
#define DSIZE 8 // 더블 워드(8byte) 사이즈
#define CHUNKSIZE (1 << 12) // 초기 가용 블록과 힙 확장을 위한 기본 크기 (1_000_000_000_000)

#define MAX(x, y) ((x) > (y) ? (x) : (y))

/* 가용 리스트에 접근하고 방문하는 작은 매크로들 */
#define PACK(size, alloc) ((size) | (alloc)) // 크기와 할당 비트를 통합 -> 헤더와 푸터에 저장

#define GET(p) (*(unsigned int *)(p)) // p가 참조하는 워드 리턴 / 인자 p는 (void*) 이므로 역참조는 불가하다.
#define PUT(p, val) (*(unsigned int *)(p) = (val)) // p에 val을 저장

#define GET_SIZE(p) (GET(p) & ~0x7) // 헤더 또는 푸터의 사이즈 리턴
#define GET_ALLOC(p) (GET(p) & 0x1) // 할당 비트 리턴

#define HDRP(bp) ((char *)(bp) - WSIZE) // 블록 헤더를 가리키는 포인터 리턴
#define FTRP(bp) ((char *)(bp) + GET_SIZE(HDRP(bp)) - DSIZE) // 블록 푸터를 가리키는 포인터 리턴
// ↳ HDRP(bp)에 헤더 사이즈(w)도 포함되어 있어서 DSIZE를 빼준다.

#define NEXT_BLKP(bp) ((char *)(bp) + GET_SIZE(((char *)(bp) - WSIZE))) // 다음 블록의 포인터
#define PREV_BLKP(bp) ((char *)(bp) - GET_SIZE(((char *)(bp) - DSIZE))) // 이전 블록의 포인터
```

<br>

- 기본 상수와 매크로 : WSIZE, DSIZE, CHUNKSIZE, MAX
- 가용 리스트에 접근하고 방문하는 작은 매크로 : PACK, GET, PUT, GET_SIZE, GET_ALLOC, HDRP, FTRP, NEXT_BLKP, PREV_BLKP

<br>

- NEXT_BLKP, PREV_BLKP 를 사용하는 방법
  EX. `GET_SIZE(HDRP(NEXT_BLKP(bp)));`

<br>

<br>

---

## 1. [INIT] 초기 가용 리스트 만들기

- 대표 함수(1) : `mm_init()`
  - 설명 : 초기화를 완료하고 할당과 반환 요청을 받을 준비를 완료한다.
  - 리턴
    - `int` : 성공 시 0, 실패 시 -1

↳ 내부 함수(2) : `extend_heap(size)`

- 설명 : 힙의 크기를 확장한다.
- 파라미터
  - `size` : 확장하고자 하는 힙 사이즈 (words)
- 리턴
  - `void*` 확장하며 생긴 가용 블록의 포인터

<br>

<br>

### 1-1. `mm_init()`

<br>

- source code

```c
int mm_init(void)
{
    heap_listp = mem_sbrk(4 * WSIZE); // 초기 상태를 저장하기 위한 사이즈 지정

    if (heap_listp == (void *)-1) // 오류 (sbrk와 동일)
        return -1;

    // 1. 빈 가용 리스트를 만들 수 있도록 초기화한다.
    PUT(heap_listp, 0); // 미사용 패딩 워드
    PUT(heap_listp + (1*WSIZE), PACK(DSIZE, 1)); // 프롤로그 헤더
    PUT(heap_listp + (2*WSIZE), PACK(DSIZE, 1)); // 프롤로그 푸터
    PUT(heap_listp + (3*WSIZE), PACK(0, 1)); // 에필로그 헤더
    heap_listp += (2*WSIZE); // heap_listp 위치를 프롤로그 헤더 뒤로 옮긴다.
    next_heap_listp = heap_listp; // next_fit에서 사용하기 위해 초기 포인터 위치를 넣어준다.

    if (extend_heap(CHUNKSIZE/WSIZE) == NULL) // 2. 이후 일반 블록을 저장하기 위해 힙을 확장한다.
        return -1;

    return 0;
}
```

<br>

- 메모리에서 4워드를 가져와서 빈 가용 리스트를 만들 수 있도록 초기화한다.

<br>

<br>

### 1-2. `extend_heap(size)`

<br>

- source code

```c
static void *extend_heap(size_t words)
{
    char *bp; // 블록 포인터
    size_t size;

    // 더블 워드 정렬 제한 조건을 적용하기 위해 반드시 짝수 사이즈(8byte)의 메모리만 할당한다.(반올림)
    size = (words % 2) ? (words + 1) * WSIZE : words * WSIZE;
    bp = mem_sbrk(size); // mem_sbrk return : 이전 brk(epilogue block 뒤 포인터) 반환
    // ↳ 실제 brk : 확장 후 힙의 맨 끝 포인터

    if ((long)bp ==  -1) // mem_sbrk err return
        return NULL;

    PUT(HDRP(bp), PACK(size, 0)); // size 만큼의 가용 블록의 헤더를 생성한다.
    PUT(FTRP(bp), PACK(size, 0)); // size 만큼의 가용 블록의 푸터를 생성한다.
    PUT(HDRP(NEXT_BLKP(bp)), PACK(0, 1)); // 새로운 에필로그 헤더를 생성한다.

    return coalesce(bp); // 이전 힙이 가용 블록이라면 연결 수행
}
```

<br>

- extend_heap이 호출되는 경우
  - CASE 1. 힙이 초기화 될 때 → ☑️ 현재 경우
  - CASE 2. `mm_malloc` 이 적합한 fit 공간을 찾지 못했을 때

<br>

- 힙은 더블 워드 정렬 제한 조건에 따라 더블 워드 경계에서 시작한다.
- extend_heap으로 가는 모든 호출은 더블 워드(8byte)의 배수인 블록을 리턴한다.
  - `mem_sbrk` 또한 더블 워드로 정렬된 메모리 블록을 리턴한다.

<br>

- 새로 추가된 사이즈 만큼의 가용 블록을 생성한다.
  - 가용 블록의 헤더, 푸터 생성
  - 에필로그의 위치를 확장된 힙 사이즈에 맞춰서 맨 뒤에 새로 생성한다.

<br>

- 만약 이전 힙이 가용 블록이라면 힙을 확장하며 만들어진 현재 가용 블록과 합친다.

<br>

<br>

---

## 2. [COALESCE] 블록 연결

- 대표 함수(1) : `coalesce(bp)`
  - 설명 : 인접 가용 블록들을 경계 태그 연결 기술을 사용해서 통합한다.
  - 리턴
    - `int` : 성공 시 0, 실패 시 -1

<br>

<br>

### 2-1. `coalesce(bp)`

<br>

- source code

```c
static void *coalesce(void *bp)
{
    size_t prev_alloc = GET_ALLOC(FTRP(PREV_BLKP(bp))); // 이전 블록 푸터에서 할당 여부 파악
    size_t next_alloc = GET_ALLOC(HDRP(NEXT_BLKP(bp))); // 다음 블록 헤더에서 할당 여부 파악
    size_t size = GET_SIZE(HDRP(bp)); // (할당 비트를 제외한) 블록 사이즈

    // @pb : 이전 블록, @cb : 현재 블록, @nb : 다음 블록
    // [CASE 1] : pb, nb - 둘 다 할당 상태
    if (prev_alloc && next_alloc)
    {
        return bp;
    }
    // [CASE 2] : pb - 할당 상태 / nb - 가용 상태
    // resize block size : cb + nb
    else if (prev_alloc && !next_alloc)
    {
        size += GET_SIZE(HDRP(NEXT_BLKP(bp))); // 현재 블록 사이즈에 다음 블록 사이즈 더함
        PUT(HDRP(bp), PACK(size, 0));
        PUT(FTRP(bp), PACK(size, 0));
    }
    // [CASE 3] : pb - 가용 상태 / nb - 할당 상태
    // resize block size : pb + cb
    else if (!prev_alloc && next_alloc)
    {
        size += GET_SIZE(HDRP(PREV_BLKP(bp)));
        PUT(FTRP(bp), PACK(size, 0));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
        bp = PREV_BLKP(bp);
    }
    // [CASE 4] : pb - 가용 상태 / nb - 가용 상태
    // resize block size : pb + nb
    else {
        size += GET_SIZE(HDRP(PREV_BLKP(bp))) +
            GET_SIZE(FTRP(NEXT_BLKP(bp)));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
        PUT(FTRP(NEXT_BLKP(bp)), PACK(size, 0));
        bp = PREV_BLKP(bp);
    }
    // next_heap_listp가 속해있는 블록이 이전 블록과 합쳐진다면
    // next_heap_listp에 해당하는 블록을 찾아갈 수 없으므로
    // 새로 next_heap_listp를 이전 블록 위치로 지정해준다.
    // next_heap_listp = bp;
    return bp;
}
```

<br>

- “current block”을 반환하면서 발생할 수 있는 연결 경우의 수
  - B : 이전 블록 / C : 현재 블록 / A : 다음 블록 을 의미한다.
- Case 1. B와 A 모두 할당 블록이다.
  - 변동 없음
- Case 2. B는 할당 블록, A는 가용 블록이다.
  - C의 헤더 : C 크기 + A 크기
  - A의 푸터 : C 크기 + A 크기
- Case 3. B는 가용 블록, A는 할당 블록이다.
  - B의 헤더 : B 크기 + C 크기
  - C의 푸터 : B 크기 + C 크기
- Case 4. B와 A 모두 가용 블록이다.
  - B의 헤더 : B 크기 + A 크기
  - A의 푸터 : B 크기 + A 크기

<br>

<img src="https://user-images.githubusercontent.com/67156494/206301905-0f83743a-16a9-4509-aadf-65c76085cc22.png" width=300>

<br>

위의 4가지 경우에 수에 따라 최종적으로 블록을 통합한다.

<br>

<br>

---

## 3. [FREE] 블록 반환

- 대표 함수(1) : `mm_free(bp)`
  - 설명 : 요청한 블록을 반환한다.
  - 파라미터
    - `void*` : 반환하려는 블록의 포인터

<br>

<br>

### 3-1. `mm_free(bp)`

<br>

- source code

```c
void mm_free(void *ptr)
{
    size_t size = GET_SIZE(HDRP(ptr)); // 반환하려는 블록의 사이즈

    PUT(HDRP(ptr), PACK(size, 0)); // 헤더의 할당 비트를 0으로 설정한다.
    PUT(FTRP(ptr), PACK(size, 0)); // 헤더의 할당 비트를 0으로 설정한다.
    coalesce(ptr); // 인접 가용 블록들에 대한 연결을 수행한다.
}
```

- 이전에 할당한 블록을 해당 함수를 호출해서 반환한다.
- 요청한 블록(bp)를 반환하고 `coalesce(bp)` 를 호출하여 인접 가용 블록을 연결한다.

<br>

<br>

---

## 4. [MALLOC] 블록 할당

- 대표 함수(1) : `mm_malloc(size)`
  - 설명 : 새 블록을 할당한다.
  - 리턴
    - `int` : 성공 시 0, 실패 시 -1

↳ 내부 함수(2) : `find_fit(size)`

- 설명 : first-fit, next-fit, best-fit 방식으로 가용 블록을 탐색한다.(선택)
- 파라미터
  - `size` : 필요한 가용 블록의 사이즈 (byte)

↳ 내부 함수(3) : `extend_heap(size)`

- 설명 : 힙의 크기를 확장한다.
- 파라미터
  - `size` : 확장하고자 하는 힙 사이즈 (words)
- 리턴
  - 확장하며 생긴 가용 블록의 포인터

↳ 내부 함수(4) : `place(bp, size)`

- 설명 : 찾은 가용 블록에 대해 할당 블록과 가용 블록으로 분할한다.
- 파라미터
  - `bp` : 찾은 가용 블록의 포인터
  - `size` : 블록 할당에 필요한 사이즈 (byte)

<br>

<br>

### 4-1. `mm_malloc(size)`

<br>

- default code

```c
void *mm_malloc(size_t size)
{
    int newsize = ALIGN(size + SIZE_T_SIZE);
    void *p = mem_sbrk(newsize);
    if (p == (void *)-1)
    return NULL;
    else {
        *(size_t *)p = size;
        return (void *)((char *)p + SIZE_T_SIZE);
    }
}
```

<br>

- source code

```c
void *mm_malloc(size_t size)
{
    size_t asize;
    size_t extendsize;
    char *bp;

    if (size == 0) // 사이즈 0 요청 처리
        return NULL;

    // 더블 워드 정렬 제한 조건을 만족 시키기위해 더블 워드 단위로 크기를 설정한다.
    if (size <= DSIZE) // 최소 크기인 16바이트(헤더, 푸터, 페이로드 포함)로 설정한다.
        asize = 2 * DSIZE;
    else // 8바이트를 넘는 요청 처리
        asize = DSIZE * ((size + (DSIZE) + (DSIZE - 1)) / DSIZE);

    // 조정한 크기(asize)에 대해 가용 리스트에서 적절한 가용 블록을 찾는다.
    bp = next_fit(asize); // Choice fit-method : first_fit, next_fit, best_fit

    if (bp != NULL) {
        place(bp, asize); // 초과부분을 분할한다.
        next_heap_listp = bp;
        return bp; // 새롭게 할당한 블록을 리턴한다.
    }

    // 적합한 fit 공간을 찾지 못했을 때 heap을 확장한다.
    extendsize = MAX(asize, CHUNKSIZE);
    if ((bp = extend_heap(extendsize/WSIZE)) == NULL)
        return NULL;
    place(bp, asize);
    next_heap_listp = bp;
    return bp;
}
```

1. 사이즈 0에 대한 메모리 할당 요청이 들어오면 NULL을 리턴한다.

   - _기존 시스템 `malloc` 함수의 리턴과 동일하다._

2. 사이즈 조정 `asize`

   1. 만약 최소 크기인 16바이트보다 작은 사이즈의 요청이 들어오면 할당할 메모리 사이즈를 16바이트로 조정한다.
   2. 8바이트를 넘는 요청에 대해서는 정렬 조건을 만족하기 위해 더블 워드 단위로 사이즈를 조정한다.

3. `find_fit(asize)` 을 이용해 가용 리스트에서 적절한 가용 블록을 찾는다.

   1. 3가지 방법(first-fit, next-fit, best-fit) 중 한 가지 방법을 선택하여 함수명을 기입한다.
   2. `find_fit(asize)` 를 통해 적절한 공간을 찾았으면 새롭게 블록을 할당한다.

4. 적절한 공간을 찾지 못했으면 heap을 확장한다.
   1. 만약 힙을 확장할 수 없는 상태라면 NULL을 리턴한다.
   2. 힙을 확장할 수 있다면 블록을 분할하여 할당한다.

<br>

<br>

### 4-2. `find_fit(size)`

1.first-fit

```c
static void *first_fit(size_t asize)
{
    void *bp;

    // 에필로그 블록의 헤더를 0으로 넣어줬으므로 에필로그 블록을 만날 때까지 탐색을 진행한다.
    for (bp = heap_listp; GET_SIZE(HDRP(bp)) > 0; bp = NEXT_BLKP(bp))
        if (!GET_ALLOC(HDRP(bp)) && (asize <= GET_SIZE(HDRP(bp))))
            return bp;

    return NULL;
}
```

2.next-fit

```c
static void *next_fit(size_t asize)
{
    char *bp;

    // next_fit 포인터에서 탐색을 시작한다.
    for (bp = NEXT_BLKP(next_heap_listp); GET_SIZE(HDRP(bp)) > 0; bp = NEXT_BLKP(bp))
        if (!GET_ALLOC(HDRP(bp)) && (asize <= GET_SIZE(HDRP(bp))))
            return bp;

    for (bp = heap_listp; bp <= next_heap_listp; bp = NEXT_BLKP(bp))
        if (!GET_ALLOC(HDRP(bp)) && (asize <= GET_SIZE(HDRP(bp))))
            return bp;

    return NULL;
}
```

3.best-fit

```c
static void *best_fit(size_t asize)
{
    void *bp;
    void *best_fit = NULL;

    for (bp = heap_listp; GET_SIZE(HDRP(bp)) > 0; bp = NEXT_BLKP(bp))
        if (!GET_ALLOC(HDRP(bp)) && (asize <= GET_SIZE(HDRP(bp))))
            // 기존에 할당하려던 공간보다 더 최적의 공간이 나타났을 경우 리턴 블록 포인터 갱신
            if (!best_fit || GET_SIZE(HDRP(bp)) < GET_SIZE(HDRP(best_fit)))
                best_fit = bp;

    return best_fit;
}
```

<br>

<br>

### 4-3. `extend_heap(size)`

<br>

- extend_heap이 호출되는 경우
  - CASE 1. 힙이 초기화 될 때
  - CASE 2. `mm_malloc` 이 적합한 fit 공간을 찾지 못했을 때 → ☑️ 현재 경우

<br>

<2-1>의 함수 설명과 동일하다.

<br>

<br>

### 4-4. `place(bp, size)`

<br>

- source code

```c
static void place(void *bp, size_t asize)
{
    size_t csize = GET_SIZE(HDRP(bp)); // 가용 블록의 사이즈

    if ((csize - asize) >= (2 * DSIZE)) {
        // asize만큼의 할당 블록을 생성한다.
        PUT(HDRP(bp), PACK(asize, 1));
        PUT(FTRP(bp), PACK(asize, 1));
        bp = NEXT_BLKP(bp);
        next_heap_listp = bp; // 분할 이후 그 다음 블록
        // 새로운 할당 블록(전체 가용 블록 - 할당 블록)의 뒷 부분을 가용 블록으로 만든다.
        PUT(HDRP(bp), PACK(csize - asize, 0));
        PUT(FTRP(bp), PACK(csize - asize, 0));
    } else {
        PUT(HDRP(bp), PACK(csize, 1));
        PUT(FTRP(bp), PACK(csize, 1));
        next_heap_listp = NEXT_BLKP(bp); // 분할 이후 그 다음 블록
    }
}
```

<br>

- 가용 공간을 찾고 해당 가용 블록이 할당하려는 메모리 공간보다 클 경우 할당 블록과 가용 블록으로 블록을 분할한다.

<br>

1. 할당하려는 메모리 공간이 만약 _헤더와 푸터 오버헤드 사이즈(8byte) + 데이터(최소 8byte)_ 보다 크거나 같은 경우에는 앞 부분을 할당 공간으로, 뒷 부분을 가용 공간을 분할한다.
2. 만약 할당하려는 메모리 공간이 _헤더와 푸터 오버헤드 사이즈(8byte) + 데이터(최소 8byte)_ 보다 작은 경우에는 가용 블록 전체를 할당 블록으로 만든다.

<br>

<br>

---

## 5. [REALLOC] 블록 재할당

- 대표 함수(1) : `mm_realloc(bp, size)`
  - 설명 : 이전에 할당한 메모리의 크기를 재조정한다.
  - 파라미터
    - `bp` : 재조정하려는 공간의 포인터
    - `size` : 재조정하려는 사이즈
  - 리턴
    - `void*` : 재조정한 메모리의 포인터

<br>

<br>

### 5-1. `mm_realloc(bp, size)`

<br>

- default code

```c
void *mm_realloc(void *ptr, size_t size)
{
    void *oldptr = ptr;
    void *newptr;
    size_t copySize;

    newptr = mm_malloc(size);
    if (newptr == NULL)
        return NULL;
    copySize = *(size_t *)((char *)oldptr - SIZE_T_SIZE);
    if (size < copySize)
        copySize = size;
    memcpy(newptr, oldptr, copySize);
    mm_free(oldptr);
    return newptr;
}
```

<br>

- source code

```c
void *mm_realloc(void *ptr, size_t size)
{
    void *oldptr = ptr; // 이전 포인터
    void *newptr; // 새로 메모리 할당할포인터

    size_t originsize = GET_SIZE(HDRP(oldptr)); // 원본 사이즈
    size_t newsize = size + DSIZE; // 새 사이즈

    // size 가 더 작은 경우
    if (newsize <= originsize) {
        return oldptr;
    } else {
        size_t addSize = originsize + GET_SIZE(HDRP(NEXT_BLKP(oldptr))); // 추가 사이즈 -> 헤더 포함 사이즈
        if (!GET_ALLOC(HDRP(NEXT_BLKP(oldptr))) && (newsize <= addSize)){ // 가용 블록이고 사이즈 충분
            PUT(HDRP(oldptr), PACK(addSize, 1)); // 새로운 헤더
            PUT(FTRP(oldptr), PACK(addSize, 1)); // 새로운 푸터
            return oldptr;
        } else {
            newptr = mm_malloc(newsize);
            if (newptr == NULL)
                return NULL;
            memmove(newptr, oldptr, newsize); // memcpy 사용 시, memcpy-param-overlap 발생
            mm_free(oldptr);
            return newptr;
        }
    }
}
```

<br>

- 기존 방법의 경우, 무조건 새로운 메모리를 할당한 뒤 데이터를 복사하는 방식으로 구현되어져 있었다.
  ∴ 반복적인 메모리 할당으로 코드의 효율성이 떨어진다.

<br>

- 개선

  - 새로 크기를 조정하려는 블록의 다음 블록이 가용 블록이라면 새로 메모리 할당을 안해줘도 된다.
  - 단순히 헤더, 푸터의 사이즈 정보만 갱신해준다.

  - `memcpy` 를 쓸 경우, 메모리 겹침 관련 에러가 나서 `memmove` 로 바꿔줬다!

<br>

<img src="https://user-images.githubusercontent.com/67156494/206302007-3e4d1d98-cb13-4c19-b618-4f200debe861.png" width=600>


- `memcpy` 🆚 `memmove`

[오류: memcpy-param-overlap](https://learn.microsoft.com/ko-kr/cpp/sanitizers/error-memcpy-param-overlap?view=msvc-170)

<br>

<br>

<br>

---

## 🎉 Result

- CASE 1. implicit, first-fit (54/100)

```bash
Results for mm malloc:
trace  valid  util     ops      secs  Kops
 0       yes   99%    5694  0.008875   642
 1       yes   99%    5848  0.008249   709
 2       yes   99%    6648  0.013674   486
 3       yes  100%    5380  0.010171   529
 4       yes   66%   14400  0.000139103523
 5       yes   92%    4800  0.009748   492
 6       yes   92%    4800  0.008707   551
 7       yes   55%   12000  0.233884    51
 8       yes   51%   24000  0.388684    62
 9       yes   27%   14401  0.119375   121
10       yes   34%   14401  0.003534  4075
Total          74%  112372  0.805040   140

Perf index = 44 (util) + 9 (thru) = 54/100
```

<br>

- CASE 2. implicit, next-fit (82/100)

```bash
Results for mm malloc:
trace  valid  util     ops      secs  Kops
 0       yes   85%    5694  0.000114 50123
 1       yes   86%    5848  0.000118 49643
 2       yes   87%    6648  0.000149 44617
 3       yes   90%    5380  0.000123 43883
 4       yes   66%   14400  0.000152 94737
 5       yes   85%    4800  0.000740  6486
 6       yes   82%    4800  0.000761  6306
 7       yes   55%   12000  0.000191 62794
 8       yes   51%   24000  0.000280 85561
 9       yes   27%   14401  0.122128   118
10       yes   47%   14401  0.003468  4152
Total          69%  112372  0.128224   876

Perf index = 42 (util) + 40 (thru) = 82/100
```

<br>

- CASE 3. implicit, next-fit, improved realloc (86/100)

```bash
Results for mm malloc:
trace  valid  util     ops      secs  Kops
 0       yes   90%    5694  0.002171  2623
 1       yes   91%    5848  0.001442  4057
 2       yes   95%    6648  0.004505  1476
 3       yes   97%    5380  0.004612  1166
 4       yes   66%   14400  0.000159 90509
 5       yes   92%    4800  0.005821   825
 6       yes   90%    4800  0.005282   909
 7       yes   55%   12000  0.023977   500
 8       yes   51%   24000  0.010755  2231
 9       yes   76%   14401  0.000260 55388
10       yes   46%   14401  0.000147 98300
Total          77%  112372  0.059131  1900

Perf index = 46 (util) + 40 (thru) = 86/100
```

<br>

<br>

<br>
