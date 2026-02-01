# FleetFlag 시스템 아키텍처 상세 설계

## 1. 전체 시스템 구성도 (Detailed Architecture)

```
┌────────────────────────── Internet ──────────────────────────────┐
│                                                                   │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │ Admin User   │    │ Tier1 API    │    │ Vehicle      │       │
│  │ (Browser)    │    │ Client       │    │ (IGN ON)     │       │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘       │
│         │                   │                    │               │
└─────────┼───────────────────┼────────────────────┼───────────────┘
          │                   │                    │
          │ HTTPS             │ HTTPS/gRPC         │ gRPC/mTLS
          │                   │                    │
┌─────────▼───────────────────▼────────────────────▼───────────────┐
│                 Azure Front Door (CDN + WAF)                      │
│                 - DDoS Protection                                 │
│                 - SSL/TLS Termination                             │
│                 - Geographic Routing (NA, EU, KR)                 │
└─────────┬──────────────────────────────────────────────────┬─────┘
          │                                                  │
┌─────────▼───────────────────────────────────────────┐     │
│        Azure Application Gateway (WAF)              │     │
│        - Rate Limiting (10K req/min per API key)    │     │
│        - JWT Validation                             │     │
│        - IP Whitelisting                            │     │
└─────────┬───────────────────────────────────────────┘     │
          │                                                  │
┌─────────▼──────────────────────────────────────────────────▼─────┐
│                      Azure AKS (Kubernetes)                       │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                   Ingress Controller                       │  │
│  │                 (NGINX / Traefik)                          │  │
│  └────┬─────────────────────────┬────────────────────────┬───┘  │
│       │                         │                        │       │
│  ┌────▼─────────┐   ┌──────────▼─────────┐   ┌─────────▼────┐  │
│  │              │   │                    │   │              │  │
│  │  Dashboard   │   │  FleetFlag API     │   │  Flipt       │  │
│  │  Service     │   │  Server            │   │  Engine      │  │
│  │  (Static)    │   │  (Rust/Axum)       │   │  (Go)        │  │
│  │              │   │                    │   │              │  │
│  │  2 pods      │   │  3 pods            │   │  2 pods      │  │
│  │  Port: 80    │   │  Port: 8080        │   │  Port: 9000  │  │
│  └──────────────┘   └─────┬──────────────┘   └──────┬───────┘  │
│                           │                          │           │
│  ┌────────────────────────▼──────────────────────────▼────────┐  │
│  │                     Service Mesh (Istio)                   │  │
│  │  - mTLS between services                                   │  │
│  │  - Traffic Management                                      │  │
│  │  - Telemetry Collection                                    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
└──────────────┬────────────────────────────┬───────────────────────┘
               │                            │
┌──────────────▼───────────┐   ┌───────────▼──────────────────────┐
│  Data Layer              │   │  Monitoring & Observability      │
│                          │   │                                  │
│  ┌────────────────────┐  │   │  ┌────────────────────────────┐ │
│  │ PostgreSQL         │  │   │  │ Prometheus                 │ │
│  │ (Managed)          │  │   │  │ - Metrics Collection       │ │
│  │                    │  │   │  │ - Alert Rules              │ │
│  │ Primary + Replica  │  │   │  └────────────────────────────┘ │
│  │ Auto-backup        │  │   │                                  │
│  └────────────────────┘  │   │  ┌────────────────────────────┐ │
│                          │   │  │ Grafana                    │ │
│  ┌────────────────────┐  │   │  │ - Dashboards               │ │
│  │ Redis Cluster      │  │   │  │ - Visualization            │ │
│  │                    │  │   │  └────────────────────────────┘ │
│  │ Master + 2 Replicas│  │   │                                  │
│  │ Sentinel HA        │  │   │  ┌────────────────────────────┐ │
│  └────────────────────┘  │   │  │ Azure Log Analytics        │ │
│                          │   │  │ - Centralized Logging      │ │
│  ┌────────────────────┐  │   │  │ - Query & Search           │ │
│  │ ClickHouse         │  │   │  └────────────────────────────┘ │
│  │ (VM Scale Set)     │  │   │                                  │
│  │                    │  │   │  ┌────────────────────────────┐ │
│  │ Sharded by Region  │  │   │  │ Jaeger                     │ │
│  └────────────────────┘  │   │  │ - Distributed Tracing      │ │
│                          │   │  └────────────────────────────┘ │
└──────────────────────────┘   └──────────────────────────────────┘
```

