# FleetFlag Mock-UI Collection

현대자동차 SDV 전환을 위한 Feature Flag 시스템의 UI Mock-up 모음입니다.

## 📋 목차

1. [BOM 의존성 매트릭스](#1-bom-의존성-매트릭스)
2. [Safety Gatekeeper 대시보드](#2-safety-gatekeeper-대시보드)
3. [CodeBeamer 추적성 관리](#3-codebeamer-추적성-관리)
4. [Kill Switch 제어 센터](#4-kill-switch-제어-센터)
5. [Rollout 전략 에디터](#5-rollout-전략-에디터)
6. [Analytics & Monitoring 대시보드](#6-analytics--monitoring-대시보드)
7. [End-to-End 시스템 시뮬레이터](#7-end-to-end-시스템-시뮬레이터)
8. [시스템 연동 맵](#8-시스템-연동-맵)
9. [Physical Deployment Viewer](#9-physical-deployment-viewer) ⭐ **NEW**

---

## 1. BOM 의존성 매트릭스

**파일명**: `01-bom-dependency-matrix.html`

### 목적
복잡한 하드웨어 부품(BOM) 간 의존성을 시각화하고 관리하여, Feature Flag 배포 시 호환성 검증을 자동화합니다.

### 주요 기능
- **의존성 매트릭스**: Feature와 HW 부품 간 Required/Optional/Exclusive 관계 표시
- **PLM 연동**: 실시간 BOM 데이터 동기화
- **충돌 감지**: 배타적 부품 조합 자동 경고
- **영향도 분석**: Flag 변경 시 영향받는 차량 대수 실시간 계산

### 핵심 화면 요소
- 요약 카드: 총 부품 수, 활성 의존성, 충돌 경고, PLM 연동 상태
- 의존성 테이블: 드래그앤드롭으로 관계 설정
- 충돌 알림: 빨간색 배지로 즉시 표시
- BOM 동기화 버튼: 수동/자동 동기화 지원

### 기술 요구사항
- PLM System API 연동 필요
- 실시간 WebSocket 업데이트 권장

---

## 2. Safety Gatekeeper 대시보드

**파일명**: `02-safety-gatekeeper-dashboard.html`

### 목적
ASIL 등급 기반 안전 규칙을 실시간 모니터링하고, 위험 상황 발생 시 자동 차단(Override) 기능을 제공합니다.

### 주요 기능
- **실시간 차량 컨텍스트 모니터링**: 배터리 전압, DTC 오류, 주행 속도, GPS 위치
- **안전 규칙 엔진**: 조건식 기반 자동 차단 (예: 주행 중 + 배터리 < 10% → 기능 비활성화)
- **ASIL 등급별 처리**: ASIL-D는 이중 검증, ASIL-B는 단일 검증
- **긴급 정지**: Emergency Stop 버튼으로 모든 안전 기능 즉시 차단

### 핵심 화면 요소
- Critical Alerts: 빨간색 깜빡임 효과로 긴급 상황 강조
- 차량 컨텍스트 게이지: 실시간 센서 값 시각화
- Override 이력: 자동 차단 발생 내역 타임라인
- 통계 카드: 오늘 총 체크 횟수, Override 건수, 평균 응답 시간

### 기술 요구사항
- 차량 텔레메트리 데이터 실시간 스트리밍 (WebSocket/SSE)
- <3초 응답시간 보장 (Kill Switch 요구사항)

---

## 3. CodeBeamer 추적성 관리

**파일명**: `03-codebeamer-traceability.html`

### 목적
ISO 26262 및 SOB 규정 준수를 위한 요구사항-Feature-Flag 간 추적성(Traceability)을 관리합니다.

### 주요 기능
- **양방향 추적**: 요구사항 → Feature → Flag → 테스트 케이스
- **CodeBeamer 연동**: 자동 동기화 및 상태 업데이트
- **ASIL 검증**: 각 단계별 ASIL 등급 일관성 검증
- **감사 로그**: 변경 이력 자동 기록

### 핵심 화면 요소
- 추적성 트리: 계층 구조 시각화 (Collapsible Tree)
- 연동 상태: CodeBeamer와의 동기화 상태 실시간 표시
- 요구사항 카드: ID, 제목, ASIL, 상태, 담당자
- 검증 상태: 각 링크의 검증 완료 여부 체크

### 기술 요구사항
- CodeBeamer API 통합 (REST/GraphQL)
- 대용량 트리 구조 렌더링 최적화 필요

---

## 4. Kill Switch 제어 센터

**파일명**: `04-kill-switch-control-center.html`

### 목적
긴급 상황 발생 시 3시간 이내 전 차량에 Feature를 차단하는 Kill Switch 시스템을 제어합니다.

### 주요 기능
- **즉시 발동**: 실시간 WebSocket 푸시로 3시간 이내 전파
- **2단계 인증**: ASIL-D 등급은 SMS/OTP 인증 필수
- **승인 체인**: ASIL 등급별 승인자 자동 할당
- **영향도 예측**: 대상 차량, 전파 시간, 의존성 충돌 사전 분석

### 핵심 화면 요소
- 활성 Kill Switch 카드: 빨간색 배지로 긴급 상황 강조
- 발동 패널: 사유, 범위, 실행 시각, 2FA 입력
- 타임라인: 최근 Kill Switch 이력 시각화
- 통계: 발동 횟수, 평균 전파 시간, 성공률

### 기술 요구사항
- WebSocket 서버 (wss://fleetflag.azure.hyundai.com/ws)
- Redis Pub/Sub로 전 서버 동기화
- Slack Webhook 알림 통합

---

## 5. Rollout 전략 에디터

**파일명**: `05-rollout-strategy-editor.html`

### 목적
단계별 Canary 배포 전략을 설계하고, BOM 의존성 기반 자동 타겟팅을 설정합니다.

### 주요 기능
- **다단계 Canary**: 1% → 5% → 10% → 50% → 100% 점진적 배포
- **자동 롤백 조건**: 오류율, 크래시, 만족도 기준 설정
- **BOM 필터링**: 호환되지 않는 차량 자동 제외
- **타임라인 시뮬레이션**: 배포 일정 예측 및 시각화

### 핵심 화면 요소
- 전략 선택: Canary / Blue-Green / Big Bang 라디오 버튼
- 단계 설정 카드: 타겟 비율, 대기 시간, 타겟팅 규칙, 롤백 조건
- 대상 차량 미리보기: BOM 호환 차량 필터링 결과
- 위험도 평가: 기술적/안전/사용자 영향 게이지

### 기술 요구사항
- 실시간 차량 BOM 데이터 조회 API
- 단계별 배포 스케줄러 (Cron/Kubernetes CronJob)

---

## 6. Analytics & Monitoring 대시보드

**파일명**: `06-analytics-monitoring-dashboard.html`

### 목적
Feature Flag의 성능, A/B 테스트 결과, 지역별 통계를 실시간 모니터링합니다.

### 주요 기능
- **실시간 KPI**: 활성 차량, 총 Flag 수, 평가 횟수, 오류율, A/B 테스트 수
- **성능 차트**: 지연시간(P50/P95/P99), 평가 요청량, 캐시 적중률
- **A/B 테스트 결과**: 통계적 유의성 검증, ROI 예측
- **지역별 분포**: 히트맵 + 지역별 순위 테이블

### 핵심 화면 요소
- KPI 카드: 그라데이션 배경 + 실시간 업데이트
- 지연시간 차트: Chart.js Line Chart (P50/P95/P99)
- A/B 테스트 카드: Control vs Variant 비교 테이블
- 성능 테이블: Top 10 Feature Flag 순위

### 기술 요구사항
- ClickHouse 분석 데이터베이스 연동
- Chart.js / D3.js 차트 라이브러리
- Leaflet.js 지도 + Heatmap Layer 플러그인

---

## 7. End-to-End 시스템 시뮬레이터 ⭐

**파일명**: `07-end-to-end-simulator.html`

### 목적
FleetFlag 시스템의 **전체 동작 흐름을 시각적으로 시뮬레이션**하여, Admin Console에서 Feature Flag 생성부터 차량 배포, 분석 수집까지의 **End-to-End 프로세스**를 실시간으로 확인합니다.

### 주요 기능
- **3가지 시나리오**: Feature 생성/배포, Kill Switch 긴급 발동, A/B 테스트 실행
- **인터랙티브 흐름도**: 7개 컴포넌트 간 데이터 흐름 시각화 (노드 활성화 + 화살표 애니메이션)
- **실시간 로그**: 각 단계의 로그 메시지 스트리밍
- **데이터 페이로드**: JSON 형식으로 실제 데이터 구조 표시
- **성능 메트릭**: 지연시간, 차량 수, 평가 횟수, 캐시 적중률 실시간 업데이트

### 핵심 화면 요소
- 시나리오 선택: 3개 카드형 라디오 버튼
- 시스템 흐름도: SVG 화살표 + 8개 노드 (Admin, API, Flipt, PostgreSQL, Redis, ClickHouse, Vehicle, SDK)
- 실행 단계: 7단계 프로그레스 바 (회색 → 파란색 → 녹색)
- 실시간 로그: 타임스탬프 + 담당자 + 메시지
- 데이터 페이로드: 구문 강조 JSON Viewer
- 성능 메트릭: 4개 KPI 카드

### 시나리오 상세

#### Scenario 1: Feature 생성 및 배포
```
Admin → API Server (검증) → Flipt Engine → PostgreSQL 
→ Redis Pub/Sub → Vehicle Agent (Delta Sync) 
→ SDK (로컬 평가 2.1ms) → ClickHouse (분석)
```
- **소요시간**: 5초
- **영향 차량**: 1,250대
- **평가 횟수**: 4,523회
- **캐시 적중률**: 98%

#### Scenario 2: Kill Switch 긴급 발동
```
Admin (CTO 승인) → API Server (Emergency) → PostgreSQL 
→ Redis (CRITICAL) → WebSocket Push (2.8초) 
→ All Vehicles (12,800대) → Slack 알림
```
- **전파 시간**: 2.8초
- **영향 차량**: 12,847대 (47대 오프라인)
- **전파 완료율**: 99.6%

#### Scenario 3: A/B 테스트 실행
```
Admin → Sticky Session 설정 → 3개 그룹 할당 (33/33/34%) 
→ 14일간 메트릭 수집 → ClickHouse 분석 
→ 통계적 유의성 검증 (p-value < 0.001)
```
- **실험 기간**: 14일
- **총 평가**: 2,847,231회
- **승자**: Variant A (GPT-4o, 인식률 +4.5%p)

### 기술 요구사항
- 순수 JavaScript (프레임워크 없음)
- SVG 애니메이션 (60fps)
- CSS Keyframes (pulse, dash, slideIn)
- 2초 간격 단계 자동 진행

### 활용 사례
- **신규 개발자 온보딩**: 전체 시스템 아키텍처 학습
- **Stakeholder 데모**: CTO, Safety Team에게 시각적 시연
- **문제 해결 훈련**: 각 단계별 데이터 확인 및 디버깅
- **교육 자료**: 비기술 팀에게 시스템 동작 설명

### 확장 가능성
- 새 시나리오 추가 (`scenarios` 객체 확장)
- 타이밍 조정 (`sleep()` 시간 변경)
- 커스텀 메트릭 추가
- 실패 시나리오 시뮬레이션

**상세 문서**: [END_TO_END_SIMULATOR.md](./END_TO_END_SIMULATOR.md)

---

## 8. 시스템 연동 맵

**파일명**: `08-system-integration-map.html`

### 목적
FleetFlag 시스템과 외부 시스템 간의 **연동 관계**를 네트워크 그래프로 시각화하여, 시스템 간 의존성, API 엔드포인트, 데이터 흐름을 한눈에 확인합니다.

### 주요 기능
- **D3.js Force-Directed Graph**: 11개 시스템 노드 + 연결선 시각화
- **실시간 헬스체크**: 각 시스템 상태 (🟢 정상 / 🟡 경고 / 🔴 장애)
- **Detail Panel**: 클릭 시 API 엔드포인트, 연동 스펙, 데이터 샘플 표시
- **의존성 강도**: 선 굵기로 강한/중간/약한 의존성 구분

**상세 문서**: [SYSTEM_INTEGRATION_MAP_PROPOSAL.md](./SYSTEM_INTEGRATION_MAP_PROPOSAL.md)

---

## 9. Physical Deployment Viewer ⭐

**파일명**: `09-physical-deployment-viewer.html`

### 목적
FleetFlag 시스템의 **물리적 배포 구성**을 시각화하여, 온프렘 IDC 랙 구성, 차량 내부 ECU 배치, 네트워크 토폴로지를 입체적으로 확인합니다.

### 주요 기능
- **4개 탭 뷰**:
  1. **Data Center (IDC)**: 3개 랙 (Frontend/Backend/Data) 서버 배치도
  2. **Vehicle Architecture**: 차량 내부 HPC → CCU → ECU 계층 구조
  3. **Network Topology**: IDC ↔ Vehicle 네트워크 흐름도
  4. **3D Rack Viewer**: Three.js 기반 3D 랙 시각화

- **실시간 하드웨어 모니터링**:
  - CPU/RAM/Storage 사용률
  - 네트워크 처리량
  - 전력 소비량
  - 헬스 상태 (🟢 정상 / 🟡 경고 / 🔴 장애)

- **서버 상세 정보**:
  - 하드웨어 스펙 (CPU 모델, 코어 수, RAM, 스토리지)
  - 실시간 메트릭 (CPU 사용률, 디스크 I/O, 네트워크 처리량)
  - 가동 시간 (Uptime)

- **차량 컴포넌트 상세**:
  - HPC/CCU/ADAS/Cluster/TCU ECU 스펙
  - Feature Flag 적용 상태
  - 실시간 메트릭 (CPU, RAM, 캐시 적중률)

- **네트워크 토폴로지**:
  - Azure Front Door → IDC Firewall → DMZ/App/Data Zone
  - 차량 CCU (LTE/5G) → VPN 터널
  - 암호화 프로토콜 (TLS 1.3, VPN IPsec)

- **3D Rack Viewer (Three.js)**:
  - 3개 랙 3D 시각화 (Frontend/Backend/Data)
  - 각 랙당 4대 서버 배치
  - LED 인디케이터 (녹색 점멸)
  - 마우스 드래그 회전, 휠 확대/축소

### 핵심 화면 요소

#### Tab 1: Data Center (IDC)
- **3개 랙 구성**:
  - Rack 1 (Frontend): Load Balancer (2), API Gateway (2)
  - Rack 2 (Backend): FleetFlag API (2), Flipt Engine (2)
  - Rack 3 (Data): PostgreSQL (2), Redis Cluster (1), ClickHouse (1)

- **서버 카드**:
  - 상단: 서버 이름, 헬스 인디케이터
  - 중간: CPU/RAM 배지, 사용률 텍스트
  - 하단: 전력 소비량 바 (색상 그라데이션)

- **서버 상세 패널** (클릭 시):
  - 하드웨어 스펙 (CPU 모델, 코어, RAM, 스토리지, 네트워크, 전력)
  - 실시간 상태 (CPU/RAM 사용률, Disk I/O, 네트워크 처리량, Uptime)

#### Tab 2: Vehicle Architecture
- **차량 다이어그램** (Genesis GV80):
  - HPC (최상단)
  - ↓ Ethernet 100Mbps
  - CCU (중단)
  - ↓ CAN Bus 500kbps
  - ADAS ECU / Cluster ECU / TCU ECU (하단)

- **컴포넌트 상세** (클릭 시):
  - HPC: Tegra Xavier, 32GB RAM, Kubernetes, FleetFlag Agent
  - CCU: i.MX 8 QuadMax, 8GB RAM, LTE/5G Modem, VPN Client
  - ECU: 각 ECU별 프로세서, RAM, 기능, Feature Flag 상태

#### Tab 3: Network Topology
- **네트워크 노드 맵**:
  - Internet Cloud → Azure Front Door → IDC Firewall
  - ↓ DMZ Zone (Load Balancer, API Gateway)
  - ↓ App Zone (FleetFlag API, Flipt)
  - ↓ Data Zone (PostgreSQL, Redis, ClickHouse)
  - ↓ Vehicle CCU (LTE/5G + VPN)

- **연결 라벨**:
  - HTTPS/WSS, VPN IPsec, TLS 1.3, LTE/5G + VPN

- **네트워크 통계**:
  - Total Throughput: 2.3 Gbps
  - Avg Latency: 8.2 ms
  - Active Connections: 125,847
  - Uptime: 99.98%

#### Tab 4: 3D Rack Viewer
- **Three.js 3D Scene**:
  - 3개 랙 (Rack 1/2/3) 3D 박스
  - 각 랙당 4개 서버 (색상: 파란색/보라색/녹색)
  - LED 인디케이터 (녹색 구체, Pulsing 애니메이션)
  - 랙 상단 라벨 (텍스트 스프라이트)

- **컨트롤**:
  - Rotate Left / Rotate Right 버튼
  - Reset View 버튼
  - OrbitControls (마우스 드래그, 휠 확대/축소)

### 기술 스택
- **Three.js 0.160.0**: 3D 렌더링
- **OrbitControls**: 3D 카메라 제어
- **Tailwind CSS**: 스타일링
- **Font Awesome**: 아이콘
- **Vanilla JavaScript**: 인터랙션

### 데이터베이스
- **서버 스펙 DB**: 12대 서버 상세 정보 (CPU, RAM, Storage, Network, Power)
- **차량 컴포넌트 DB**: HPC, CCU, ADAS, Cluster, TCU 스펙
- **네트워크 노드 DB**: Azure, Firewall, DMZ, App, Data Zone, Vehicle CCU

### 활용 사례
- **IDC 용량 계획**: 서버 리소스 사용률 모니터링
- **장애 대응**: 실시간 헬스체크로 장애 서버 즉시 파악
- **차량 아키텍처 교육**: 신규 개발자 온보딩 자료
- **네트워크 최적화**: 병목 구간 식별 및 대역폭 증설 계획
- **Stakeholder 데모**: CTO/인프라팀에게 물리적 구성 시연

### 확장 가능성
- 실시간 메트릭 업데이트 (Prometheus/Grafana 연동)
- 서버 추가/제거 시뮬레이션
- 전력 소비량 예측 (Phase별)
- 차량 대수 증가 시뮬레이션 (10,000 → 100,000대)
- 네트워크 트래픽 애니메이션 (데이터 패킷 흐름)

**관련 문서**: [Physical Deployment Architecture](../02-architecture/physical-deployment.md)

---

## 🎨 디자인 시스템

### 색상 팔레트
- **Primary (Hyundai Navy)**: `#002c5f`
- **Secondary (Hyundai Blue)**: `#00aad2`
- **Success**: `#10b981` (Green-600)
- **Warning**: `#f59e0b` (Orange-500)
- **Danger**: `#dc2626` (Red-600)
- **Info**: `#3b82f6` (Blue-500)

### ASIL 등급별 색상
- **ASIL-QM (비안전)**: `#6b7280` (Gray-500)
- **ASIL-A**: `#3b82f6` (Blue-500)
- **ASIL-B**: `#10b981` (Green-600)
- **ASIL-C**: `#f59e0b` (Orange-500)
- **ASIL-D**: `#8b5cf6` (Purple-600) 또는 `#dc2626` (Red-600)

### 타이포그래피
- **헤더**: Pretendard/Inter, Bold, 24-32px
- **본문**: Pretendard/Inter, Regular, 14-16px
- **코드/VIN**: Mono (Courier New), 12px

### 아이콘
- **Font Awesome 6.4.0**: https://fontawesome.com
- 주요 아이콘: `fa-flag`, `fa-car`, `fa-shield-alt`, `fa-power-off`, `fa-chart-line`

---

## 🛠️ 기술 스택

### 프론트엔드
- **HTML5 + Tailwind CSS 3.x**: CDN 방식 (빠른 프로토타이핑)
- **Chart.js 4.x**: 차트 시각화
- **Font Awesome 6.4.0**: 아이콘
- **Vanilla JavaScript**: 인터랙션 (프레임워크 없음)

### 백엔드 (향후 구현)
- **Hono Framework**: Cloudflare Workers 기반 API
- **PostgreSQL**: 메타데이터 저장
- **Redis**: 실시간 캐시 + Pub/Sub
- **ClickHouse**: 분석 데이터

### 배포
- **Cloudflare Pages**: 정적 호스팅
- **Cloudflare Workers**: Edge API

---

## 📐 인터랙션 패턴

### 1. Kill Switch 발동 흐름
```
사용자 클릭 → Feature 선택 → 사유 입력 (50자+) → 범위 선택 (전체/부분) 
→ 2FA 인증 (ASIL-D) → 승인 체인 (CTO) → 즉시 실행 
→ WebSocket 푸시 → Redis Pub/Sub → 전 차량 전파 (3시간 이내) 
→ Slack 알림 → 감사 로그 기록
```

### 2. Rollout 전략 설정 흐름
```
Feature 선택 → 전략 선택 (Canary) → Stage 추가 (1%, 5%, 50%) 
→ 타겟팅 규칙 (지역/BOM/VIN) → 롤백 조건 (오류율 > 5%) 
→ BOM 호환성 자동 검증 → 타임라인 시뮬레이션 → 승인 → 배포 시작 
→ 단계별 자동 진행 → 실시간 모니터링
```

### 3. A/B 테스트 결과 분석
```
실험 선택 → Control vs Variant 비교 → 통계적 유의성 검증 (p-value < 0.05) 
→ 신뢰도 계산 (95%) → ROI 예측 → 승자 선택 → 전체 배포 또는 롤백
```

---

## 🔗 연동 시스템

| 시스템 | 용도 | 인터페이스 | 주기 |
|--------|------|-----------|------|
| **PLM System** | BOM 데이터 동기화 | REST API | 실시간/주기적 |
| **CodeBeamer** | 요구사항 추적성 | REST/GraphQL | 실시간 |
| **차량 텔레메트리** | 센서 데이터 수집 | WebSocket | 실시간 (1초) |
| **Azure Log Analytics** | 로그 수집 및 분석 | Log Stream | 실시간 |
| **Slack** | 알림 및 협업 | Webhook | 이벤트 기반 |
| **SUMS (SW Update)** | OTA 업데이트 연동 | REST API | 배포 시 |

---

## 📱 반응형 지원

모든 Mock-UI는 반응형 디자인을 적용하여 다양한 화면 크기에서 동작합니다:

- **Desktop (1920x1080)**: 기본 레이아웃
- **Laptop (1366x768)**: 그리드 조정 (3열 → 2열)
- **Tablet (768x1024)**: 사이드바 접기, 카드 1열 배치
- **Mobile (375x667)**: 하단 네비게이션, 모달 전체 화면

---

## 🚀 실행 방법

### 로컬에서 실행
```bash
# 1. 파일 다운로드
git clone <repository_url>
cd fleetflag-specification/07-mock-ui

# 2. 웹 서버 실행 (Python 3.x)
python3 -m http.server 8000

# 3. 브라우저에서 접속
# http://localhost:8000/01-bom-dependency-matrix.html
# http://localhost:8000/02-safety-gatekeeper-dashboard.html
# http://localhost:8000/03-codebeamer-traceability.html
# http://localhost:8000/04-kill-switch-control-center.html
# http://localhost:8000/05-rollout-strategy-editor.html
# http://localhost:8000/06-analytics-monitoring-dashboard.html
```

### Cloudflare Pages 배포 (향후)
```bash
# wrangler 설치
npm install -g wrangler

# Cloudflare 로그인
wrangler login

# 배포
wrangler pages publish . --project-name fleetflag-ui
```

---

## 📝 주의사항

### 보안
- **API 키/토큰**: 절대 하드코딩 금지, 환경 변수 사용
- **2FA 인증**: ASIL-D 등급 필수 적용
- **감사 로그**: 모든 중요 작업 기록

### 성능
- **평가 지연**: P95 < 50ms 목표
- **Kill Switch 전파**: 3시간 이내 100% 완료
- **차트 렌더링**: 1만 개 이상 데이터 포인트 시 가상화 필요

### 접근성
- **키보드 네비게이션**: Tab/Enter 지원
- **스크린 리더**: ARIA 라벨 추가
- **WCAG 2.1 AA 준수**: 명도 대비 4.5:1 이상

---

## 📄 라이선스

본 Mock-UI는 현대자동차 SDV 전환 프로젝트의 일부이며, 내부 사용 목적으로 제작되었습니다.

---

## 👥 제작

- **프로젝트**: 현대자동차 Feature Flag 시스템 (FleetFlag)
- **작성자**: FleetFlag 개발팀
- **작성일**: 2026-02-01
- **버전**: v1.0.0

---

## 📞 문의

- **기술 문의**: fleetflag-dev@hyundai.com
- **보안 문의**: security@hyundai.com
- **긴급 상황**: Safety Manager (이성룡 부장)
