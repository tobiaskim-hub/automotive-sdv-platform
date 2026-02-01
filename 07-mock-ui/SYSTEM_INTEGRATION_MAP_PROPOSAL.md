# FleetFlag 시스템 연동 아키텍처 시각화 도구 - 기획서

> **목적**: FleetFlag Feature Flag System이 현대자동차의 다양한 내부/외부 시스템들과 어떻게 연동되며, 실시간으로 데이터를 주고받는지를 **전체 그림**으로 시각화

---

## 📋 Executive Summary

### 핵심 문제
- FleetFlag는 **독립 시스템이 아님** → 10개 이상의 시스템과 긴밀히 연동
- 각 시스템의 **역할과 데이터 흐름**이 명확하지 않음
- 장애 발생 시 **어느 시스템이 원인인지** 파악 어려움
- 신규 개발자/협력사가 **전체 구조 이해**에 시간 소요

### 솔루션
**"FleetFlag System Integration Map"** - 인터랙티브 시스템 연동 지도
- 11개 연동 시스템을 **실시간 네트워크 그래프**로 시각화
- 각 시스템의 **역할, 데이터 타입, 프로토콜, 의존성** 표시
- **클릭 → 상세 정보** 패널로 연동 스펙 확인
- **실시간 헬스체크** 및 장애 시뮬레이션

---

## 🎯 목표 사용자

| 사용자 그룹 | 니즈 | 기대 효과 |
|------------|------|----------|
| **CTO/임원진** | 전체 시스템 구조 한눈에 파악 | 의사결정 속도 향상, 투자 우선순위 결정 |
| **시스템 아키텍트** | 연동 포인트 및 병목 지점 식별 | 아키텍처 최적화, 확장성 설계 |
| **개발자 (Backend)** | API 엔드포인트 및 데이터 스펙 확인 | 개발 시간 단축, 오류 감소 |
| **운영팀 (DevOps)** | 시스템 간 의존성 및 장애 영향 분석 | 장애 대응 시간 단축, 예방 관리 |
| **협력사 (Tier 1)** | 연동 방법 및 요구사항 이해 | 연동 개발 기간 단축, 품질 향상 |
| **보안팀** | 데이터 흐름 및 보안 경계 확인 | 보안 취약점 사전 발견, 컴플라이언스 |

---

## 🏗️ 시스템 연동 구조 (11개 시스템)

### 핵심 연동 시스템

```
                    ┌─────────────────────────────────────┐
                    │   🏢 현대자동차 내부 시스템          │
                    └─────────────────────────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
   ┌────▼────┐              ┌──────▼──────┐            ┌──────▼──────┐
   │CodeBeamer│              │ PLM System  │            │   SUMS      │
   │(ALM)    │              │(BOM 관리)   │            │(OTA Update) │
   └────┬────┘              └──────┬──────┘            └──────┬──────┘
        │                           │                           │
        └───────────────────────────┼───────────────────────────┘
                                    │
                            ┌───────▼────────┐
                            │   FleetFlag    │ 🎯 중심 시스템
                            │  Feature Flag  │
                            │    System      │
                            └───────┬────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
   ┌────▼────┐              ┌──────▼──────┐            ┌──────▼──────┐
   │ Azure   │              │  Vehicles   │            │ ClickHouse  │
   │ Services│              │   (Fleet)   │            │(Analytics)  │
   │ (Infra) │              │  125,000대  │            │             │
   └────┬────┘              └──────┬──────┘            └──────┬──────┘
        │                           │                           │
        └───────────────────────────┼───────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
              ┌─────▼─────┐                  ┌─────▼─────┐
              │   Slack   │                  │  Grafana  │
              │ (알림/협업)│                  │(모니터링) │
              └───────────┘                  └───────────┘
```

---

## 📊 연동 시스템 상세 명세

### 1. **CodeBeamer (ALM - Application Lifecycle Management)**

