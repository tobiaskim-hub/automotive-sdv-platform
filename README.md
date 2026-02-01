# FleetFlag - SDV Feature Management Platform Specification

> 현대자동차 SDV 전환을 위한 Feature Flag Management System 기술 명세서

## 📋 문서 개요

본 저장소는 **FleetFlag** 프로젝트의 공식 기술 명세 문서를 관리합니다.

- **프로젝트명**: FleetFlag - SDV Feature Management Platform
- **발주처**: 현대자동차 (Hyundai Motor Company)
- **개발사**: (주) 무투스랩
- **버전**: v1.0
- **작성일**: 2026-02-01
- **문서 상태**: 🟡 Draft (검토 중)

---

## 🎯 프로젝트 목표

1. **안전 크리티컬 Feature Flag 관리**: ISO 26262 준수
2. **협력사(Tier 1) 블랙박스 SW 제어**: API Gateway 레벨 제어
3. **Kill Switch 기능**: 3초 이내 긴급 비활성화
4. **오프라인 안정성**: 7일 캐시로 네트워크 단절 시에도 안정적 동작

---

## 📁 문서 구조

### 1️⃣ 요구사항 (Requirements)
- **[시스템 요구사양서](01-requirements/SRS-system-requirements.md)** - 전체 시스템 요구사항
- **[기능 요구사항](01-requirements/functional-requirements.md)** - 기능별 상세 요구사항
- **[비기능 요구사항](01-requirements/non-functional-requirements.md)** - 성능, 보안, 가용성

### 2️⃣ 아키텍처 (Architecture)
- **[시스템 아키텍처 설계서](02-architecture/system-architecture.md)** - 전체 시스템 구조
- **[컴포넌트 상세 설계](02-architecture/component-design.md)** - 각 컴포넌트 내부 구조
- **[데이터 플로우](02-architecture/data-flow.md)** - 데이터 흐름 및 통신 프로토콜
- **[배포 아키텍처](02-architecture/deployment-architecture.md)** - Azure 배포 구조

### 3️⃣ 인터페이스 (Interface Specification)
- **[REST API 명세서](03-interface/api-specification.md)** - Admin API 엔드포인트
- **[gRPC 프로토콜](03-interface/grpc-protocol.md)** - Vehicle-Cloud 통신 프로토콜
- **[SDK 인터페이스](03-interface/sdk-interface.md)** - C++/Python/Rust SDK
- **[OpenAPI Spec](03-interface/openapi.yaml)** - OpenAPI 3.0 형식 스펙

### 4️⃣ 데이터베이스 (Database)
- **[스키마 설계서](04-database/schema-design.md)** - DB 스키마 설계 문서
- **[PostgreSQL DDL](04-database/postgresql-schema.sql)** - Control Plane DB
- **[SQLite DDL](04-database/sqlite-schema.sql)** - Vehicle Agent 캐시

### 5️⃣ 보안 (Security)
- **[보안 설계서](05-security/security-design.md)** - 보안 아키텍처
- **[인증/인가](05-security/authentication.md)** - JWT, API Key, mTLS
- **[암호화 전략](05-security/encryption.md)** - 데이터 암호화

### 6️⃣ UI/UX 설계 (User Interface)
- **[UI 와이어프레임](06-ui-ux/ui-wireframes.md)** - Admin Dashboard UI
- **[디자인 시스템](06-ui-ux/design-system.md)** - 현대차 브랜드 컬러
- **[사용자 플로우](06-ui-ux/user-flows.md)** - 주요 기능 플로우

### 7️⃣ Mock-UI (Interactive Prototypes)
- **[Mock-UI 모음](07-mock-ui/README.md)** - HTML/CSS/JS 인터랙티브 프로토타입
- **[BOM 의존성 매트릭스](07-mock-ui/01-bom-dependency-matrix.html)** - 하드웨어 의존성 관리
- **[Safety Gatekeeper](07-mock-ui/02-safety-gatekeeper-dashboard.html)** - 실시간 안전 모니터링
- **[CodeBeamer 추적성](07-mock-ui/03-codebeamer-traceability.html)** - 요구사항 추적성
- **[Kill Switch 제어](07-mock-ui/04-kill-switch-control-center.html)** - 긴급 차단 시스템
- **[Rollout 전략](07-mock-ui/05-rollout-strategy-editor.html)** - 단계별 배포 설계
- **[Analytics 대시보드](07-mock-ui/06-analytics-monitoring-dashboard.html)** - 성능 분석