---

## 2. 컴포넌트별 상세 설계

### 2.1 FleetFlag API Server (Rust/Axum)

#### 책임 (Responsibilities)
- Flag CRUD API 제공
- Flipt 엔진과의 통합
- 인증/인가 처리
- Audit Log 기록
- WebSocket Push Notification (Kill Switch)

#### 내부 구조
```
src/
├── main.rs                    # 애플리케이션 진입점
├── api/
│   ├── mod.rs
│   ├── flags.rs               # Flag CRUD endpoints
│   ├── evaluation.rs          # Flag evaluation API
│   ├── rules.rs               # Targeting rules API
│   ├── analytics.rs           # Analytics endpoints
│   ├── vehicles.rs            # Vehicle management
│   └── websocket.rs           # Kill Switch push notifications
├── services/
│   ├── mod.rs
│   ├── flag_service.rs        # 비즈니스 로직
│   ├── flipt_client.rs        # Flipt gRPC 클라이언트
│   ├── audit_service.rs       # Audit logging
│   └── notification_service.rs # Slack/Teams 알림
├── models/
│   ├── mod.rs
│   ├── flag.rs                # Flag 데이터 모델
│   ├── rule.rs                # Rule 데이터 모델
│   ├── vehicle.rs             # Vehicle 데이터 모델
│   └── user.rs                # User 데이터 모델
├── db/
│   ├── mod.rs
│   ├── postgres.rs            # PostgreSQL 커넥션 풀
│   ├── redis.rs               # Redis 커넥션 풀
│   └── migrations/            # DB 마이그레이션
├── middleware/
│   ├── mod.rs
│   ├── auth.rs                # JWT 인증
│   ├── rate_limit.rs          # Rate limiting
│   └── cors.rs                # CORS 처리
└── utils/
    ├── mod.rs
    ├── crypto.rs              # 암호화 유틸
    └── telemetry.rs           # OpenTelemetry 설정
```

#### 핵심 의존성
```toml
[dependencies]
# Web Framework
axum = "0.7"
tower = "0.4"
tower-http = "0.5"

# Async Runtime
tokio = { version = "1.35", features = ["full"] }

# gRPC (Flipt 통신용)
tonic = "0.11"
prost = "0.12"

# Database
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio-rustls", "uuid", "time"] }
redis = { version = "0.24", features = ["tokio-comp", "cluster-async"] }

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# Authentication
jsonwebtoken = "9.2"
argon2 = "0.5"

# Observability
tracing = "0.1"
tracing-subscriber = "0.3"
opentelemetry = "0.21"

# UUID & Time
uuid = { version = "1.6", features = ["v4", "serde"] }
time = "0.3"
```

---

### 2.2 Flipt Engine (Go - 커스터마이징)

#### 커스터마이징 포인트
1. **차량 VIN 기반 Sticky Assignment**
   - VIN 해시를 시드로 사용하여 일관된 분배
   - 동일 차량은 항상 동일 Variant 할당

2. **ASIL 등급 지원**
   - Flag에 Safety Level 필드 추가
   - ASIL-D Flag는 2단계 승인 workflow

3. **오프라인 캐시 TTL**
   - 기본 Flipt는 캐시 만료 개념 없음
   - TTL 필드 추가 (7일 기본값)

4. **Kill Switch Override**
   - 최우선 순위 규칙 타입 추가
   - 모든 타겟팅 무시하고 강제 비활성화

#### 통신 프로토콜 (gRPC)
```protobuf
syntax = "proto3";
package fleetflag.flipt.v1;

service EvaluationService {
  // 단일 Flag 평가
  rpc EvaluateFlag(EvaluateRequest) returns (EvaluateResponse);
  
  // 다중 Flag 평가 (배치)
  rpc BatchEvaluate(BatchEvaluateRequest) returns (BatchEvaluateResponse);
  
  // 차량 전체 Flag 동기화
  rpc SyncVehicleFlags(SyncRequest) returns (SyncResponse);
}

message EvaluateRequest {
  string flag_key = 1;
  string entity_id = 2;  // VIN
  map<string, string> context = 3;
}

message EvaluateResponse {
  bool match = 1;
  string variant_key = 2;
  map<string, string> variant_attachment = 3;
  string reason = 4;  // "TARGET_MATCH", "KILL_SWITCH", "DEFAULT"
}
```

