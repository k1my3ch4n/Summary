---
created: 2026-11-13
updated: 2026-11-13
domain: General
tags: [browser, network, rendering, http, dns, tcp, tls]
---

# 브라우저에서 URL 입력부터 화면 렌더링까지

## 📖 핵심 개념 및 동작 원리

> [!tip] 한 줄 요약
> `URL 파싱 → DNS 조회 → TCP/TLS 연결 → HTTP 요청/응답 → HTML 파싱 → DOM/CSSOM → Render Tree → Layout → Paint → Composite`
> 브라우저는 "문서를 받아오는 단계(네트워크)"와 "픽셀로 그리는 단계(렌더링)"로 크게 나뉩니다.

---

### 1단계. URL 입력 및 전처리

| 처리 | 설명 |
|---|---|
| 스킴 판별 | 검색어인지 URL인지 판단 → `google.com`은 `https://google.com`으로 보정 |
| URL 파싱 | scheme / host / port / path / query / fragment 분해 |
| HSTS 확인 | `google.com`은 HSTS Preload 목록에 있어 **http 요청 없이 즉시 https로 전환** |
| 퓨니코드 변환 | 한글·유니코드 도메인은 `xn--` 형태로 변환 |

### 2단계. DNS 조회 (도메인 → IP)

조회는 **캐시 계층을 순서대로** 탐색합니다.

1. 브라우저 자체 DNS 캐시
2. OS 캐시 (`/etc/hosts` 포함)
3. 라우터 캐시
4. ISP의 리커시브 DNS 서버
5. 권한 서버 체인: Root → TLD(`.com`) → Authoritative(`ns.google.com`)

> [!important] 실무 포인트
> - DNS 결과는 TTL 동안 캐시되므로 **DNS 레코드 변경 시 즉시 반영되지 않습니다.**
> - 느린 DNS는 FCP를 직접 늦춥니다. `<link rel="dns-prefetch">`, `preconnect`로 선조회를 유도할 수 있습니다.
> - 최근에는 프라이버시를 위해 DoH(DNS over HTTPS)를 쓰는 경우도 많습니다.

### 3단계. TCP 연결 (3-way Handshake)

```
Client → SYN       → Server
Client ← SYN+ACK   ← Server
Client → ACK       → Server   # 연결 성립
```

- 1 RTT 소요. 지연(latency)이 큰 환경일수록 비용이 큽니다.
- HTTP/3(QUIC)는 TCP 대신 **UDP 기반**이라 이 단계와 TLS를 합쳐 0~1 RTT로 줄입니다.

### 4단계. TLS Handshake (HTTPS 암호화)

1. ClientHello: 지원 암호 스위트, TLS 버전, **SNI(접속하려는 호스트명)** 전달
2. ServerHello + 인증서 전달
3. 인증서 검증: CA 체인 신뢰 여부, 도메인 일치, 만료일, 폐기(OCSP) 확인
4. 키 교환(ECDHE) → **세션 키** 생성 → 이후 대칭키 암호화 통신

> [!note] TLS 1.3는 1 RTT, 세션 재개(resumption) 시 0-RTT까지 가능합니다.

### 5단계. HTTP 요청 / 응답

요청 예시:

```http
GET / HTTP/2
Host: www.google.com
User-Agent: Mozilla/5.0 ...
Accept: text/html,application/xhtml+xml,...
Accept-Encoding: gzip, deflate, br
Cookie: ...
```

응답 예시:

```http
HTTP/2 200 OK
Content-Type: text/html; charset=UTF-8
Content-Encoding: br
Cache-Control: private, max-age=0
...

<!doctype html><html>...
```

- 실제로 `google.com` 입력 시 `www.google.com`으로 **301/302 리다이렉트**가 한 번 발생합니다 → 리다이렉트마다 추가 RTT 비용.
- 응답 본문은 gzip/brotli로 압축되어 오고, 브라우저가 해제합니다.
- 응답은 한 번에 오지 않고 **TCP 세그먼트 단위로 스트리밍**됩니다. 브라우저는 다 받기 전에 파싱을 시작합니다.

### 6단계. HTML 파싱 → DOM 생성

- 바이트 → 문자(charset 디코딩) → 토큰 → 노드 → **DOM Tree**
- 파싱 도중 리소스를 만나면:

| 리소스 | 파싱 블로킹 | 설명 |
|---|---|---|
| `<link rel="stylesheet">` | 렌더링 블로킹 | CSSOM 완성 전에는 화면을 그리지 않음 |
| `<script>` (일반) | **파싱 블로킹** | 다운로드+실행 동안 HTML 파싱 중단 |
| `<script defer>` | 비블로킹 | 파싱 완료 후 순서대로 실행 |
| `<script async>` | 비블로킹 | 다운로드 완료 즉시 실행(순서 보장 X) |
| `<img>` | 비블로킹 | 레이아웃에만 영향 |