#### 역할
- **요구사항 관리**: 각 Feature Flag를 요구사항 ID와 연결
- **추적성(Traceability)**: 요구사항 → Feature → Flag → 테스트 케이스 양방향 추적
- **ASIL 검증**: 각 단계의 ASIL 등급 일관성 자동 검증
- **변경 이력**: 요구사항 변경 시 관련 Flag 자동 업데이트

#### 데이터 흐름
```
CodeBeamer → FleetFlag: 요구사항 생성/변경 이벤트 (Webhook)
FleetFlag → CodeBeamer: Flag 상태 업데이트 (REST API)
```

#### 연동 스펙
- **프로토콜**: REST API + Webhook
- **인증**: OAuth 2.0 (Client Credentials)
- **데이터 포맷**: JSON
- **주기**: 실시간 (Webhook 트리거)
- **엔드포인트**:
  - `GET /api/v1/requirements/{id}` - 요구사항 조회
  - `POST /api/v1/requirements/{id}/flags` - Flag 연결
  - `PUT /api/v1/flags/{id}/requirement` - 요구사항 업데이트

#### 데이터 예시
```json
{
  "requirement_id": "REQ-2024-0523",
  "title": "자율주행 L3 고속도로 모드",
  "asil_level": "ASIL-D",
  "status": "approved",
  "assigned_to": "이성룡 부장",
  "linked_flags": ["FLAG-2024-0523"]
}
```

#### 의존성
- **상향 의존**: FleetFlag는 CodeBeamer의 요구사항 데이터에 의존
- **장애 영향**: CodeBeamer 다운 시 새 Flag 생성 불가 (기존 Flag는 정상 동작)

---

### 2. **PLM System (Product Lifecycle Management - BOM 관리)**

#### 역할
- **BOM 데이터 동기화**: 차량별 하드웨어 부품 목록 실시간 동기화
- **의존성 검증**: Feature Flag 배포 전 BOM 호환성 자동 체크
- **부품 정보**: 부품 번호, 버전, 제조사, 설치 위치
- **변경 알림**: BOM 변경 시 영향받는 Flag 목록 자동 계산

#### 데이터 흐름
```
PLM → FleetFlag: BOM 데이터 일괄 동기화 (배치 API, 매일 02:00)
PLM → FleetFlag: BOM 변경 이벤트 (Webhook, 실시간)
FleetFlag → PLM: 차량별 BOM 조회 (REST API, 온디맨드)
```

#### 연동 스펙
- **프로토콜**: REST API + FTP (대용량 배치)
- **인증**: API Key + IP Whitelisting
- **데이터 포맷**: XML (레거시) + JSON (신규)
- **주기**: 
  - 배치 동기화: 매일 02:00 (전체 125,000대)
  - 실시간 이벤트: Webhook
- **엔드포인트**:
  - `GET /api/v1/vehicles/{vin}/bom` - 차량 BOM 조회
  - `POST /api/v1/bom/sync` - 수동 동기화 트리거

#### 데이터 예시
```json
{
  "vin": "KMHXX00BXPU123456",
  "model": "GN7",
  "model_year": 2026,
  "bom_revision": "3.2",
  "components": [
    {
      "part_number": "ECU-HW-0234",
      "name": "Mobileye EyeQ5",
      "version": "v2.3.1",
      "manufacturer": "Mobileye",
      "install_date": "2025-11-23"
    },
    {
      "part_number": "HPC-CPU-0012",
      "name": "NVIDIA Drive AGX Orin",
      "ram_gb": 8,
      "manufacturer": "NVIDIA"
    }
  ]
}
```

#### 의존성
- **상향 의존**: FleetFlag의 BOM 필터링 기능은 PLM 데이터에 의존
- **장애 영향**: PLM 다운 시 BOM 기반 타겟팅 불가 (VIN 기반은 가능)

---

### 3. **SUMS (Software Update Management System - OTA)**

#### 역할
- **OTA 업데이트 연동**: Feature Flag 변경 시 OTA 패키지 자동 생성
- **업데이트 스케줄링**: Flag 배포 일정과 OTA 일정 동기화
- **버전 관리**: 차량 SW 버전과 Flag 호환성 매칭
- **롤백 지원**: Flag 롤백 시 이전 SW 버전으로 복원