---

### 2.3 Vehicle Agent (Rust Core)

#### 상태 머신 (State Machine)
```
┌──────────────────────────────────────────────────────────────┐
│                     Agent State Machine                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  INIT                                                        │
│   │                                                          │
│   ├─ Load local config                                      │
│   ├─ Initialize SQLite cache                                │
│   └─▶ CONNECTING                                            │
│                                                              │
│  CONNECTING                                                  │
│   │                                                          │
│   ├─ Establish gRPC connection (mTLS)                       │
│   ├─ Retry with exponential backoff (1s → 2s → 4s → 8s)    │
│   ├─ Max 5 attempts                                         │
│   │                                                          │
│   ├─ Success ──▶ SYNCING                                    │
│   └─ Failure ──▶ OFFLINE_MODE                               │
│                                                              │
│  SYNCING                                                     │
│   │                                                          │
│   ├─ Call SyncVehicleFlags(VIN)                             │
│   ├─ Store to SQLite (with TTL)                             │
│   ├─ Subscribe to WebSocket (Kill Switch channel)           │
│   └─▶ READY                                                 │
│                                                              │
│  READY (Normal Operation)                                   │
│   │                                                          │
│   ├─ Serve flag evaluations from local cache (< 0.1ms)     │
│   ├─ Background sync every 5 minutes                        │
│   ├─ Listen to WebSocket for real-time updates             │
│   │                                                          │
│   ├─ Network lost ──▶ OFFLINE_MODE                          │
│   └─ Kill Switch received ──▶ EMERGENCY_UPDATE             │
│                                                              │
│  OFFLINE_MODE                                                │
│   │                                                          │
│   ├─ Use local SQLite cache                                 │
│   ├─ Check cache TTL (7 days)                               │
│   ├─ If expired → fallback to default value                 │
│   ├─ Retry connection every 60s                             │
│   └─ Network restored ──▶ CONNECTING                        │
│                                                              │
│  EMERGENCY_UPDATE (Kill Switch)                             │
│   │                                                          │
│   ├─ Update local cache immediately                         │
│   ├─ Disable affected flags (< 3 seconds)                   │
│   ├─ Log event to local storage                             │
│   └─▶ READY                                                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

#### 내부 구조
```
agent/src/
├── lib.rs                     # 공개 API
├── client.rs                  # gRPC 클라이언트 (Tonic)
├── cache.rs                   # SQLite 로컬 캐시
├── evaluator.rs               # 로컬 평가 엔진
├── state_machine.rs           # 상태 머신 구현
├── sync.rs                    # 동기화 로직
├── websocket.rs               # WebSocket 클라이언트 (Kill Switch)
├── telemetry.rs               # 메트릭 수집
├── ffi/                       # Foreign Function Interface
│   ├── mod.rs
│   ├── cpp.rs                 # C++ 바인딩
│   └── python.rs              # Python 바인딩 (PyO3)
└── no_std/                    # RTOS 지원 (옵션)
    ├── mod.rs
    └── allocator.rs           # 커스텀 할당자
```

#### 메모리 최적화 전략
```rust
// 1. 고정 크기 버퍼 사용 (힙 할당 최소화)
const MAX_FLAGS: usize = 1024;
const MAX_RULES_PER_FLAG: usize = 16;

struct FlagCache {
    flags: [Option<Flag>; MAX_FLAGS],  // Stack allocation
    count: usize,
}

// 2. Copy-on-Write 패턴
struct Flag {
    key: Box<str>,           // Immutable, shared
    value: AtomicBool,       // Mutable, 1 byte
    ttl: AtomicU64,          // 8 bytes
}

