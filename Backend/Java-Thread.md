---
created: 2026-11-23
updated: 2026-11-23
domain: Backend
tags: [java, thread, concurrency, jvm, executor, virtual-thread]
---

# Java Thread (자바 쓰레드)

## 📖 핵심 개념 및 동작 원리

> [!tip] 한 줄 요약
> 자바의 쓰레드는 **JVM이 OS 네이티브 쓰레드에 1:1로 매핑**하는 실행 단위입니다(Java 21부터는 Virtual Thread로 M:N도 가능). 쓰레드는 **Stack/PC는 각자 갖고, Heap/Method Area는 공유**하기 때문에 "공유 상태"를 어떻게 다루느냐가 동시성 프로그래밍의 전부입니다.

---

### 1. 프로세스 vs 쓰레드, 그리고 JVM 메모리

| 구분 | 프로세스 | 쓰레드 |
|---|---|---|
| 메모리 | 독립적인 주소 공간 | **Heap·Method Area 공유**, Stack은 개별 |
| 생성 비용 | 큼 | 상대적으로 작음(그래도 OS 쓰레드는 ~1MB 스택) |
| 통신 | IPC 필요 | 공유 메모리로 즉시 (대신 동기화 문제 발생) |
| 장애 전파 | 격리됨 | 한 쓰레드의 OOM/치명적 오류가 프로세스 전체에 영향 |

JVM 기준 쓰레드별 소유 영역:

| 영역 | 공유 여부 | 내용 |
|---|---|---|
| JVM Stack | **쓰레드별** | 지역 변수, 파라미터, 호출 프레임 |
| PC Register | **쓰레드별** | 현재 실행 중인 바이트코드 주소 |
| Native Method Stack | **쓰레드별** | JNI 호출 스택 |
| Heap | 공유 | 객체 인스턴스, 배열 |
| Method Area(Metaspace) | 공유 | 클래스 메타데이터, static 변수, 상수 풀 |

> [!important] 동시성 버그의 근원
> 지역 변수는 Stack에 있어 **본질적으로 thread-safe**하지만, 인스턴스 필드·static 필드는 Heap/Method Area에 있어 **여러 쓰레드가 동시에 건드릴 수 있습니다.**

---

### 2. 쓰레드 생성 방법

```java
// 1) Thread 상속 — 단일 상속을 써버리므로 비권장
class MyThread extends Thread {
    @Override public void run() { System.out.println("run"); }
}
new MyThread().start();

// 2) Runnable 구현 — 반환값 없음, 예외 던지기 불가
Runnable task = () -> System.out.println("run");
new Thread(task).start();

// 3) Callable + ExecutorService — 반환값 O, 예외 전파 O (실무 표준)
ExecutorService pool = Executors.newFixedThreadPool(4);
Future<Integer> f = pool.submit(() -> 1 + 1);
Integer result = f.get();   // 블로킹 대기
pool.shutdown();

// 4) Virtual Thread (Java 21+)
Thread.startVirtualThread(() -> System.out.println("virtual"));
```

> [!warning] `start()` vs `run()`
> `run()`을 직접 호출하면 **새 쓰레드가 생기지 않고 현재 쓰레드에서 그냥 메서드 실행**됩니다. 반드시 `start()`를 호출해야 하며, 같은 Thread 객체에 `start()`를 두 번 호출하면 `IllegalThreadStateException`이 발생합니다.

| 방식 | 반환값 | 체크 예외 | 비고 |
|---|---|---|---|
| `Runnable` | X | 던질 수 없음 | 함수형 인터페이스 |
| `Callable<V>` | O | `throws Exception` 가능 | `Future`로 결과 수신 |

---

### 3. 쓰레드 생명주기 (Thread.State)

```
NEW ──start()──▶ RUNNABLE ◀──────────────┐
                   │  │                  │
   synchronized 획득 실패 │  │ wait()/join()/   │ notify()/
                   ▼  ▼ sleep()          │ 시간만료/
               BLOCKED  WAITING          │ interrupt()
                        TIMED_WAITING ───┘
                   │
                   ▼
              TERMINATED
```

| 상태 | 진입 조건 |
|---|---|
| `NEW` | 객체 생성, `start()` 전 |
| `RUNNABLE` | 실행 중 또는 CPU 스케줄링 대기 (Java는 Running을 구분하지 않음) |
| `BLOCKED` | **monitor lock**(synchronized) 획득 대기 |
| `WAITING` | `wait()`, `join()`, `LockSupport.park()` — 무기한 대기 |
| `TIMED_WAITING` | `sleep(n)`, `wait(n)`, `join(n)` — 시간 제한 대기 |
| `TERMINATED` | `run()` 종료 또는 예외로 중단 |

> [!note] `BLOCKED`와 `WAITING` 구분이 실무에서 중요합니다. 스레드 덤프에서 `BLOCKED`가 많으면 **락 경합**, `WAITING`이 많으면 **외부 I/O·커넥션 풀 고갈**을 의심합니다.

---

### 4. 동시성 문제의 3대 원인

#### (1) 가시성(Visibility)
각 쓰레드는 CPU 캐시에 값을 들고 있어, 다른 쓰레드의 변경을 못 볼 수 있습니다.

```java
// 문제: flag 변경이 다른 쓰레드에 영원히 안 보일 수 있음
private boolean running = true;        // ❌
private volatile boolean running = true; // ✅ 메인 메모리 직접 읽기/쓰기
```