#### 데이터 흐름
```
FleetFlag → SUMS: Flag 변경 이벤트 전송 (REST API)
SUMS → FleetFlag: OTA 업데이트 상태 알림 (Webhook)
FleetFlag → SUMS: 차량 SW 버전 조회 (REST API)
```

#### 연동 스펙
- **프로토콜**: REST API + Webhook
- **인증**: mTLS (상호 인증)
- **데이터 포맷**: JSON
- **주기**: 실시간 + 배치 (OTA 패키지는 매일 00:00)
- **엔드포인트**:
  - `POST /api/v1/ota/deploy` - OTA 배포 요청
  - `GET /api/v1/vehicles/{vin}/sw-version` - SW 버전 조회
  - `POST /api/v1/ota/rollback` - 롤백 요청

#### 데이터 예시
```json
{
  "deployment_id": "OTA-20260201-001",
  "flag_id": "FLAG-2024-0812",
  "target_vehicles": 10234,
  "sw_package": "sw-v5.3.2-feature-voice-ai.tar.gz",
  "schedule": "2026-02-05T02:00:00Z",
  "status": "scheduled"
}
```

#### 의존성
- **양방향 의존**: FleetFlag와 SUMS는 서로 의존 (긴밀한 통합)
- **장애 영향**: SUMS 다운 시 새 OTA 배포 불가 (기존 Flag는 정상 동작)

---

### 4. **Azure Services (Cloud Infrastructure)**

#### 역할
- **인프라 제공**: AKS (Kubernetes), PostgreSQL, Redis, Application Gateway
- **보안 경계**: Front Door (WAF), Virtual Network (VPN), Key Vault
- **모니터링**: Azure Monitor, Log Analytics
- **스케일링**: HPA (Auto Scaling), Load Balancer

#### 데이터 흐름
```
FleetFlag API → Azure AKS: 컨테이너 실행
FleetFlag API → Azure PostgreSQL: 데이터 저장
FleetFlag API → Azure Redis: 캐시 읽기/쓰기
FleetFlag → Azure Monitor: 메트릭 전송
```

#### 연동 스펙
- **프로토콜**: Azure SDK, kubectl (Kubernetes API)
- **인증**: Managed Identity, Service Principal
- **주기**: 실시간 (항상 연결)

#### 의존성
- **강한 의존**: FleetFlag는 Azure 없이 동작 불가
- **장애 영향**: Azure 다운 시 전체 시스템 정지

---

### 5. **Vehicle Fleet (125,000대 차량)**

#### 역할
- **Flag 평가**: 로컬 SDK에서 Feature Flag 평가 (2.1ms)
- **텔레메트리 전송**: 사용 빈도, 오류율, 센서 데이터 전송
- **Delta Sync**: 변경된 Flag만 동기화 (효율성)
- **오프라인 모드**: 7일간 로컬 캐시로 동작

#### 데이터 흐름
```
FleetFlag → Vehicle: Flag 동기화 (gRPC, Delta Sync)
Vehicle → FleetFlag: 텔레메트리 전송 (gRPC Stream)
FleetFlag → Vehicle: Kill Switch 푸시 (WebSocket)
```

#### 연동 스펙
- **프로토콜**: gRPC (동기화), WebSocket (Kill Switch)
- **인증**: mTLS (VIN 기반 인증서)
- **데이터 포맷**: Protobuf (gRPC)
- **주기**: 
  - Delta Sync: 5분마다
  - 텔레메트리: 30초마다 배치 전송
  - Kill Switch: 즉시 (< 3초)

#### 데이터 예시
```protobuf
message TelemetryEvent {
  string vin = 1;
  string flag_key = 2;
  bool evaluation_result = 3;
  int64 latency_ms = 4;
  string context_data = 5; // JSON
  int64 timestamp = 6;
}
```

#### 의존성
- **양방향 의존**: FleetFlag와 Vehicle은 서로 의존
- **장애 영향**: Vehicle 오프라인 시 로컬 캐시로 7일간 동작

---

### 6. **ClickHouse (Analytics Database)**

