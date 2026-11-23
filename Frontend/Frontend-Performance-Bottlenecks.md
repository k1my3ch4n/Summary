---
created: 2026-11-23
updated: 2026-11-23
domain: Frontend
tags: [performance, typescript, javascript, react, rendering, web-worker, optimization]
---

# 프론트엔드 성능 병목과 최적화 전략

## 📖 핵심 개념 및 동작 원리

> [!important] 가장 먼저 알아야 할 전제 — 프론트엔드는 백엔드와 병목 구조가 다릅니다
> **JS는 메인 스레드 하나로 JS 실행 + 스타일 계산 + 레이아웃 + 페인트를 전부 처리합니다.**
> 백엔드는 "CPU를 얼마나 쓰느냐(처리량)"가 문제라면, 프론트엔드는 **"메인 스레드를 얼마나 오래 붙잡느냐(응답성)"**가 문제입니다.
> 60fps를 유지하려면 한 프레임당 예산이 **16.7ms**뿐이고, 여기서 브라우저 작업을 빼면 **실제 JS 예산은 약 10ms**입니다. 이 한 줄이 프론트엔드 최적화의 전부라고 해도 과언이 아닙니다.

> [!warning] 질문에서 언급하신 "불필요한 for문 수정, 상수 밖으로 빼기"에 대해
> **백엔드와 마찬가지로 프론트엔드에서도 이런 미세 최적화의 효과는 거의 0에 가깝습니다.**
> 1,000개짜리 배열을 두 번 순회하든 한 번 순회하든 사람이 느낄 수 있는 차이는 없습니다(0.1ms 수준).
> 프론트엔드 체감 성능의 **90%는 ① 네트워크(번들 크기) ② 불필요한 리렌더링 ③ 메인 스레드 블로킹**에서 결정됩니다.
> 단, **예외가 있습니다**: 그 코드가 **렌더링 함수 안에서 매 렌더마다 실행된다면** 이야기가 완전히 달라집니다(아래 4장).

---

## 1. 병목 지도 — 사용자가 화면을 보기까지의 4단계

```
[1단계] 네트워크        [2단계] 파싱·실행        [3단계] 렌더링        [4단계] 런타임
HTML/JS/CSS 다운로드  →  JS 파싱·컴파일·실행  →  Layout/Paint/     →  상호작용·
API 응답 대기            하이드레이션             Composite            리렌더링
    ↓                       ↓                      ↓                    ↓
"화면이 안 뜸"          "흰 화면이 오래"        "깜빡이고 밀림"      "클릭이 버벅임"
    ↓                       ↓                      ↓                    ↓
 LCP / TTFB              FCP / TBT               CLS                  INP
```

| 지표 | 의미 | 목표 | 주 원인 |
|---|---|---|---|
| **TTFB** | 첫 바이트 도착 | < 800ms | 서버 응답, DNS/TLS, 리다이렉트 |
| **FCP** | 첫 콘텐츠 표시 | < 1.8s | 렌더 블로킹 CSS/JS |
| **LCP** | 최대 콘텐츠 표시 | < 2.5s | 큰 이미지, 느린 API, 번들 크기 |
| **CLS** | 레이아웃 이동 | < 0.1 | 크기 미지정 이미지, 늦게 삽입되는 요소 |
| **INP** | 상호작용 응답성 | < 200ms | 긴 JS 태스크, 과도한 리렌더링 |
| **TBT** | 메인 스레드 차단 시간 | < 200ms | 50ms 넘는 Long Task |

> [!tip] 증상 → 지표 매핑
> - "화면이 늦게 나와요" → **LCP / FCP** (네트워크·번들)
> - "흰 화면이 한참 떠 있어요" → **FCP / TBT** (JS 번들·하이드레이션)
> - "클릭해도 반응이 없어요" → **INP** (메인 스레드 블로킹·리렌더링)
> - "스크롤이 버벅여요" → **프레임 드랍** (리플로우·리페인트)
> - "화면이 덜컹 밀려요" → **CLS** (레이아웃 이동)

---

## 2. [1단계] 네트워크 병목 — 체감 성능의 최대 지분

