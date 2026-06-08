# TALKI : AI 기반 실시간 발표 분석·피드백 서비스

## 프로젝트 개요

TALKI는 발표·면접에 대한 사회불안을 가진 사용자를 위한 AI 기반 발표 트레이닝 웹 서비스다. 인지행동치료(CBT) 원리를 바탕으로, 사용자가 실전 발표 환경을 반복 체험하며 불안을 완화할 수 있도록 설계했다.

카메라·마이크로 수집한 영상을 서버에 업로드하면 AI가 시선 집중도·자세 안정성·발화 속도·주제 적절성·추임새 등을 자동 분석하고, 점수와 LLM 기반 텍스트 피드백을 제공한다. 실시간 발표 중에는 WebSocket으로 분석 신호를 수신해 화면에 즉시 피드백을 표시하는 기능도 포함되어 있다.

팀원 4인이 함께 개발 중인 서비스다.

---

## 기술 스택

| 분류 | 사용 기술 |
|------|-----------|
| 프레임워크 | React 19, TypeScript 5.9, Vite 7 |
| 라우팅 | React Router DOM 7 |
| 스타일링 | Tailwind CSS 3 |
| AI·미디어 처리 | MediaPipe FaceMesh, MediaPipe Pose, MediaPipe Camera Utils, Web MediaRecorder API |
| 실시간 통신 | WebSocket (브라우저 내장) |
| 시각화 | Recharts, 커스텀 SVG 삼각형 차트, 커스텀 CircleProgress |
| PDF 내보내기 | html2canvas + jsPDF |
| HTTP 통신 | 커스텀 fetch 클라이언트 (JWT 자동 갱신 포함) |
| 영상 저장 | AWS S3 Pre-signed URL |

---

## 전체 기능 요약

- **랜딩 페이지**: 서비스 소개, FAQ, CTA. 스크롤 위치에 따라 내비게이션 바 전환
- **홈 페이지**: 연속 연습 스트릭, 주간 달성 현황, 이전 마음가짐 기록, 최근 실전 리포트 요약
- **실전 발표 페이지**: 카메라+마이크 활성화 → MediaPipe로 실시간 얼굴·자세 추적 → WebSocket으로 서버 전송 → 발표 종료 시 S3에 영상 업로드
- **분석 결과 페이지**: 총점·항목별 점수·LLM 피드백 표시. 프리미엄 회원은 세부 분석(삼각형 차트, 타임라인, 반복어 분석)까지 열람 가능. PDF 다운로드 지원
- **연습 페이지**: 즉흥 말하기 / 키워드 기반 구성 / 핵심 파악 연습. 준비 카운트다운 → 녹음 → 피드백의 3단계 플로우
- **가이드라인 / 카테고리 / 튜토리얼**: 실전 진입 전 환경 체크 및 발표 유형 선택

---

## 내가 담당한 부분

### 1. 메인 랜딩 페이지 (`/`)

서비스의 첫인상을 결정하는 페이지로, HeroSection · WorrySection · PracticeSection · RequirementsSection · FaqSection · CtaSection · Footer 총 7개 섹션을 구성하고 조합했다.

스크롤 위치에 따라 내비게이션 바를 동적으로 전환하는 방식을 구현할 때, 화면 밖에 보이지 않는 `div`(sentinel)를 페이지에 배치하고 `getBoundingClientRect().top`으로 그 위치를 감지해 메인 내비게이션과 일반 내비게이션 사이를 자연스럽게 전환했다. 전체 페이지는 CSS `snap-mandatory`로 섹션 단위 스냅 스크롤을 적용해 UX의 흐름감을 높였다.

HeroSection에서는 SVG 웨이브 애니메이션(중첩 `feTurbulence` 필터와 translate 루프)과 일러스트 부유 효과, 그리고 호버 시 빛이 화살표를 따라 흐르는 CTA 버튼 애니메이션을 직접 SVG와 Tailwind keyframe으로 구현했다.

