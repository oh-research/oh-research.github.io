# 원뿔 곡선 (Conic Sections) 3D 시각화 — 구현 계획서

## 1. 프로젝트 개요

원뿔(이중 원뿔)을 평면으로 절단할 때 생성되는 원뿔 곡선(원, 타원, 포물선, 쌍곡선)을 Three.js 기반 3D 시각화로 보여주는 단일 페이지 웹 애플리케이션.

- **파일 구조**: `index.html` 단일 파일 (HTML + CSS + JS 인라인)
- **언어**: 한국어 UI, 영문 병기
- **외부 의존성**: Three.js 0.170.0 (CDN), MathJax 3 (CDN)

---

## 2. 기술 스택

| 구분 | 선택 | 이유 |
|------|------|------|
| 3D 렌더링 | Three.js 0.170.0 (ES Module, import map) | 브라우저 네이티브, 별도 빌드 불필요 |
| 수식 렌더링 | MathJax 3 (tex-svg) | LaTeX 수식을 SVG로 렌더링 |
| 카메라 제어 | OrbitControls (Three.js addon) | 마우스 드래그 회전/줌/팬 |
| 스타일링 | CSS 변수 기반 다크 테마, 인라인 `<style>` | 단일 파일 유지 |

---

## 3. 레이아웃 구조

```
┌──────────────┬────────────────────────────────┐
│  LEFT PANEL  │         3D CANVAS AREA          │
│  (300px)     │                                 │
│              │                                 │
│ - 제목       │     Three.js WebGL Canvas       │
│ - 곡선 유형  │                                 │
│ - 슬라이더   │              ┌──────────┐       │
│ - 수식       │              │ 2D 오버레이│      │
│ - 단계 설명  │              │ (토글)    │       │
│ - 2D 버튼   │              └──────────┘       │
└──────────────┴────────────────────────────────┘
```

### 3.1 왼쪽 패널 (`#panel`)
- **헤더**: 프로젝트 제목 + 부제
- **곡선 유형 배지** (`#conic-badge`): 현재 곡선 유형을 색상 점 + 이름으로 표시
- **슬라이더 섹션**: 절단 각도(θ: 0°~90°), 절단 위치(d: 0~3.0)
- **각도 표시기** (`#angle-indicator`): 반각(α=45°) 대비 θ/α 비율 표시
- **수식 박스** (`#formula-box`): MathJax로 렌더링되는 표준 방정식
- **단계별 설명** (`#step-list`): 원→타원→포물선→쌍곡선 4단계, 현재 유형 하이라이트
- **2D 토글 버튼**: 2D 단면 투영 오버레이 ON/OFF

### 3.2 3D 캔버스 영역 (`#canvas-wrap`)
- Three.js WebGL 캔버스
- 우측 상단 코너 힌트 (조작 안내)
- 우측 하단 2D 오버레이 (280x280px, 토글 시 표시)

### 3.3 반응형 (모바일, ≤768px)
- flex-direction을 column으로 변경
- 캔버스 상단(48vh), 패널 하단(52vh)
- 2D 오버레이 숨김

---

## 4. CSS 디자인 시스템

### 4.1 CSS 변수 (`:root`)
```
--panel-width: 300px
--bg-dark: #0f1117          (배경)
--bg-panel: #1a1d27         (패널 배경)
--bg-card: #22263a          (카드 배경)
--text-primary: #e8eaf6     (주요 텍스트)
--text-secondary: #9fa8da   (보조 텍스트)
--text-muted: #5c6bc0       (흐린 텍스트)
--accent: #7986cb           (강조색)
--border: #2e3454           (테두리)
--color-circle: #2196F3     (원 - 파랑)
--color-ellipse: #4CAF50    (타원 - 초록)
--color-parabola: #FF9800   (포물선 - 주황)
--color-hyperbola: #F44336  (쌍곡선 - 빨강)
--radius: 10px
--transition: 0.25s ease
```