#### 역할
- **대용량 분석**: 평가 이벤트 (2.8억 건/월) 실시간 분석
- **A/B 테스트**: 통계적 유의성 검증 (p-value, 신뢰도)
- **성능 메트릭**: 지연시간 P50/P95/P99, 캐시 적중률
- **대시보드**: Grafana 연동으로 실시간 차트

#### 데이터 흐름
```
Vehicle → FleetFlag → ClickHouse: 텔레메트리 배치 삽입 (1만 건/배치)
FleetFlag → ClickHouse: 분석 쿼리 실행 (A/B 테스트 결과)
Grafana → ClickHouse: 대시보드 쿼리
```

#### 연동 스펙
- **프로토콜**: Native TCP (ClickHouse 전용)
- **인증**: User/Password + IP Whitelisting
- **데이터 포맷**: TSV (Tab-Separated Values) 또는 JSON
- **주기**: 배치 삽입 (1분마다 또는 1만 건마다)

#### 데이터 예시
```sql
-- evaluation_events 테이블
CREATE TABLE evaluation_events (
  event_time DateTime,
  vin String,
  flag_key String,
  result UInt8,
  latency_ms UInt16,
  cache_hit UInt8
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_time, vin);
```

#### 의존성
- **하향 의존**: ClickHouse는 FleetFlag 데이터를 수신만 함
- **장애 영향**: ClickHouse 다운 시 분석 불가, Flag 동작은 정상

---

### 7. **Slack (Collaboration & Notification)**

#### 역할
- **긴급 알림**: Kill Switch 발동 시 #safety-critical 채널 알림
- **배포 알림**: 새 Flag 배포 시 #feature-flags 채널 알림
- **오류 알림**: 오류율 > 5% 시 #alerts 채널 알림
- **승인 요청**: ASIL-D Flag는 CTO 승인 요청 (인터랙티브 버튼)

#### 데이터 흐름
```
FleetFlag → Slack: Webhook으로 메시지 전송
Slack → FleetFlag: 버튼 클릭 시 Callback API 호출
```

#### 연동 스펙
- **프로토콜**: Slack Webhook (Incoming Webhook)
- **인증**: Webhook URL (Secret Token 포함)
- **데이터 포맷**: JSON (Slack Block Kit)
- **주기**: 이벤트 기반 (즉시)

#### 데이터 예시
```json
{
  "channel": "#safety-critical",
  "username": "FleetFlag Alert",
  "icon_emoji": ":rotating_light:",
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "🔴 *Kill Switch Activated*\n*Flag*: FLAG-2024-0523 (자율주행 L3)\n*Reason*: Mobileye 센서 결함\n*Vehicles*: 12,847대"
      }
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": { "type": "plain_text", "text": "상세 보기" },
          "url": "https://fleetflag.azure.hyundai.com/flags/FLAG-2024-0523"
        }
      ]
    }
  ]
}
```

#### 의존성
- **약한 의존**: Slack 다운 시에도 FleetFlag는 정상 동작
- **장애 영향**: Slack 다운 시 알림 누락 (로그는 정상 기록)

---

### 8. **Grafana (Monitoring & Observability)**

#### 역할
- **실시간 대시보드**: 활성 차량, Flag 수, 평가 횟수, 오류율
- **성능 모니터링**: API 지연시간, DB 커넥션, Redis 메모리
- **알림 규칙**: 오류율 > 5% 시 Slack 알림
- **커스텀 쿼리**: PromQL (Prometheus), ClickHouse SQL

#### 데이터 흐름
```
Prometheus → FleetFlag: 메트릭 스크래핑 (/metrics 엔드포인트)
Grafana → Prometheus: PromQL 쿼리
Grafana → ClickHouse: SQL 쿼리 (A/B 테스트 결과)
```

#### 연동 스펙
- **프로토콜**: HTTP (Prometheus Exporter), TCP (ClickHouse)
- **인증**: Bearer Token (Grafana API)
- **데이터 포맷**: Prometheus 메트릭 포맷
- **주기**: 15초마다 스크래핑

