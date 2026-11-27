---
created: 2026-11-27
updated: 2026-11-27
domain: Frontend
tags: [react, error-handling, error-boundary, resilience, ux]
---

# Error Boundary (에러 바운더리)

## 📖 핵심 개념 및 동작 원리

### 1. 한 줄 정의

**Error Boundary**는 자식 컴포넌트 트리에서 발생한 **렌더링 중 에러를 잡아내어(catch)**, 앱 전체가 흰 화면(White Screen of Death)으로 죽는 대신 **대체 UI(fallback UI)를 보여주는 [[React]] 컴포넌트**입니다.

JS의 `try/catch`가 명령형 코드를 감싸듯, Error Boundary는 **선언적 UI 트리**를 감쌉니다.

### 2. 등장 배경

React 16부터 렌더링 중 에러가 발생하면 **컴포넌트 트리 전체를 언마운트**하는 정책으로 바뀌었습니다.

> [!important]
> 이유: 깨진 상태(corrupted state)로 UI를 계속 렌더링하면, 잘못된 금액 표시·잘못된 대상에게 메시지 전송 같은 **조용한 오동작**이 발생할 수 있기 때문입니다. "아무것도 안 보여주는 것"이 "틀린 것을 보여주는 것"보다 안전하다는 판단입니다.

따라서 개발자가 명시적으로 "이 영역이 깨지면 여기까지만 격리하고 이런 UI를 보여줘"라고 선언하는 장치가 필요해졌고, 그것이 Error Boundary입니다.

### 3. 동작 원리 (2개의 라이프사이클)

클래스 컴포넌트에서 아래 둘 중 **하나 이상**을 구현하면 Error Boundary가 됩니다.

| 메서드 | 호출 시점 | 역할 | 비고 |
|---|---|---|---|
| `static getDerivedStateFromError(error)` | **렌더 단계** | 에러 상태로 전환해 fallback UI 렌더 | 순수해야 함(사이드이펙트 금지) |
| `componentDidCatch(error, errorInfo)` | **커밋 단계** | 로깅/모니터링 전송 등 사이드이펙트 | `errorInfo.componentStack` 제공 |

```tsx
class ErrorBoundary extends React.Component<Props, State> {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };        // 1) fallback UI로 전환
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    logToSentry(error, info.componentStack); // 2) 에러 리포팅
  }

  render() {
    if (this.state.hasError) return this.props.fallback;
    return this.props.children;
  }
}
```

에러는 **트리를 타고 위로 전파(bubbling)** 되며, 가장 가까운 상위 Error Boundary가 잡습니다. 아무도 잡지 않으면 루트까지 올라가 앱 전체가 언마운트됩니다.

> [!note] 왜 아직도 클래스인가?
> 2026년 현재까지도 Error Boundary에 대응하는 공식 Hook은 없습니다. 실무에서는 직접 만들기보다 **`react-error-boundary`** 라이브러리를 쓰는 것이 사실상 표준입니다.

### 4. ⚠️ 잡지 못하는 에러 (가장 흔한 오해)

> [!warning]
> Error Boundary는 **렌더 단계의 동기적 에러만** 잡습니다. 아래는 잡지 못합니다.

| 잡히지 않는 경우 | 이유 | 대응 방법 |
|---|---|---|
| **이벤트 핸들러** (`onClick` 등) | 렌더링 흐름 밖에서 실행 | `try/catch` + 상태로 에러 표시 |
| **비동기 코드** (`setTimeout`, `Promise`, `fetch`) | 콜스택이 분리됨 | `.catch()` / `try-catch` + `await` |
| **SSR (서버 사이드 렌더링)** | 클라이언트 라이프사이클 미적용 | 프레임워크별 에러 처리(`error.tsx` 등) |
| **Error Boundary 자신의 에러** | 자기 자신은 못 잡음 | 상위에 또 하나 배치 |

```tsx
// 비동기 에러를 Boundary로 끌어올리는 패턴
const { showBoundary } = useErrorBoundary(); // react-error-boundary
try { await api.save(); } catch (e) { showBoundary(e); }
```

### 5. 배치 전략 (Granularity)

핵심은 **"어디까지 깨져도 괜찮은가"** 라는 서비스 관점의 결정입니다.