#### (2) 원자성(Atomicity)
`count++`는 **읽기 → 증가 → 쓰기** 3단계라 중간에 끼어들 수 있습니다.

```java
count++;                          // ❌ volatile로도 해결 안 됨
synchronized { count++; }         // ✅
atomicInteger.incrementAndGet();  // ✅ CAS 기반, 더 빠름
```

#### (3) 재배치(Reordering)
컴파일러·CPU가 성능을 위해 명령 순서를 바꿉니다. → **happens-before** 규칙(JMM)으로 보장.

> [!important] volatile의 한계
> `volatile`은 **가시성과 순서는 보장하지만 원자성은 보장하지 않습니다.** 카운터 증가 같은 복합 연산에는 `synchronized`나 `AtomicXXX`를 써야 합니다.

---

### 5. 동기화 도구 비교

| 도구 | 특징 | 사용 시점 |
|---|---|---|
| `synchronized` | JVM 내장, 블록 벗어나면 자동 해제 | 단순 임계 영역 |
| `ReentrantLock` | `tryLock()`, 타임아웃, 공정성, 인터럽트 가능 | 세밀한 제어 필요 시 |
| `ReadWriteLock` | 읽기 다중 허용 / 쓰기 배타 | 읽기 >> 쓰기 비율 |
| `Atomic*` | CAS 기반 락프리 | 단일 변수 카운터/플래그 |
| `Concurrent*` 컬렉션 | `ConcurrentHashMap` 등 | 공유 자료구조 |
| `CountDownLatch` | N개 완료 대기(1회용) | 초기화 대기, 병렬 작업 취합 |
| `Semaphore` | 동시 접근 개수 제한 | 리소스 쓰로틀링 |
| `CyclicBarrier` | N개가 모두 도달하면 재개(재사용) | 단계별 병렬 연산 |

```java
// synchronized — 메서드 / 블록
public synchronized void add() { ... }   // 인스턴스 락(this)
public static synchronized void f() {}   // 클래스 락(Class 객체)
synchronized (lockObj) { ... }           // 특정 객체 락 (범위 최소화 권장)

// ReentrantLock — finally에서 반드시 unlock
Lock lock = new ReentrantLock();
lock.lock();
try { ... } finally { lock.unlock(); }

// tryLock으로 데드락 회피
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try { ... } finally { lock.unlock(); }
}
```

> [!warning] 피해야 할 패턴
> - `synchronized(this)`로 외부에 노출된 객체 잠금 → 외부에서 같은 락을 걸면 교착 가능. **private final Object lock = new Object()** 권장
> - `String` 리터럴이나 `Integer` 박싱 객체를 락으로 사용 → 상수 풀 공유로 전역 락이 됨
> - 임계 영역 안에서 I/O나 외부 API 호출 → 락 보유 시간 폭증

---

### 6. 쓰레드 풀 (ExecutorService)

매 요청마다 쓰레드를 만들면 생성 비용 + 컨텍스트 스위칭 + OOM 위험이 있습니다. 풀로 재사용합니다.

```java
ExecutorService pool = new ThreadPoolExecutor(
    10,                              // corePoolSize
    50,                              // maximumPoolSize
    60L, TimeUnit.SECONDS,           // keepAliveTime (초과 쓰레드 유휴 제한)
    new LinkedBlockingQueue<>(100),  // workQueue
    new ThreadPoolExecutor.CallerRunsPolicy() // 포화 정책
);
```

**작업 투입 순서 (중요)**
1. 현재 쓰레드 수 < `corePoolSize` → **새 쓰레드 생성**
2. core가 찼으면 → **큐에 적재**
3. 큐도 꽉 차면 → `maximumPoolSize`까지 **쓰레드 추가 생성**
4. 그래도 안 되면 → **거부 정책(RejectedExecutionHandler)** 실행

| 포화 정책 | 동작 |
|---|---|
| `AbortPolicy`(기본) | `RejectedExecutionException` 던짐 |
| `CallerRunsPolicy` | 호출한 쓰레드가 직접 실행 → 자연스러운 백프레셔 |
| `DiscardPolicy` | 조용히 버림 |
| `DiscardOldestPolicy` | 큐의 가장 오래된 작업 버리고 투입 |

> [!warning] `Executors` 팩토리 메서드 주의
> - `newFixedThreadPool` / `newSingleThreadExecutor` → **무한 큐(LinkedBlockingQueue)** → 작업이 쌓여 **OOM**
> - `newCachedThreadPool` → **무한 쓰레드 생성** → OOM
> 실무에서는 `ThreadPoolExecutor`를 직접 설정하는 것이 안전합니다.

**풀 크기 가늠**
- CPU 바운드: `코어 수 + 1`
- I/O 바운드: `코어 수 × (1 + 대기시간/연산시간)`

**종료**

```java
pool.shutdown();                                  // 새 작업 거부, 기존 작업은 완료
if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
    pool.shutdownNow();                           // 인터럽트로 강제 중단 시도
}
```

---

### 7. CompletableFuture (비동기 조합)