#### 의존성
- **약한 의존**: Grafana 다운 시에도 FleetFlag는 정상 동작
- **장애 영향**: Grafana 다운 시 모니터링 불가

---

### 9. **Prometheus (Metrics Collection)**

#### 역할
- **메트릭 수집**: FleetFlag API의 /metrics 엔드포인트 스크래핑
- **시계열 저장**: 메트릭 데이터 15일 보관
- **알림 규칙**: Alertmanager 연동

#### 데이터 흐름
```
FleetFlag → Prometheus: /metrics 엔드포인트 노출
Prometheus → Alertmanager: 알림 규칙 트리거
```

#### 연동 스펙
- **프로토콜**: HTTP (Pull 방식)
- **데이터 포맷**: Prometheus 메트릭 포맷
- **주기**: 15초마다

---

### 10. **Azure Log Analytics (Centralized Logging)**

#### 역할
- **로그 수집**: FleetFlag API, Flipt, Vehicle Agent 로그 중앙 집계
- **로그 검색**: Kusto Query Language (KQL)
- **알림**: 특정 오류 패턴 감지 시 알림

#### 데이터 흐름
```
FleetFlag → Azure Log Analytics: 로그 전송 (Serilog Sink)
```

#### 연동 스펙
- **프로토콜**: HTTP (Azure Monitor API)
- **인증**: Workspace ID + Shared Key
- **주기**: 실시간 (비동기 전송)

---

### 11. **GitHub (Source Code & CI/CD)**

#### 역할
- **소스 코드 관리**: FleetFlag API, SDK, Admin Dashboard
- **CI/CD**: GitHub Actions로 자동 빌드/테스트/배포
- **버전 관리**: Tag 기반 릴리스

#### 데이터 흐름
```
Developer → GitHub: 코드 푸시
GitHub Actions → Azure AKS: 자동 배포
GitHub → Slack: 배포 알림
```

---

## 🎨 시각화 도구 설계

### 화면 구성 (3-Panel Layout)

```
┌─────────────────────────────────────────────────────────────┐
│  FleetFlag System Integration Map                           │
│  [실시간 헬스체크: 🟢 정상] [마지막 업데이트: 14:35:22]      │
└─────────────────────────────────────────────────────────────┘
┌──────────────────────────┬──────────────────────────────────┐
│                          │                                  │
│   🗺️ System Network      │   📊 Detail Panel               │
│      (Interactive)       │                                  │
│                          │   🔍 Selected: CodeBeamer       │
│   [11개 시스템 노드]      │                                  │
│   [연결선 + 데이터 흐름]  │   역할: 요구사항 관리 (ALM)      │
│                          │   프로토콜: REST API + Webhook   │
│   [클릭으로 상세 보기]    │   인증: OAuth 2.0               │
│                          │   상태: 🟢 정상 (응답 12ms)     │
│                          │                                  │
│                          │   엔드포인트:                    │
│                          │   GET /api/v1/requirements/{id}  │
│                          │                                  │
│                          │   데이터 샘플:                   │
│                          │   {                              │
│                          │     "requirement_id": "REQ-..."  │
│                          │     "asil_level": "ASIL-D"       │
│                          │   }                              │
│                          │                                  │
│                          │   [연동 테스트] [문서 보기]       │
├──────────────────────────┴──────────────────────────────────┤
│  📈 Real-time Metrics Dashboard                             │
│  [활성 연동: 10/11] [평균 응답: 45ms] [오류율: 0.02%]        │
└─────────────────────────────────────────────────────────────┘
```

### 주요 기능

#### 1. **인터랙티브 네트워크 그래프**
- **라이브러리**: D3.js Force-Directed Graph
- **노드**:
  - 중심: FleetFlag (큰 원, 파란색)
  - 주변: 11개 시스템 (중간 원, 색상별 분류)
  - 크기: 데이터 전송량에 비례
- **연결선**:
  - 실선: 동기식 (REST API, gRPC)
  - 점선: 비동기식 (Webhook, Pub/Sub)
  - 화살표: 데이터 흐름 방향
  - 굵기: 트래픽 양에 비례
  - 색상: 상태 (녹색: 정상, 노란색: 지연, 빨간색: 오류)