// 3. Zero-copy serialization (SQLite)
// rusqlite의 FromSql trait 활용
impl FromSql for Flag {
    fn column_result(value: ValueRef<'_>) -> Result<Self> {
        // Borrow data from SQLite directly (no copy)
    }
}

// 총 메모리 사용량 추산:
// - Code: ~10MB
// - Data: ~50MB (1024 flags * ~50KB each)
// - Stack: ~2MB
// - SQLite cache: ~2MB (mmap)
// Total: ~64MB 이하
```

---

### 2.4 Dashboard (React + TypeScript)

#### 컴포넌트 구조
```
src/
├── main.tsx                   # 애플리케이션 진입점
├── App.tsx                    # 루트 컴포넌트
├── pages/
│   ├── Dashboard.tsx          # 홈 대시보드
│   ├── Flags/
│   │   ├── FlagList.tsx
│   │   ├── FlagDetail.tsx
│   │   └── FlagEditor.tsx
│   ├── Analytics.tsx
│   ├── Vehicles.tsx
│   ├── KillSwitch.tsx
│   └── Settings.tsx
├── components/
│   ├── layout/
│   │   ├── Header.tsx
│   │   ├── Sidebar.tsx
│   │   └── Footer.tsx
│   ├── flags/
│   │   ├── FlagCard.tsx
│   │   ├── RolloutProgress.tsx
│   │   └── TargetingRules.tsx
│   ├── charts/
│   │   ├── LineChart.tsx
│   │   └── BarChart.tsx
│   └── common/
│       ├── Button.tsx
│       ├── Modal.tsx
│       └── Table.tsx
├── api/
│   ├── client.ts              # Axios 설정
│   ├── flags.ts               # Flag API 호출
│   ├── analytics.ts
│   └── vehicles.ts
├── hooks/
│   ├── useFlags.ts            # React Query hooks
│   ├── useAnalytics.ts
│   └── useWebSocket.ts        # Kill Switch 실시간 수신
├── stores/
│   ├── authStore.ts           # Zustand 상태 관리
│   └── uiStore.ts
└── types/
    ├── flag.ts                # TypeScript 타입 정의
    ├── rule.ts
    └── vehicle.ts
```

#### 실시간 업데이트 (WebSocket)
```typescript
// useWebSocket.ts
import { useEffect } from 'react';
import { useQueryClient } from '@tanstack/react-query';

export function useWebSocket() {
  const queryClient = useQueryClient();

  useEffect(() => {
    const ws = new WebSocket('wss://fleetflag.azure.hyundai.com/ws');

    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);

      if (message.type === 'KILL_SWITCH') {
        // Kill Switch 이벤트 수신
        queryClient.invalidateQueries(['flags']);
        
        // Toast 알림 표시
        toast.error(`Kill Switch: ${message.flag_key} 비활성화됨`);
      }

      if (message.type === 'FLAG_UPDATE') {
        // Flag 업데이트 수신
        queryClient.setQueryData(['flags', message.flag_key], message.data);
      }
    };

    return () => ws.close();
  }, [queryClient]);
}
```

---

## 3. 데이터 플로우 (Data Flow)

### 3.1 Flag 생성 플로우
```
Admin (Browser)
  │
  ├─ 1. POST /api/v1/flags
  │    Body: { key: "new_feature", ... }
  │
  ▼
FleetFlag API Server
  │
  ├─ 2. Validate input (schema check)
  ├─ 3. Check ASIL level (if ASIL-D, require approval)
  ├─ 4. Insert to PostgreSQL
  │      INSERT INTO flags (id, key, name, ...) VALUES (...)
  │
  ├─ 5. Call Flipt gRPC
  │      CreateFlag(key, name, enabled, ...)
  │
  ├─ 6. Write Audit Log
  │      INSERT INTO audit_logs (user_id, action, target, ...)
  │
  ├─ 7. Invalidate Redis cache
  │      DEL cache:flag:new_feature
  │
  └─ 8. Return response
       { id: "uuid", key: "new_feature", ... }
```

### 3.2 차량 Flag 평가 플로우 (Fast Path)
```
Vehicle Application (C++ SDK)
  │
  ├─ 1. ff_client->IsEnabled("adas_lane_keep")
  │
  ▼