### 2. 메인 홈 페이지 (`/home`)

로그인한 사용자가 서비스에 진입한 뒤 가장 먼저 보는 대시보드 성격의 화면이다. 왼쪽 패널과 오른쪽 패널로 나뉘며, 오른쪽은 캘린더 뷰와 대시보드 뷰를 토글로 전환할 수 있다.

데이터 흐름을 설계할 때 닉네임 요청과 연습 이력 요청을 의도적으로 분리했다. 닉네임은 `/profile/get` 엔드포인트로, 연습 이력·마음가짐·최근 리포트는 `/personal/home` 엔드포인트로 각각 독립적인 `useEffect`에서 가져온다. 두 요청이 완전히 분리되어 있어 한쪽이 실패해도 나머지 정보는 정상 표시된다.

왼쪽 패널(`LeftPanel`)에서는 이번 주 요일별 달성 현황을 원형 인디케이터로 시각화하고, 연속 연습 스트릭·이전 마음가짐 기록·최근 실전 리포트를 카드 형태로 배치했다. 모바일 환경에서는 실전·연습 기능이 PC에서만 동작함을 안내하는 별도 카드가 표시된다.

### 3. 실시간 실전 발표 페이지 (`/live`)

서비스의 핵심 기능을 담당하는 페이지다. 구조는 크게 두 컴포넌트로 나뉜다.

`LiveFeedbackTracker`는 화면에는 보이지 않는 숨겨진 `<video>` 요소 위에서 동작한다. 카메라 스트림을 열고, MediaPipe FaceMesh(홍채 포함 468개 랜드마크)와 MediaPipe Pose를 순차 실행해 시선·자세 데이터를 추출한다. 1초 단위로 누적한 오디오를 Float32 → Int16 PCM으로 변환하고 Base64로 인코딩해 WebSocket을 통해 백엔드로 전송한다. 서버는 `session_start` 이벤트로 발표 세션 ID를 응답하고, 이후 실시간 피드백 텍스트를 `feedback` 이벤트로 내려준다.

`LiveFeedback`은 배경 발표 영상, 음성 파형 인디케이터, 실시간 피드백 메시지, 실시간 피드백·돌발 상황 토글, 종료 버튼을 포함한 UI 레이어다. 종료 버튼을 누르면 `MediaRecorder`가 기록한 Blob을 S3 Pre-signed URL을 통해 직접 업로드하고, 완료 후 분석 로딩 페이지로 이동한다.

`canRecord` prop으로 카운트다운 종료 전에는 `MediaRecorder.start()`가 호출되지 않도록 녹화 시작 시점을 제어한다(트러블슈팅 섹션 참고).

### 4. 실시간 분석 세부 결과 페이지 (`/result`)

발표가 끝난 뒤 AI 분석 결과를 보여주는 페이지다. 분석에 시간이 걸리기 때문에 결과가 준비될 때까지 10초 간격으로 폴링하고, 응답을 받으면 폴링을 중단한다.

기본 결과(`AnalysisResult`)는 총점, 항목별 원형 프로그레스 바(발화 속도·시선 집중도·주제 적절성·제스처 안정성), 추임새 점수 바, LLM이 생성한 텍스트 피드백(장점·성장 포인트·연습 추천)으로 구성된다.

세부 결과(`AnalysisResultDetail`)는 프리미엄 회원에게만 제공된다. SVG로 직접 그린 삼각형 레이더 차트(시선 집중도·주제 적절성·제스처 안정성을 꼭지점으로), 타임라인 기반 발표 구간 분석, 반복어 분석, 실제 발표 영상 플레이어(S3 다운로드 URL 사용)가 포함된다. BASIC 회원이 세부 결과 버튼을 누르면 무료 체험 모달이 열리고, 동의 시 `/profile/type/update?type=PREMIUM`을 호출해 유저 타입을 업그레이드한다.