- **애니메이션**:
  - 데이터 전송 시 점이 연결선을 따라 이동
  - 노드 hover 시 관련 시스템 강조

#### 2. **Detail Panel (우측)**
- **시스템 정보**:
  - 이름, 역할, 담당자
  - 프로토콜, 인증 방식
  - 엔드포인트 목록
- **상태 모니터링**:
  - 실시간 헬스체크 (🟢/🟡/🔴)
  - 평균 응답시간
  - 오류율 (최근 1시간)
- **데이터 샘플**:
  - JSON/XML 구문 강조
  - 실제 연동 데이터 예시
- **액션 버튼**:
  - [연동 테스트]: API 호출 테스트
  - [문서 보기]: Swagger/OpenAPI 문서로 이동
  - [로그 확인]: 최근 100건 로그

#### 3. **Metrics Dashboard (하단)**
- **KPI 카드**:
  - 활성 연동: 10/11 (Slack 다운)
  - 평균 응답시간: 45ms
  - 오류율: 0.02%
  - 데이터 전송량: 2.3GB/hour
- **실시간 차트**:
  - Line Chart: 시스템별 응답시간 (최근 1시간)
  - Bar Chart: 시스템별 트래픽 (최근 5분)

---

## 🎯 핵심 기능 상세

### 1. **실시간 헬스체크**

```javascript
// 각 시스템에 주기적으로 Ping 전송
setInterval(async () => {
  const systems = [
    { name: 'CodeBeamer', endpoint: '/api/v1/health' },
    { name: 'PLM', endpoint: '/ping' },
    // ...
  ];
  
  for (const sys of systems) {
    const startTime = Date.now();
    try {
      await fetch(sys.endpoint, { timeout: 3000 });
      const latency = Date.now() - startTime;
      updateNodeStatus(sys.name, 'healthy', latency);
    } catch (error) {
      updateNodeStatus(sys.name, 'unhealthy', null);
    }
  }
}, 30000); // 30초마다
```

### 2. **장애 시뮬레이션**

```
사용자 액션:
  1. 노드 우클릭 → "장애 시뮬레이션"
  2. PLM 시스템 선택
  3. 장애 유형: "완전 다운" / "응답 지연" / "오류율 증가"

시뮬레이션 결과:
  - PLM 노드: 빨간색으로 변경
  - 연결선: 빨간색 점선으로 변경
  - Detail Panel: "장애 시뮬레이션 중" 표시
  - 영향 분석:
    ✓ BOM 기반 타겟팅 불가
    ✓ VIN 기반 타겟팅은 정상
    ✓ 기존 Flag 동작은 정상
    ⚠️ 신규 Flag 생성 시 경고 표시
```

### 3. **데이터 흐름 추적**

```
사용자 액션:
  1. "Feature 생성 흐름 추적" 버튼 클릭

애니메이션:
  1초: Admin → FleetFlag (POST /api/v1/flags)
  2초: FleetFlag → CodeBeamer (연동 체크)
  3초: FleetFlag → PLM (BOM 검증)
  4초: FleetFlag → PostgreSQL (저장)
  5초: FleetFlag → Redis (캐시)
  6초: FleetFlag → Vehicle (Delta Sync)
  7초: Vehicle → ClickHouse (텔레메트리)
  
각 단계마다:
  - 연결선 위로 점 이동 애니메이션
  - Detail Panel에 현재 단계 정보 표시
  - 예상 소요시간 vs 실제 소요시간 비교
```

### 4. **의존성 분석**