| 병목               | 증상                 | 해결책                                                       |
| ---------------- | ------------------ | --------------------------------------------------------- |
| **번들 크기 과다**     | 흰 화면 지속, 느린 FCP    | 코드 스플리팅, 트리 셰이킹, 번들 분석                                    |
| **무거운 라이브러리**    | 번들 급증              | 경량 대체(`moment`→`date-fns`/`Temporal`, `lodash`→개별 import) |
| **Waterfall 요청** | API가 순차 호출됨        | `Promise.all`로 병렬화                                        |
| **N+1 API 호출**   | 목록마다 상세 호출         | 백엔드에 배치 API 요청, BFF 도입                                    |
| **최적화 안 된 이미지**  | LCP 지연             | WebP/AVIF, `srcset`, lazy loading, CDN 리사이징               |
| **캐시 미활용**       | 매번 재다운로드           | `Cache-Control`, 해시 파일명, SWR/React Query                  |
| **폰트 로딩**        | 텍스트 깜빡임(FOUT/FOIT) | `font-display: swap`, `preload`, subset                   |

```typescript
// ❌ Waterfall — 순차 실행으로 총 900ms
const user = await fetchUser(id);       // 300ms
const posts = await fetchPosts(id);     // 300ms
const stats = await fetchStats(id);     // 300ms

// ✅ 병렬 실행 — 총 300ms
const [user, posts, stats] = await Promise.all([
  fetchUser(id), fetchPosts(id), fetchStats(id),
]);

// ✅ 일부 실패를 허용해야 한다면
const results = await Promise.allSettled([...]);
```

```typescript
// ❌ 라이브러리 전체 import → 번들에 전부 포함
import _ from 'lodash';                 // ~70KB
import moment from 'moment';            // ~290KB (로케일 포함)

// ✅ 필요한 것만
import debounce from 'lodash/debounce'; // ~2KB
import { format } from 'date-fns';      // 트리 셰이킹 가능
```

```typescript
// ✅ 코드 스플리팅 — 초기 번들에서 제외
const HeavyChart = lazy(() => import('./HeavyChart'));
const Editor = lazy(() => import('./Editor'));   // 라우트·모달 단위가 효과적

<Suspense fallback={<Skeleton />}>
  <HeavyChart />
</Suspense>
```

> [!important] 번들 최적화의 출발점은 **측정**입니다
> ```bash
> npx vite-bundle-visualizer        # Vite
> npx webpack-bundle-analyzer       # Webpack
> npx source-map-explorer build/**/*.js
> ```
> 트리맵에서 가장 큰 블록부터 제거하세요. 보통 **상위 3개가 전체의 절반**을 차지합니다.

---

## 3. [2~3단계] 렌더링 병목 — 레이아웃 스래싱

> [!warning] 프론트엔드 고유의 함정: **Forced Synchronous Layout (Layout Thrashing)**
> DOM 읽기(`offsetHeight` 등)와 쓰기(`style.x = ...`)를 **번갈아 하면**, 브라우저가 매번 레이아웃을 강제로 다시 계산합니다. 루프 안에서 이걸 하면 **O(n)이 사실상 O(n) × 레이아웃 비용**이 되어 치명적입니다.

```typescript
// ❌ 읽기-쓰기 교차 → 요소 개수만큼 강제 리플로우 발생
elements.forEach(el => {
  el.style.height = el.offsetHeight + 10 + 'px';   // 읽고 → 쓰고 → 읽고 → 쓰고...
});

// ✅ 읽기와 쓰기를 분리 (Read → Write 배치)
const heights = elements.map(el => el.offsetHeight);  // 1) 전부 읽기
elements.forEach((el, i) => {                          // 2) 전부 쓰기
  el.style.height = heights[i] + 10 + 'px';
});
```

**렌더링 단계별 비용** — Java 노트의 "알고리즘 복잡도" 개념과 같은 위치입니다.

| 변경하는 속성 | 재실행되는 단계 | 비용 |
|---|---|---|
| `width`, `height`, `top`, `margin`, `font-size` | **Layout → Paint → Composite** | 가장 비쌈 |
| `color`, `background`, `box-shadow`, `border-radius` | **Paint → Composite** | 중간 |
| `transform`, `opacity`, `filter` | **Composite만** | 가장 쌈 ✅ |

