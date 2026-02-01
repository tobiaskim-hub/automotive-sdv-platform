# FleetFlag 개발 가이드 종합 문서

> **현대자동차 SDV 전환을 위한 Feature Flag System 구축 가이드**  
> 작성일: 2026-02-01 | 버전: v1.0.0 | 상태: 🟢 Complete

---

## 📋 Executive Summary

본 문서는 현대자동차의 **Software-Defined Vehicle (SDV)** 전환을 위한 **Feature Flag Management System (FleetFlag)** 구축을 위한 종합 개발 가이드입니다.

### 핵심 목표
1. **온프렘(On-Premise) 보안 규정 준수**: 현대차 내부망 기반 배포
2. **BOM 의존성 관리**: 하드웨어 부품 간 복잡한 의존성 자동 검증
3. **ASIL 보장**: ISO 26262 안전 표준 준수
4. **전사적 제어 타워**: 통합 거버넌스 및 Kill Switch 긴급 대응

### 시스템 비전
단순한 온/오프 스위치가 아닌, **현대차의 SDV 생태계 전반을 제어하는 엔터프라이즈급 플랫폼**으로 구축합니다.

---

## 🎯 프로젝트 범위

### In-Scope (포함 사항)
- ✅ Feature Flag 생성/관리/배포 (CRUD)
- ✅ 다차원 타겟팅 (VIN, 지역, BOM, SW 버전)
- ✅ BOM 의존성 매트릭스 관리 및 자동 검증
- ✅ Safety Gatekeeper (실시간 차량 컨텍스트 기반 Override)
- ✅ Kill Switch (3시간 이내 긴급 차단)
- ✅ 단계별 Rollout (Canary/Blue-Green 배포)
- ✅ A/B 실험 및 통계 분석
- ✅ CodeBeamer 연동 (요구사항 추적성)
- ✅ 오프라인 모드 (7일 로컬 캐시)
- ✅ Admin Dashboard (웹 기반 UI)

### Out-of-Scope (제외 사항)
- ❌ OTA 업데이트 자체 구현 (SUMS 시스템 연동만)
- ❌ 차량 제어 명령 직접 전송 (Flag 평가 결과만 제공)
- ❌ 실시간 비디오/음성 스트리밍
- ❌ AI 모델 학습/추론 (AI 플랫폼과 별도)

---

## 🏗️ 시스템 아키텍처

### Physical Architecture (물리적 배치)

```
┌─────────────────────────────────────────────────────────────┐
│  Control Plane (현대차 연구소 내부망 IDC)                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ Admin Console│  │  FleetFlag   │  │  Flipt       │       │
│  │  (React UI)  │◄─┤  API Server  │◄─┤  Engine      │       │
│  │              │  │  (Rust/Axum) │  │  (Go)        │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│         │                 │                   │              │
│         └─────────────────┼───────────────────┘              │
│                           ↓                                  │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  PostgreSQL (Primary + Replica) + Redis Cluster         ││
│  │  ClickHouse (Analytics)                                 ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
                           │
                     VPN / TLS 암호화
                           │
                           ↓
┌─────────────────────────────────────────────────────────────┐
│  Vehicle Plane (차량 내 ECU/HPC)                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ Feature Flag │  │  Local Cache │  │  Safety      │       │
│  │  SDK         │◄─┤  (SQLite)    │◄─┤  Gatekeeper  │       │
│  │  (C++/Rust)  │  │              │  │  (Rust)      │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│         │                                     │              │
│         └────────── Application Code ─────────┘              │
└─────────────────────────────────────────────────────────────┘
```

### Logical Architecture (논리적 흐름)

```
Admin Console → Policy Engine → Audit Log → Integration Gateway
                      ↓
                 Sync Agent
                      ↓
            Feature Flag SDK (Vehicle)
                      ↓
            Local Cache + Safety Gatekeeper
                      ↓
              Application Code
```

### 핵심 구성요소