```java
CompletableFuture
    .supplyAsync(() -> callApiA(), pool)          // 비동기 실행
    .thenApply(a -> transform(a))                 // 결과 변환
    .thenCompose(b -> callApiB(b))                // 다른 Future와 연결(flatMap)
    .thenCombine(other, (x, y) -> merge(x, y))    // 두 Future 병합
    .exceptionally(ex -> fallback())              // 예외 처리
    .thenAccept(System.out::println);

// 여러 작업 전부 대기
CompletableFuture.allOf(f1, f2, f3).join();
```

> [!tip] `Future.get()`은 블로킹이라 비동기의 장점을 죽입니다. 콜백 체이닝(`thenXXX`)으로 이어 붙이세요. 그리고 `Async` 접미사 메서드에 **커스텀 풀을 명시**하지 않으면 공용 `ForkJoinPool.commonPool`을 쓰게 되어 전역 영향이 생깁니다.

---

### 8. Virtual Thread (Java 21+, Project Loom)

| 구분 | Platform Thread | Virtual Thread |
|---|---|---|
| 매핑 | OS 쓰레드 1:1 | JVM이 캐리어 쓰레드에 **M:N** 매핑 |
| 생성 비용 | 비쌈 (~1MB 스택) | 매우 저렴 (힙에 스택 저장, 수백만 개 가능) |
| 블로킹 I/O | OS 쓰레드가 점유되어 낭비 | **언마운트**되어 캐리어 쓰레드 반환 |
| 적합 작업 | CPU 바운드 | **I/O 바운드** |

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i ->
        executor.submit(() -> { httpCall(); return null; }));
}
```

> [!important] Virtual Thread 주의사항
> - **풀링하지 마세요.** 작업당 하나씩 새로 만드는 것이 설계 의도입니다.
> - `synchronized` 블록 안에서 블로킹하면 **pinning**이 발생해 캐리어 쓰레드가 묶입니다 → `ReentrantLock` 사용 권장 (Java 24에서 상당 부분 개선)
> - CPU 바운드 작업에는 이점이 없습니다.

---

### 9. 실무 트러블슈팅

**데드락 4대 조건** — 하나만 깨도 해결됩니다.
1. 상호 배제 2. 점유와 대기 3. 비선점 4. **순환 대기** ← 보통 "락 획득 순서를 전역에서 통일"해서 해결

**스레드 덤프 분석**
```bash
jps -l                  # PID 확인
jstack <PID> > dump.txt # 스레드 덤프
# 또는
jcmd <PID> Thread.print
```
- `deadlock` 키워드 검색 → JVM이 자동 탐지해 표시
- `BLOCKED` 다수 + 동일 락 주소 → 경합 지점 특정

**ThreadLocal 메모리 누수**
```java
try {
    ctx.set(value);
    ...
} finally {
    ctx.remove();   // ✅ 풀 쓰레드는 재사용되므로 반드시 정리
}
```
쓰레드 풀에서 `remove()`를 빠뜨리면 **이전 요청 데이터가 다음 요청에 보이는 보안 사고**로도 이어집니다.

**예외 처리**
```java
// run() 안의 예외는 호출 쓰레드로 전파되지 않음
Thread.setDefaultUncaughtExceptionHandler((t, e) -> log.error("{} died", t.getName(), e));
// ExecutorService.submit()의 예외는 Future에 갇히므로 반드시 get() 하거나 작업 내부에서 try-catch
```

---

> [!tip] 핵심 정리
> 1. `start()`가 새 쓰레드, `run()`은 그냥 메서드 호출
> 2. `volatile` = 가시성만, `synchronized`/`Atomic` = 원자성까지
> 3. `Executors` 팩토리는 무한 큐/무한 쓰레드 → OOM. `ThreadPoolExecutor` 직접 설정
> 4. `ThreadLocal`은 `finally`에서 `remove()`
> 5. I/O 바운드 대량 처리는 Virtual Thread, CPU 바운드는 여전히 Platform Thread

관련 키워드: [[JVM-Memory-Structure]] · [[Java-Memory-Model]] · [[Deadlock]] · [[CompletableFuture]] · [[Virtual-Thread]] · [[Spring-Async]]

---

## 💬 Q&A 기록

### Q1. Java 언어에서 쓰레드에 대해 자세히 알려줘

**A.** 위 핵심 개념 섹션 1~9가 전체 답변입니다. 학습 순서로 압축하면:

| 단계 | 알아야 할 것 |
|---|---|
| 기초 | Stack은 쓰레드별 / Heap은 공유 → 공유 상태가 문제의 근원 |
| 생성 | `Runnable`(반환X) vs `Callable`(반환O) vs `ExecutorService`(실무 표준) |
| 생명주기 | `BLOCKED`(락 경합) vs `WAITING`(I/O·대기) 구분이 장애 분석의 출발점 |
| 동시성 3요소 | 가시성(`volatile`) · 원자성(`synchronized`/`Atomic`) · 재배치(happens-before) |
| 쓰레드 풀 | core → 큐 → max → 거부 정책 순서. `Executors` 팩토리는 OOM 위험 |
| 비동기 | `CompletableFuture`로 논블로킹 체이닝, 커스텀 풀 명시 |
| 최신 | Java 21 Virtual Thread — I/O 바운드에 한해 게임 체인저, 풀링 금지 |

> [!note] 가장 흔한 실무 실수 3가지
> 1. `Executors.newFixedThreadPool()`의 무한 큐로 인한 OOM
> 2. 쓰레드 풀 환경에서 `ThreadLocal.remove()` 누락 → 데이터 오염/누수
> 3. `volatile`로 카운터를 보호할 수 있다고 착각 → 원자성 미보장

---

### Q2. 루프를 아주 많이 돌아야 하는 작업이 있어서 그동안 다른 작업을 못 하는데, 쓰레드로 해결할 수 있나?

**A.** **네, 다만 "무엇이 해결되는지"를 정확히 나눠서 봐야 합니다.** 쓰레드가 해결해 주는 것과 해결해 주지 못하는 것이 분명히 다릅니다.

#### 1) 두 가지 목표를 먼저 구분

| 목표                      | 의미                                                 | 쓰레드로 해결되나?                       |
| ----------------------- | -------------------------------------------------- | -------------------------------- |
| **응답성(Responsiveness)** | 무거운 루프가 도는 동안에도 다른 작업(UI, API 응답, 다른 요청)이 멈추지 않는 것 | ✅ **확실히 해결됨**                    |
| **처리량(Throughput)**     | 루프 자체가 더 빨리 끝나는 것                                  | ⚠️ **코어 수만큼만** 가능, 그 이상은 오히려 느려짐 |

질문하신 "그동안 다른 작업을 진행할 수 없다"는 **응답성 문제**이고, 이건 쓰레드로 바로 해결됩니다. 무거운 루프를 별도 쓰레드로 보내면 메인 쓰레드는 즉시 다음 일을 합니다.

```java
ExecutorService pool = Executors.newFixedThreadPool(2);