```css
/* ❌ 매 프레임 레이아웃 재계산 */
.box { transition: left 0.3s; }
/* ✅ GPU 합성만 — 60fps 유지 */
.box { transition: transform 0.3s; will-change: transform; }
```

| 기타 렌더링 병목 | 해결책 |
|---|---|
| 긴 리스트 전체 렌더 (수천 개 DOM) | **가상 스크롤**(`react-window`, `TanStack Virtual`) |
| 이미지 크기 미지정 → CLS | `width`/`height` 또는 `aspect-ratio` 명시 |
| 스크롤·리사이즈 핸들러 과다 호출 | `throttle`, `requestAnimationFrame`, `IntersectionObserver` |
| 복잡한 CSS 선택자, 과도한 `box-shadow`/`filter` | 선택자 단순화, 효과 범위 축소 |
| `will-change` 남용 | 필요한 요소에만. 남용 시 메모리 폭증 |

```typescript
// ❌ 스크롤 이벤트는 초당 수십~수백 회 발생
window.addEventListener('scroll', () => heavyWork());

// ✅ rAF로 프레임당 1회로 제한
let ticking = false;
window.addEventListener('scroll', () => {
  if (ticking) return;
  ticking = true;
  requestAnimationFrame(() => { heavyWork(); ticking = false; });
}, { passive: true });   // passive: 스크롤 블로킹 방지

// ✅ 요소 노출 감지는 스크롤 계산 대신 IntersectionObserver
const io = new IntersectionObserver(entries => { /* ... */ });
```

---

## 4. [4단계] 런타임 병목 — 여기서 "불필요한 for문"이 진짜 문제가 됩니다

> [!important] 질문하신 코드 레벨 최적화가 유효해지는 지점
> 같은 코드라도 **어디에 있느냐**에 따라 영향이 100배 달라집니다.
> - 이벤트 핸들러 안에서 1회 실행되는 루프 → **거의 무의미**
> - **렌더링 함수 본문에서 매 렌더마다 실행되는 루프** → **치명적** (초당 수십 번 실행)
> - **리스트 아이템 1,000개 각각에서 실행되는 루프** → **치명적** (n × m)

```tsx
// ❌ 매 렌더마다 재생성 + 전체 재계산
function ProductList({ products, keyword }: Props) {
  const CATEGORIES = ['전자', '의류', '식품'];           // 매 렌더마다 새 배열
  const filtered = products.filter(p => p.name.includes(keyword))
                           .sort((a, b) => b.price - a.price);  // 매 렌더마다 정렬
  const handleClick = (id: string) => select(id);        // 매 렌더마다 새 함수
  return <>{filtered.map(p => <Item key={p.id} onClick={handleClick} />)}</>;
}

// ✅
const CATEGORIES = ['전자', '의류', '식품'];              // ① 모듈 스코프로 추출
function ProductList({ products, keyword }: Props) {
  const filtered = useMemo(                               // ② 의존성 변경 시에만 계산
    () => products.filter(p => p.name.includes(keyword))
                  .sort((a, b) => b.price - a.price),
    [products, keyword],
  );
  const handleClick = useCallback((id: string) => select(id), []); // ③ 참조 안정화
  return <>{filtered.map(p => <Item key={p.id} onClick={handleClick} />)}</>;
}
```

> [!tip] 즉, 질문하신 "상수를 밖으로 빼기"는 **프론트엔드에서는 백엔드보다 훨씬 중요합니다.**
> 다만 이유가 다릅니다. **연산 비용을 아끼려는 게 아니라, 참조 동일성(reference identity)을 유지해 자식 컴포넌트의 불필요한 리렌더링을 막기 위해서**입니다.
> JS에서 `[] !== []`, `{} !== {}`, `() => {} !== () => {}` 이기 때문에, 매 렌더마다 새로 만들면 `React.memo`가 전부 무력화됩니다.

### 리렌더링 병목 체크리스트

