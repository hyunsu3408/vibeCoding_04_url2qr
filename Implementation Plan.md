# 구현 계획서 (Implementation Plan)

## 1. 개요 및 프로젝트 목표
본 프로젝트는 사용자가 웹사이트 URL을 입력하면 실시간으로 고화질 QR코드를 생성하고, 클릭 한 번으로 `.jpg` 형식의 이미지로 다운로드할 수 있는 웹 기반 도구입니다.

* **브랜드 일체화**: 상단 로고(`img/logo.png`)의 고유 색상인 **골든 앰버 옐로우(`#FFC000`) & 딥 블랙**을 웹사이트 전체의 톤앤매너로 계승.
* **인터랙티브 시각 경험**: 정적인 배경 대신 **마우스 커서의 위치에 반응하여 유기적으로 색조가 변하는 동적 오로라 그라데이션** 적용.
* **직관적인 UX 전환**: 초기 화면 중앙의 미니멀한 검색창 형태에서 확인 버튼 클릭 시 자연스럽게 하단 플로팅 독으로 이동하고 정중앙에 QR코드가 팝업되는 시각적 상태 전이.

---

## 2. 기술 스택 및 선정 이유

| 기술 영역 | 사용 기술 | 선정 이유 및 역할 |
| :--- | :--- | :--- |
| **마크업** | HTML5 (Semantic) | 가볍고 빠른 로딩, 검색/접근성 친화적 구조, 로컬 파일 직접 실행 지원 |
| **스타일링** | CSS3 (Custom Properties) | CSS 변수를 통한 JavaScript 실시간 색상 연동, 글래스모피즘, 60fps 하드웨어 가속 전환 |
| **로직** | Vanilla JavaScript (ES6+) | 외부 프레임워크 종속성 제로, 브라우저 단독 실행 호환, 부드러운 물리 관성 보간(Lerp) 연산 |
| **QR 엔진** | QRCode.js (Local Packaging) | 외부 CDN 차단 환경에서도 100% 동작하는 로컬 번들링, 최고 복원율(Level H) 지원 |
| **폰트** | Pretendard | 현대적이고 가독성이 뛰어난 시스템 최적화 웹폰트 적용 |

---

## 3. 디자인 시스템 (Design System)

### 3.1 컬러 팔레트
* **Brand Primary**: `#FFC000` (Warm Sun Gold / Amber)
* **Primary Gradient**: `linear-gradient(135deg, #ffc820 0%, #ff9900 100%)`
* **Base Canvas**: `#FFFDF9` (Clean Warm Sunlight Cream)
* **Surface**: `rgba(255, 255, 255, 0.92)` + `backdrop-filter: blur(20px)`
* **Typography & Dark Point**: `#0F172A` (Deep Slate Black)
* **Accent Badge**: `#FEF3C7` (Soft Honey Yellow) / 텍스트: `#92400E`

### 3.2 인터랙티브 그라데이션 수학적 모델
마우스의 위치 $P(x, y)$를 화면 기준 정규화 좌표 $(-1.0 \le \text{normX}, \text{normY} \le 1.0)$로 변환하여 로고 색조를 중심으로 실시간 변조합니다:

$$\text{Hue}_1 = 44 + (12 \times \text{normX}) + (5 \times \text{normY}) \quad (32^\circ \sim 56^\circ, \text{Warm Peach to Golden Yellow})$$
$$\text{Hue}_2 = 28 + (12 \times \text{normY}) - (8 \times \text{normX}) \quad (16^\circ \sim 44^\circ, \text{Honey Coral to Amber})$$
$$\text{Hue}_3 = 50 - (10 \times \text{normX}) \quad (40^\circ \sim 60^\circ, \text{Amber to Sunny Lemon})$$

커서의 물리적 추적에는 지연 계수 $\alpha = 0.08$의 선형 보간(Lerp) 알고리즘을 적용하여 끊김 없는 60fps 부드러운 움직임을 보장합니다:
$$\text{Current} \leftarrow \text{Current} + (\text{Target} - \text{Current}) \times 0.08$$

---

## 4. 핵심 기능 구현 설계

### 4.1 상태 전이 (State Transitions)
* **초기 상태 (`body:not(.state-generated)`)**:
  - 안내 문구 및 URL 입력창이 뷰포트의 정중앙에 위치.
  - QR코드 결과 영역은 비활성화(`display: none; opacity: 0;`).
* **생성 완료 상태 (`body.state-generated`)**:
  - 안내 문구가 부드럽게 축소되며 숨김 처리.
  - URL 입력창이 화면 하단 2rem 지점으로 미끄러져 내려가 플로팅 독으로 고정.
  - 정중앙에 고해상도 QR코드 카드가 스케일 업 애니메이션과 함께 등장.

### 4.2 스마트 URL 보정 파이프라인
* 입력값이 도메인 형태(`naver.com`, `google.com` 등)인 경우 자동으로 `https://`를 접두사로 추가.
* 모바일 카메라가 텍스트가 아닌 웹사이트 링크로 즉시 인식하도록 최적화.

### 4.3 캔버스 기반 JPG 다운로드 아키텍처
* **문제점 해결**: 일반 Canvas to JPEG 변환 시 투명한 영역이 기본적으로 검은색(Black)으로 채워지는 현상 방지.
* **해결 파이프라인**:
  1. 가상 캔버스(`exportCanvas`) 생성.
  2. 배경 전체를 순수 화이트(`#FFFFFF`)로 채색.
  3. 사방 40px의 안전 여백(Quiet Zone) 확보.
  4. 원본 QR 모듈을 중앙에 렌더링.
  5. `canvas.toDataURL('image/jpeg', 0.95)`를 통해 고압축 선명한 `.jpg` 파일 생성 및 자동 다운로드.