```
사용자 액션:
  1. FleetFlag 노드 클릭
  2. "의존성 분석" 버튼

분석 결과:
┌─────────────────────────────────────┐
│ FleetFlag 의존성 분석                │
├─────────────────────────────────────┤
│ 강한 의존 (다운 시 시스템 정지):     │
│  - Azure AKS (인프라)                │
│  - PostgreSQL (메타데이터)           │
│  - Redis (캐시)                      │
├─────────────────────────────────────┤
│ 중간 의존 (일부 기능 제한):          │
│  - CodeBeamer (신규 Flag 생성)       │
│  - PLM (BOM 기반 타겟팅)             │
│  - SUMS (OTA 배포)                   │
├─────────────────────────────────────┤
│ 약한 의존 (알림/모니터링만 영향):    │
│  - Slack (알림)                      │
│  - Grafana (모니터링)                │
│  - ClickHouse (분석)                 │
└─────────────────────────────────────┘
```

---

## 🛠️ 기술 스택

### 프론트엔드
- **D3.js v7**: Force-Directed Graph, 네트워크 시각화
- **React 18**: UI 컴포넌트
- **TailwindCSS**: 스타일링
- **Recharts**: 메트릭 차트
- **Monaco Editor**: JSON/XML 뷰어

### 백엔드 (Healthcheck API)
- **Hono (Cloudflare Workers)**: 경량 헬스체크 API
- **Redis**: 상태 캐싱 (30초)

### 데이터 소스
- **실시간**: Prometheus 메트릭 (응답시간, 오류율)
- **정적**: 연동 스펙 JSON 파일

---

## 📐 구현 우선순위

### Phase 1: 기본 시각화 (1주)
- [x] 11개 시스템 노드 배치
- [x] 연결선 그리기
- [x] 정적 데이터 표시
- [x] Detail Panel 기본 정보

### Phase 2: 인터랙션 (1주)
- [ ] 노드 클릭 → Detail Panel 업데이트
- [ ] 노드 드래그 & 줌/팬
- [ ] 연결선 hover → 데이터 흐름 툴팁

### Phase 3: 실시간 데이터 (1주)
- [ ] 헬스체크 API 통합
- [ ] 실시간 상태 업데이트 (WebSocket)
- [ ] 메트릭 차트

### Phase 4: 고급 기능 (1주)
- [ ] 장애 시뮬레이션
- [ ] 데이터 흐름 추적 애니메이션
- [ ] 의존성 분석

---

## 🎓 활용 시나리오

### Scenario 1: 신규 협력사 온보딩
```
협력사: "우리 시스템을 FleetFlag와 연동하려면 어떻게 해야 하나요?"
→ Integration Map 열기
→ 비슷한 시스템 (예: SUMS) 클릭
→ 연동 스펙 확인 (REST API, mTLS, JSON)
→ Swagger 문서로 이동
→ 연동 테스트 실행
```

### Scenario 2: 장애 대응
```
알림: "PLM 시스템 응답 없음"
→ Integration Map 열기
→ PLM 노드 빨간색으로 표시
→ "영향 분석" 버튼 클릭
→ 결과: BOM 기반 타겟팅 불가, VIN 기반은 정상
→ 임시 조치: VIN 리스트 기반 배포로 전환
→ PLM 팀에 Slack 알림 전송
```

### Scenario 3: 성능 최적화
```
개발자: "어느 시스템이 병목인가요?"
→ Integration Map 열기
→ Metrics Dashboard 확인
→ PLM 시스템 응답시간 250ms (다른 시스템은 30ms)
→ PLM 노드 클릭 → 로그 확인
→ 원인: PLM BOM 조회 API 느림
→ 해결: FleetFlag에 BOM 캐시 추가 (Redis, 5분 TTL)
```

---

## 📞 다음 단계

### 즉시 실행 가능
1. **Mock 데이터로 프로토타입 제작** (2-3일)
2. **Stakeholder 데모** (CTO, 아키텍트)
3. **피드백 수집 및 개선**

### 중기 계획
1. **실시간 헬스체크 API 개발** (1주)
2. **Prometheus 메트릭 통합** (1주)
3. **장애 시뮬레이션 기능** (1주)

---

이 기획서를 바탕으로 **"System Integration Map"을 제작**해 드릴까요?
- HTML/CSS/JS로 인터랙티브 프로토타입
- 11개 시스템 네트워크 그래프
- Detail Panel + Metrics Dashboard
- 실제 연동 스펙 데이터 포함