세부 분석 섹션들은 IntersectionObserver로 뷰포트 진입 시점을 감지해 애니메이션을 순차적으로 실행한다. PDF 다운로드는 html2canvas와 jsPDF를 조합해 기본 분석과 전체 분석 두 가지 버전을 지원한다.

### 5. 연습 페이지 3종 (`/practice/impromptu`, `/practice/keyword`, `/practice/core`)

세 연습 페이지는 공통 플로우를 공유한다: `idle → preparing(10초 카운트다운) → recording(30초) → finished`의 4단계 상태 머신으로 진행된다. `PracticeStep` 타입으로 현재 단계를 관리하고, `useEffect`에서 타이머를 구동해 단계를 자동으로 전환한다. 마이크 권한은 연습 시작 시점에 `getUserMedia`로 요청하고, `MediaRecorder`로 오디오를 캡처한다.

각 연습의 특성은 다음과 같다.

**즉흥 말하기 연습**: 3개 질문 중 하나를 무작위로 선택해 제시한다. 10초 준비 후 30초 동안 자유롭게 말하면 되며, 완료 후 코치 버블에 간단 피드백이 표시된다.

**키워드 기반 구성 연습**: 제시된 3개 키워드(협업·문제해결·성장)를 모두 포함해 말하는 연습이다. 완료 후 사용된 키워드 수와 평가 결과를 `KeywordAnalysis` 컴포넌트로 시각화한다. `KeywordCards`에서 사용된 키워드는 상태가 변경되어 시각적으로 구분된다.

**핵심 파악 연습**: 긴 지문을 읽고 핵심을 요약해 말하는 연습이다. `CoreTextCard` 컴포넌트에 지문과 핵심 키워드를 함께 표시하며, 연습 완료 후 파악된 키워드 수를 분석해 보여준다.

세 페이지 모두 `PracticeLayout` 공통 래퍼와 `CoachBubble`, `MicButton`, `TimerSection` 등 재사용 컴포넌트를 공유하도록 구성해 코드 중복을 줄이고 일관된 UX를 제공했다.

---

## 문제 해결 경험

### 카운트다운 5초가 발표 데이터에 포함되는 문제

실전 발표 페이지에 처음 진입하면 "5초 후에 시작됩니다"라는 카운트다운이 표시된다. 사용자가 이 시간 동안 자세를 잡고 환경을 점검하는데, 초기 구현에서는 카운트다운이 화면에 뜨는 순간 이미 카메라·마이크·MediaRecorder가 모두 활성화되어 데이터 수집이 시작되고 있었다. 그 결과 사용자가 카운트다운 중 자세를 고치거나 움직이는 장면이 발표 영상에 포함되어 분석 결과의 정확도를 떨어뜨렸다.

원인을 정리하면, 카메라 준비(스트림 열기, MediaPipe 초기화)와 녹화 시작(`MediaRecorder.start()`)이 하나의 흐름에서 동시에 이루어지고 있었기 때문이다.

해결 방법으로 두 시점을 분리했다. `LiveFeedbackTracker`에 `canRecord` prop을 추가하고, 카운트다운이 진행 중인 동안(`showCountdown === true`)에는 `canRecord={false}`로 설정했다. 카메라 스트림과 MediaPipe는 페이지 진입 시 정상적으로 초기화되지만, `canRecordRef.current`가 `false`인 동안에는 `startRecordingFromStream()`이 호출되지 않는다. `canRecord`가 `true`로 바뀌는 순간, 즉 카운트다운이 완전히 끝나는 `onFinish` 콜백이 호출된 직후에야 `MediaRecorder`가 시작된다.

```tsx
// LiveFeedbackTracker.tsx
useEffect(() => {
  canRecordRef.current = canRecord
  if (canRecord && streamRef.current && !mediaRecorderRef.current) {
    startRecordingFromStream(streamRef.current)
  }
}, [canRecord])
```

```tsx
// LiveFeedback.tsx
<LiveFeedbackTracker
  canRecord={!showCountdown}   // 카운트다운이 끝나야 true로 전환
  ...
/>
{showCountdown && <CountdownOverlay onFinish={() => setShowCountdown(false)} />}
```