#### 1. Control Plane (서버 사이드)
- **위치**: 현대차 연구소 내부망 IDC
- **역할**: 정책 관리, 승인, 배포 패키징
- **구성요소**:
  - **Admin Console (React UI)**: Feature Flag 관리 웹 대시보드
  - **FleetFlag API Server (Rust/Axum)**: RESTful API 제공
  - **Flipt Engine (Go)**: OpenFeature 표준 평가 엔진
  - **Policy Engine**: 승인 워크플로우 및 ASIL 검증
  - **Audit Log**: 모든 변경 이력 기록
  - **Integration Gateway**: 외부 시스템 연동 (CodeBeamer, PLM, SUMS)
- **데이터 저장소**:
  - PostgreSQL (메타데이터, Primary+Replica)
  - Redis Cluster (실시간 캐시, Pub/Sub)
  - ClickHouse (분석 데이터)
- **배포 환경**: Kubernetes (Azure AKS)

#### 2. Vehicle Plane (차량 사이드)
- **위치**: 차량 내 ECU/HPC
- **역할**: 런타임 Flag 평가 및 안전 제어
- **구성요소**:
  - **Feature Flag SDK**: 애플리케이션 코드에 임베디드 (C++/Python/Rust)
  - **Evaluator**: 로컬에서 Flag 평가 (ms 단위 응답)
  - **Local Cache (SQLite)**: 최근 7일 정책 저장 (오프라인 모드)
  - **Safety Gatekeeper**: 실시간 차량 컨텍스트 확인 후 Override
  - **Sync Agent**: 서버와 주기적 동기화 (Delta Sync)
- **통신**: gRPC/WebSocket (VPN/TLS 암호화)
- **메모리**: ≤ 64MB
- **CPU**: ≤ 2%

### 데이터 플로우

#### 1. Flag 생성 및 배포
```
Admin → API Server → Validation → DB Insert → Flipt Sync 
→ Audit Log → Cache Invalidate → Vehicle Sync (Delta)
```

#### 2. 실시간 평가 (Fast Path)
```
Application Code → SDK.evaluate() → Local Cache Lookup 
→ Rules Evaluation → Safety Gatekeeper Check → Return Boolean
```

#### 3. Kill Switch 발동
```
Admin (Emergency) → API POST /kill-switch → DB Update 
→ Redis Pub/Sub → WebSocket Push → All Vehicles (< 3시간) 
→ Slack Notification → Audit Log
```

---

## 📊 시스템 요구사항

### 기능 요구사항 (Functional Requirements)

#### FR-001: Feature Flag 생성 및 관리
- **설명**: Feature Flag의 전체 생명주기 관리 (CRUD)
- **세부 기능**:
  - Flag Key (고유 ID), Display Name, Description, Type (Boolean/String/Number/JSON)
  - ASIL 등급 설정 (QM/A/B/C/D)
  - Default Value 설정
  - 생성자/수정자 추적
  - 태그 및 카테고리 분류

#### FR-002: 다차원 타겟팅 규칙
- **설명**: 복합 조건 기반 Feature 제어
- **타겟팅 차원**:
  - **VIN**: 화이트리스트/블랙리스트
  - **지역/법규**: GPS 좌표, 국가 코드
  - **HW BOM**: 부품 번호, 버전
  - **SW 버전**: Semantic Versioning (≥, ≤, ==)
  - **차량 모델**: GN7, G90, IONIQ 등
  - **Percentage Rollout**: 0-100% 단계별 배포

#### FR-003: BOM 의존성 매트릭스 관리
- **설명**: Feature와 HW 부품 간 의존 관계 정의 및 검증
- **의존성 타입**:
  - **Required**: 필수 부품 (없으면 Feature 비활성화)
  - **Optional**: 선택 부품 (있으면 추가 기능)
  - **Exclusive**: 배타적 부품 (동시 존재 불가)
- **자동 검증**: 배포 전 모든 대상 차량의 BOM 자동 확인
- **충돌 경고**: Exclusive 관계 위반 시 빨간색 경고

#### FR-004: Safety Gatekeeper
- **설명**: 서버 On 명령에도 불구하고 차량 내부 SDK가 실시간 안전 체크
- **체크 항목**:
  - 배터리 전압 (< 10% → 전력 소모 기능 차단)
  - DTC 오류 코드 (센서 고장 → 해당 기능 차단)
  - 주행 속도 (> 80km/h → 운전자 방해 기능 차단)
  - GPS 위치 (터널/지하 → 통신 필요 기능 대기)