### 4.2 주요 스타일 구현
- 커스텀 range 슬라이더 (둥근 thumb, 어두운 트랙)
- 스크롤바 커스터마이징 (thin, 테두리 색상)
- 카드형 UI 요소 (border-radius, border, backdrop-filter)
- step-item의 active 상태 트랜지션

---

## 5. 3D 씬 구성

### 5.1 상수
```
HALF_ANGLE_DEG = 45          (원뿔 반각)
CONE_HEIGHT = 3.2            (각 원뿔 높이)
CURVE_SEGMENTS = 512         (곡선 분할 수)
```

### 5.2 씬 요소

#### a) 이중 원뿔
- `ConeGeometry(radius, height, 64, 1, open=true)` 2개
- 꼭짓점(apex)이 원점에서 만나도록 배치
  - 상부 원뿔: apex y=0, base y=+CONE_HEIGHT (180° X축 회전 후 y방향 이동)
  - 하부 원뿔: apex y=0, base y=-CONE_HEIGHT (회전 없이 y방향 이동)
- 반투명 Phong 재질(opacity 0.22) + 와이어프레임 오버레이(opacity 0.18)

#### b) 절단 평면
- `PlaneGeometry(8, 8, 8, 8)` + 반투명 파랑 재질
- 엣지 아웃라인 (`EdgesGeometry` + `LineSegments`)
- 그리드 헬퍼 (16분할)
- 위치/회전은 θ, d 파라미터에 따라 갱신

#### c) 교차 곡선
- `TubeGeometry` (두께 0.035)로 렌더링하여 가시성 확보
- `CatmullRomCurve3`로 부드러운 스플라인 생성
- 쌍곡선의 경우 두 번째 가지(lower nappe)를 별도 튜브로 추가

#### d) 조명
- AmbientLight (0x8899cc, 강도 0.6)
- DirectionalLight 키 라이트 (흰색, 강도 1.2, 그림자)
- DirectionalLight 필 라이트 (파랑 계열, 강도 0.4)

#### e) 기타
- AxesHelper (길이 1.5, 반투명)
- OrbitControls (감쇠, 거리 제한 2~18)

---

## 6. 수학적 모델

### 6.1 원뿔 방정식
```
x² + z² = (y · tan(α))²     (α = 45°이므로 tan(α) = 1)
→ x² + z² = y²
```

### 6.2 절단 평면 방정식
```
y · cos(θ) + z · sin(θ) = d
```
- θ: 절단 각도 (0° = 수평, 90° = 수직)
- d: 원점으로부터의 거리 (오프셋)
- 법선 벡터가 y-z 평면에 위치

### 6.3 교차점 파라메트릭 계산
원뿔 위의 점을 매개변수 t로 표현:
```
x = r·cos(t),  z = r·sin(t),  y = r/tan(α)
```
평면에 대입하면:
```
r · [cos(θ)/tan(α) + sin(θ)·sin(t)] = d
→ r = d / [cos(θ)/tan(α) + sin(θ)·sin(t)]
```

### 6.4 곡선 유형 분류 (`classifyType`)
| 조건 | 유형 |
|------|------|
| d < 0.01 | degenerate (점 또는 직선) |
| θ/α < 0.02 | circle (원) |
| θ/α < 1 - ε | ellipse (타원) |
| θ/α ≈ 1 (±ε) | parabola (포물선) |
| θ/α > 1 + ε | hyperbola (쌍곡선) |

여기서 ε = 0.025 (전환 영역의 허용 오차)

### 6.5 Nappe별 계산
- `nappe = +1` (상부): r ≥ 0인 점만 수집
- `nappe = -1` (하부): r ≤ 0인 점만 수집
- |y| > CONE_HEIGHT인 점은 제외 (원뿔 범위 초과)
- 분모 ≈ 0인 점은 건너뜀 (점근선 근처)

### 6.6 곡선 구성
- 인접 점 간 거리 > 1.2이면 세그먼트 분리 (열린 곡선 처리)
- 가장 긴 세그먼트 선택
- 시작점과 끝점이 가까우면 닫힌 곡선으로 처리 (원, 타원)