FleetFlag Agent (Rust)
  │
  ├─ 2. Check local SQLite cache
  │      SELECT * FROM flags WHERE key = 'adas_lane_keep'
  │
  ├─ 3. Check TTL (7 days)
  │      IF now - cached_at < 7 days
  │      THEN use cached value
  │      ELSE fallback to default
  │
  ├─ 4. Evaluate rules locally
  │      - Check Kill Switch (highest priority)
  │      - Check Percentage Rollout (VIN hash)
  │      - Check Vehicle Target
  │
  └─ 5. Return result (< 0.1ms)
       true / false

// 네트워크 호출 없음! (Fast Path)
// Latency: ~0.05ms (메모리/SQLite만 접근)
```

### 3.3 Kill Switch 플로우 (Emergency Path)
```
Admin (Browser)
  │
  ├─ 1. POST /api/v1/flags/{id}/kill-switch
  │    Body: { reason: "Safety issue", ... }
  │
  ▼
FleetFlag API Server
  │
  ├─ 2. Update flag status (enabled=false)
  │      UPDATE flags SET enabled=false WHERE id=...
  │
  ├─ 3. Publish to Redis Pub/Sub
  │      PUBLISH kill-switch:channel '{"flag_key": "adas_lane_keep", ...}'
  │
  ├─ 4. WebSocket broadcast to all connected vehicles
  │      For each ws_connection:
  │        ws.send({ type: "KILL_SWITCH", flag_key: "..." })
  │
  ├─ 5. Send Slack notification
  │      POST https://hooks.slack.com/...
  │      "🚨 Kill Switch: adas_lane_keep (영향 차량: 7,500대)"
  │
  └─ 6. Write Audit Log
       INSERT INTO audit_logs (action='KILL_SWITCH', ...)

Vehicle Agent (Rust)
  │
  ├─ 7. Receive WebSocket message
  │      ws.onmessage({ type: "KILL_SWITCH", ... })
  │
  ├─ 8. Update local SQLite cache
  │      UPDATE flags SET enabled=false WHERE key=...
  │
  ├─ 9. Notify all SDK clients
  │      For each registered_callback:
  │        callback.on_kill_switch(flag_key)
  │
  └─ 10. Log event locally
        Total time: < 3 seconds
```

### 3.4 A/B 실험 결과 수집 플로우
```
Vehicle Application
  │
  ├─ 1. Report metric
  │      client.reportMetric("lane_keep", "accuracy", 0.95)
  │
  ▼
FleetFlag Agent
  │
  ├─ 2. Buffer metrics (batch)
  │      metrics_queue.push({ flag: "lane_keep", metric: "accuracy", value: 0.95 })
  │
  ├─ 3. Send batch every 60 seconds
  │      POST /api/v1/metrics/batch
  │      Body: [{ ... }, { ... }]  (최대 1000개)
  │
  ▼
FleetFlag API Server
  │
  ├─ 4. Insert to ClickHouse (async)
  │      INSERT INTO metrics (timestamp, flag_key, metric_name, value, vin, variant, ...)
  │
  └─ 5. Trigger analytics pipeline
       - Calculate aggregates (hourly)
       - Update A/B test results
       - Alert if anomaly detected
```

---

## 4. 확장성 설계 (Scalability)

### 4.1 Horizontal Scaling

#### API Server
```yaml
# k8s/api-server-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fleetflag-api
spec:
  replicas: 3  # 초기 3개 pod
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: api
        image: fleetflag-api:v1.0
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 2000m
            memory: 2Gi
---
# Auto-scaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fleetflag-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fleetflag-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### 4.2 Database Sharding

#### PostgreSQL (Read Replicas)
```
Primary (Write)
  ├─ Replica 1 (Read - Region: NA)
  ├─ Replica 2 (Read - Region: EU)
  └─ Replica 3 (Read - Region: KR)

// 읽기 트래픽 분산
// 쓰기는 Primary에만
```

#### ClickHouse (By Region)
```
Shard 1: North America
Shard 2: Europe
Shard 3: Korea

// 지역별 데이터 격리
// 로컬 쿼리로 latency 감소
```