```
<RootBoundary>              ← 최후의 보루: "오류가 발생했습니다" 전체 화면
  <Header />
  <PageBoundary>            ← 페이지 단위: 헤더/네비는 살리고 본문만 복구 UI
    <WidgetBoundary>        ← 위젯 단위: 추천 영역만 숨기고 나머지는 정상
      <RecommendList />
    </WidgetBoundary>
  </PageBoundary>
</RootBoundary>
```

| 전략 | 장점 | 단점 |
|---|---|---|
| **루트 1개만** | 구현 단순, 누락 없음 | 작은 위젯 에러로 전체 화면 날아감 |
| **페이지/라우트 단위** | 네비게이션 유지, 복구 쉬움 | 페이지 내부 영향 범위는 여전히 큼 |
| **위젯/섹션 단위** | 장애 격리 최적, UX 최상 | 보일러플레이트 증가, fallback 설계 비용 |

> [!tip] 실무 권장
> **루트 + 라우트 단위**를 기본으로 깔고, **외부 API 의존도가 높거나 비핵심인 영역**(광고, 추천, 실시간 차트, 서드파티 위젯)에만 선택적으로 위젯 단위를 추가합니다.

### 6. 생태계 연계

- **[[React-Suspense]]와의 조합**: `Suspense`는 "로딩 중", `ErrorBoundary`는 "실패". 데이터 패칭 UI의 양 축입니다. [[TanStack-Query]]의 `throwOnError: true`, `useSuspenseQuery`와 함께 쓰면 로딩/에러 분기를 JSX 바깥으로 걷어낼 수 있습니다.
- **[[Next.js]] App Router**: `error.tsx`(세그먼트 단위, 내부적으로 Error Boundary), `global-error.tsx`(루트 레이아웃 에러)로 파일 컨벤션화되어 있습니다.
- **[[Vue]] / [[Angular]]**: Vue는 `errorCaptured` 훅 + `onErrorCaptured`, Angular는 `ErrorHandler` 프로바이더로 유사 역할을 수행합니다. 다만 React처럼 "트리 언마운트" 정책은 아닙니다.
- **모니터링**: [[Sentry]] 등은 `componentStack`을 함께 받아 어떤 컴포넌트에서 터졌는지 추적합니다.

### 7. 체크리스트

- [ ] 루트에 최후의 보루 Boundary가 있는가
- [ ] fallback UI에 **복구 수단**(재시도 버튼, 홈으로 가기)이 있는가
- [ ] 라우트 이동 시 에러 상태가 **리셋**되는가 (`resetKeys` 활용)
- [ ] 에러가 **모니터링 도구로 전송**되는가
- [ ] 비동기/이벤트 핸들러 에러는 **별도 경로**로 처리되는가
- [ ] fallback 자체가 에러를 던지지 않는가

---

## 💬 Q&A 기록

### Q1. 상태 관리를 사실상 TanStack Query가 담당하고 있는데, Error Boundary를 따로 적용하는 게 맞을까?

**A.** **네, 적용하는 쪽이 맞습니다. 다만 "모든 에러를 Boundary로"가 아니라 역할을 나눠야 합니다.**

#### 1) 둘은 경쟁 관계가 아니라 보완 관계

| 구분 | TanStack Query | Error Boundary |
|---|---|---|
| 담당 범위 | **서버 상태**의 fetch 실패, 재시도, 캐시 무효화 | **렌더링 단계**에서 터진 모든 에러 |
| 잡는 것 | 네트워크 4xx/5xx, timeout, `queryFn` throw | `undefined.map()`, 잘못된 타입 접근, 자식 컴포넌트 버그 |
| 못 잡는 것 | 렌더 중 터지는 JS 에러 | fetch 실패(기본적으로, `throwOnError` 없으면) |

> [!important]
> TanStack Query가 `isError`를 잘 처리해 줘도, **데이터가 성공적으로 왔는데 그 데이터 모양이 예상과 달라 렌더에서 터지는 경우**(`data.items.map` 인데 `items`가 `null`)는 Query가 전혀 막아주지 못합니다. 이게 실무에서 흰 화면을 만드는 1순위 원인입니다. 최소한 **루트 + 라우트 단위 Boundary는 보험 성격으로 반드시** 있어야 합니다.