---

## 7. 절단 평면 변환

Three.js PlaneGeometry의 기본 법선은 (0,0,1).
목표 법선은 (0, cos(θ), sin(θ)).

```
변환: Rx(θ - π/2)
위치: (0, d·cos(θ), d·sin(θ))
```

---

## 8. UI 업데이트 로직

### 8.1 슬라이더 연동
- `slider-angle` → `thetaDeg` (0~90)
- `slider-offset` → `offsetD` (0~3.0)
- input 이벤트마다 `fullUpdate()` 호출

### 8.2 `fullUpdate()` 흐름
```
fullUpdate()
  ├── updatePlaneTransform()    // 절단 평면 위치/회전 갱신
  ├── updateIntersectionCurve() // 교차 곡선 재계산 및 메시 교체
  ├── updateUI()                // 배지, 수식, 단계 하이라이트 갱신
  └── draw2D()                  // 2D 오버레이 갱신 (활성 시)
```

### 8.3 UI 요소 갱신 (`updateUI`)
- 곡선 유형이 변경된 경우에만 DOM 갱신 (최적화)
- 배지 색상/텍스트, 수식 (MathJax re-typeset), step-item active 클래스 토글
- 각 곡선 유형별 수식:
  - 원: `x² + y² = r²`
  - 타원: `x²/a² + y²/b² = 1`
  - 포물선: `y² = 4px`
  - 쌍곡선: `x²/a² - y²/b² = 1`

---

## 9. 2D 단면 투영

- Canvas 2D API로 렌더링 (280x280px 오버레이)
- x-z 평면 투영 (탑뷰)
- 배경 그리드(20px 간격) + 축선
- 3D 교차점을 2D 좌표로 변환 (scale = W/2 / 4.5)
- 쌍곡선의 경우 두 가지 모두 그리기
- 좌측 하단에 곡선 유형 라벨 표시

---

## 10. 애니메이션 루프

```javascript
function animate() {
  requestAnimationFrame(animate);
  controls.update();        // 감쇠 효과 적용
  renderer.render(scene, camera);
}
```

---

## 11. 퇴행 케이스 (Degenerate) 처리

d ≈ 0일 때 평면이 꼭짓점을 지나감:
- θ = 0: 점 (교차 없음)
- θ > 0: 원뿔의 모선(generator) 2개
  - `sin(t₀) = -cos(θ) / (tan(α) · sin(θ))` 풀어서 방향 계산
  - t₀와 π - t₀ 방향으로 직선 2개 생성

---

## 12. 초기화 순서

1. MathJax 설정 (window.MathJax 객체)
2. Three.js import map 선언
3. CSS 로드 (인라인 `<style>`)
4. HTML DOM 구성
5. `<script type="module">` 실행:
   - Three.js + OrbitControls import
   - 상수/상태 선언
   - 렌더러, 씬, 카메라, 컨트롤 초기화
   - 조명 추가
   - 이중 원뿔 생성 및 추가
   - 절단 평면 생성 및 추가
   - 슬라이더 이벤트 리스너 등록
   - 2D 토글 버튼 이벤트 리스너 등록
   - `fullUpdate()` 최초 실행
   - `animate()` 루프 시작
6. window load 이벤트에서 MathJax typeset 실행

---

## 13. 성능 고려사항

- `CURVE_SEGMENTS = 512`: 부드러운 곡선과 성능의 균형
- `TubeGeometry` 최대 600 세그먼트로 제한
- pixelRatio 최대 2로 제한 (`Math.min(devicePixelRatio, 2)`)
- 곡선 유형 변경 시에만 DOM 갱신 (불필요한 MathJax re-typeset 방지)
- 이전 TubeGeometry는 `dispose()` 후 씬에서 제거 (메모리 누수 방지)
- depthWrite: false로 반투명 객체 렌더링 최적화
