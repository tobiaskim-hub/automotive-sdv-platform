# FleetFlag - 데이터베이스 스키마 설계

**Database Schema Design**

---

## 문서 정보

| 항목 | 내용 |
|------|------|
| 문서명 | FleetFlag 데이터베이스 스키마 설계 |
| 버전 | v1.0 |
| 작성일 | 2026-02-01 |
| 작성자 | FleetFlag Team |
| 상태 | 🟡 Review |
| 분류 | CONFIDENTIAL |

---

## 목차

1. [개요](#1-개요)
2. [PostgreSQL 스키마 (Control Plane)](#2-postgresql-스키마-control-plane)
3. [SQLite 스키마 (Vehicle Agent)](#3-sqlite-스키마-vehicle-agent)
4. [인덱스 전략](#4-인덱스-전략)
5. [파티셔닝 전략](#5-파티셔닝-전략)
6. [마이그레이션 계획](#6-마이그레이션-계획)

---

## 1. 개요

### 1.1 데이터베이스 구성

| 데이터베이스 | 용도 | 엔진 | 위치 |
|------------|------|------|------|
| PostgreSQL | 메타데이터 (Flags, Rules, Users, Audit) | PostgreSQL 15 | Azure Managed (Primary + 2 Replicas) |
| Redis | 캐시 (Hot Flags, Session) | Redis 7.0 | Azure Cache (Sentinel HA) |
| ClickHouse | 시계열 메트릭, A/B 실험 결과 | ClickHouse 23.8 | Azure VM Scale Set |
| SQLite | 차량 로컬 캐시 | SQLite 3.42 | 차량 Agent (64MB 제한) |

---

## 2. PostgreSQL 스키마 (Control Plane)

### 2.1 ER 다이어그램

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│    users     │       │    flags     │       │    rules     │
├──────────────┤       ├──────────────┤       ├──────────────┤
│ id (PK)      │       │ id (PK)      │       │ id (PK)      │
│ email        │       │ key (UNIQUE) │       │ flag_id (FK) │
│ name         │◀──────│ created_by   │       │ type         │
│ role         │       │ updated_by   │       │ priority     │
│ password     │       │ name         │───────│ conditions   │
│ created_at   │       │ description  │       │ value        │
│ updated_at   │       │ type         │       │ created_at   │
└──────────────┘       │ default_value│       └──────────────┘
                       │ enabled      │
                       │ safety_level │
                       │ environment  │
                       │ created_at   │
                       │ updated_at   │
                       └──────────────┘
                              │
                              │
                       ┌──────▼───────┐
                       │ audit_logs   │
                       ├──────────────┤
                       │ id (PK)      │
                       │ user_id (FK) │
                       │ flag_id (FK) │
                       │ action       │
                       │ before_value │
                       │ after_value  │
                       │ ip_address   │
                       │ user_agent   │
                       │ created_at   │
                       └──────────────┘

┌──────────────┐       ┌──────────────┐
│  vehicles    │       │ experiments  │
├──────────────┤       ├──────────────┤
│ vin (PK)     │       │ id (PK)      │
│ model        │       │ flag_id (FK) │
│ sw_version   │       │ name         │
│ region       │       │ status       │
│ status       │       │ start_date   │
│ last_seen    │       │ end_date     │
│ created_at   │       │ sample_size  │
│ updated_at   │       │ confidence   │
└──────────────┘       │ created_at   │
                       └──────────────┘
```

---

### 2.2 테이블 상세 정의

#### 2.2.1 users (사용자)

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    password_hash VARCHAR(255) NOT NULL, -- Argon2id
    role VARCHAR(20) NOT NULL CHECK (role IN ('ADMIN', 'WRITE', 'READ', 'SAFETY')),
    department VARCHAR(100),
    is_active BOOLEAN DEFAULT true,
    last_login_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 인덱스
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_is_active ON users(is_active);

-- 코멘트
COMMENT ON TABLE users IS '관리자 사용자 계정';
COMMENT ON COLUMN users.role IS 'ADMIN: 모든 권한, WRITE: Flag 생성/수정, READ: 조회만, SAFETY: ASIL-D 승인';
```

---

#### 2.2.2 flags (Feature Flags)

```sql
CREATE TABLE flags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key VARCHAR(100) UNIQUE NOT NULL, -- 예: "adas_lane_keep_v2"
    name VARCHAR(200) NOT NULL,
    description TEXT,
    type VARCHAR(20) NOT NULL CHECK (type IN ('boolean', 'string', 'number', 'json')),
    default_value JSONB NOT NULL, -- {"value": false}
    enabled BOOLEAN DEFAULT false,
    safety_level VARCHAR(20) NOT NULL CHECK (safety_level IN ('Non-Critical', 'Safety-Relevant', 'ASIL-B', 'ASIL-D')),
    environment VARCHAR(20) NOT NULL CHECK (environment IN ('development', 'staging', 'production')),
    
    -- 통계
    total_evaluations BIGINT DEFAULT 0,
    active_vehicles INTEGER DEFAULT 0,
    last_evaluation_at TIMESTAMP WITH TIME ZONE,
    
    -- 메타데이터
    created_by UUID REFERENCES users(id) ON DELETE SET NULL,
    updated_by UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- 삭제 대신 아카이브
    archived_at TIMESTAMP WITH TIME ZONE
);

-- 인덱스
CREATE INDEX idx_flags_key ON flags(key);
CREATE INDEX idx_flags_enabled ON flags(enabled);
CREATE INDEX idx_flags_safety_level ON flags(safety_level);
CREATE INDEX idx_flags_environment ON flags(environment);
CREATE INDEX idx_flags_created_at ON flags(created_at DESC);
CREATE INDEX idx_flags_archived_at ON flags(archived_at) WHERE archived_at IS NULL;

-- 복합 인덱스 (자주 쓰는 쿼리)
CREATE INDEX idx_flags_env_enabled ON flags(environment, enabled) WHERE archived_at IS NULL;

-- 전문 검색 (이름/설명)
CREATE INDEX idx_flags_search ON flags USING gin(to_tsvector('english', name || ' ' || COALESCE(description, '')));

-- 코멘트
COMMENT ON TABLE flags IS 'Feature Flag 메타데이터';
COMMENT ON COLUMN flags.key IS '고유 식별자 (immutable)';
COMMENT ON COLUMN flags.safety_level IS 'ISO 26262 안전 등급';
```

---

#### 2.2.3 rules (타겟팅 규칙)

```sql
CREATE TABLE rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_id UUID NOT NULL REFERENCES flags(id) ON DELETE CASCADE,
    type VARCHAR(50) NOT NULL CHECK (type IN (
        'percentage_rollout',
        'vehicle_whitelist',
        'vehicle_target',
        'user_attribute',
        'kill_switch'
    )),
    priority INTEGER NOT NULL DEFAULT 100, -- 높을수록 우선순위 높음
    enabled BOOLEAN DEFAULT true,
    
    -- 규칙 조건 (JSONB로 유연하게 저장)
    conditions JSONB NOT NULL, -- {"rollout_percentage": 75} 또는 {"models": ["Ioniq 5"], "sw_version": ">= 2.5.0"}
    value JSONB, -- Variant 값 (옵션)
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- 제약: 동일 Flag에 동일 type+priority는 불가
    CONSTRAINT unique_flag_type_priority UNIQUE(flag_id, type, priority)
);

-- 인덱스
CREATE INDEX idx_rules_flag_id ON rules(flag_id);
CREATE INDEX idx_rules_type ON rules(type);
CREATE INDEX idx_rules_priority ON rules(priority DESC);
CREATE INDEX idx_rules_enabled ON rules(enabled);

-- JSONB GIN 인덱스 (조건 검색 최적화)
CREATE INDEX idx_rules_conditions ON rules USING gin(conditions);

-- 코멘트
COMMENT ON TABLE rules IS 'Feature Flag 타겟팅 규칙';
COMMENT ON COLUMN rules.priority IS '높을수록 먼저 평가 (First Match Wins)';
COMMENT ON COLUMN rules.conditions IS 'JSON 형식의 규칙 조건';
```

**conditions 예시:**

```json
// percentage_rollout
{
  "rollout_percentage": 75,
  "sticky": true
}

// vehicle_target
{
  "models": ["Ioniq 5 Gen2", "Ioniq 6"],
  "sw_version": ">= 2.5.0",
  "regions": ["NA", "EU"]
}

// vehicle_whitelist
{
  "vins": ["KMH1234567890ABCD", "KMH9876543210ZYXW"]
}

// kill_switch
{
  "force_disabled": true,
  "reason": "Safety issue"
}
```

---

#### 2.2.4 audit_logs (감사 로그)

```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    flag_id UUID REFERENCES flags(id) ON DELETE SET NULL,
    action VARCHAR(50) NOT NULL CHECK (action IN (
        'CREATE_FLAG', 'UPDATE_FLAG', 'DELETE_FLAG', 'ARCHIVE_FLAG',
        'CREATE_RULE', 'UPDATE_RULE', 'DELETE_RULE',
        'KILL_SWITCH', 'ENABLE_FLAG', 'DISABLE_FLAG'
    )),
    
    -- 변경 전/후 값 (JSONB로 저장)
    before_value JSONB,
    after_value JSONB,
    
    -- 요청 메타데이터
    ip_address INET,
    user_agent TEXT,
    request_id UUID, -- 추적용
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 인덱스
CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_flag_id ON audit_logs(flag_id);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at DESC);

-- 파티셔닝 (월별) - ISO 26262 요구사항: 7년 보존
CREATE TABLE audit_logs_2026_01 PARTITION OF audit_logs
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- 코멘트
COMMENT ON TABLE audit_logs IS '모든 Flag 변경 이력 (ISO 26262: 7년 보존)';
COMMENT ON COLUMN audit_logs.before_value IS '변경 전 값 (JSON)';
COMMENT ON COLUMN audit_logs.after_value IS '변경 후 값 (JSON)';
```

---

#### 2.2.5 vehicles (차량)

```sql
CREATE TABLE vehicles (
    vin VARCHAR(17) PRIMARY KEY CHECK (vin ~ '^[A-HJ-NPR-Z0-9]{17}$'), -- VIN 형식 검증
    model VARCHAR(100) NOT NULL,
    sw_version VARCHAR(50) NOT NULL,
    region VARCHAR(10) NOT NULL,
    
    -- 상태
    status VARCHAR(20) NOT NULL CHECK (status IN ('online', 'offline', 'maintenance')),
    last_seen_at TIMESTAMP WITH TIME ZONE,
    
    -- Agent 정보
    agent_version VARCHAR(20),
    agent_memory_mb INTEGER,
    agent_cpu_percent NUMERIC(5,2),
    cache_size_mb NUMERIC(8,2),
    
    -- 메타데이터
    registered_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 인덱스
CREATE INDEX idx_vehicles_model ON vehicles(model);
CREATE INDEX idx_vehicles_sw_version ON vehicles(sw_version);
CREATE INDEX idx_vehicles_region ON vehicles(region);
CREATE INDEX idx_vehicles_status ON vehicles(status);
CREATE INDEX idx_vehicles_last_seen_at ON vehicles(last_seen_at DESC);

-- 코멘트
COMMENT ON TABLE vehicles IS '차량 정보 및 상태';
COMMENT ON COLUMN vehicles.vin IS 'Vehicle Identification Number (17자리)';
```

---

#### 2.2.6 experiments (A/B 실험)

```sql
CREATE TABLE experiments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_id UUID NOT NULL REFERENCES flags(id) ON DELETE CASCADE,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    
    -- 실험 상태
    status VARCHAR(20) NOT NULL CHECK (status IN ('draft', 'running', 'completed', 'cancelled')),
    
    -- 실험 기간
    start_date TIMESTAMP WITH TIME ZONE NOT NULL,
    end_date TIMESTAMP WITH TIME ZONE,
    
    -- 통계 설정
    sample_size INTEGER NOT NULL,
    confidence_level NUMERIC(3,2) DEFAULT 0.95 CHECK (confidence_level BETWEEN 0 AND 1),
    
    -- Primary Metric
    primary_metric VARCHAR(100) NOT NULL,
    
    -- 결과 (실험 완료 후)
    winner_variant VARCHAR(50), -- 'control' or 'treatment'
    p_value NUMERIC(10,8),
    improvement_percent NUMERIC(8,4),
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 인덱스
CREATE INDEX idx_experiments_flag_id ON experiments(flag_id);
CREATE INDEX idx_experiments_status ON experiments(status);
CREATE INDEX idx_experiments_start_date ON experiments(start_date DESC);

-- 코멘트
COMMENT ON TABLE experiments IS 'A/B 실험 메타데이터';
COMMENT ON COLUMN experiments.confidence_level IS '신뢰 수준 (기본: 95%)';
```

---

#### 2.2.7 api_keys (Tier 1 협력사 API Key)

```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL, -- 예: "Tier1_CompanyA"
    key_hash VARCHAR(255) UNIQUE NOT NULL, -- SHA-256 해시
    key_prefix VARCHAR(20) NOT NULL, -- 예: "ff_api_key_tier1_companyA"
    
    -- 권한
    role VARCHAR(20) NOT NULL CHECK (role IN ('READ', 'WRITE', 'ADMIN')),
    
    -- Rate Limiting
    rate_limit_per_min INTEGER DEFAULT 10000,
    
    -- 상태
    is_active BOOLEAN DEFAULT true,
    last_used_at TIMESTAMP WITH TIME ZONE,
    
    -- 만료
    expires_at TIMESTAMP WITH TIME ZONE,
    
    created_by UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 인덱스
CREATE INDEX idx_api_keys_key_hash ON api_keys(key_hash);
CREATE INDEX idx_api_keys_is_active ON api_keys(is_active);
CREATE INDEX idx_api_keys_expires_at ON api_keys(expires_at);

-- 코멘트
COMMENT ON TABLE api_keys IS 'Tier 1 협력사 API Key';
COMMENT ON COLUMN api_keys.key_hash IS 'SHA-256 해시 (원본 키는 저장하지 않음)';
```

---

## 3. SQLite 스키마 (Vehicle Agent)

### 3.1 개요

차량 Agent는 오프라인 모드를 위해 로컬 SQLite 캐시를 사용합니다.

**제약사항:**
- 총 DB 크기: < 2MB
- 메모리 사용량: < 10MB (mmap 포함)
- Flag 개수: 최대 100개 (차량당)

---

### 3.2 테이블 정의

```sql
-- flags_cache (Flag 캐시)
CREATE TABLE flags_cache (
    key TEXT PRIMARY KEY NOT NULL,
    enabled INTEGER NOT NULL, -- 0 or 1 (Boolean)
    variant TEXT NOT NULL, -- 'control', 'treatment', 'enabled'
    default_value TEXT NOT NULL, -- JSON string
    
    -- TTL (Time To Live)
    cached_at INTEGER NOT NULL, -- Unix timestamp (seconds)
    ttl_seconds INTEGER NOT NULL DEFAULT 604800, -- 7일 = 604800초
    
    -- 메타데이터
    flag_type TEXT NOT NULL, -- 'boolean', 'string', 'number', 'json'
    metadata TEXT -- JSON string (옵션)
);

-- 인덱스
CREATE INDEX idx_flags_cache_enabled ON flags_cache(enabled);
CREATE INDEX idx_flags_cache_cached_at ON flags_cache(cached_at);

-- agent_state (Agent 상태)
CREATE TABLE agent_state (
    key TEXT PRIMARY KEY NOT NULL,
    value TEXT NOT NULL,
    updated_at INTEGER NOT NULL -- Unix timestamp (seconds)
);

-- 초기 상태 값
INSERT INTO agent_state (key, value, updated_at) VALUES
    ('last_sync_at', '0', 0),
    ('sync_count', '0', 0),
    ('agent_version', '1.0.0', 0);

-- metrics_queue (메트릭 대기열)
CREATE TABLE metrics_queue (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    flag_key TEXT NOT NULL,
    metric_name TEXT NOT NULL,
    value REAL NOT NULL,
    timestamp INTEGER NOT NULL, -- Unix timestamp (seconds)
    tags TEXT, -- JSON string (옵션)
    sent INTEGER NOT NULL DEFAULT 0 -- 0: 대기, 1: 전송 완료
);

-- 인덱스
CREATE INDEX idx_metrics_queue_sent ON metrics_queue(sent);
CREATE INDEX idx_metrics_queue_timestamp ON metrics_queue(timestamp);

-- 코멘트 (SQLite는 COMMENT 미지원, 대신 주석으로 작성)
-- flags_cache: Flag 로컬 캐시 (TTL 7일)
-- agent_state: Agent 내부 상태 (last_sync_at 등)
-- metrics_queue: 전송 대기 중인 메트릭 (오프라인 시 버퍼링)
```

---

### 3.3 SQLite 최적화 설정

```sql
-- WAL 모드 (Write-Ahead Logging)
PRAGMA journal_mode = WAL;

-- 메모리 맵 크기 (10MB)
PRAGMA mmap_size = 10485760;

-- 캐시 크기 (2MB)
PRAGMA cache_size = -2000;

-- 동기화 모드 (성능 우선)
PRAGMA synchronous = NORMAL;

-- Auto-vacuum
PRAGMA auto_vacuum = INCREMENTAL;
```

---

## 4. 인덱스 전략

### 4.1 PostgreSQL 인덱스

#### 자주 사용하는 쿼리 패턴

1. **Flag 목록 조회 (환경별, 활성화 상태별)**
   ```sql
   SELECT * FROM flags 
   WHERE environment = 'production' 
     AND enabled = true 
     AND archived_at IS NULL;
   ```
   → 인덱스: `idx_flags_env_enabled`

2. **Flag 검색 (이름/설명)**
   ```sql
   SELECT * FROM flags 
   WHERE to_tsvector('english', name || ' ' || description) @@ to_tsquery('lane keep');
   ```
   → 인덱스: `idx_flags_search` (GIN)

3. **타겟팅 규칙 조회 (Flag별, 우선순위순)**
   ```sql
   SELECT * FROM rules 
   WHERE flag_id = 'uuid' 
     AND enabled = true 
   ORDER BY priority DESC;
   ```
   → 인덱스: `idx_rules_flag_id`, `idx_rules_priority`

4. **감사 로그 조회 (사용자별, 최근순)**
   ```sql
   SELECT * FROM audit_logs 
   WHERE user_id = 'uuid' 
   ORDER BY created_at DESC 
   LIMIT 100;
   ```
   → 인덱스: `idx_audit_logs_user_id`, `idx_audit_logs_created_at`

---

### 4.2 인덱스 모니터링

```sql
-- 사용되지 않는 인덱스 찾기
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;

-- 중복 인덱스 찾기
SELECT 
    pg_size_pretty(SUM(pg_relation_size(idx))::BIGINT) AS size,
    (array_agg(idx))[1] AS idx1,
    (array_agg(idx))[2] AS idx2,
    (array_agg(idx))[3] AS idx3,
    (array_agg(idx))[4] AS idx4
FROM (
    SELECT 
        indexrelid::regclass AS idx,
        (indrelid::text ||E'\n'|| indclass::text ||E'\n'|| 
         indkey::text ||E'\n'|| COALESCE(indexprs::text,'')||E'\n'|| 
         COALESCE(indpred::text,'')) AS key
    FROM pg_index
) sub
GROUP BY key 
HAVING COUNT(*) > 1
ORDER BY SUM(pg_relation_size(idx)) DESC;
```

---

## 5. 파티셔닝 전략

### 5.1 audit_logs 월별 파티셔닝

ISO 26262 요구사항: 7년 보존

```sql
-- 부모 테이블 (이미 위에서 정의)
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- ... (필드 생략)
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- 2026년 파티션 생성 (자동화 스크립트)
CREATE TABLE audit_logs_2026_01 PARTITION OF audit_logs
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE audit_logs_2026_02 PARTITION OF audit_logs
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- ... (12개월)

-- 2027년 파티션 생성
CREATE TABLE audit_logs_2027_01 PARTITION OF audit_logs
    FOR VALUES FROM ('2027-01-01') TO ('2027-02-01');

-- ... (반복)
```

---

### 5.2 자동 파티션 생성 (PL/pgSQL)

```sql
CREATE OR REPLACE FUNCTION create_audit_log_partition()
RETURNS void AS $$
DECLARE
    partition_date DATE;
    partition_name TEXT;
    start_date TEXT;
    end_date TEXT;
BEGIN
    -- 다음 달 파티션 생성
    partition_date := date_trunc('month', NOW() + INTERVAL '1 month');
    partition_name := 'audit_logs_' || to_char(partition_date, 'YYYY_MM');
    start_date := partition_date::TEXT;
    end_date := (partition_date + INTERVAL '1 month')::TEXT;
    
    -- 파티션 존재 여부 확인
    IF NOT EXISTS (
        SELECT 1 FROM pg_tables WHERE tablename = partition_name
    ) THEN
        EXECUTE format(
            'CREATE TABLE %I PARTITION OF audit_logs FOR VALUES FROM (%L) TO (%L)',
            partition_name, start_date, end_date
        );
        RAISE NOTICE 'Created partition: %', partition_name;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Cron Job으로 매달 실행 (예: pg_cron)
SELECT cron.schedule('create-audit-partition', '0 0 1 * *', 'SELECT create_audit_log_partition()');
```

---

## 6. 마이그레이션 계획

### 6.1 마이그레이션 도구

**선택**: [sqlx-cli](https://github.com/launchbadge/sqlx/tree/main/sqlx-cli) (Rust 기반)

```bash
# 설치
cargo install sqlx-cli --no-default-features --features postgres

# 마이그레이션 생성
sqlx migrate add create_users_table

# 마이그레이션 실행
sqlx migrate run --database-url postgres://user:pass@localhost/fleetflag

# 롤백
sqlx migrate revert --database-url postgres://user:pass@localhost/fleetflag
```

---

### 6.2 마이그레이션 파일 구조

```
migrations/
├── 20260201_001_create_users_table.sql
├── 20260201_002_create_flags_table.sql
├── 20260201_003_create_rules_table.sql
├── 20260201_004_create_audit_logs_table.sql
├── 20260201_005_create_vehicles_table.sql
├── 20260201_006_create_experiments_table.sql
├── 20260201_007_create_api_keys_table.sql
├── 20260202_008_add_indexes.sql
└── 20260203_009_setup_partitioning.sql
```

---

### 6.3 예시: 첫 번째 마이그레이션

**파일**: `migrations/20260201_001_create_users_table.sql`

```sql
-- Up Migration
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) NOT NULL CHECK (role IN ('ADMIN', 'WRITE', 'READ', 'SAFETY')),
    department VARCHAR(100),
    is_active BOOLEAN DEFAULT true,
    last_login_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_is_active ON users(is_active);

COMMENT ON TABLE users IS '관리자 사용자 계정';

-- 초기 관리자 계정 생성 (Argon2id 해시)
INSERT INTO users (email, name, password_hash, role, department) VALUES
    ('admin@hyundai.com', 'System Admin', '$argon2id$v=19$m=19456,t=2,p=1$...', 'ADMIN', 'IT');
```

---

## 7. 백업 및 복구 전략

### 7.1 백업 계획

| 데이터베이스 | 백업 주기 | 보존 기간 | 방법 |
|------------|---------|----------|------|
| PostgreSQL | 매일 (자동) | 30일 | Azure Automated Backup |
| PostgreSQL (수동) | 주 1회 | 1년 | pg_dump (PITR 포함) |
| ClickHouse | 주 1회 | 90일 | clickhouse-backup |
| SQLite (차량) | 동기화 시 | 7일 | Cloud 동기화 |

---

### 7.2 복구 절차

#### PostgreSQL Point-in-Time Recovery (PITR)

```bash
# 1. Azure Portal에서 복구 시점 선택
# 2. 새 서버 인스턴스로 복구
# 3. 연결 문자열 업데이트

# 또는 수동 복구
pg_restore -d fleetflag -v fleetflag_backup.dump
```

---

## 8. 성능 모니터링 쿼리

### 8.1 느린 쿼리 찾기

```sql
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    max_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

### 8.2 테이블 크기 모니터링

```sql
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS index_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

---

## 9. 변경 이력

| 버전 | 날짜 | 작성자 | 변경 내용 |
|------|------|--------|-----------|
| v1.0 | 2026-02-01 | FleetFlag Team | 초안 작성 |

---

**문서 끝**