| 병목 | 원인 | 해결책 |
|---|---|---|
| 전체 트리 리렌더 | 최상위에 자주 바뀌는 state | state를 **사용하는 곳 가까이로 내리기** |
| Context 전체 구독 | 값 하나 바뀌어도 모든 구독자 리렌더 | Context 분리, 또는 Zustand/Jotai 같은 선택적 구독 |
| `memo`가 안 먹음 | props로 매번 새 객체/함수 전달 | `useMemo`/`useCallback`, 원시값으로 전달 |
| 리스트 전체 리렌더 | `key`에 index 사용 | **안정적인 고유 id**를 key로 |
| 입력할 때마다 전체 리렌더 | 상위 state에 입력값 보관 | 입력 컴포넌트로 state 격리, `debounce` |
| 비싼 계산 매 렌더 반복 | 메모이제이션 부재 | `useMemo` |

```tsx
// ❌ 입력 1글자마다 부모 전체 + 무거운 리스트까지 리렌더
const [q, setQ] = useState('');
return <><input value={q} onChange={e => setQ(e.target.value)} /><HeavyList /></>;

// ✅ 검색 반영을 지연 (React 18+)
const deferredQ = useDeferredValue(q);         // 입력은 즉시, 리스트는 뒤따라
return <><input value={q} onChange={...} /><HeavyList query={deferredQ} /></>;
```

### 자료구조 — 여기서는 백엔드와 완전히 동일합니다

```typescript
// ❌ O(n²) — 1,000 × 1,000 = 100만 회
const merged = orders.map(o => ({
  ...o,
  user: users.find(u => u.id === o.userId),   // 루프 안의 find
}));

// ✅ O(n) — Map 인덱싱
const userMap = new Map(users.map(u => [u.id, u]));
const merged = orders.map(o => ({ ...o, user: userMap.get(o.userId) }));
```
> 루프 안의 `Array.find`, `Array.includes`, `indexOf`는 모두 **O(n)**입니다. `Map`/`Set`으로 바꾸면 O(1)이 됩니다. 중첩 루프를 발견하면 가장 먼저 의심하세요.

---

## 5. 메인 스레드 블로킹 — TypeScript의 "쓰레드"는 Web Worker입니다

> [!important] Java 쓰레드 ↔ 프론트엔드 대응 관계
> | Java | TypeScript/브라우저 | 비고 |
> |---|---|---|
> | `Thread` / `ExecutorService` | **Web Worker** | **진짜 병렬**. 별도 스레드에서 실행 |
> | `CompletableFuture` | `Promise` / `async-await` | **병렬이 아님**. 단일 스레드 내 비동기 스케줄링 |
> | 공유 메모리 + `synchronized` | `postMessage`(구조화 복제) | 기본은 **메모리 비공유** → 데이터 경쟁이 거의 없음 |
> | `volatile`/`Atomic` | `SharedArrayBuffer` + `Atomics` | 특수 케이스에서만 |

> [!warning] 가장 흔한 오해
> **`async/await`는 CPU 작업을 해결해주지 않습니다.**
> `await`는 **I/O 대기 중에만** 메인 스레드를 양보합니다. 무거운 `for` 루프를 `async` 함수에 넣어도 **메인 스레드는 그대로 멈춥니다.** 화면이 얼어붙는 건 동일합니다.
> 순수 CPU 연산을 메인 스레드에서 빼내려면 **Web Worker가 유일한 정답**입니다.

```typescript
// ❌ async를 붙여도 메인 스레드는 그대로 블로킹 → 화면이 얼어붙음
async function process(data: number[]) {
  return data.map(heavyTransform);   // 100ms 걸리면 100ms 동안 클릭도 안 됨
}

// ✅ Web Worker로 분리 — UI는 계속 반응
// worker.ts
self.onmessage = (e: MessageEvent<number[]>) => {
  self.postMessage(e.data.map(heavyTransform));
};

// main.ts
const worker = new Worker(new URL('./worker.ts', import.meta.url), { type: 'module' });
worker.postMessage(data);
worker.onmessage = (e) => setResult(e.data);
```

**Web Worker가 맞는 경우 / 아닌 경우**