---

## 🏗️ 기술 스택

### Control Plane (Cloud)
- **언어**: Rust (API Server), Go (Flipt Engine)
- **프레임워크**: Axum (Rust), Flipt (Go)
- **데이터베이스**: PostgreSQL (메타데이터), Redis (캐시), ClickHouse (분석)
- **인프라**: Azure AKS (Kubernetes)
- **표준**: OpenFeature Specification

### Vehicle Plane (차량)
- **언어**: Rust (Core Agent)
- **통신**: gRPC (Tonic), WebSocket
- **캐시**: SQLite (로컬 캐시)
- **SDK**: C++ (AUTOSAR Adaptive), Python (AI/ML), Rust (UMOS Core)

### Admin Dashboard (관리자)
- **프레임워크**: React 18 + TypeScript
- **빌드**: Vite
- **스타일**: TailwindCSS
- **상태 관리**: TanStack Query (React Query)

---

## 📊 주요 KPI

| 항목 | 목표 | 측정 방법 |
|------|------|-----------|
| API 응답 시간 (P95) | < 50ms | Prometheus + Grafana |
| Kill Switch 전파 시간 | < 3초 | WebSocket 이벤트 타임스탬프 |
| 차량 Agent 메모리 사용량 | ≤ 64MB | 차량 텔레메트리 |
| 차량 Agent CPU 사용률 | ≤ 2% | 차량 텔레메트리 |
| 시스템 가용성 | 99.9% | Azure Monitor |
| 지원 차량 수 | 100,000대 | Phase 3 목표 |

---

## 🗓️ 개발 로드맵

### Phase 1: MVP (12개월) - 2026.03 ~ 2027.02
- Rust API Server 개발
- Rust Vehicle Agent 개발
- Admin Dashboard 기본 기능
- C++ SDK (AUTOSAR)
- Docker Compose 개발 환경

**예산**: 6억 원  
**지원 차량**: 10대 (PoC)

### Phase 2: Production (12개월) - 2027.03 ~ 2028.02
- Flipt 엔진 커스터마이징
- 다중 환경 지원 (Dev/Staging/Prod)
- A/B 실험 기능
- Python/Rust SDK
- Azure AKS 배포

**예산**: 12억 원  
**지원 차량**: 100대 (Beta)

### Phase 3: Scale-Up (6개월) - 2028.03 ~ 2028.08
- 전사 확대 (10,000대)
- 협력사(Tier 1) API Gateway 통합
- 고급 분석 기능 (ClickHouse)
- ISO 26262 인증

**예산**: 5억 원  
**지원 차량**: 10,000대 (양산)

**총 예산**: 23억 원 (부가세 제외)

---

## 📝 문서 작성 규칙

1. **Markdown 포맷 사용**: 모든 문서는 `.md` 형식
2. **버전 관리**: Git으로 모든 변경 이력 추적
3. **코드 블록**: 기술 명세는 코드 블록으로 명시
4. **다이어그램**: `assets/diagrams/`에 이미지 저장
5. **검토 상태**: 🔴 Draft, 🟡 Review, 🟢 Approved

---

## 👥 문서 담당자

| 역할 | 담당자 | 연락처 |
|------|--------|--------|
| 프로젝트 매니저 | - | - |
| 아키텍처 설계 | - | - |
| 기술 명세 작성 | - | - |
| 보안 검토 | - | - |

---

## 📞 연락처

- **이메일**: team@fleetflag.dev
- **Slack**: #fleetflag-project
- **이슈 트래킹**: GitHub Issues

---

## 📜 라이선스

본 문서는 현대자동차 및 (주) 무투스랩의 기밀 문서입니다.  
**CONFIDENTIAL** - 무단 복제 및 배포 금지

---

**최종 업데이트**: 2026-02-01  
**문서 버전**: v1.0