- **Preload Scanner**: 메인 파서가 스크립트에 막혀 있어도, 별도 스캐너가 앞쪽 HTML을 훑어 리소스를 미리 다운로드합니다.

### 7단계. CSSOM 생성 & Render Tree 구성

- CSS 파싱 → **CSSOM Tree**
- DOM + CSSOM 결합 → **Render Tree** (화면에 그려질 노드만 포함)
  - `display: none` → Render Tree에서 제외
  - `visibility: hidden` → 포함됨(공간 차지)

### 8단계. Layout(Reflow) → Paint → Composite

| 단계 | 하는 일 |
|---|---|
| Layout | 각 노드의 **위치와 크기**를 뷰포트 기준으로 계산 (%, em → px 확정) |
| Paint | 텍스트/색/그림자/테두리 등을 **그리기 명령 목록**으로 변환 |
| Layerize | `transform`, `will-change`, `position: fixed` 등을 별도 레이어로 분리 |
| Composite | GPU가 레이어들을 합성해 최종 픽셀을 화면에 출력 |

> [!important] 성능 최적화 핵심
> - `width`, `top` 변경 → **Layout부터 재실행** (가장 비쌈)
> - `background-color` 변경 → Paint부터
> - `transform`, `opacity` 변경 → **Composite만** 실행 (가장 쌈, 애니메이션에 권장)

### 9단계. JS 실행과 이벤트

- `DOMContentLoaded`: DOM 파싱 완료 (이미지 대기 X)
- `load`: 이미지·폰트 등 **모든 서브 리소스** 로드 완료
- 이후 JS가 DOM을 변경하면 → 변경 범위에 따라 Layout/Paint/Composite가 다시 수행됨

---

### 전체 흐름 요약도

```
URL 입력
  └─ HSTS/URL 정규화
      └─ DNS 조회 (캐시 → 리커시브 → 권한 서버)
          └─ TCP 3-way handshake
              └─ TLS handshake (인증서 검증 + 세션키)
                  └─ HTTP 요청 → 응답(HTML 스트리밍)
                      ├─ HTML 파싱 → DOM
                      ├─ CSS 파싱 → CSSOM
                      │   └─ Render Tree
                      │       └─ Layout → Paint → Composite → 화면 표시
                      └─ JS 실행 → DOM 변경 시 재렌더링
```

> [!tip] 면접 단골 포인트
> 1. DNS는 **캐시 계층**을 순서대로 탐색한다
> 2. CSS는 **렌더링 블로킹**, 동기 `<script>`는 **파싱 블로킹**
> 3. `transform`/`opacity` 애니메이션은 Layout/Paint를 건너뛴다
> 4. 브라우저는 HTML을 **다 받기 전에 점진적으로 파싱·렌더링**한다

관련 키워드: [[DNS]] · [[TCP]] · [[TLS]] · [[HTTP]] · [[Critical-Rendering-Path]] · [[Reflow-Repaint]] · [[CDN]]

---

## 💬 Q&A 기록

### Q1. 유저가 google.com에 접속했을 때, 네트워크를 통해 HTML을 받아 사용자에게 표출하는 과정은?

**A.** 위 핵심 개념 섹션의 1~9단계가 전체 답변입니다. 압축하면:

1. **주소 해석**: `google.com` → HSTS로 https 확정 → URL 파싱
2. **DNS**: 캐시 계층 탐색 후 권한 서버에서 IP 획득
3. **연결**: TCP 3-way handshake → TLS handshake(인증서 검증 + 세션키 교환)
4. **요청/응답**: `GET /` 전송 → `www.google.com`으로 리다이렉트 → 압축된 HTML을 스트리밍으로 수신
5. **파싱**: HTML → DOM, CSS → CSSOM, 둘을 합쳐 Render Tree
6. **렌더링**: Layout(위치·크기) → Paint(그리기 명령) → Composite(GPU 합성) → 화면 출력
7. **후처리**: JS 실행, `DOMContentLoaded` / `load` 이벤트, 이후 DOM 변경 시 부분 재렌더링

> [!note] 핵심은 "순차적 완료"가 아니라 **파이프라인**이라는 점입니다. HTML을 전부 받기 전에 파싱이 시작되고, Preload Scanner가 리소스를 선취하며, 화면은 점진적으로 그려집니다.

---
