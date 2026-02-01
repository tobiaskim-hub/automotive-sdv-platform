# FleetFlag Mock-UI Collection

현대자동차 SDV 전환을 위한 Feature Flag 시스템의 UI Mock-up 모음입니다.

## 📋 목차

1. [BOM 의존성 매트릭스](#1-bom-의존성-매트릭스)
2. [Safety Gatekeeper 대시보드](#2-safety-gatekeeper-대시보드)
3. [CodeBeamer 추적성 관리](#3-codebeamer-추적성-관리)
4. [Kill Switch 제어 센터](#4-kill-switch-제어-센터)
5. [Rollout 전략 에디터](#5-rollout-전략-에디터)
6. [Analytics & Monitoring 대시보드](#6-analytics--monitoring-대시보드)

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
