# TALKI 프론트엔드 포트폴리오

> AI 기반 실시간 발표 피드백 서비스 — 프론트엔드 개발 담당

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [기술 스택](#2-기술-스택)
3. [서비스 아키텍처](#3-서비스-아키텍처)
4. [주요 기능](#4-주요-기능)
5. [담당 기능 상세](#5-담당-기능-상세)
6. [기술적 성과](#6-기술적-성과)
7. [트러블 슈팅](#7-트러블-슈팅)
8. [프로젝트를 통해 얻은 경험](#8-프로젝트를-통해-얻은-경험)
9. [핵심 요약](#9-핵심-요약)

---

## 1. 프로젝트 개요

### 프로젝트 소개

**TALKI**는 발표 불안을 겪는 사용자가 실제 발표 환경과 유사한 조건에서 반복적으로 연습하고, AI 기반 분석을 통해 즉각적이고 구체적인 피드백을 받을 수 있는 웹 서비스입니다.

### 개발 목적

발표를 앞두고 막연한 불안을 느끼는 사람들이 많지만, 실질적인 연습 환경을 갖추기 어렵습니다. TALKI는 다음 문제를 해결하고자 기획되었습니다.

- **환경 재현**: 화상 회의, 강당, 강의실 등 상황별 발표 환경을 배경 영상으로 시뮬레이션
- **즉각적 피드백**: 발표 도중 시선·자세·말 속도를 WebSocket으로 실시간 분석
- **데이터 기반 개선**: 발표 종료 후 AI 리포트로 강점·성장 포인트·연습 방향 제시
- **단계적 훈련**: 즉흥 말하기·키워드 구성·핵심 파악 등 체계적인 사전 연습 제공

### 해결하고자 한 문제

| 문제 | TALKI의 접근 |
|------|-------------|
| 혼자 연습할 환경이 없다 | 상황별 배경 영상 + 카메라 피드 |
| 내가 잘하고 있는지 모른다 | 실시간 WebSocket 피드백 |
| 막연한 "잘하자"가 아닌 구체적 개선점 필요 | LLM 기반 5개 항목 정량 분석 리포트 |
| 부담 없는 사전 훈련이 필요하다 | 3가지 유형의 연습 모드 |

---

## 2. 기술 스택

### Frontend

| 분류 | 기술 |
|------|------|
| UI 라이브러리 | React 19.2.0 |
| 언어 | TypeScript 5.9.3 |
| 빌드 도구 | Vite 7.2.4 |
| 라우팅 | React Router DOM 7.12.0 |
| 스타일링 | Tailwind CSS 3.4.19 |
| 차트 | Recharts 3.7.0 |
| 브라우저 API | MediaRecorder, Web Audio API, WebSocket, IntersectionObserver, ResizeObserver |
| 컴퓨터 비전 | MediaPipe FaceMesh, Pose, Camera Utils |
| PDF 생성 | html2canvas 1.4.1 + jsPDF 4.2.0 |
| 아이콘 | react-icons 5.5.0 |

### Backend (연동)

- REST API: `http://43.201.182.246:8080`
- WebSocket: `ws://43.201.182.246:8080/realtime?type={presentationType}`
- 파일 스토리지: AWS S3 (Presigned URL 방식)

### 개발 환경

- 코드 품질: ESLint 9, Prettier 3.8 + prettier-plugin-tailwindcss
- 경로 별칭: `@/` → `src/` (Vite alias)
- API 프록시: 개발 환경에서 `/api` → 백엔드 서버 (CORS 우회)

---

## 3. 서비스 아키텍처

### 전체 구조

```
사용자 브라우저
    │
    ├─ React SPA (Vite 빌드)
    │       │
    │       ├─ REST API (fetchClient.tsx)
    │       │       └─ JWT 자동 갱신 (401 → /auth/reissue → 재시도)
    │       │
    │       ├─ WebSocket (LiveFeedbackTracker)
    │       │       └─ 실시간 face/pose/audio 페이로드 전송
    │       │
    │       └─ S3 Presigned URL
    │               └─ PUT 직접 업로드 (영상 파일)
    │
    └─ MediaPipe (CDN WASM)
            ├─ FaceMesh: 468랜드마크 (시선 추적)
            └─ Pose: 33랜드마크 (자세 분석)
```

### 폴더 구조

```
src/
├── pages/                  # 라우트 단위 페이지
│   ├── LandingPage/        # 서비스 소개 랜딩
│   ├── HomePage/           # 사용자 대시보드
│   ├── LiveFeedback/       # 실시간 실전 페이지
│   ├── FeedbackResult/     # 분석 결과 페이지
│   ├── Practice/           # 연습 모드 (3가지 유형)
│   ├── GuideLine/          # 카메라/마이크 환경 점검
│   ├── Category/           # 발표 환경 설정
│   ├── Login/ Signup/      # 인증
│   └── AnalysisLoading/    # 분석 대기 화면
├── components/             # 재사용 가능한 공통 컴포넌트
│   ├── Nav/                # 헤더 네비게이션
│   ├── CircleProgress/     # 원형 진행률 차트
│   ├── LinearProgressBar/  # 선형 진행률 바
│   └── Practice/           # 연습 레이아웃·사이드바
├── api/                    # API 클라이언트
│   ├── fetchClient.tsx     # JWT 자동 갱신 포함 HTTP 클라이언트
│   └── personal.tsx        # 개인 홈 데이터 도메인 API
└── utils/
    ├── pdfDownload.tsx     # html2canvas + jsPDF 유틸
    └── images.tsx          # 에셋 경로 상수
```

### 컴포넌트 계층 구조

```
App (라우터)
├── LandingMain
│   ├── HeroSection           (SVG 웨이브 애니메이션)
│   ├── PracticeSection       (스크롤 기반 카드 scatter 애니메이션)
│   ├── WorrySection / FaqSection / CtaSection
│   └── Footer
│
├── Home
│   ├── LeftPanel             (스트릭, 마음가짐, 최근 리포트)
│   ├── CalendarView          (연습 이력 달력)
│   └── DashboardView + GrowthGraph
│
├── LiveFeedback
│   ├── LiveFeedbackTracker   (hidden — 카메라/오디오/WS/MediaPipe)
│   ├── VoiceWaveIndicator    (음성 감지 파형 시각화)
│   ├── CountdownOverlay      (5초 카운트다운)
│   └── TutorialModal
│
├── FeedbackResult
│   ├── AnalysisResult        (총점, CircleProgress ×4, 피드백 텍스트)
│   └── AnalysisResultDetail  (5개 세부 섹션 + TriangleChart + VideoPlayer)
│
└── Practice (3종)
    ├── ImpromptuPractice
    ├── KeywordPractice
    └── CoreUnderstandingPractice
        (공통: PracticeLayout > TitleSection + 콘텐츠 + MicButton + CoachBubble)
```

---

## 4. 주요 기능

| 기능 | 설명 |
|------|------|
| 실시간 실전 발표 | 상황별 배경 영상 + 카메라 피드 + WebSocket 실시간 분석 |
| 실시간 피드백 | WebSocket으로 수신한 피드백 메시지를 화면에 3초 표시 |
| 발표 영상 업로드 | MediaRecorder로 녹화 → S3 Presigned PUT 업로드 |
| AI 분석 리포트 | 시선·자세·말 속도·주제 적절성·추임새 5개 항목 정량 점수 |
| 세부 분석 (Premium) | WPM 그래프, 카메라 응시율, 자세 불안정 비율, 삼각형 레이더 차트 |
| 타임스탬프 영상 리뷰 | 약점 구간·돌발 질문 타임스탬프 클릭 시 해당 시점으로 영상 이동 |
| 연습 모드 (3종) | 즉흥 말하기 / 키워드 기반 구성 / 핵심 파악 |
| 스트릭 & 홈 대시보드 | 연속 연습일 수, 이번 주 연습 달력, 최근 리포트 요약 |
| PDF 다운로드 | html2canvas + jsPDF로 분석 결과 PDF 저장 |
| 토큰 자동 갱신 | 401 응답 시 refreshToken으로 재발급 후 원본 요청 재시도 |

---

## 5. 담당 기능 상세

---

### 5.1 메인 랜딩 페이지

#### 구현 내용

서비스의 첫인상을 결정하는 랜딩 페이지를 전체 구현했습니다. `HeroSection`, `WorrySection`, `PracticeSection`, `RequirementsSection`, `FaqSection`, `CtaSection`, `Footer` 7개 섹션으로 구성되며, 서비스의 가치 제안을 시각적으로 전달합니다.

#### 기술적 특징 — 스크롤 연동 네비게이션 전환

```tsx
// LandingMain.tsx
const sentinelRef = useRef<HTMLDivElement>(null)
const [useMainNav, setUseMainNav] = useState(true)

const handleScroll = () => {
  const sentinelTop = sentinelRef.current.getBoundingClientRect().top
  setUseMainNav(sentinelTop > 0)
}

// HeroSection 직후에 보이지 않는 sentinel div 배치
<main onScroll={handleScroll} className="h-screen snap-y snap-mandatory overflow-y-auto">
  {useMainNav ? <MainNav /> : <Nav />}
  <HeroSection />
  <div ref={sentinelRef} />  {/* 이 div가 뷰포트를 벗어나면 Nav 전환 */}
  <PracticeSection />
  ...
</main>
```

`IntersectionObserver` 대신 `getBoundingClientRect().top`을 스크롤 이벤트에서 직접 읽어 sentinel div의 뷰포트 통과 여부를 판별합니다. Hero 섹션이 지나가면 투명 배경의 `MainNav`에서 흰 배경의 `Nav`로 전환됩니다.

#### 기술적 특징 — 스크롤 기반 카드 scatter 애니메이션

`PracticeSection`의 핵심 인터랙션은 스크롤에 따라 쌓인 카드가 순서대로 펼쳐지는 효과입니다.

```tsx
// PracticeSection.tsx
function easeOutCubic(t: number): number {
  return 1 - Math.pow(1 - t, 3)
}

// 1단계: useLayoutEffect로 각 카드의 실제 높이를 측정하여 spread 목표 위치 계산
useLayoutEffect(() => {
  const heights = cardRefs.current.map((ref) => ref?.offsetHeight ?? 0)
  let y = 0
  heights.forEach((h) => {
    positions.push(y)
    y += h + GAP  // GAP = 48px
  })
  setNaturalPositions(positions)
  setNaturalHeight(y - GAP)  // 컨테이너의 실제 필요 높이
}, [currentCards])

// 2단계: 스크롤 progress (0→1) 계산
// progress = 섹션이 얼마나 스크롤되었는지 비율
const scrolled = -cardsContainerRef.current.getBoundingClientRect().top
const total = Math.max(1, naturalHeight - vh)
setScrollProgress(Math.max(0, Math.min(1, scrolled / total)))

// 3단계: 카드별 애니메이션 계산
// 카드 i는 progress의 [(i-1)/n ~ i/n] 구간에서만 이동 (순차 펼침)
const cardP = index === 0
  ? 1
  : Math.max(0, Math.min(1, (scrollProgress - (index - 1) / n) * n))

const eased = easeOutCubic(cardP)
const top = fromTop + (toTop - fromTop) * eased  // stacked → spread 위치 보간
const scale = 1 - (1 - eased) * index * 0.02    // 뒤 카드일수록 약간 축소
const opacity = index === 0 ? 1 : 0.65 + 0.35 * eased
```

설계 포인트:
- **useLayoutEffect**: DOM이 페인트되기 전에 카드 높이를 측정해 레이아웃 깜빡임 방지
- **순차 펼침**: 전체 스크롤 범위를 카드 수로 분할해 각 카드가 독립적인 구간에서 이동
- **willChange: 'top, transform, opacity'**: GPU 레이어 분리로 합성 단계에서 처리
- **easeOutCubic**: 카드 착지 시 자연스러운 감속 효과

#### 성능 최적화

- 배경 SVG 웨이브 애니메이션은 `transform`과 `filter`만 변경 (reflow 없음)
- 실전/연습 탭 전환 시 `currentCards` 참조만 교체 → 컨테이너 재계산 최소화
- 스크롤 이벤트에 `passive: true` 옵션으로 스크롤 블로킹 방지

#### 사용자 경험 개선

- `snap-y snap-mandatory`로 섹션 단위 스냅 스크롤 구현
- 모바일/데스크톱 CTA 버튼 분기 (플랫폼별 진입 경로 차별화)
- Hero → PracticeSection 전환 시 네비게이션이 자연스럽게 교체되어 몰입감 유지

---

### 5.2 메인 홈페이지

#### 구현 내용

로그인 후 진입하는 대시보드 페이지입니다. 사용자의 연속 연습 스트릭, 이번 주 연습 이력, 마음가짐 기록, 최근 리포트 요약을 한눈에 확인할 수 있으며, 달력 뷰와 대시보드 뷰를 토글로 전환할 수 있습니다.

#### 구현 구조

```tsx
// Home.tsx — 두 API를 독립적으로 병렬 호출
useEffect(() => {
  const fetchUserName = async () => {
    const data = await api.get('/profile/get')
    setUserName(data.userName ?? data.userId ?? null)
  }
  fetchUserName()
}, [])

useEffect(() => {
  const fetchHomeData = async () => {
    const data = await getPersonalHome()
    setHomeData(data)
    setStreak(data.practiceDays.streakDays || 0)
    // ISO 날짜 문자열 → Date 객체 변환
    const dates = data.practiceDays.practiceDays.map((d: string) => {
      const [y, m, day] = d.split('-').map(Number)
      return new Date(y, m - 1, day)  // 로컬 타임존 기준
    })
    setPracticedDates(dates)
  }
  fetchHomeData()
}, [])
```

#### 기술적 특징 — API 장애 격리

`/personal/home` 엔드포인트와 `/profile/get`을 별도 `useEffect`로 분리했습니다. 홈 데이터 API가 실패해도 닉네임 표시는 영향을 받지 않으며, 반대로 프로필 API 실패가 홈 데이터 로딩을 막지 않습니다.

#### 데이터 처리 방식

```tsx
// LeftPanel.tsx — 이번 주 월~일 날짜 배열 계산
const dayOfWeek = today.getDay()  // 0=일, 1=월 ...
const monday = new Date(today)
monday.setDate(today.getDate() - (dayOfWeek === 0 ? 6 : dayOfWeek - 1))

const currentWeek = Array.from({ length: 7 }, (_, i) => {
  const d = new Date(monday)
  d.setDate(monday.getDate() + i)
  return d
})

// 연습일 판별: 연도·월·일 모두 일치 확인
const isSameDay = (a: Date, b: Date) =>
  a.getFullYear() === b.getFullYear() &&
  a.getMonth() === b.getMonth() &&
  a.getDate() === b.getDate()
```

- 서버에서 받은 `"YYYY-MM-DD"` 문자열을 `new Date(y, m-1, day)`로 로컬 타임존 기준 Date로 변환 (`new Date("YYYY-MM-DD")`는 UTC 기준이므로 시차 버그 방지)
- `isSameDay` 헬퍼로 연습일 여부를 O(n) 탐색

#### 반응형 레이아웃

- 모바일: `order-2` (LeftPanel 하단) / `order-1` (우측 패널 상단)
- 데스크톱: 사이드 바이사이드 (`lg:w-1/3` / `lg:w-2/3`)
- 모달: 모바일은 인라인 대체, 데스크톱은 `absolute` 오버레이

---

### 5.3 실시간 실전 페이지

#### 구현 내용

서비스의 핵심 기능입니다. 사용자가 카메라 앞에서 발표하는 동안 MediaPipe로 시선·자세 데이터를 수집하고, Web Audio API로 음성을 처리해 WebSocket으로 백엔드에 전송합니다. 동시에 MediaRecorder로 영상을 녹화합니다.

#### 핵심 아키텍처 — LiveFeedbackTracker

화면에는 보이지 않는 `<video>` 하나만 렌더링하는 컴포넌트입니다. 카메라·음성·WebSocket·MediaPipe의 모든 복잡한 처리를 캡슐화하고, 부모에게는 `stopRecording(): Promise<Blob>` 하나만 노출합니다.

```tsx
// 부모가 접근 가능한 인터페이스
export interface LiveFeedbackTrackerRef {
  stopRecording: () => Promise<Blob>
}

// 컴포넌트 외부(부모)로 노출
useImperativeHandle(ref, () => ({ stopRecording }))
```

#### 실시간 처리 방식 — 데이터 파이프라인

```
getUserMedia (video+audio)
    │
    ├─ [video track] ─→ MediaPipe Camera → FaceMesh (468 랜드마크)
    │                                   → Pose (33 랜드마크)
    │                                            │
    ├─ [audio track] ─→ AudioContext ──────────→ buildPayload()
    │   ScriptProcessor                           │
    │   (1초 누적 후 전송)                        │
    │                                             ▼
    │                                    WebSocket.send(JSON)
    │
    └─ [video+audio] ─→ MediaRecorder → Blob chunks (1초 단위)
                                      → stopRecording() → Blob
```

```tsx
// buildPayload: 16개 얼굴 랜드마크 + 4개 자세 랜드마크 + base64 오디오
const FACE_INDICES = [468, 469, 470, 471, 473, 474, 475, 476, 33, 133, 362, 263, 159, 386, 145, 374]
const POSE_INDICES = [13, 14, 15, 16]  // 양 팔꿈치·손목 (제스처 판단)

function buildPayload(base64Audio: string) {
  const face: Record<string, { x: number; y: number }> = {}
  FACE_INDICES.forEach((idx) => {
    face[String(idx)] = faceLandmarks?.[idx]
      ? { x: Number(faceLandmarks[idx].x.toFixed(3)), y: Number(faceLandmarks[idx].y.toFixed(3)) }
      : { x: 0, y: 0 }
  })
  return { face, pose, audio: base64Audio, timestamp: Date.now() }
}
```

```tsx
// 오디오: Float32 PCM → Int16 → Base64 변환 후 전송
function float32ToInt16(float32: Float32Array): Int16Array {
  const int16 = new Int16Array(float32.length)
  for (let i = 0; i < float32.length; i++) {
    const s = Math.max(-1, Math.min(1, float32[i]))
    int16[i] = s < 0 ? s * 0x8000 : s * 0x7fff
  }
  return int16
}
// 1초(sampleRate 개) 누적 후 한 번에 전송 → 불필요한 WS 메시지 최소화
```

#### 상태 관리

LiveFeedbackTracker는 9개의 ref를 사용합니다. state 대신 ref를 선택한 이유는, 카메라 프레임마다(~30fps) 갱신되는 `faceDataRef`, `poseDataRef`를 state로 관리하면 불필요한 리렌더링이 발생하기 때문입니다.

```tsx
const videoRef       = useRef<HTMLVideoElement>()     // 카메라 스트림 표시
const mediaRecorderRef = useRef<MediaRecorder>()      // 녹화기
const recordedChunksRef = useRef<Blob[]>([])          // 영상 청크
const streamRef      = useRef<MediaStream>()          // 원본 스트림
const cameraRef      = useRef<Camera>()               // MediaPipe 카메라
const faceDataRef    = useRef<FaceMeshResults>()      // 얼굴 랜드마크 (매 프레임 갱신)
const poseDataRef    = useRef<PoseResults>()          // 자세 랜드마크 (매 프레임 갱신)
const wsRef          = useRef<WebSocket>()            // WebSocket 연결
const audioContextRef = useRef<AudioContext>()        // 오디오 컨텍스트
```

#### 카운트다운 기간 녹화 제외

5초 카운트다운 동안 녹화가 포함되면 결과가 왜곡됩니다. 이를 해결하기 위해 `canRecord` prop을 추가하고, 카메라 초기화와 녹화 시작을 분리했습니다.

```tsx
// LiveFeedback.tsx
<LiveFeedbackTracker canRecord={!showCountdown} ... />

// LiveFeedbackTracker.tsx
// 카메라 초기화 시: canRecord가 false면 녹화 보류
if (canRecordRef.current) {
  startRecordingFromStream(stream)
}

// canRecord가 true로 바뀌는 순간 (카운트다운 종료) 녹화 시작
useEffect(() => {
  canRecordRef.current = canRecord
  if (canRecord && streamRef.current && !mediaRecorderRef.current) {
    startRecordingFromStream(streamRef.current)
  }
}, [canRecord])
```

#### 영상 업로드 플로우

```tsx
// 1. 녹화 종료 → Blob 획득
const blob = await trackerRef.current.stopRecording()

// 2. 백엔드에서 S3 Presigned URL 발급
const resData = await api.post('/videos/upload-url', {
  presentationId, filename, contentType: 'video/mp4',
  userId, presentationType, topic
})

// 3. S3에 직접 PUT 업로드 (인증 헤더 불필요)
const uploadRes = await fetch(resData.uploadUrl, {
  method: 'PUT',
  body: blob,
  headers: { 'Content-Type': 'video/mp4' }
})

// 4. 분석 대기 페이지로 이동
navigate('/analysis-loading', { state: { presentationId, key: resData.key } })
```

#### 주요 기술 — MediaPipe WASM 충돌 방지

```tsx
// 잘못된 방식 (WASM 전역 Module 충돌 발생)
await Promise.all([faceMesh.send({ image }), pose.send({ image })])

// 올바른 방식 (순차 실행)
await faceMesh.send({ image })
await pose.send({ image })
```

MediaPipe FaceMesh와 Pose는 내부적으로 같은 전역 WASM Module 객체를 공유하므로 `Promise.all`로 병렬 실행 시 충돌합니다. 순차 실행으로 해결했습니다.

---

### 5.4 실시간 분석 세부 결과 페이지

#### 구현 내용

발표 종료 후 AI가 분석한 결과를 5개 섹션으로 제공합니다. 기본 분석(무료)과 세부 분석(프리미엄)으로 나뉘며, 세부 분석에는 WPM 시각화, 영상 타임스탬프 리뷰, 삼각형 레이더 차트가 포함됩니다.

#### 데이터 시각화

**① 원형 진행률 차트 (CircleProgress)**

4개 항목(발화 속도·시선 집중도·주제 적절성·제스처 안정성)을 원형 SVG로 시각화합니다.

```tsx
// 1500ms 동안 60 스텝으로 선형 증가 애니메이션
const duration = 1500
const steps = 60
const stepDuration = duration / steps  // 25ms 간격

// SVG stroke-dashoffset으로 진행률 표현
const circumference = normalizedRadius * 2 * Math.PI
const strokeDashoffset = -(circumference - (animatedPercentage / 100) * circumference)

// 점수 구간별 색상 코딩
const getStrokeColor = (score: number) =>
  score < 40 ? '#DB1013' : score < 70 ? '#FFA956' : '#4DCB56'
```

**② WPM 바 차트**

발화 속도를 0~200 WPM 트랙 위에 표시합니다. 120~160 WPM 구간을 초록색으로 마킹하고, 현재 WPM 위치에 삼각형 마커를 표시합니다.

```tsx
// 적정 범위 초록 구간 (120~160 / 200 = 40%~80%)
<div className="absolute h-full bg-[#A8D8A8]" style={{ left: '40%', width: '20%' }} />
// 현재 WPM 바
<div className="absolute h-full bg-[#5650FF]"
     style={{ width: `${Math.min((wpm / 200) * 100, 100)}%` }} />
// 삼각형 마커 (CSS border trick)
<div className="h-0 w-0 border-x-4 border-b-[6px] border-x-transparent border-b-[#3B3B3B]" />
```

**③ 삼각형 레이더 차트 (TriangleChart)**

시선 집중도·제스처 안정성·주제 적절성 3개 항목을 정삼각형 레이더 차트로 시각화합니다. 기존 이미지 파일을 실제 SVG 컴포넌트로 교체했습니다.

```tsx
// 정삼각형 꼭지점: 위(-90°), 우하단(30°), 좌하단(150°)
const vtx = (angle: number, scale = 1): [number, number] => [
  cx + r * scale * Math.cos(angle),
  cy + r * scale * Math.sin(angle),
]

// 외부 격자 삼각형 5단계 (20%~100%)
{[0.2, 0.4, 0.6, 0.8, 1.0].map((level) => (
  <polygon points={gridPoints(level)} fill="none"
    stroke={level === 1 ? '#C8C5F0' : '#EEEDF8'} />
))}

// 데이터 삼각형: 각 꼭지점을 점수 비율(0~1)로 스케일
const [dtx, dty] = vtx(topAngle, gazeScore / 100)
const [dbrx, dbry] = vtx(brAngle, topicScore / 100)
const [dblx, dbly] = vtx(blAngle, postureScore / 100)

// 오렌지 그라디언트 채우기
<defs>
  <linearGradient id="triGrad" x1="0.5" y1="0" x2="0.5" y2="1">
    <stop offset="0%" stopColor="#FFAB76" stopOpacity="0.8" />
    <stop offset="100%" stopColor="#FFC78A" stopOpacity="0.4" />
  </linearGradient>
</defs>
```

#### 결과 분석 로직 — IntersectionObserver 기반 지연 렌더링

5개 섹션이 한꺼번에 렌더링되면 초기 로드가 무거워집니다. `IntersectionObserver`로 섹션이 뷰포트에 진입할 때만 애니메이션을 적용합니다.

```tsx
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        const index = parseInt(entry.target.getAttribute('data-index') || '0')
        setVisibleSections((prev) => [...new Set([...prev, index])])
      }
    })
  },
  { threshold: 0.2 }
)
```

```tsx
// FeedbackSection.tsx — CSS transition으로 부드러운 진입
className={`transition-all duration-700 ${
  isVisible ? 'translate-y-0 opacity-100' : 'translate-y-10 opacity-0'
}`}
```

#### 영상 타임스탬프 리뷰

약점 구간과 돌발 질문에 타임스탬프를 표시하고, 클릭 시 해당 시점으로 영상을 이동합니다.

```tsx
// "m:ss" 또는 "h:mm:ss" → 초 변환
const parseTimestamp = (ts: string): number => {
  const parts = ts.split(':').map(Number)
  if (parts.length === 2) return parts[0] * 60 + parts[1]
  return parts[0] * 3600 + parts[1] * 60 + parts[2]
}

// 클릭 시 영상 currentTime 변경
videoRef.current.currentTime = parseTimestamp(item.time)
videoRef.current.play()
```

#### 분석 결과 폴링

백엔드 분석에 시간이 소요되므로 10초 간격으로 결과를 폴링합니다. 컴포넌트 언마운트 시 레이스 컨디션을 방지하는 `isCancelled` 플래그를 사용합니다.

```tsx
let isCancelled = false
let timeoutId: ReturnType<typeof setTimeout>

const poll = async () => {
  if (isCancelled) return
  const res = await api.get(`/analyze/getResult?presentationId=${presentationId}`)
  if (res) {
    setAnalysisData(processedData)
    setLoading(false)
    return  // 성공 시 폴링 중단
  }
  if (!isCancelled) {
    timeoutId = setTimeout(poll, 10000)  // 실패 시 10초 후 재시도
  }
}

return () => {
  isCancelled = true
  clearTimeout(timeoutId)
}
```

---

### 5.5 즉흥 말하기 연습

#### 구현 내용

3개의 즉흥 질문 중 무작위로 1개가 출제되며, 10초 준비 → 30초 발표의 단계적 흐름으로 진행됩니다. 마이크로 음성을 녹음하고, 완료 후 간단한 피드백을 제공합니다.

#### 사용자 흐름

```
idle → (마이크 버튼 클릭) → preparing (10초 카운트다운)
     → recording (30초 녹음) → finished (피드백 표시)
```

```tsx
export type PracticeStep = 'idle' | 'preparing' | 'recording' | 'finished'

// 준비 → 녹음 자동 전환
useEffect(() => {
  let timer: ReturnType<typeof setInterval>
  if (step === 'preparing') {
    if (prepTimeLeft > 0) {
      timer = setInterval(() => setPrepTimeLeft((prev) => prev - 1), 1000)
    } else {
      startRecording()  // 0초 도달 시 자동 전환
    }
  }
  return () => clearInterval(timer)
}, [step, prepTimeLeft])
```

#### 구현 기술

- **MediaRecorder API**: 마이크 오디오 스트림 캡처 (`audio/webm` 형식)
- **PracticeLayout**: 사이드바 스텝 인디케이터 + 탐색 잠금 공통 컴포넌트
- **잠금 처리**: 준비/녹음 중에는 `isLocked=true`로 다음 단계 이동 차단
- **랜덤 출제**: 진입 시 `useEffect`에서 `Math.random()`으로 1~3번 질문 선택
- **CoachBubble**: 단계별 다른 안내 메시지 (진행 중 → 완료 후 피드백)

---

### 5.6 키워드 기반 구성 연습

#### 구현 내용

3개의 키워드(협업·문제해결·성장)를 모두 포함하여 30초 발표를 구성하는 연습입니다. 발표 종료 후 사용된 키워드 수와 평가를 시각적으로 표시합니다.

#### 데이터 처리 방식

```tsx
interface KeywordType {
  id: number
  text: string
  isUsed: boolean
}

// 완료 후 사용 키워드 업데이트 (불변성 유지)
setKeywords((prev) =>
  prev.map((kw, i) => (i === 0 || i === 2 ? { ...kw, isUsed: true } : kw))
)

// KeywordAnalysis: 사용 비율에 따른 평가 레이블
<KeywordAnalysis
  usedCount={keywords.filter((k) => k.isUsed).length}  // 2
  totalCount={keywords.length}                           // 3
  evaluation="양호"
  detail="연결 자연스러움"
/>
```

#### 구현 기술

- 준비/녹음 타이머를 `setInterval` 대신 `setTimeout` 재귀 호출로 구현 (정밀도 향상)
- 언마운트 시 `useEffect` cleanup에서 `MediaRecorder.stop()` 강제 호출 (스트림 누수 방지)
- 키워드 알약(pill) UI: 사용됨 여부에 따라 초록색 강조

---

### 5.7 핵심 파악 연습

#### 구현 내용

긴 지문을 읽고 핵심 키워드 3개(의사소통·경청·협업)를 포함해 30초 동안 요약 발표하는 연습입니다. 준비 시간 동안 지문을 충분히 읽을 수 있도록 스크롤 가능한 텍스트 카드를 제공합니다.

#### 구현 기술

```tsx
// CoreTextCard: 지문 표시 + 키워드 하이라이팅
// keywords 배열을 기반으로 원문에서 해당 단어 강조 표시
```

- `isCompact={true}` prop으로 타이머를 소형 레이아웃으로 전환 → 지문 읽기 공간 확보
- 동일한 `PracticeStep` 타입과 타이머 로직을 3가지 연습 페이지에 공통으로 재사용
- CoachBubble에 단계별 "요약 가이드" 체크리스트 제공 (원문 핵심 파악 → 키워드 포함 → 간결한 표현)

#### 사용자 경험 고려사항

3가지 연습 모드(즉흥·키워드·핵심파악)는 `PracticeLayout` 공통 래퍼 아래 동일한 UX 흐름(idle → preparing → recording → finished)을 유지합니다. 사용자는 처음 한 번만 인터페이스를 익히면 3가지 모드를 일관되게 사용할 수 있습니다.

---

## 6. 기술적 성과

### 재사용 가능한 컴포넌트 설계

| 컴포넌트 | 재사용 위치 |
|---------|-----------|
| `PracticeLayout` | ImpromptuPractice, KeywordPractice, CoreUnderstandingPractice |
| `CircleProgress` | AnalysisResult (4개), AnalysisLoading |
| `FeedbackSection` | AnalysisResultDetail (5개 섹션) |
| `MicButton` | 3가지 연습 페이지 공통 |
| `TimerSection` | 3가지 연습 페이지 공통 (`isCompact` prop으로 레이아웃 분기) |
| `CoachBubble` | 3가지 연습 페이지 단계별 메시지 |
| `TitleSection` | 3가지 연습 페이지 헤더 |

### JWT 자동 갱신

```tsx
// fetchClient.tsx — 모든 API 요청에 자동 적용
if (response.status === 401) {
  const reissueRes = await fetch(`${BASE_URL}/auth/reissue`, { ... })
  if (reissueRes.ok) {
    const { newAccess, newRefresh } = await reissueRes.json()
    localStorage.setItem('accessToken', newAccess)
    localStorage.setItem('refreshToken', newRefresh)
    // 원본 요청 재시도
    response = await fetch(url, { ...config, headers })
  }
}
```

모든 API 요청이 단일 `fetchClient`를 통과하므로, 401 처리 로직을 한 곳에서 관리합니다.

### PDF 생성 — 비디오 요소 처리

```tsx
// html2canvas는 video 요소를 캡처하지 못하는 문제 해결
onclone: (clonedDoc) => {
  clonedDoc.querySelectorAll('video').forEach((video) => {
    const canvas = clonedDoc.createElement('canvas')
    canvas.getContext('2d')?.drawImage(video, 0, 0)
    video.parentNode?.replaceChild(canvas, video)
  })
}
```

---

## 7. 트러블 슈팅

### 문제 1: 카운트다운 중 녹화 데이터 포함

**문제 상황**  
5초 카운트다운 동안에도 녹화가 진행되어, 사용자가 화면을 응시하지 않는 초반 5초가 발표 데이터에 포함됨.

**원인 분석**  
`LiveFeedbackTracker`가 마운트와 동시에 `initCamera() → startRecordingFromStream()`을 호출하므로, 부모 컴포넌트에서 `showCountdown` 상태와 관계없이 즉시 녹화가 시작됨.

**해결 과정**  
1. `startRecordingFromStream` 함수를 `useEffect` 클로저 외부(컴포넌트 레벨)로 이동하여 다른 `useEffect`에서도 호출 가능하도록 변경
2. `canRecord: boolean` prop 추가
3. `initCamera`에서 `canRecordRef.current`가 true일 때만 녹화 시작
4. `canRecord` 변화를 감지하는 별도 `useEffect` 추가 — 스트림이 이미 준비된 상태에서 `canRecord`가 true로 바뀌면 즉시 녹화 시작
5. 부모에서 `canRecord={!showCountdown}` 전달

**결과**  
카운트다운 5초가 완전히 지나야 녹화가 시작되어 분석 데이터 정확도 향상.

---

### 문제 2: 홈 화면 닉네임 미표시

**문제 상황**  
홈 화면 사이드바의 `"OOO님, 반가워요!"` 문구에서 닉네임 대신 "사용자"가 표시됨. Nav 컴포넌트에서는 정상 표시됨.

**원인 분석**  
`getPersonalHome()` → `/personal/home` 엔드포인트가 404를 반환했고, catch 블록이 실행되면서 `userName`이 null 상태로 유지됨. `LeftPanel`에서 `userName || '사용자'`로 폴백되어 "사용자"가 표시됨.

**해결 과정**  
Nav가 `/profile/get`에서 닉네임을 정상적으로 가져오는 것을 확인. `userName` fetch를 `/personal/home`과 분리하여 독립적인 `useEffect`로 구현.

```tsx
useEffect(() => {
  const fetchUserName = async () => {
    const data = await api.get('/profile/get')  // Nav와 동일한 엔드포인트
    setUserName(data.userName ?? data.userId ?? null)
  }
  fetchUserName()
}, [])
```

**결과**  
`/personal/home` 엔드포인트 상태와 무관하게 닉네임이 항상 표시됨. 두 API의 장애가 서로 영향을 주지 않는 구조로 개선.

---

### 문제 3: MediaPipe FaceMesh + Pose 동시 실행 시 충돌

**문제 상황**  
FaceMesh와 Pose를 `Promise.all`로 동시에 실행하자 "WASM Module already loaded" 오류와 함께 분석이 중단됨.

**원인 분석**  
MediaPipe FaceMesh와 Pose는 내부적으로 동일한 전역 WASM Module 인스턴스를 공유함. 병렬 실행 시 두 태스크가 같은 Module 초기화를 동시에 시도하여 충돌.

**해결 과정**  
```tsx
// 변경 전 (충돌 발생)
await Promise.all([faceMesh.send({ image }), pose.send({ image })])

// 변경 후 (순차 실행)
await faceMesh.send({ image })
await pose.send({ image })
```

**결과**  
WASM 충돌 없이 두 모델이 안정적으로 동작. 프레임당 약 33ms(30fps) 이내에서 순차 처리 완료.

---

## 8. 프로젝트를 통해 얻은 경험

### 기술적 성장

- **브라우저 미디어 API 심화**: MediaRecorder, Web Audio API, MediaStream의 생명주기를 직접 관리하면서 스트림 누수, 녹화 상태 관리, 오디오 포맷 변환(Float32 → Int16 → Base64)을 경험했습니다.
- **실시간 데이터 처리**: WebSocket 연결 이후에만 오디오 전송을 시작하는 패턴, 1초 누적 후 일괄 전송하는 배치 처리, 컴포넌트 언마운트 시 완전한 클린업을 통해 실시간 데이터 파이프라인 설계 능력을 키웠습니다.
- **SVG 기반 데이터 시각화**: Recharts 같은 외부 라이브러리 없이 순수 SVG와 삼각법으로 레이더 차트를 구현하면서, 좌표 계산과 그라디언트 적용을 직접 다루었습니다.
- **스크롤 기반 애니메이션**: `useLayoutEffect`로 DOM 측정 → 스크롤 progress 계산 → easeOutCubic 보간 → inline style 업데이트 파이프라인을 구현하면서 CSS 애니메이션 없이 60fps 수준의 인터랙션을 만드는 방법을 익혔습니다.

### 협업 경험

- 팀 공통 API 클라이언트(`fetchClient.tsx`)와 공통 컴포넌트(`CircleProgress`, `PracticeLayout`)를 먼저 설계하여 팀원들의 개발 속도를 높였습니다.
- 백엔드 API 스펙이 변경될 때(엔드포인트 404, 응답 필드명 변경) 프론트엔드에서 폴백 전략을 적용하여 서비스 중단 없이 대응했습니다.

### 사용자 관점에서의 개선 경험

- 카운트다운 중 녹화 데이터 포함 문제는 분석 정확도에 직결되는 문제였습니다. 기능이 "동작한다"는 것과 "올바르게 동작한다"는 것의 차이를 인식하고 세부 요구사항까지 챙기는 습관을 기를 수 있었습니다.
- 실시간 피드백 메시지도 카운트다운 종료 전에는 표시하지 않도록 처리하여, 사용자가 화면 세팅 시간에 불필요한 알림을 받지 않도록 개선했습니다.

---

## 9. 핵심 요약

TALKI 프로젝트에서 MediaPipe 컴퓨터 비전·Web Audio API·WebSocket을 연동한 실시간 발표 분석 파이프라인을 설계하고 구현했습니다. 단순한 기능 구현을 넘어, 카운트다운 중 녹화 제외·닉네임 API 장애 격리·WASM 충돌 해결 등 실제 동작 환경에서 발생하는 엣지 케이스를 직접 발견하고 수정했습니다. 랜딩 페이지의 스크롤 기반 카드 애니메이션부터 SVG 삼각형 레이더 차트까지, 외부 라이브러리에 의존하지 않고 브라우저 API와 수학적 계산으로 구현하는 경험을 통해 프론트엔드의 근본 원리에 대한 이해를 높였습니다.