#### 2) 그래서 "어디까지 Boundary로 넘길 것인가"가 진짜 질문

TanStack Query는 `throwOnError` 옵션으로 **쿼리 에러를 Error Boundary로 던질지** 선택할 수 있습니다.

```tsx
// A. Query가 자체 처리 (throwOnError: false, 기본값)
const { data, isError, error, refetch } = useQuery({ queryKey, queryFn });
if (isError) return <InlineError onRetry={refetch} />;

// B. Boundary로 위임 (throwOnError: true)
const { data } = useSuspenseQuery({ queryKey, queryFn }); // data 타입이 non-nullable
// 로딩 → <Suspense>, 에러 → <ErrorBoundary> 가 담당
```

**선택 기준 (권장)**

| 상황 | 권장 | 이유 |
|---|---|---|
| 페이지의 **핵심 데이터**가 실패 (상세 조회, 대시보드 메인) | **B. Boundary 위임** | 어차피 화면을 못 그림 → 분기 코드 제거, `data` non-nullable로 타입 안전 |
| **비핵심 위젯** (추천, 배너, 사이드 통계) | A. Query 자체 처리 | 그 영역만 조용히 숨기거나 작은 에러 표시 |
| **Mutation** (저장, 삭제) | A. 자체 처리 | 화면은 멀쩡함 → 토스트/인라인 메시지가 UX상 적절 |
| **401/403 인증 에러** | 둘 다 아님 | 인터셉터/`QueryCache.onError`에서 전역 처리(로그인 리다이렉트) |

#### 3) 조건부 적용 패턴

```tsx
throwOnError: (error) => error.status >= 500  // 서버 장애만 Boundary로
```

4xx(유효성, 권한 등)는 인라인으로 안내하고, 5xx·네트워크 장애만 Boundary로 올리는 방식이 균형이 좋습니다.

#### 4) 리셋 연동은 필수

Boundary로 던졌다면 **재시도 경로**를 반드시 연결해야 합니다. 안 그러면 사용자가 새로고침 말고는 할 게 없습니다.

```tsx
<QueryErrorResetBoundary>
  {({ reset }) => (
    <ErrorBoundary onReset={reset} resetKeys={[location.pathname]}
      fallbackRender={({ resetErrorBoundary }) => (
        <button onClick={resetErrorBoundary}>다시 시도</button>
      )}>
      <Suspense fallback={<Skeleton />}>
        <Page />
      </Suspense>
    </ErrorBoundary>
  )}
</QueryErrorResetBoundary>
```

#### 5) 트레이드오프 정리

**Boundary 위임의 장점**
- 컴포넌트에서 `isLoading`/`isError` 분기가 사라져 JSX가 단순해짐
- `useSuspenseQuery`로 `data` 타입이 non-nullable → `data?.` 옵셔널 체이닝 지옥 해소
- 에러 UI가 한 곳에 모여 일관성·디자인 품질 확보

**Boundary 위임의 단점**
- 에러 발생 시 해당 서브트리가 **통째로 언마운트** → 사용자가 입력 중이던 폼 상태, 스크롤 위치 소실
- fallback이 너무 크면 "작은 실패에 과한 반응"으로 보임
- `resetKeys` 설계를 안 하면 에러 상태가 라우트 이동 후에도 남는 사고 발생

> [!tip] 결론
> **"루트/라우트 Boundary는 무조건 깐다 + 쿼리별로 `throwOnError`를 의도적으로 선택한다"** 가 현실적인 답입니다. TanStack Query가 상태 관리를 다 해주니 Boundary가 필요 없다 — 는 **네트워크 에러와 렌더 에러를 동일시하는 오해**입니다.

---

### Q2. 데이터 형태가 달라져서 에러가 터지는 순간, 원래는 화면이 다 깨지지만 Error Boundary는 "그 부분만" 막아주는 것인가?

**A.** **방향은 정확합니다. 다만 "그 부분"의 범위를 결정하는 건 React가 아니라 개발자입니다.** 두 가지를 정확히 구분해야 합니다.

#### 1) 맞는 부분 — 전파를 끊어준다