### 4.3 Redis Cluster (HA)
```
Master 1 (Write)
  └─ Replica 1-1 (Read)
  └─ Replica 1-2 (Read)

Master 2 (Write)
  └─ Replica 2-1 (Read)
  └─ Replica 2-2 (Read)

Sentinel (3 nodes)
  - Health check
  - Auto failover
```

---

## 5. 장애 복구 시나리오 (Failure Scenarios)

### 5.1 API Server 장애
```
시나리오: API Server pod 크래시

1. Kubernetes detects pod failure (liveness probe fail)
   ↓
2. Terminates old pod
   ↓
3. Spawns new pod (< 10 seconds)
   ↓
4. New pod becomes Ready
   ↓
5. Ingress routes traffic to new pod

영향: 없음 (다른 2개 pod가 처리)
복구 시간: ~10 seconds
```

### 5.2 PostgreSQL Primary 장애
```
시나리오: Primary 데이터베이스 노드 다운

1. Azure detects failure (health check)
   ↓
2. Promotes Replica to Primary (automatic)
   ↓
3. Updates connection string
   ↓
4. API Server reconnects to new Primary

영향: 쓰기 불가 (1-2분)
읽기는 계속 가능 (Replica 사용)
복구 시간: ~2 minutes
```

### 5.3 Redis Cluster 장애
```
시나리오: Redis Master 노드 다운

1. Sentinel detects failure (3-node quorum)
   ↓
2. Elects new Master from Replicas
   ↓
3. Reconfigures clients (automatic)

영향: 캐시 미스 증가 → DB 로드 상승
      (성능 저하 있으나 기능은 정상)
복구 시간: ~30 seconds
```

### 5.4 전체 클라우드 장애
```
시나리오: Azure 리전 전체 장애

1. Vehicle Agent detects connection failure
   ↓
2. Switches to OFFLINE_MODE
   ↓
3. Uses local SQLite cache (7-day TTL)
   ↓
4. Cloud restores
   ↓
5. Agent reconnects and syncs

영향: 차량은 캐시로 계속 동작 (7일간 안전)
새로운 Flag 변경은 반영 불가
복구 시간: Depends on Azure (차량은 영향 없음)
```

---

## 6. 보안 아키텍처 (Security Architecture)

### 6.1 인증 계층
```
┌────────────────────────────────────────────────────┐
│  Layer 1: Network (Azure Front Door)              │
│  - DDoS Protection                                 │
│  - IP Whitelisting (Tier1 suppliers)               │
└────────────────────────────────────────────────────┘
                     │
┌────────────────────▼───────────────────────────────┐
│  Layer 2: Application Gateway                      │
│  - WAF Rules (OWASP Top 10)                        │
│  - Rate Limiting (10K req/min per API key)         │
└────────────────────────────────────────────────────┘
                     │
┌────────────────────▼───────────────────────────────┐
│  Layer 3: API Server                               │
│  - JWT Validation (Admin users)                    │
│  - API Key Validation (Tier1 suppliers)            │
│  - RBAC (Role-Based Access Control)                │
└────────────────────────────────────────────────────┘
                     │
┌────────────────────▼───────────────────────────────┐
│  Layer 4: Vehicle Communication                    │
│  - mTLS (Mutual TLS)                               │
│  - VIN-based certificate                           │
│  - Certificate revocation (CRL)                    │
└────────────────────────────────────────────────────┘
```

### 6.2 데이터 암호화
```
┌──────────────────────────────────────────────────┐
│  At Rest:                                        │
│  - PostgreSQL: Transparent Data Encryption (TDE) │
│  - Redis: AOF/RDB encryption                     │
│  - ClickHouse: Disk encryption (Azure)           │
│  - SQLite (Vehicle): AES-256                     │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│  In Transit:                                     │
│  - TLS 1.3 (minimum)                             │
│  - mTLS for vehicle-cloud communication          │
│  - Certificate pinning (optional)                │
└──────────────────────────────────────────────────┘
```

---

## 다음 단계

시스템 아키텍처 상세 설계 완료! 이제:
1. **데이터베이스 스키마 설계** (PostgreSQL, SQLite)
2. **API 명세서 작성** (REST, gRPC)
3. **보안 설계 심화** (인증/인가 구체화)

어느 것부터 진행할까요?