- **Override 로직**: 조건 미충족 시 로컬에서 강제 비활성화
- **Audit**: Override 발생 시 텔레메트리로 서버에 보고

#### FR-005: Kill Switch
- **설명**: 긴급 상황 시 3시간 이내 전 차량 기능 비활성화
- **프로세스**:
  1. Admin이 Kill Switch 발동 (사유 입력 필수)
  2. DB 업데이트 + Redis Pub/Sub 발행
  3. WebSocket으로 모든 차량에 푸시 (우선순위 최상)
  4. 차량 즉시 반영 (로컬 캐시 무시)
  5. Slack 알림 (#safety-critical 채널)
  6. Audit Log 기록
- **전파 시간**: 3시간 이내 100% (목표: 2h 41m)
- **승인**: ASIL-D 등급은 CTO 승인 필수

#### FR-006: 단계별 Rollout 전략
- **Canary 배포**: 1% → 5% → 10% → 50% → 100%
- **Blue-Green**: 50:50 A/B 테스트
- **Big Bang**: 즉시 100% (비권장)
- **자동 롤백 조건**:
  - 오류율 > 5% (10분 연속)
  - 크래시 발생 > 10건
  - 사용자 만족도 < 3.0 (5점 척도)
- **BOM 필터링**: 호환되지 않는 차량 자동 제외

#### FR-007: A/B 실험 및 통계 분석
- **실험 설계**: Control vs Variant (최대 3개 Variant)
- **메트릭 수집**: 사용 빈도, 오류율, 만족도, 컨버전율
- **통계 검증**: p-value < 0.05, 신뢰도 95%
- **ROI 예측**: 예상 고객만족도 증가, NPS 향상

#### FR-008: CodeBeamer 연동 (추적성)
- **양방향 추적**: 요구사항 → Feature → Flag → 테스트 케이스
- **자동 동기화**: CodeBeamer Webhook 수신
- **ASIL 검증**: 각 단계의 ASIL 등급 일관성 검증
- **Audit**: 변경 이력 자동 기록

#### FR-009: 오프라인 모드
- **로컬 캐시**: SQLite에 최근 7일 정책 저장
- **TTL 기반 갱신**: 7일 경과 시 서버 재동기화 필요
- **Graceful Degradation**: 네트워크 단절 시 캐시 사용
- **재연결 시 Delta Sync**: 변경된 Flag만 동기화

### 비기능 요구사항 (Non-Functional Requirements)

#### NFR-001: 성능 (Performance)
- **API 응답시간**: P95 < 50ms
- **Flag 평가 지연**: P99 < 10ms (로컬 캐시)
- **Kill Switch 전파**: < 3시간 (전 차량 100%)
- **Throughput**: 10,000 req/sec (API Server)

#### NFR-002: 확장성 (Scalability)
- **지원 차량**: Phase 1: 10대 → Phase 3: 100,000대
- **지원 Flag 수**: 10,000개
- **HPA**: CPU 70%, Memory 80% 기준 자동 확장
- **Database Sharding**: 지역별 PostgreSQL Replica

#### NFR-003: 가용성 (Availability)
- **목표**: 99.9% (연간 다운타임 < 8.76시간)
- **Failover**: PostgreSQL Primary-Replica 자동 전환
- **Redis HA**: Sentinel 기반 Cluster
- **Multi-Region**: Azure Korea Central + East Asia

#### NFR-004: 보안 (Security)
- **네트워크**: Azure Front Door (CDN/WAF), Application Gateway
- **인증**: JWT (Admin), API Key (SDK), mTLS (Vehicle)
- **암호화**: TLS 1.3+ (전송), TDE (저장)
- **RBAC**: Role-Based Access Control
- **Audit**: 모든 변경 이력 감사 로그

#### NFR-005: 메모리 사용량 (Vehicle Agent)
- **목표**: ≤ 64MB
- **구성**: SDK 4MB + Cache 10MB + Runtime 50MB
- **최적화**: Rust zero-copy, 메모리 풀

#### NFR-006: CPU 사용률 (Vehicle Agent)
- **목표**: ≤ 2% (idle), ≤ 5% (peak)
- **측정**: 차량 텔레메트리로 실시간 모니터링

#### NFR-007: 오프라인 안정성
- **캐시 TTL**: 7일
- **재연결 시**: Delta Sync로 변경분만 동기화
- **Fallback**: 캐시 만료 시 Default Value 사용

---

## 🎨 UI/UX 설계 가이드

### 디자인 원칙
1. **안전 최우선**: Kill Switch는 빨간색으로 항상 눈에 띄게
2. **3클릭 내 접근**: 모든 핵심 기능은 3클릭 이내 도달
3. **실시간 피드백**: 변경 사항 즉시 반영 (1초 이내)
4. **명확한 상태 표시**: 색상 코드로 상태 구분 (Active/Inactive/Killed)

### 현대차 디자인 시스템
- **Primary 색상**: Hyundai Navy `#002c5f`
- **Secondary 색상**: Hyundai Blue `#00aad2`
- **상태 색상**:
  - Success: `#10b981` (Green-600)
  - Warning: `#f59e0b` (Orange-500)
  - Danger: `#dc2626` (Red-600)
- **ASIL 등급 색상**:
  - ASIL-QM: `#6b7280` (Gray-500)
  - ASIL-A: `#3b82f6` (Blue-500)
  - ASIL-B: `#10b981` (Green-600)
  - ASIL-C: `#f59e0b` (Orange-500)
  - ASIL-D: `#8b5cf6` (Purple-600)

### 핵심 화면 (6개 Mock-UI 제공)

#### 1. **BOM 의존성 매트릭스** (`01-bom-dependency-matrix.html`)
- Feature와 HW 부품 간 의존 관계 시각화
- 드래그앤드롭으로 관계 설정
- 충돌 감지 및 경고
- PLM 시스템과 실시간 동기화

#### 2. **Safety Gatekeeper 대시보드** (`02-safety-gatekeeper-dashboard.html`)
- 실시간 차량 컨텍스트 모니터링 (배터리, DTC, 속도, GPS)
- 안전 규칙 엔진 (조건식 기반 자동 차단)
- Override 이력 타임라인
- Emergency Stop 버튼

#### 3. **CodeBeamer 추적성 관리** (`03-codebeamer-traceability.html`)
- 요구사항 → Feature → Flag → 테스트 케이스 추적
- 양방향 링크 시각화 (Collapsible Tree)
- ASIL 등급 일관성 검증
- 자동 동기화 상태 표시

#### 4. **Kill Switch 제어 센터** (`04-kill-switch-control-center.html`)
- 긴급 차단 발동 패널 (사유 입력 필수)
- 2단계 인증 (SMS/OTP)
- 영향도 예측 (대상 차량, 전파 시간)
- 실시간 전파 상태 모니터링
- 최근 이력 타임라인

#### 5. **Rollout 전략 에디터** (`05-rollout-strategy-editor.html`)
- 단계별 Canary 배포 설정 (1% → 5% → 50% → 100%)
- 타겟팅 규칙 (지역, BOM, VIN, SW 버전)
- 자동 롤백 조건 (오류율, 크래시)
- 배포 타임라인 시뮬레이션
- 위험도 평가 게이지

#### 6. **Analytics & Monitoring 대시보드** (`06-analytics-monitoring-dashboard.html`)
- 실시간 KPI 카드 (활성 차량, Flag 수, 평가 횟수, 오류율)
- 성능 차트 (지연시간 P50/P95/P99, 평가 요청량)
- A/B 테스트 결과 (통계적 유의성, ROI)
- 지역별 히트맵
- Top 10 Feature Flag 순위

---

## 🔗 시스템 인터페이스 명세

### REST API (Admin API Server)

#### 인증 (Authentication)
```
POST /api/v1/auth/login
POST /api/v1/auth/logout
POST /api/v1/auth/refresh
```

#### Feature Flags (CRUD)
```
GET    /api/v1/flags              # 전체 목록
GET    /api/v1/flags/{id}         # 상세 조회
POST   /api/v1/flags              # 생성
PATCH  /api/v1/flags/{id}         # 수정
DELETE /api/v1/flags/{id}         # 삭제 (Soft Delete)
```

#### Kill Switch
```
POST   /api/v1/flags/{id}/kill-switch/activate    # 긴급 차단
POST   /api/v1/flags/{id}/kill-switch/deactivate  # 복구
GET    /api/v1/kill-switches                      # 이력 조회
```

#### Evaluation (평가)
```
POST   /api/v1/evaluate              # 단일 평가
POST   /api/v1/evaluate/batch        # 배치 평가
```

#### Analytics (분석)
```
GET    /api/v1/analytics/flags/{id}/metrics       # Flag 메트릭
GET    /api/v1/analytics/experiments/{id}/results # A/B 결과
```

#### Vehicles (차량 관리)
```
GET    /api/v1/vehicles              # 전체 차량 목록
GET    /api/v1/vehicles/{vin}        # VIN별 상세
POST   /api/v1/vehicles/{vin}/sync   # 수동 동기화
```

### gRPC Protocol (Vehicle-Cloud 통신)

```protobuf
service EvaluationService {
  rpc EvaluateFlag(EvaluateRequest) returns (EvaluateResponse);
  rpc BatchEvaluate(BatchEvaluateRequest) returns (stream EvaluateResponse);
  rpc SyncFlags(SyncRequest) returns (stream FlagConfig);
}

service TelemetryService {
  rpc ReportMetrics(stream MetricEvent) returns (Ack);
  rpc ReportOverride(OverrideEvent) returns (Ack);
}
```

### SDK Interface (차량 SDK)

#### C++ SDK (AUTOSAR Adaptive)
```cpp
#include <fleetflag/sdk.h>

FleetFlagClient client("wss://fleetflag.azure.hyundai.com", "VIN123");
bool isEnabled = client.evaluate("feature-autopilot-l3", false);

if (isEnabled) {
  // Feature logic
}
```

#### Python SDK (AI/ML)
```python
from fleetflag import FleetFlagClient

client = FleetFlagClient(url="wss://...", vin="VIN123")
is_enabled = client.evaluate("feature-voice-ai", default=False)

if is_enabled:
    # Feature logic
```

#### Rust SDK (UMOS Core)
```rust
use fleetflag::Client;

let client = Client::new("wss://...", "VIN123").await?;
let is_enabled = client.evaluate("feature-ota-update", false).await?;

if is_enabled {
    // Feature logic
}
```

### WebSocket Protocol (Kill Switch 실시간 푸시)

```json
// Server → Vehicle
{
  "type": "kill_switch",
  "flag_id": "FLAG-2024-0523",
  "action": "activate",
  "reason": "Sensor fault detected",
  "timestamp": "2026-02-01T14:35:22Z"
}

// Vehicle → Server (Ack)
{
  "type": "ack",
  "flag_id": "FLAG-2024-0523",
  "vin": "KMHXX00BXPU123456",
  "status": "applied",
  "timestamp": "2026-02-01T14:35:23Z"
}
```

---

## 🔐 보안 설계

### 4계층 보안 아키텍처

#### Layer 1: Network Security
- **Azure Front Door**: DDoS 방어, WAF, CDN
- **Application Gateway**: Rate Limiting, IP Whitelisting, JWT Validation
- **VPN**: 차량-서버 간 암호화 터널

#### Layer 2: Application Security
- **Admin**: JWT (HS256, 15분 만료, Refresh Token 7일)
- **SDK**: API Key (SHA-256, 환경 변수 저장)
- **RBAC**: Admin, Engineer, Viewer 역할 기반 권한

#### Layer 3: Vehicle Security
- **mTLS**: 차량 인증서 기반 상호 인증
- **VIN-based Certificate**: 차량별 고유 인증서
- **Certificate Pinning**: 중간자 공격 방어

#### Layer 4: Data Security
- **전송 암호화**: TLS 1.3+
- **저장 암호화**: PostgreSQL TDE, Redis 암호화
- **Audit Log**: 모든 변경 이력 암호화 저장

---

## 🚀 개발 로드맵 및 예산

### Phase 1: MVP (12개월) - 2026.03 ~ 2027.02
- **목표**: PoC 완성 및 내부 검증
- **개발 항목**:
  - Rust API Server (기본 CRUD)
  - Rust Vehicle Agent (로컬 평가)
  - Admin Dashboard (React)
  - C++ SDK (AUTOSAR)
  - Docker Compose 개발 환경
- **예산**: 6억 원
- **지원 차량**: 10대

### Phase 2: Production (12개월) - 2027.03 ~ 2028.02
- **목표**: 양산 준비 및 Beta 테스트
- **개발 항목**:
  - Flipt 엔진 커스터마이징
  - Kill Switch + Safety Gatekeeper
  - A/B 실험 기능
  - Python/Rust SDK
  - Azure AKS 배포
  - CodeBeamer 연동
- **예산**: 12억 원
- **지원 차량**: 100대

### Phase 3: Scale-Up (6개월) - 2028.03 ~ 2028.08
- **목표**: 전사 확대 및 대량 양산
- **개발 항목**:
  - 100,000대 확장 (HPA, Sharding)
  - ClickHouse 분석 기능
  - ISO 26262 인증
  - Tier 1 협력사 API Gateway 통합
- **예산**: 5억 원
- **지원 차량**: 10,000대 → 100,000대

**총 예산**: 23억 원 (부가세 제외)

---

## 📞 연락처 및 다음 단계

### 문의처
- **이메일**: team@fleetflag.dev
- **Slack**: #fleetflag-project
- **긴급**: Safety Manager (이성룡 부장)

### 다음 단계
1. ✅ **문서 검토 완료** (본 문서)
2. 🔲 **GitHub 저장소 생성 및 Push**
3. 🔲 **Stakeholder 승인** (CTO, Safety Team)
4. 🔲 **Phase 1 Kick-off** (2026.03)

---

## 📂 저장소 구조

```
fleetflag-specification/
├── README.md                          # 프로젝트 개요
├── DEVELOPMENT_GUIDE.md               # 본 문서 (종합 가이드)
├── 01-requirements/                   # 요구사항
│   └── SRS-system-requirements.md
├── 02-architecture/                   # 아키텍처
│   └── system-architecture.md
├── 03-interface/                      # 인터페이스
│   ├── api-specification.md
│   └── system-integration.md
├── 04-database/                       # 데이터베이스
│   └── schema-design.md
├── 05-security/                       # 보안
│   └── security-design.md
├── 06-ui-ux/                          # UI/UX
│   └── ui-wireframes.md
└── 07-mock-ui/                        # Mock-UI (HTML)
    ├── README.md
    ├── 01-bom-dependency-matrix.html
    ├── 02-safety-gatekeeper-dashboard.html
    ├── 03-codebeamer-traceability.html
    ├── 04-kill-switch-control-center.html
    ├── 05-rollout-strategy-editor.html
    └── 06-analytics-monitoring-dashboard.html
```

---

## ✅ 체크리스트

### 문서화 완료 항목
- [x] 시스템 요구사양서 (SRS)
- [x] 시스템 아키텍처 설계서
- [x] API 인터페이스 명세서
- [x] 데이터베이스 스키마 설계
- [x] 보안 설계서
- [x] UI 와이어프레임
- [x] Mock-UI 6개 (HTML/CSS/JS)
- [x] 종합 개발 가이드 (본 문서)

### 구현 대기 항목
- [ ] Rust API Server 개발
- [ ] Rust Vehicle Agent 개발
- [ ] React Admin Dashboard
- [ ] C++/Python/Rust SDK
- [ ] gRPC Protocol 구현
- [ ] WebSocket Kill Switch
- [ ] CodeBeamer 연동
- [ ] Azure AKS 배포

---

**최종 업데이트**: 2026-02-01  
**문서 버전**: v1.0.0  
**상태**: 🟢 Complete  
**작성자**: FleetFlag 개발팀