```
[Boundary 없음]                        [Boundary 있음]
RecommendList 에서 터짐                 RecommendList 에서 터짐
      ↓ 위로 계속 전파                        ↓ 위로 전파
      ↓                                 WidgetBoundary 가 가로챔 ✋
   루트까지 도달                              ↓
= 앱 전체 언마운트 (흰 화면)            = 추천 영역만 fallback,
                                         헤더·본문·다른 위젯은 멀쩡
```

에러는 **위로만(상위로) 전파**되고, 가장 가까운 상위 Boundary에서 멈춥니다. 그 Boundary 바깥(형제 컴포넌트, 상위 레이아웃)은 **아예 영향을 받지 않고 그대로 살아있습니다.** 이 부분이 Error Boundary의 본질입니다.

#### 2) 주의할 부분 ① — 격리 단위 = 내가 친 울타리

> [!warning]
> React가 "에러난 최소 범위"를 똑똑하게 찾아주는 게 아닙니다. **가장 가까운 상위 Boundary까지가 통째로 날아갑니다.**

루트에만 Boundary를 두면, 작은 위젯 하나가 터져도 **여전히 전체 화면이 fallback으로 바뀝니다.** "화면이 다 깨지는 대신 에러 화면이 뜬다" 정도의 차이일 뿐입니다.

```tsx
// ❌ 루트에만 → 추천 위젯 하나 터져도 전체 화면 교체
<RootBoundary><Page /></RootBoundary>

// ✅ 위젯을 감싸야 → 추천 영역만 교체
<RootBoundary>
  <Header />
  <MainContent />
  <WidgetBoundary fallback={null}><RecommendList /></WidgetBoundary>
</RootBoundary>
```

**즉, "그 부분만 막힌다"는 결과를 얻으려면 내가 그 부분에 울타리를 쳐 둬야 합니다.**

#### 3) 주의할 부분 ② — 부분 복구가 아니라 "서브트리 전체 언마운트"

Boundary 내부에서도 **에러난 한 줄만 빼고 나머지는 렌더되는 방식이 아닙니다.** 해당 Boundary의 `children` 서브트리는 **통째로 언마운트**되고 fallback으로 교체됩니다.

| 오해 | 실제 |
|---|---|
| 에러난 항목만 빠지고 리스트 나머지는 보인다 | Boundary 안 서브트리 **전부** 사라지고 fallback만 남음 |
| 컴포넌트 state가 유지된 채 에러 UI가 덧씌워진다 | 언마운트되므로 **내부 state·입력값·스크롤 전부 소실** |

> [!tip]
> 리스트에서 **특정 아이템만 깨져도 나머지는 보이게** 하려면, Boundary를 리스트가 아니라 **아이템 단위로** 내려야 합니다.
> ```tsx
> {items.map(item => (
>   <ErrorBoundary key={item.id} fallback={<BrokenItem />}>
>     <Item data={item} />
>   </ErrorBoundary>
> ))}
> ```

#### 4) 그런데 타입 불일치는 Boundary "만"으로 끝낼 문제는 아니다

Boundary는 **증상 완화(사용자가 흰 화면을 안 봄 + 에러 리포팅)** 장치이지, 원인 해결책이 아닙니다. 실무에서는 3단 방어를 권합니다.

| 계층 | 수단 | 역할 |
|---|---|---|
| **예방** | [[Zod]] 등 런타임 스키마 검증 (`queryFn`에서 `schema.parse(res)`) | 잘못된 데이터를 **렌더까지 못 가게** 차단 → Query의 에러로 전환되어 재시도·인라인 처리 가능 |
| **방어** | 옵셔널 체이닝 / 기본값 (`items ?? []`) | 사소한 누락은 조용히 흡수 |
| **최후** | **Error Boundary** | 그래도 뚫린 경우 격리 + Sentry 리포팅 |

> [!important] 요약
> - ✅ "전파를 끊어 다른 영역을 지킨다" → **맞음**
> - ⚠️ "그 부분"의 크기 = **내가 Boundary를 어디에 뒀는지**로 결정됨
> - ⚠️ Boundary 안쪽은 부분 복구가 아니라 **통째 언마운트 + fallback 교체**
> - 💡 타입/형태 불일치는 **스키마 검증으로 예방**하고, Boundary는 최후의 안전망으로 두는 게 정석

---