// 무거운 루프를 백그라운드로 분리
Future<Long> future = pool.submit(() -> {
    long sum = 0;
    for (long i = 0; i < 10_000_000_000L; i++) sum += i;
    return sum;
});

// 메인 쓰레드는 막히지 않고 계속 진행
doOtherWork();

long result = future.get();  // 결과가 필요한 시점에만 대기
```

> [!important] 핵심
> `future.get()`을 **바로 호출하면 의미가 없습니다.** 그 순간 블로킹되어 단일 쓰레드와 똑같아집니다. "결과가 실제로 필요해지는 가장 늦은 시점"까지 미루거나, 콜백(`CompletableFuture.thenAccept`)으로 넘기세요.

#### 2) 루프 자체를 빠르게 하려면 — 분할(Parallelism)

작업을 쪼갤 수 있다면 코어 수만큼 나눠서 동시에 돌립니다.

```java
// 방법 A: parallelStream — 가장 간단
long sum = LongStream.range(0, 10_000_000_000L)
                     .parallel()
                     .sum();

// 방법 B: 수동 분할
int n = Runtime.getRuntime().availableProcessors();
ExecutorService pool = Executors.newFixedThreadPool(n);
List<Future<Long>> futures = new ArrayList<>();
long chunk = total / n;
for (int i = 0; i < n; i++) {
    long start = i * chunk, end = (i == n - 1) ? total : start + chunk;
    futures.add(pool.submit(() -> compute(start, end)));  // 구간별 부분합
}
long sum = 0;
for (Future<Long> f : futures) sum += f.get();            // 취합
pool.shutdown();
```

> [!warning] 쓰레드를 늘린다고 무한히 빨라지지 않습니다
> **CPU 바운드 작업의 상한선은 물리 코어 수**입니다. 8코어에서 100개 쓰레드를 만들면 컨텍스트 스위칭 비용만 늘어 **오히려 느려집니다.**
> - CPU 바운드 → 풀 크기 `코어 수` 또는 `코어 수 + 1`
> - I/O 바운드 → 코어 수보다 훨씬 크게 (대기 시간이 길어 CPU가 논다)

#### 3) 내 작업이 어느 쪽인지 판별

| 유형 | 예시 | 최적 전략 |
|---|---|---|
| **CPU 바운드** | 수치 연산, 암복호화, 이미지 처리, 대량 정렬 | 코어 수만큼 분할 (`parallelStream`, `ForkJoinPool`) |
| **I/O 바운드** | DB 조회, 외부 API 호출, 파일 읽기 | 쓰레드 수를 크게, **또는 Virtual Thread** |

> [!note] Virtual Thread는 이 문제의 답이 아닙니다
> Virtual Thread는 **블로킹 I/O 중에 캐리어 쓰레드를 반환**해서 이득을 봅니다. 순수 CPU 루프는 반환할 대기 구간이 없어 **이점이 전혀 없습니다.** 질문하신 "루프를 많이 도는" 작업이 계산 위주라면 Platform Thread + 분할이 정답입니다.

#### 4) 분할이 불가능한 루프라면

이전 결과가 다음 계산에 필요한 **순차 의존(sequential dependency)** 루프는 병렬화가 안 됩니다. 이때는 방향을 바꿉니다.

- **알고리즘 개선**: O(n²) → O(n log n)이 쓰레드 8배보다 훨씬 큽니다. **최우선 검토 대상**
- **응답성만 확보**: 백그라운드 쓰레드로 보내고 진행률만 보고
- **작업 자체를 분리**: 배치 서버·메시지 큐(Kafka/SQS)로 넘겨 요청 처리와 분리

#### 5) 반드시 챙겨야 할 것 — 취소와 진행 상황

긴 루프는 **중단할 수 있어야** 합니다. 안 그러면 서버 종료가 안 되거나 좀비 작업이 남습니다.

```java
for (long i = 0; i < huge; i++) {
    if (Thread.currentThread().isInterrupted()) {   // ✅ 주기적 인터럽트 확인
        log.info("작업 취소됨");
        return;                                     // 자원 정리 후 종료
    }
    heavyWork(i);
    if (i % 1_000_000 == 0) progress.set(i);        // 진행률 보고(AtomicLong)
}
```

```java
// 호출 측: 타임아웃 + 취소
try {
    result = future.get(10, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    future.cancel(true);   // true = 실행 중인 쓰레드에 인터럽트 전달
}
```

> [!warning] `Thread.stop()`은 **절대 사용 금지**입니다(deprecated·제거됨). 락을 들고 있는 상태로 즉사시켜 데이터가 깨집니다. 반드시 인터럽트 기반 협조적 취소를 쓰세요.

#### 6) 상황별 결론

| 상황 | 권장 해법 |
|---|---|
| 다른 작업이 멈추는 게 문제 | `ExecutorService`로 백그라운드 분리 + `CompletableFuture` 콜백 |
| 루프를 더 빨리 끝내고 싶음 (분할 가능) | `parallelStream()` 또는 코어 수만큼 수동 분할 |
| 분할 불가(순차 의존) | 알고리즘 개선 우선, 그다음 백그라운드 분리 |
| 루프 안에 DB·API 호출이 섞임 | 쓰레드 수 크게 or Virtual Thread |
| 몇 분~몇 시간짜리 작업 | 요청/응답에서 떼어내 배치·메시지 큐로 |

> [!tip] 한 줄 정리
> 쓰레드는 **"다른 일을 못 하는 문제"는 거의 항상 해결**해 주지만, **"루프 자체를 빠르게 하는 문제"는 작업을 쪼갤 수 있을 때, 코어 수 한도 내에서만** 해결해 줍니다.

---

### Q3. CPU 사용량이 높은 원인이 루프로 인한 "인터럽트" 때문이라고 생각하는데, 쓰레드로 해결할 수 있나?

**A.** 먼저 **용어에 혼동이 있어서 바로잡고 시작**해야 합니다. 원인을 잘못 지목하면 해결책도 어긋나기 때문입니다.

#### 1) 용어 정리 — 세 가지가 전부 다른 개념입니다

| 용어 | 실제 의미 | CPU 사용량과의 관계 |
|---|---|---|
| **하드웨어 인터럽트(IRQ)** | 디스크·네트워크 카드 등이 CPU에 보내는 신호. 커널이 처리 | 일반 애플리케이션 루프와 **무관**. `top`의 `hi`/`si` 항목 |
| **컨텍스트 스위칭** | OS가 쓰레드를 교체하며 레지스터·스택을 저장/복원 | 쓰레드가 **많을수록 증가** (오버헤드) |
| **`Thread.interrupt()`** | 자바에서 쓰레드에 "중단 요청" 플래그를 세우는 **협조적 신호** | CPU 사용량과 **직접 관련 없음** |

> [!important] 가장 중요한 교정
> **루프가 CPU를 먹는 이유는 인터럽트 때문이 아니라, 그냥 "CPU가 쉬지 않고 계산하고 있기 때문"입니다.**
> `for` 루프가 1억 번 돈다면 CPU는 1억 번의 명령을 실제로 실행합니다. 중간에 인터럽트가 끼어들어서 느린 게 아니라, **일 자체가 그만큼 있는 것**입니다.

#### 2) 핵심 질문: 쓰레드가 CPU 사용량을 줄여주나?

> [!warning] 대부분의 경우 **아니오. 오히려 늘어납니다.**
> 쓰레드는 **일을 나눠서 동시에 처리**할 뿐, **총 연산량을 줄이지 않습니다.**
> - 1코어가 100% 쓰다가 → 4쓰레드로 나누면 **4코어가 각각 100%** (총 CPU 사용량은 오히려 증가)
> - 줄어드는 건 **벽시계 시간(wall-clock time)**이지, **CPU 시간(CPU time)**이 아닙니다
> - 여기에 컨텍스트 스위칭 비용까지 **추가**됩니다

쓰레드는 "CPU 사용량을 줄이는 도구"가 아니라 **"CPU를 놀리지 않게 하거나, 일을 빨리 끝내는 도구"**입니다.

#### 3) 단, **예외가 하나 있습니다** — Busy-Wait(스핀 루프)

이 경우에만 쓰레드/동기화 도구로 **CPU 사용량이 실제로 급감**합니다. 질문하신 상황이 여기에 해당할 가능성이 꽤 높습니다.

```java
// ❌ Busy-Waiting: 할 일이 없는데도 CPU를 100% 태움
while (true) {
    if (!queue.isEmpty()) {
        process(queue.poll());
    }
    // 아무것도 안 하면서 계속 루프 → CPU 코어 하나가 풀로 점유됨
}
```

```java
// ❌ sleep 폴링: 나아졌지만 여전히 낭비 + 지연 발생
while (true) {
    if (!queue.isEmpty()) process(queue.poll());
    Thread.sleep(10);   // CPU는 덜 쓰지만 최대 10ms 지연, 여전히 깨어남
}
```

```java
// ✅ 블로킹: 할 일이 없으면 쓰레드가 WAITING 상태로 잠듦 → CPU 사용량 0%
BlockingQueue<Task> queue = new LinkedBlockingQueue<>();
while (!Thread.currentThread().isInterrupted()) {
    Task t = queue.take();   // 데이터가 올 때까지 OS가 쓰레드를 재워둠
    process(t);
}
```

> [!tip] 판별법
> **"그 루프는 실제로 의미 있는 계산을 하고 있는가, 아니면 조건이 만족되기를 기다리며 헛돌고 있는가?"**
> - 헛돌고 있다 → **`BlockingQueue` / `wait-notify` / `CountDownLatch` / 이벤트 기반으로 전환하면 CPU가 극적으로 떨어집니다**
> - 실제 계산 중이다 → 쓰레드로는 CPU 사용량이 안 줄어듭니다. 알고리즘을 봐야 합니다

#### 4) 추측하지 말고 — 원인부터 측정하세요

> [!important] "루프 때문일 것이다"는 **가설**입니다. 측정 없이 고치면 엉뚱한 곳을 건드리게 됩니다.

**① 어떤 쓰레드가 CPU를 먹는지 특정**
```bash
top -H -p <PID>          # 쓰레드별 CPU 사용률 (Linux)
# 범인 TID를 16진수로 변환
printf "%x\n" <TID>
jstack <PID> | grep -A 30 "nid=0x<위에서_나온_16진수>"
```
→ 해당 쓰레드가 지금 **어느 코드 줄에서** 돌고 있는지 정확히 나옵니다.

**② 더 정확하게는 프로파일러**
```bash
# async-profiler — 플레임 그래프로 CPU 점유 지점을 시각화
./profiler.sh -d 30 -e cpu -f flame.html <PID>
```
→ 가장 넓은 막대가 범인입니다. JFR(`jcmd <PID> JFR.start`)도 좋습니다.

**③ 체크할 다른 원인들** — 루프가 아닌 경우가 의외로 많습니다

| 증상 | 의심 지점 | 확인법 |
|---|---|---|
| CPU 톱니파, 주기적 급등 | **GC 과다** | `jstat -gcutil <PID> 1000`, GC 로그 |
| `BLOCKED` 쓰레드 다수 | **락 경합**(스핀 포함) | 스레드 덤프에서 동일 락 주소 |
| 특정 API 호출 시만 급등 | **비효율 알고리즘 / N+1 쿼리** | 프로파일러 |
| 상시 1코어 100% 고정 | **busy-wait 루프** | `top -H` → 단일 쓰레드 고정 |
| 쓰레드 수가 비정상적으로 많음 | 컨텍스트 스위칭 폭주 | `vmstat 1`의 `cs` 컬럼 |

#### 5) 상황별 정리 — 쓰레드가 답인가?

| 실제 원인 | 쓰레드로 해결되나 | 올바른 해법 |
|---|---|---|
| **Busy-wait / 폴링 루프** | ✅ **예** | `BlockingQueue`, `wait/notify`, 이벤트 기반 전환 |
| 다른 요청이 밀리는 응답성 문제 | ✅ 예 | 무거운 작업을 별도 풀로 분리 |
| 알고리즘이 비효율적 | ❌ 아니오 | 복잡도 개선, 캐싱, 쿼리 튜닝 |
| 연산량 자체가 많음 | ⚠️ 시간만 단축 | 코어 수만큼 분할 (총 CPU는 그대로) |
| GC 과다 | ❌ 아니오 | 힙 튜닝, 객체 생성 억제 |
| 락 경합 | ⚠️ 부분적 | 임계 영역 축소, `Atomic`/락프리 자료구조 |
| 쓰레드가 너무 많음 | ❌ **역효과** | 풀 크기 축소 |

> [!tip] 결론 세 줄
> 1. **CPU 사용량이 높은 건 인터럽트 때문이 아니라, 실제로 연산을 하고 있기 때문**입니다
> 2. 쓰레드는 **CPU 사용량을 줄이지 않습니다.** 단, **헛도는 busy-wait 루프라면 블로킹으로 바꿔 극적으로 줄일 수 있습니다**
> 3. 무엇보다 **`top -H` + 스레드 덤프 또는 프로파일러로 원인을 먼저 특정**하세요. 추측 기반 최적화는 거의 빗나갑니다

관련 키워드: [[Busy-Waiting]] · [[BlockingQueue]] · [[Context-Switching]] · [[JVM-Profiling]] · [[GC-Tuning]]

---

### Q4. 몇만 건 처리에도 CPU가 높은데, 수십만·수억 건이나 이미지를 처리하는 곳은 어떻게 하나? 쓰레드나 for→while 변경으로 효율화할 수 있나?

**A.** 두 가지를 먼저 분명히 하고 시작하겠습니다.

> [!warning] ① `for` → `while` 변경은 **효과가 전혀 없습니다**
> 두 문법은 컴파일되면 **완전히 동일한 바이트코드**(`goto` + 조건 분기)가 됩니다. JIT 컴파일러 관점에서도 차이가 없습니다. 반복문 문법은 **가독성의 문제이지 성능의 문제가 아닙니다.**

> [!important] ② 몇만 건에 CPU가 높다면, 그건 **"데이터가 많아서"가 아닙니다**
> 현대 CPU는 **단순 루프를 초당 수억~수십억 번** 돕니다. 몇만 건은 단순 연산이라면 **수 밀리초**면 끝납니다.
> CPU가 높다는 건 **루프를 도는 횟수가 아니라, 루프 "안에서" 뭔가 비싼 일을 하고 있다**는 뜻입니다. 여기가 진짜 범인입니다.

#### 1) 최적화 효과 크기 순서 — 위에서부터 보세요

| 순위 | 영역 | 기대 효과 | 비고 |
|---|---|---|---|
| 1 | **알고리즘 복잡도** | 10~10,000배 | O(n²) → O(n). 압도적 1위 |
| 2 | **I/O 패턴**(쿼리·네트워크) | 10~1,000배 | N+1 제거, 배치 조회 |
| 3 | **메모리·GC** | 2~10배 | 불필요한 객체 생성 제거 |
| 4 | **병렬화(쓰레드)** | **코어 수 배(최대)** | 8코어면 최대 8배. **여기가 한계** |
| 5 | 미세 최적화(문법) | ~0% | `for`/`while`, `++i` 등 — **무의미** |

> [!tip] 쓰레드는 4순위입니다
> 8코어 서버에서 쓰레드로 얻을 수 있는 최대 이득은 **8배**입니다. 반면 O(n²)를 O(n log n)으로 바꾸면 10만 건 기준 **수천 배**가 빨라집니다. **1~3번을 먼저 처리하고 그래도 부족할 때 쓰레드를 꺼내세요.**

#### 2) 몇만 건에 CPU가 튀는 전형적 범인들 (체크리스트)

**① N+1 쿼리 — 가장 흔하고 가장 치명적**
```java
// ❌ 1만 건이면 쿼리 1만 번 → 네트워크 왕복만 수십 초
for (Order o : orders) {
    User u = userRepository.findById(o.getUserId());   // 매 루프마다 DB 호출
}

// ✅ 한 번에 조회 후 Map으로 O(1) 조회
Map<Long, User> users = userRepository.findAllById(userIds).stream()
        .collect(toMap(User::getId, identity()));
for (Order o : orders) { User u = users.get(o.getUserId()); }
```

**② 중첩 루프 = O(n²)**
```java
// ❌ 1만 × 1만 = 1억 번 비교
for (A a : listA) for (B b : listB) if (a.getId().equals(b.getAId())) { ... }

// ✅ Map으로 인덱싱 → 1만 + 1만 = 2만 번
Map<Long, B> idx = listB.stream().collect(toMap(B::getAId, identity()));
for (A a : listA) { B b = idx.get(a.getId()); }
```
> `List.contains()`를 루프 안에서 쓰는 것도 같은 함정입니다 → `HashSet`으로 교체.

**③ 문자열 연결**
```java
String s = "";
for (...) s += item;              // ❌ 매번 새 String 생성 → O(n²)
StringBuilder sb = new StringBuilder();
for (...) sb.append(item);        // ✅ O(n)
```

**④ 루프 안의 반복 비용**
```java
for (String x : list) {
    if (x.matches("\\d+")) { ... }                 // ❌ 매번 정규식 재컴파일
    log.debug("처리: " + buildHeavyString(x));      // ❌ 로그 레벨과 무관하게 문자열 생성
}
// ✅
Pattern p = Pattern.compile("\\d+");               // 루프 밖에서 1회
for (String x : list) {
    if (p.matcher(x).matches()) { ... }
    if (log.isDebugEnabled()) log.debug(...);       // 또는 log.debug("처리: {}", x)
}
```

**⑤ 오토박싱 / 불필요한 객체 생성**
```java
Long sum = 0L;
for (long i = 0; i < n; i++) sum += i;   // ❌ 매 반복마다 Long 객체 생성 → GC 폭주
long sum = 0L;                           // ✅ 원시 타입 사용
```
> 컬렉션도 `List<Integer>` 대신 `int[]`를 쓰면 메모리가 **4~5배** 줄고 캐시 효율이 올라갑니다.

**⑥ 컬렉션 초기 용량 미지정**
```java
new ArrayList<>();          // ❌ 10만 건 삽입 시 내부 배열 복사 수십 회
new ArrayList<>(100_000);   // ✅ 예상 크기 지정
new HashMap<>(expected / 0.75f + 1);
```

> [!note] 몇만 건에서 CPU가 높다면 위 6가지 중 하나일 확률이 **매우 높습니다.** 쓰레드를 붙이기 전에 이것부터 확인하세요. 쓰레드는 **비효율을 병렬로 복제할 뿐**입니다.

#### 3) 데이터 규모별 전략 — 임계점이 다릅니다

| 규모 | 핵심 전략 | 주의점 |
|---|---|---|
| **~1만 건** | 단일 쓰레드로 충분. 위 체크리스트만 지키면 끝 | 쓰레드는 오버엔지니어링 |
| **~10만 건** | 알고리즘 정리 + 배치 조회 + `parallelStream` 고려 | 메모리 전체 적재 가능 여부 확인 |
| **~100만 건** | **청크(chunk) 처리** 필수, 커서 기반 조회, 병렬 분할 | **전체를 메모리에 올리면 OOM** |
| **1000만~억 건** | 애플리케이션 밖으로: **DB 집계, 배치 프레임워크, 분산 처리** | 자바 단일 프로세스로 풀지 않음 |

> [!important] 가장 중요한 전환점: **"전부 메모리에 올리지 않기"**
> `findAll()`로 100만 건을 `List`에 담는 순간 Heap이 터지거나 GC가 CPU를 다 먹습니다. 대용량의 핵심은 속도가 아니라 **스트리밍/청크 처리**입니다.

```java
// ❌ 전체 적재
List<Row> all = repository.findAll();        // OOM 위험
// ✅ 청크 단위 처리 — 메모리 일정하게 유지
int page = 0, size = 1000;
Page<Row> chunk;
do {
    chunk = repository.findAll(PageRequest.of(page++, size));
    process(chunk.getContent());
    entityManager.clear();                   // JPA 1차 캐시 비우기 (중요)
} while (chunk.hasNext());
```
> JPA라면 `entityManager.clear()`를 빠뜨리면 **영속성 컨텍스트에 객체가 계속 쌓여 결국 OOM**이 납니다. 대용량 배치에서 가장 흔한 사고입니다.

#### 4) 그래서 쓰레드는 언제 쓰나

**쓸 만한 경우**
- 체크리스트를 다 정리했는데도 **순수 CPU 연산량**이 많다 (이미지 변환, 암복호화, 집계)
- 건별 처리가 **서로 독립적**이다 (앞 결과가 뒤에 영향 없음)
- 코어가 놀고 있다 (`top`에서 1코어만 100%, 나머지 유휴)

```java
// 청크 단위 병렬 처리 — 메모리와 병렬성을 동시에 관리
int cores = Runtime.getRuntime().availableProcessors();
ExecutorService pool = Executors.newFixedThreadPool(cores);
try {
    for (List<Row> chunk : partition(rows, 1000)) {
        pool.submit(() -> process(chunk));   // 청크 단위로 분배
    }
} finally {
    pool.shutdown();
    pool.awaitTermination(1, TimeUnit.HOURS);
}
```

> [!warning] `parallelStream()`의 함정
> 기본적으로 **전역 `ForkJoinPool.commonPool`**(코어 수 - 1)을 씁니다. 웹 서버에서 남발하면 **다른 요청까지 같이 느려집니다.** 풀을 지정하려면 커스텀 `ForkJoinPool` 안에서 제출하거나, 그냥 `ExecutorService`를 쓰세요.
> 또한 `parallelStream`은 **I/O 작업에 부적합**합니다(블로킹이 공용 풀을 고갈시킴).

**쓰면 안 되는 경우**
- DB가 병목인데 쓰레드만 늘림 → **커넥션 풀 고갈 + DB 부하 폭증**으로 더 느려짐
- 순차 의존이 있는 처리
- 이미 코어가 모두 100% → 더 나눌 여유가 없음

#### 5) 이미지·파일 처리는 성격이 다릅니다

| 항목 | 포인트 |
|---|---|
| 병목 구분 | 디코딩/리사이즈 = **CPU 바운드**, 업/다운로드 = **I/O 바운드** → 두 단계를 **다른 풀**로 분리 |
| 메모리 | 4000×3000 이미지 1장 = 약 **48MB** 비압축. 동시 10장이면 480MB → **동시 처리 수 제한 필수**(`Semaphore`) |
| 스트리밍 | 파일 전체를 `byte[]`로 읽지 말고 `InputStream`으로 흘려보내기 |
| 네이티브 | 순수 자바(`ImageIO`)보다 **libvips·ImageMagick** 같은 네이티브 라이브러리가 수 배~수십 배 빠름 |
| 전략 | 대용량이라면 **실시간 처리 대신 큐에 넣고 비동기 워커로 처리** |

```java
Semaphore limit = new Semaphore(4);   // 동시 이미지 처리 4개로 제한 → OOM 방지
limit.acquire();
try { resize(image); } finally { limit.release(); }
```

#### 6) 수억 건 규모는 "자바에서 안 합니다"

| 접근 | 설명 |
|---|---|
| **DB로 밀기** | 집계·필터링은 SQL이 자바 루프보다 압도적으로 빠릅니다. 데이터를 옮기지 말고 **연산을 데이터 쪽으로** |
| **Spring Batch** | 청크 처리, 재시도, 실패 지점 재시작, 병렬 스텝을 프레임워크가 제공 |
| **메시지 큐 + 워커 수평 확장** | Kafka/SQS로 분배, 인스턴스를 늘려 처리량 확보 (쓰레드보다 확장 한계가 높음) |
| **분산 처리** | Spark 등. TB급에서 선택 |
| **사전 집계** | 매번 계산하지 말고 결과를 미리 만들어 두기(배치 집계 테이블, 캐시) |

> [!tip] 결론 정리
> 1. **`for` → `while`은 효과 0%**입니다. 문법 최적화는 의미 없습니다
> 2. **몇만 건에 CPU가 높은 건 비정상** — 루프 횟수가 아니라 **루프 안의 N+1 쿼리 / O(n²) / 객체 생성**이 범인입니다
> 3. 효과 순서는 **알고리즘 > I/O 패턴 > 메모리 > 쓰레드 > 문법**. 쓰레드는 최대 **코어 수 배**가 한계입니다
> 4. 대용량의 본질은 속도가 아니라 **"전부 메모리에 올리지 않는 것"**(청크·스트리밍)입니다
> 5. 억 건 규모는 자바 루프가 아니라 **DB 집계·배치 프레임워크·분산 처리**의 영역입니다
> 6. 무엇보다 **프로파일러로 범인을 특정한 뒤** 고치세요

관련 키워드: [[N+1-Problem]] [[Spring-Batch]] [[Chunk-Processing]] [[GC-Tuning]] [[JVM-Profiling]] [[Parallel-Stream]] [[Image-Processing]]

---
