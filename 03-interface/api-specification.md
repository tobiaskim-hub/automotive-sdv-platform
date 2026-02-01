# FleetFlag - 시스템 인터페이스 명세서

**System Interface Specification**

---

## 문서 정보

| 항목 | 내용 |
|------|------|
| 문서명 | FleetFlag 시스템 인터페이스 명세서 |
| 버전 | v1.0 |
| 작성일 | 2026-02-01 |
| 작성자 | FleetFlag Team |
| 상태 | 🟡 Review |
| 분류 | CONFIDENTIAL |

---

## 목차

1. [REST API 명세](#1-rest-api-명세)
2. [gRPC Protocol 명세](#2-grpc-protocol-명세)
3. [SDK 인터페이스](#3-sdk-인터페이스)
4. [WebSocket 프로토콜](#4-websocket-프로토콜)
5. [에러 코드](#5-에러-코드)

---

## 1. REST API 명세

### 1.1 개요

- **Base URL**: `https://fleetflag.azure.hyundai.com/api/v1`
- **인증**: Bearer Token (JWT) 또는 API Key
- **Content-Type**: `application/json`
- **Rate Limit**: 10,000 req/min per API Key

### 1.2 인증 (Authentication)

#### POST /auth/login
**관리자 로그인**

**Request:**
```http
POST /api/v1/auth/login HTTP/1.1
Content-Type: application/json

{
  "email": "admin@hyundai.com",
  "password": "secure_password"
}
```

**Response:**
```json
{
  "token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires_in": 3600,
  "user": {
    "id": "uuid",
    "email": "admin@hyundai.com",
    "name": "김철수",
    "role": "ADMIN"
  }
}
```

**Status Codes:**
- `200 OK`: 로그인 성공
- `401 Unauthorized`: 잘못된 credentials
- `429 Too Many Requests`: Rate limit 초과

---

### 1.3 Feature Flags API

#### GET /flags
**Flag 목록 조회**

**Headers:**
```http
Authorization: Bearer {jwt_token}
```

**Query Parameters:**
- `environment` (string, optional): `development`, `staging`, `production`
- `enabled` (boolean, optional): `true`, `false`
- `page` (integer, optional): 페이지 번호 (default: 1)
- `limit` (integer, optional): 페이지 크기 (default: 20)
- `search` (string, optional): Flag key/name 검색

**Response:**
```json
{
  "data": [
    {
      "id": "uuid",
      "key": "adas_lane_keep_v2",
      "name": "ADAS Lane Keep Algorithm v2",
      "description": "차선 유지 알고리즘 개선 버전",
      "type": "boolean",
      "default_value": false,
      "enabled": true,
      "safety_level": "ASIL-D",
      "environment": "production",
      "rollout_percentage": 75,
      "created_at": "2026-01-15T10:00:00Z",
      "updated_at": "2026-02-01T10:23:00Z",
      "created_by": {
        "id": "uuid",
        "name": "김철수",
        "email": "kim@hyundai.com"
      }
    }
  ],
  "pagination": {
    "current_page": 1,
    "total_pages": 5,
    "total_items": 127,
    "items_per_page": 20
  }
}
```

---

#### GET /flags/{flag_id}
**Flag 상세 조회**

**Response:**
```json
{
  "id": "uuid",
  "key": "adas_lane_keep_v2",
  "name": "ADAS Lane Keep Algorithm v2",
  "description": "차선 유지 알고리즘 개선 버전",
  "type": "boolean",
  "default_value": false,
  "enabled": true,
  "safety_level": "ASIL-D",
  "environment": "production",
  "rules": [
    {
      "id": "rule-1",
      "type": "percentage_rollout",
      "value": 75,
      "priority": 100
    },
    {
      "id": "rule-2",
      "type": "vehicle_target",
      "conditions": {
        "models": ["Ioniq 5 Gen2", "Ioniq 6"],
        "sw_version": ">= 2.5.0",
        "regions": ["NA", "EU"]
      },
      "priority": 90
    }
  ],
  "statistics": {
    "total_evaluations": 2340000,
    "active_vehicles": 7500,
    "last_evaluation": "2026-02-01T12:00:00Z"
  },
  "created_at": "2026-01-15T10:00:00Z",
  "updated_at": "2026-02-01T10:23:00Z"
}
```

---

#### POST /flags
**Flag 생성**

**Request:**
```json
{
  "key": "battery_fast_charge_v3",
  "name": "Battery Fast Charge v3",
  "description": "고속 충전 최적화 알고리즘",
  "type": "boolean",
  "default_value": false,
  "safety_level": "Safety-Relevant",
  "environment": "development",
  "rules": [
    {
      "type": "percentage_rollout",
      "value": 10
    }
  ]
}
```

**Response:**
```json
{
  "id": "uuid",
  "key": "battery_fast_charge_v3",
  "name": "Battery Fast Charge v3",
  "created_at": "2026-02-01T13:00:00Z"
}
```

**Status Codes:**
- `201 Created`: Flag 생성 성공
- `400 Bad Request`: 유효하지 않은 입력
- `409 Conflict`: 동일한 key가 이미 존재

---

#### PATCH /flags/{flag_id}
**Flag 업데이트**

**Request:**
```json
{
  "enabled": false,
  "rules": [
    {
      "type": "percentage_rollout",
      "value": 25
    }
  ]
}
```

**Response:**
```json
{
  "id": "uuid",
  "key": "battery_fast_charge_v3",
  "updated_at": "2026-02-01T14:00:00Z"
}
```

---

#### DELETE /flags/{flag_id}
**Flag 삭제**

**Response:**
```json
{
  "message": "Flag deleted successfully"
}
```

**Status Codes:**
- `204 No Content`: 삭제 성공
- `404 Not Found`: Flag를 찾을 수 없음

---

#### POST /flags/{flag_id}/kill-switch
**Kill Switch 활성화**

**Request:**
```json
{
  "reason": "Safety issue: False positive in highway scenarios",
  "notify_slack": true
}
```

**Response:**
```json
{
  "flag_id": "uuid",
  "kill_switch_activated": true,
  "affected_vehicles": 7500,
  "notification_sent": true,
  "activated_at": "2026-02-01T10:15:00Z"
}
```

**Status Codes:**
- `200 OK`: Kill Switch 활성화 성공
- `403 Forbidden`: 권한 없음 (ASIL-D는 Safety 팀만 가능)

---

### 1.4 Evaluation API

#### POST /evaluate
**Flag 평가 (단일)**

**Headers:**
```http
X-API-Key: ff_api_key_tier1_companyA_...
```

**Request:**
```json
{
  "flag_key": "adas_lane_keep_v2",
  "context": {
    "vin": "KMH1234567890ABCD",
    "model": "Ioniq 5 Gen2",
    "sw_version": "UMOS 2.5.2",
    "region": "KR"
  }
}
```

**Response:**
```json
{
  "flag_key": "adas_lane_keep_v2",
  "enabled": true,
  "variant": "treatment",
  "reason": "PERCENTAGE_ROLLOUT",
  "evaluated_at": "2026-02-01T12:00:00Z"
}
```

---

#### POST /evaluate/batch
**Flag 평가 (배치)**

**Request:**
```json
{
  "flags": ["adas_lane_keep_v2", "battery_optimization", "new_ui_theme"],
  "context": {
    "vin": "KMH1234567890ABCD",
    "model": "Ioniq 5 Gen2",
    "sw_version": "UMOS 2.5.2"
  }
}
```

**Response:**
```json
{
  "evaluations": [
    {
      "flag_key": "adas_lane_keep_v2",
      "enabled": true,
      "variant": "treatment"
    },
    {
      "flag_key": "battery_optimization",
      "enabled": false,
      "variant": "control"
    },
    {
      "flag_key": "new_ui_theme",
      "enabled": true,
      "variant": "enabled"
    }
  ]
}
```

---

### 1.5 Analytics API

#### GET /analytics/flags/{flag_id}/metrics
**Flag 메트릭 조회**

**Query Parameters:**
- `start_date` (ISO 8601): 시작 날짜
- `end_date` (ISO 8601): 종료 날짜
- `granularity` (string): `hour`, `day`, `week`

**Response:**
```json
{
  "flag_key": "adas_lane_keep_v2",
  "metrics": {
    "total_evaluations": 2340000,
    "active_vehicles": 7500,
    "variants": {
      "treatment": {
        "vehicles": 5625,
        "evaluations": 1755000
      },
      "control": {
        "vehicles": 1875,
        "evaluations": 585000
      }
    },
    "experiment_results": {
      "primary_metric": {
        "name": "lane_departure_count",
        "control_value": 124,
        "treatment_value": 105,
        "improvement": -15.3,
        "p_value": 0.003,
        "significant": true
      }
    }
  },
  "time_series": [
    {
      "timestamp": "2026-02-01T00:00:00Z",
      "evaluations": 98000
    }
  ]
}
```

---

### 1.6 Vehicles API

#### GET /vehicles
**차량 목록 조회**

**Query Parameters:**
- `status` (string, optional): `online`, `offline`
- `model` (string, optional): 차량 모델
- `search` (string, optional): VIN 검색

**Response:**
```json
{
  "data": [
    {
      "vin": "KMH1234567890ABCD",
      "model": "Ioniq 5 Gen2",
      "sw_version": "UMOS 2.5.2",
      "region": "Seoul, KR",
      "status": "online",
      "last_seen": "2026-02-01T12:00:00Z",
      "active_flags": 12,
      "cache_expires_at": "2026-02-08T12:00:00Z"
    }
  ]
}
```

---

#### GET /vehicles/{vin}
**차량 상세 조회**

**Response:**
```json
{
  "vin": "KMH1234567890ABCD",
  "model": "Ioniq 5 Gen2",
  "sw_version": "UMOS 2.5.2",
  "region": "Seoul, KR",
  "status": "online",
  "last_seen": "2026-02-01T12:00:00Z",
  "flags": [
    {
      "flag_key": "adas_lane_keep_v2",
      "enabled": true,
      "variant": "treatment",
      "cached_at": "2026-02-01T11:55:00Z"
    }
  ],
  "telemetry": {
    "agent_version": "1.0.2",
    "memory_usage_mb": 58,
    "cpu_usage_percent": 1.5,
    "cache_size_mb": 2.1
  }
}
```

---

## 2. gRPC Protocol 명세

### 2.1 개요

- **프로토콜**: gRPC (Protocol Buffers)
- **포트**: 9000 (Control Plane), 9001 (Vehicle Plane)
- **인증**: mTLS (VIN-based certificate)

### 2.2 Proto Definition

```protobuf
syntax = "proto3";
package fleetflag.v1;

// =============================================================================
// FleetFlag Evaluation Service
// =============================================================================

service EvaluationService {
  // 단일 Flag 평가
  rpc EvaluateFlag(EvaluateRequest) returns (EvaluateResponse);
  
  // 다중 Flag 평가 (배치)
  rpc BatchEvaluate(BatchEvaluateRequest) returns (BatchEvaluateResponse);
  
  // 차량 전체 Flag 동기화 (초기 연결 시)
  rpc SyncVehicleFlags(SyncRequest) returns (SyncResponse);
  
  // Delta 동기화 (변경사항만)
  rpc DeltaSync(DeltaSyncRequest) returns (DeltaSyncResponse);
}

// -----------------------------------------------------------------------------
// EvaluateFlag
// -----------------------------------------------------------------------------

message EvaluateRequest {
  string flag_key = 1;           // Flag 고유 식별자
  string entity_id = 2;          // VIN
  map<string, string> context = 3; // 차량 속성 (model, sw_version, region)
}

message EvaluateResponse {
  bool match = 1;                // Flag 활성화 여부
  string variant_key = 2;        // Variant (control/treatment/enabled)
  map<string, string> variant_attachment = 3; // 추가 데이터
  string reason = 4;             // 평가 이유 (TARGET_MATCH, KILL_SWITCH, DEFAULT)
  int64 evaluated_at = 5;        // Unix timestamp (ms)
}

// -----------------------------------------------------------------------------
// BatchEvaluate
// -----------------------------------------------------------------------------

message BatchEvaluateRequest {
  repeated string flag_keys = 1;
  string entity_id = 2;
  map<string, string> context = 3;
}

message BatchEvaluateResponse {
  repeated EvaluateResponse evaluations = 1;
}

// -----------------------------------------------------------------------------
// SyncVehicleFlags
// -----------------------------------------------------------------------------

message SyncRequest {
  string vin = 1;
  string agent_version = 2;      // Agent 버전 (예: "1.0.2")
  int64 last_sync_at = 3;        // 마지막 동기화 시간 (Unix timestamp ms)
}

message SyncResponse {
  repeated FlagSnapshot flags = 1;
  int64 sync_timestamp = 2;
}

message FlagSnapshot {
  string key = 1;
  bool enabled = 2;
  string variant = 3;
  int64 ttl_seconds = 4;         // 캐시 TTL (기본: 604800 = 7일)
  map<string, string> metadata = 5;
}

// -----------------------------------------------------------------------------
// DeltaSync (변경사항만 전송)
// -----------------------------------------------------------------------------

message DeltaSyncRequest {
  string vin = 1;
  int64 last_sync_at = 2;        // Unix timestamp (ms)
}

message DeltaSyncResponse {
  repeated FlagChange changes = 1;
  int64 sync_timestamp = 2;
}

message FlagChange {
  string key = 1;
  ChangeType change_type = 2;
  FlagSnapshot snapshot = 3;     // change_type이 DELETE가 아닐 때만
}

enum ChangeType {
  CREATED = 0;
  UPDATED = 1;
  DELETED = 2;
  KILL_SWITCH = 3;               // 긴급 비활성화
}

// =============================================================================
// Telemetry Service (차량 → Cloud)
// =============================================================================

service TelemetryService {
  // 메트릭 배치 전송
  rpc ReportMetrics(MetricBatchRequest) returns (MetricBatchResponse);
  
  // Agent 상태 보고
  rpc ReportAgentHealth(HealthRequest) returns (HealthResponse);
}

message MetricBatchRequest {
  string vin = 1;
  repeated Metric metrics = 2;
}

message Metric {
  string flag_key = 1;
  string metric_name = 2;        // 예: "accuracy", "latency"
  double value = 3;
  int64 timestamp = 4;           // Unix timestamp (ms)
  map<string, string> tags = 5;  // 추가 메타데이터
}

message MetricBatchResponse {
  int32 accepted_count = 1;
  int32 rejected_count = 2;
}

message HealthRequest {
  string vin = 1;
  string agent_version = 2;
  AgentStatus status = 3;
  ResourceUsage resource_usage = 4;
}

message AgentStatus {
  enum State {
    INIT = 0;
    CONNECTING = 1;
    SYNCING = 2;
    READY = 3;
    OFFLINE_MODE = 4;
    ERROR = 5;
  }
  State state = 1;
  int64 uptime_seconds = 2;
  int64 last_sync_at = 3;
}

message ResourceUsage {
  uint64 memory_bytes = 1;       // RSS (Resident Set Size)
  float cpu_percent = 2;         // CPU 사용률 (0.0 ~ 100.0)
  uint64 cache_size_bytes = 3;   // SQLite 캐시 크기
}

message HealthResponse {
  bool acknowledged = 1;
}
```

---

## 3. SDK 인터페이스

### 3.1 C++ SDK (AUTOSAR Adaptive)

#### 초기화
```cpp
#include <fleetflag/client.h>

namespace fleetflag {

class Client {
public:
    // Factory method (Singleton 권장)
    static std::shared_ptr<Client> create(const std::string& config_path);
    
    // Flag 평가 (Boolean)
    bool isEnabled(const std::string& flag_key);
    
    // Flag 평가 (String Variant)
    std::string getVariant(const std::string& flag_key);
    
    // Flag 평가 (JSON)
    nlohmann::json getJsonValue(const std::string& flag_key);
    
    // Context 설정 (차량 속성)
    void setContext(const std::map<std::string, std::string>& context);
    
    // 메트릭 보고
    void reportMetric(const std::string& flag_key, 
                      const std::string& metric_name, 
                      double value);
    
    // Kill Switch 콜백 등록
    void onKillSwitch(std::function<void(const std::string&)> callback);
    
    // 동기화 강제 실행
    void forceSync();
    
    // 리소스 정리
    ~Client();
};

} // namespace fleetflag
```

#### 사용 예시
```cpp
#include <fleetflag/client.h>
#include <iostream>

int main() {
    // 1. Client 초기화
    auto client = fleetflag::Client::create("/etc/fleetflag/config.json");
    
    // 2. Context 설정 (차량 정보)
    client->setContext({
        {"vin", "KMH1234567890ABCD"},
        {"model", "Ioniq 5 Gen2"},
        {"sw_version", "UMOS 2.5.2"},
        {"region", "KR"}
    });
    
    // 3. Kill Switch 콜백 등록
    client->onKillSwitch([](const std::string& flag_key) {
        std::cerr << "🚨 Kill Switch: " << flag_key << " disabled!" << std::endl;
    });
    
    // 4. Flag 평가
    if (client->isEnabled("adas_lane_keep_v2")) {
        std::cout << "✅ Running new LKA algorithm" << std::endl;
        runNewLaneKeepAlgorithm();
    } else {
        std::cout << "⏸️  Running old LKA algorithm" << std::endl;
        runOldLaneKeepAlgorithm();
    }
    
    // 5. 메트릭 보고
    client->reportMetric("adas_lane_keep_v2", "accuracy", 0.95);
    
    return 0;
}
```

---

### 3.2 Python SDK (AI/ML 모듈)

#### 설치
```bash
pip install fleetflag-sdk
```

#### API
```python
from fleetflag import Client

# 1. Client 초기화
client = Client.from_config("/etc/fleetflag/config.json")

# 2. Context 설정
client.set_context({
    "vin": "KMH1234567890ABCD",
    "model": "Ioniq 5 Gen2",
    "sw_version": "UMOS 2.5.2"
})

# 3. Flag 평가 (Boolean)
if client.is_enabled("ml_object_detection_v3"):
    print("✅ Using new ML model")
    result = run_new_ml_model()
else:
    print("⏸️  Using old ML model")
    result = run_old_ml_model()

# 4. Variant 평가 (String)
model_variant = client.get_variant("ml_model_version")
if model_variant == "yolo_v8":
    load_yolo_v8_model()
elif model_variant == "efficientdet":
    load_efficientdet_model()

# 5. JSON 설정값
config = client.get_json("ml_hyperparameters")
# {
#   "learning_rate": 0.001,
#   "batch_size": 32,
#   "epochs": 100
# }

# 6. 메트릭 보고
client.report_metric("ml_object_detection_v3", "inference_time_ms", 45.2)
client.report_metric("ml_object_detection_v3", "accuracy", 0.92)

# 7. Kill Switch 콜백
@client.on_kill_switch
def handle_kill_switch(flag_key: str):
    print(f"🚨 Kill Switch: {flag_key} disabled!")
    rollback_to_safe_mode()
```

---

### 3.3 Rust SDK (42dot UMOS Core)

#### Cargo.toml
```toml
[dependencies]
fleetflag-sdk = "1.0"
```

#### API
```rust
use fleetflag_sdk::{Client, Context};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 1. Client 초기화
    let client = Client::from_config("/etc/fleetflag/config.json").await?;
    
    // 2. Context 설정
    let context = Context::builder()
        .vin("KMH1234567890ABCD")
        .model("Ioniq 5 Gen2")
        .sw_version("UMOS 2.5.2")
        .region("KR")
        .build();
    
    client.set_context(context).await?;
    
    // 3. Flag 평가
    if client.is_enabled("adas_lane_keep_v2").await? {
        println!("✅ Running new LKA algorithm");
        run_new_lane_keep_algorithm().await?;
    } else {
        println!("⏸️  Running old LKA algorithm");
        run_old_lane_keep_algorithm().await?;
    }
    
    // 4. 메트릭 보고
    client.report_metric("adas_lane_keep_v2", "accuracy", 0.95).await?;
    
    // 5. Kill Switch 콜백
    client.on_kill_switch(|flag_key| {
        eprintln!("🚨 Kill Switch: {} disabled!", flag_key);
    }).await;
    
    Ok(())
}
```

---

## 4. WebSocket 프로토콜

### 4.1 연결 (Connection)

**URL**: `wss://fleetflag.azure.hyundai.com/ws`

**Headers:**
```http
Authorization: Bearer {jwt_token}
Sec-WebSocket-Protocol: fleetflag-v1
```

### 4.2 메시지 포맷 (JSON)

#### Kill Switch 이벤트
```json
{
  "type": "KILL_SWITCH",
  "flag_key": "adas_lane_keep_v2",
  "timestamp": "2026-02-01T10:15:00Z",
  "reason": "Safety issue",
  "affected_vehicles": 7500
}
```

#### Flag 업데이트 이벤트
```json
{
  "type": "FLAG_UPDATE",
  "flag_key": "battery_optimization",
  "changes": {
    "enabled": false,
    "rollout_percentage": 25
  },
  "timestamp": "2026-02-01T14:00:00Z"
}
```

#### Heartbeat (Ping/Pong)
```json
{
  "type": "PING",
  "timestamp": "2026-02-01T12:00:00Z"
}
```

**Response:**
```json
{
  "type": "PONG",
  "timestamp": "2026-02-01T12:00:00Z"
}
```

---

## 5. 에러 코드

### 5.1 HTTP Status Codes

| 코드 | 의미 | 설명 |
|------|------|------|
| 200 | OK | 요청 성공 |
| 201 | Created | 리소스 생성 성공 |
| 204 | No Content | 삭제 성공 (응답 본문 없음) |
| 400 | Bad Request | 유효하지 않은 요청 |
| 401 | Unauthorized | 인증 실패 |
| 403 | Forbidden | 권한 없음 |
| 404 | Not Found | 리소스를 찾을 수 없음 |
| 409 | Conflict | 리소스 충돌 (중복 key) |
| 429 | Too Many Requests | Rate limit 초과 |
| 500 | Internal Server Error | 서버 오류 |
| 503 | Service Unavailable | 서비스 점검 중 |

### 5.2 에러 응답 포맷

```json
{
  "error": {
    "code": "FLAG_NOT_FOUND",
    "message": "Flag 'invalid_key' not found",
    "details": {
      "flag_key": "invalid_key"
    },
    "request_id": "req-uuid",
    "timestamp": "2026-02-01T12:00:00Z"
  }
}
```

### 5.3 에러 코드 목록

| 코드 | HTTP Status | 설명 |
|------|------------|------|
| `INVALID_REQUEST` | 400 | 요청 형식 오류 |
| `FLAG_NOT_FOUND` | 404 | Flag를 찾을 수 없음 |
| `FLAG_KEY_DUPLICATE` | 409 | Flag key 중복 |
| `UNAUTHORIZED` | 401 | 인증 실패 |
| `FORBIDDEN` | 403 | 권한 없음 |
| `RATE_LIMIT_EXCEEDED` | 429 | Rate limit 초과 |
| `INTERNAL_ERROR` | 500 | 서버 내부 오류 |
| `SERVICE_UNAVAILABLE` | 503 | 서비스 불가 |
| `INVALID_FLAG_TYPE` | 400 | 유효하지 않은 Flag 타입 |
| `INVALID_SAFETY_LEVEL` | 400 | 유효하지 않은 Safety Level |
| `APPROVAL_REQUIRED` | 403 | ASIL-D Flag는 승인 필요 |
| `VIN_NOT_FOUND` | 404 | 차량을 찾을 수 없음 |

---

## 6. OpenAPI 3.0 Specification

전체 OpenAPI 스펙은 별도 파일 참조: [openapi.yaml](./openapi.yaml)

**Swagger UI**: https://fleetflag.azure.hyundai.com/api/docs

---

## 7. 변경 이력

| 버전 | 날짜 | 작성자 | 변경 내용 |
|------|------|--------|-----------|
| v1.0 | 2026-02-01 | FleetFlag Team | 초안 작성 |

---

**문서 끝**