| 적합 ✅ | 부적합 ❌ |
|---|---|
| 대량 데이터 파싱(CSV/JSON 수만 건) | 단순 API 호출 (이미 비동기) |
| 이미지 처리, 암복호화 | DOM 조작 (**Worker는 DOM 접근 불가**) |
| 복잡한 정렬·집계·통계 | 10ms 이내로 끝나는 가벼운 작업 |
| 가상 스크롤용 대규모 전처리 | 큰 데이터를 자주 주고받는 경우(복제 비용) |

> [!note] `postMessage`는 **구조화 복제(structured clone)** 비용이 있습니다. 수십 MB를 왔다 갔다 하면 오히려 느려집니다. 큰 바이너리는 `Transferable`(`ArrayBuffer` 소유권 이전)을 쓰세요.

**Worker 없이 메인 스레드를 양보하는 방법 (작업 분할)**

```typescript
// 긴 작업을 청크로 쪼개 브라우저에 제어권 반환 → UI 멈춤 방지
async function processInChunks<T>(items: T[], fn: (x: T) => void, size = 500) {
  for (let i = 0; i < items.length; i += size) {
    items.slice(i, i + size).forEach(fn);
    await new Promise(r => setTimeout(r, 0));   // 양보 (yield)
  }
}

// 급하지 않은 작업은 유휴 시간에
requestIdleCallback(() => sendAnalytics());

// React 18: 긴급하지 않은 업데이트로 표시
startTransition(() => setFilter(value));   // 입력 응답성 유지
```

---

## 6. 측정 도구 — "추측하지 말고 프로파일링"

| 도구 | 용도 |
|---|---|
| **Lighthouse** | Core Web Vitals 종합 점검, 개선 항목 자동 제안 |
| **Chrome DevTools → Performance** | Long Task, 레이아웃 스래싱, 프레임 드랍 추적 (**가장 중요**) |
| **Performance Insights** | LCP/CLS 원인 요소를 직접 지목 |
| **Network 탭** | Waterfall, 번들 크기, 캐시 적중 여부 |
| **React DevTools Profiler** | 어떤 컴포넌트가 왜 리렌더됐는지(`Record why each component rendered`) |
| **번들 분석기** | 번들 구성 트리맵 |
| **web-vitals 라이브러리** | **실사용자(RUM) 데이터** 수집 — 실제 환경이 진실 |

```typescript
import { onLCP, onINP, onCLS } from 'web-vitals';
onLCP(console.log); onINP(console.log); onCLS(console.log);
```

> [!tip] DevTools Performance 읽는 법
> - **빨간 삼각형(Long Task)** = 50ms 이상 메인 스레드 점유 → INP 악화의 주범
> - **보라색 Layout 막대가 촘촘** = 레이아웃 스래싱
> - **노란 Scripting이 대부분** = JS 실행 과다 → 리렌더링 또는 무거운 연산
> - **CPU 4x throttling + Fast 3G**로 테스트하세요. 개발자 장비는 너무 빠릅니다.

---

## 7. 우선순위 — Java 노트와 동일한 원칙

| 순위 | 영역 | 기대 효과 | 예시 |
|---|---|---|---|
| 1 | **네트워크 / 번들** | 수 초 단위 | 코드 스플리팅, 이미지 최적화, API 병렬화 |
| 2 | **불필요한 리렌더링** | 수백 ms | state 위치 조정, `memo`, 참조 안정화 |
| 3 | **알고리즘 / 자료구조** | 수십~수백 ms | O(n²) 제거, `Map`/`Set` 인덱싱 |
| 4 | **렌더링 경로** | 수십 ms | `transform` 사용, 레이아웃 스래싱 제거, 가상 스크롤 |
| 5 | **Web Worker 분리** | 체감 응답성 | 무거운 CPU 연산 오프로딩 |
| 6 | 문법 미세 최적화 | **~0%** | `for` vs `forEach`, 변수 호이스팅 — **의미 없음** |

> [!warning] 가장 흔한 실수
> **1~2번을 건드리지 않고 6번부터 손대는 것.** 번들이 3MB인 상태에서 루프를 아무리 다듬어도 체감은 바뀌지 않습니다.
> 또한 **`useMemo`/`useCallback` 남발**도 안티패턴입니다. 메모이제이션 자체도 비용(메모리 + 비교 연산)이 있어서, 가벼운 계산에 붙이면 **오히려 손해**입니다. Profiler로 병목을 확인한 뒤 붙이세요.