이 수정으로 카메라는 미리 준비를 완료하고 대기하다가, 카운트다운이 종료된 이후의 데이터만 분석에 반영되도록 개선했다.

---

### 홈 화면 닉네임 미표시 문제

홈 화면의 "OOO님, 반가워요!" 인사말에 닉네임이 표시되지 않는 버그를 발견했다. 네비게이션 바에서는 같은 계정의 닉네임이 정상적으로 보였으므로, 계정 데이터 자체의 문제가 아님을 먼저 확인했다.

원인을 추적해보니, 홈 화면이 닉네임을 가져오던 `/personal/home` 엔드포인트가 오류를 반환하고 있었다. 문제는 구조에 있었다. 당시 홈 화면은 닉네임, 연습 이력, 마음가짐, 최근 리포트를 모두 단일 API 호출 하나로 묶어서 가져오고 있었다. 이 요청이 실패하자 닉네임은 물론 연습 이력 등 나머지 정보도 함께 표시되지 않았다.

해결 방법으로 닉네임 요청과 홈 데이터 요청을 완전히 분리했다. 닉네임은 내비게이션 바(`Nav`)와 동일한 `/profile/get` 엔드포인트를 사용하는 독립적인 `useEffect`로 가져오도록 수정했다. 홈 데이터 요청이 실패해도 닉네임은 정상 표시되고, 반대로 닉네임 요청이 실패해도 홈 데이터는 영향받지 않는다.

```tsx
// Home.tsx — 닉네임 요청을 별도 useEffect로 분리
useEffect(() => {
  const fetchUserName = async () => {
    try {
      const data = await api.get('/profile/get')     // Nav와 동일한 엔드포인트
      setUserName(data.userName ?? data.userId ?? null)
    } catch (err) {
      console.error('Failed to fetch profile:', err)
    }
  }
  fetchUserName()
}, [])

useEffect(() => {
  const fetchHomeData = async () => {
    try {
      const data = await getPersonalHome()           // 연습 이력·리포트는 별도 요청
      setHomeData(data)
      // streak, practicedDates 세팅...
    } catch (err) {
      console.error('Failed to fetch home data:', err)
    }
  }
  fetchHomeData()
}, [])
```

요청을 분리하는 것만으로 장애 격리(fault isolation)가 자연스럽게 이루어졌고, 하나의 API 오류가 무관한 UI 영역까지 망가뜨리는 문제를 구조적으로 해결했다.

---

## 배운 점과 성과

브라우저 미디어 API를 직접 다뤄본 경험이 이 프로젝트의 가장 큰 수확이다. `getUserMedia`, `MediaRecorder`, `AudioContext`, `ScriptProcessorNode`를 순서대로 연결하며 스트림을 제어하는 방법, 그리고 카메라 준비 시점과 실제 녹화 시작 시점을 명확하게 분리해야 한다는 점을 실제 버그를 통해 배웠다. MediaPipe 두 모델을 Promise.all이 아닌 순차 실행으로 처리해야 WASM 전역 Module 충돌이 생기지 않는다는 점도 이 프로젝트에서 직접 확인했다.

API 설계 관점에서는 여러 관심사를 하나의 요청에 묶는 것이 성능보다 장애 격리 측면에서 오히려 불리할 수 있다는 교훈을 얻었다. 닉네임 버그는 단순한 API 분리 하나로 해결됐지만, 그 과정에서 "독립적인 UI 영역은 독립적인 데이터 요청"이라는 설계 원칙을 체감했다.

프리미엄 / 베이직 회원 구분, PDF 내보내기, 폴링 기반 결과 조회 등 실제 서비스에서 발생하는 엣지 케이스를 코드 수준에서 처리해보며, 기능 완성도를 높이는 것과 에러 케이스를 대비하는 것이 동시에 이루어져야 한다는 것을 실감했다.