---

> [!tip] 핵심 정리
> 1. 프론트엔드의 적은 CPU 사용량이 아니라 **메인 스레드 점유 시간**입니다 (프레임 예산 16.7ms)
> 2. 체감 성능의 대부분은 **네트워크(번들 크기)**에서 결정됩니다 — 여기부터 보세요
> 3. "상수 밖으로 빼기"는 **연산 절약이 아니라 참조 동일성 유지**를 위해 중요합니다
> 4. `async/await`는 **병렬이 아닙니다.** CPU 작업 오프로딩은 **Web Worker**만 가능합니다
> 5. 애니메이션은 **`transform`/`opacity`**만 사용 → Composite만 실행
> 6. DOM **읽기와 쓰기를 분리**하세요 (레이아웃 스래싱 방지)
> 7. **측정 → 수정 → 재측정.** Lighthouse로 큰 그림, Performance 탭으로 범인 특정

관련 키워드: [[General/Browser-URL-To-Render.md|브라우저 렌더링 과정]] · [[Backend/Java-Thread.md|Java Thread]] · [[Core-Web-Vitals]] · [[Web-Worker]] · [[React-Rendering]] · [[Code-Splitting]] · [[Virtual-Scroll]]

---

## 💬 Q&A 기록

### Q1. Java 쓰레드/성능 최적화 관점을 TypeScript·프론트엔드에 도입하면? 주요 병목과 고려사항, 해결책은?

**A.** 위 1~7장이 전체 답변입니다. 질문의 핵심 포인트만 짚으면:

**① "불필요한 for문, 상수 밖으로 빼기"는 맞는 예시인가?**
- **부분적으로 맞습니다.** 다만 **이유가 다릅니다.**
- 일반 함수 안의 루프 최적화 → 백엔드와 마찬가지로 **효과 미미**
- **렌더링 함수 안**이라면 → **매우 중요**. 매 렌더마다 실행되고, 새 참조가 생겨 자식 리렌더를 유발합니다
- 중첩 루프 / 루프 안 `find`·`includes` → **백엔드와 동일하게 치명적**. `Map`/`Set`으로 교체

**② Java 쓰레드에 대응하는 것은?**
- `Promise`/`async-await`가 아니라 **Web Worker**입니다
- `async/await`는 단일 스레드 비동기일 뿐, **CPU 작업 중에는 화면이 그대로 멈춥니다**

**③ 프론트엔드 고유의 병목 (백엔드엔 없는 것)**
| 병목 | 설명 |
|---|---|
| 번들 크기 | 코드를 "다운로드"해야 실행되는 환경 특성 |
| 리렌더링 | 상태 변경이 UI 트리 전체로 전파 |
| 레이아웃 스래싱 | DOM 읽기/쓰기 교차 → 강제 동기 레이아웃 |
| 프레임 예산 | 16.7ms 안에 모든 걸 끝내야 60fps |
| 사용자 기기 편차 | 저사양 모바일 기준으로 측정해야 함 |

**④ 증상별 1차 점검 순서**

| 증상 | 먼저 볼 곳 |
|---|---|
| 화면이 늦게 뜬다 | Network 탭 → 번들 크기, API Waterfall, 이미지 |
| 흰 화면이 길다 | 번들 분석기 → 코드 스플리팅 여부 |
| 데이터가 늦게 온다 | 순차 `await` → `Promise.all`, 백엔드 응답 시간, 캐싱(React Query) |
| 클릭이 버벅인다 | Performance 탭 Long Task → 리렌더링 or 무거운 연산 |
| 스크롤이 끊긴다 | 레이아웃 스래싱, `transform` 미사용, 가상 스크롤 미적용 |
| 화면이 덜컹인다 | 이미지 크기 미지정(CLS) |

> [!important] 결론
> 백엔드와 **원칙은 같습니다** — "측정 먼저, 알고리즘·I/O 먼저, 미세 최적화는 마지막."
> 다만 **1순위가 다릅니다.** 백엔드는 알고리즘·쿼리, 프론트엔드는 **네트워크(번들)와 리렌더링**입니다.

---
