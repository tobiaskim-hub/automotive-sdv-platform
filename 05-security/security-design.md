# FleetFlag - 보안 설계서

**Security Design Document**

---

## 문서 정보

| 항목 | 내용 |
|------|------|
| 문서명 | FleetFlag 보안 설계서 |
| 버전 | v1.0 |
| 작성일 | 2026-02-01 |
| 작성자 | FleetFlag Team |
| 상태 | 🟡 Review |
| 분류 | CONFIDENTIAL |

---

## 목차

1. [보안 개요](#1-보안-개요)
2. [인증 및 인가](#2-인증-및-인가)
3. [암호화](#3-암호화)
4. [네트워크 보안](#4-네트워크-보안)
5. [취약점 관리](#5-취약점-관리)
6. [컴플라이언스](#6-컴플라이언스)
7. [사고 대응](#7-사고-대응)

---

## 1. 보안 개요

### 1.1 보안 목표

| 목표 | 설명 | 우선순위 |
|------|------|---------|
| **기밀성 (Confidentiality)** | 인가되지 않은 사용자의 데이터 접근 방지 | ⭐⭐⭐⭐⭐ |
| **무결성 (Integrity)** | 데이터 변조 방지 및 감지 | ⭐⭐⭐⭐⭐ |
| **가용성 (Availability)** | 서비스 중단 최소화 (99.9% 가용성) | ⭐⭐⭐⭐ |
| **추적성 (Traceability)** | 모든 작업 감사 로그 기록 (7년 보존) | ⭐⭐⭐⭐⭐ |

---

### 1.2 위협 모델 (Threat Model)

#### 1.2.1 위협 시나리오

| 위협 | 영향도 | 발생 가능성 | 대응 |
|------|--------|-----------|------|
| **차량 VIN Spoofing** | 🔴 Critical | 🟡 Medium | mTLS + VIN 인증서 |
| **API Key 유출** | 🔴 Critical | 🟡 Medium | Rate Limiting + IP 화이트리스트 |
| **SQL Injection** | 🔴 Critical | 🟢 Low | Parameterized Queries (sqlx) |
| **XSS (Admin Dashboard)** | 🟡 High | 🟢 Low | Content Security Policy |
| **DDoS 공격** | 🟠 Medium | 🟡 Medium | Azure DDoS Protection |
| **중간자 공격 (MITM)** | 🔴 Critical | 🟢 Low | TLS 1.3 + Certificate Pinning |
| **내부자 공격** | 🔴 Critical | 🟢 Low | RBAC + 감사 로그 |

---

### 1.3 보안 계층 아키텍처

```
┌──────────────────────────────────────────────────────────────┐
│  Layer 1: Network Security (Azure Front Door)                │
│  - DDoS Protection                                           │
│  - WAF (Web Application Firewall)                            │
│  - IP Whitelisting                                           │
└────────────────┬─────────────────────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────────────────────┐
│  Layer 2: Application Gateway                                │
│  - Rate Limiting (10K req/min per key)                       │
│  - JWT Validation                                            │
│  - API Key Validation                                        │
└────────────────┬─────────────────────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────────────────────┐
│  Layer 3: API Server (Rust/Axum)                             │
│  - Authentication Middleware                                 │
│  - Authorization (RBAC)                                      │
│  - Input Validation                                          │
└────────────────┬─────────────────────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────────────────────┐
│  Layer 4: Data Layer                                         │
│  - TDE (Transparent Data Encryption)                         │
│  - Encryption at Rest                                        │
│  - Backup Encryption                                         │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. 인증 및 인가

### 2.1 사용자 인증 (Admin Dashboard)

#### 2.1.1 JWT (JSON Web Token)

**알고리즘**: RS256 (RSA with SHA-256)

**Token 구조**:
```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "user-uuid",
    "email": "admin@hyundai.com",
    "role": "ADMIN",
    "department": "ADAS",
    "iat": 1735689600,
    "exp": 1735693200,  // 1시간 후 만료
    "jti": "token-uuid"
  },
  "signature": "..."
}
```

**키 관리**:
- **Private Key**: Azure Key Vault 저장 (RSA 2048-bit)
- **Public Key**: API Server에 배포 (검증용)
- **키 로테이션**: 90일마다 자동 갱신

---

#### 2.1.2 비밀번호 정책

**해싱 알고리즘**: Argon2id

```rust
use argon2::{
    password_hash::{PasswordHash, PasswordHasher, PasswordVerifier, SaltString},
    Argon2
};

// 비밀번호 해싱
let salt = SaltString::generate(&mut OsRng);
let argon2 = Argon2::default();
let password_hash = argon2.hash_password(password.as_bytes(), &salt)?
    .to_string();

// 비밀번호 검증
let parsed_hash = PasswordHash::new(&password_hash)?;
Argon2::default().verify_password(password.as_bytes(), &parsed_hash)?;
```

**파라미터**:
- **메모리**: 19 MiB
- **반복 횟수**: 2
- **병렬도**: 1

**정책**:
- 최소 길이: 12자
- 복잡도: 대소문자 + 숫자 + 특수문자
- 이전 비밀번호 3개 재사용 금지
- 90일마다 변경 권장

---

### 2.2 API Key 인증 (Tier 1 협력사)

#### 2.2.1 API Key 생성

```rust
use rand::Rng;
use sha2::{Sha256, Digest};

// API Key 생성 (192-bit random)
let random_bytes: [u8; 24] = rand::thread_rng().gen();
let api_key = format!(
    "ff_api_key_tier1_{}_{}",
    company_name,
    base64::encode_config(random_bytes, base64::URL_SAFE_NO_PAD)
);

// SHA-256 해시 저장 (원본 키는 DB에 저장하지 않음)
let mut hasher = Sha256::new();
hasher.update(api_key.as_bytes());
let key_hash = format!("{:x}", hasher.finalize());

// DB에 hash만 저장
sqlx::query!(
    "INSERT INTO api_keys (name, key_hash, key_prefix, role) VALUES ($1, $2, $3, $4)",
    company_name,
    key_hash,
    &api_key[..30], // 프리픽스만 저장 (로깅용)
    "READ"
).execute(&pool).await?;

// 생성된 키를 사용자에게 1회만 표시 (복구 불가)
println!("API Key (1회만 표시): {}", api_key);
```

**예시 키**:
```
ff_api_key_tier1_companyA_xY3bN9mK2pQrT7vL4jW8zH1cF6gS5uI0
```

---

#### 2.2.2 API Key 검증

```rust
// 요청 헤더에서 API Key 추출
let api_key = headers.get("X-API-Key")
    .and_then(|h| h.to_str().ok())
    .ok_or(AuthError::MissingApiKey)?;

// SHA-256 해시 계산
let mut hasher = Sha256::new();
hasher.update(api_key.as_bytes());
let key_hash = format!("{:x}", hasher.finalize());

// DB에서 검증
let api_key_record = sqlx::query!(
    r#"
    SELECT id, role, rate_limit_per_min, is_active, expires_at
    FROM api_keys
    WHERE key_hash = $1
    "#,
    key_hash
).fetch_optional(&pool).await?;

match api_key_record {
    Some(record) if record.is_active => {
        // Rate Limiting 체크
        check_rate_limit(&record.id, record.rate_limit_per_min).await?;
        
        // 마지막 사용 시간 업데이트
        update_last_used(&record.id).await?;
        
        Ok(ApiKeyAuth { role: record.role })
    }
    Some(_) => Err(AuthError::ApiKeyInactive),
    None => Err(AuthError::InvalidApiKey)
}
```

---

### 2.3 mTLS (Mutual TLS) - 차량 인증

#### 2.3.1 인증서 구조

```
Root CA (현대차 내부 CA)
 └─ Intermediate CA (FleetFlag Vehicles)
     └─ Vehicle Certificate (VIN: KMH1234567890ABCD)
         - Subject: CN=KMH1234567890ABCD, O=Hyundai Motor, OU=Vehicles
         - Serial Number: VIN
         - Validity: 5 years
         - Key Usage: Digital Signature, Key Encipherment
         - Extended Key Usage: Client Authentication
```

---

#### 2.3.2 mTLS Handshake

```
Vehicle                              FleetFlag Server
  |                                         |
  |-- Client Hello (TLS 1.3) -------------->|
  |                                         |
  |<-------------- Server Hello ------------|
  |<-------------- Server Certificate ------|
  |<-------------- Certificate Request -----|
  |                                         |
  |-- Client Certificate (VIN-based) ------>|
  |-- Certificate Verify ------------------>|
  |-- Finished --------------------------->|
  |                                         |
  |<-------------- Finished -----------------|
  |                                         |
  |====== Encrypted Application Data =======|
```

---

#### 2.3.3 서버 측 인증서 검증

```rust
use rustls::{ServerConfig, Certificate, PrivateKey};
use rustls::server::AllowAnyAuthenticatedClient;
use x509_parser::prelude::*;

// mTLS 설정
let mut config = ServerConfig::builder()
    .with_safe_defaults()
    .with_client_cert_verifier(
        AllowAnyAuthenticatedClient::new(root_cert_store)
    )
    .with_single_cert(server_cert, server_key)?;

// 클라이언트 인증서에서 VIN 추출
fn extract_vin_from_cert(cert: &Certificate) -> Result<String> {
    let (_, x509) = X509Certificate::from_der(&cert.0)?;
    let cn = x509.subject()
        .iter_common_name()
        .next()
        .and_then(|cn| cn.as_str().ok())
        .ok_or("No CN in certificate")?;
    
    // VIN 형식 검증 (17자리)
    if cn.len() == 17 && cn.chars().all(|c| c.is_ascii_alphanumeric()) {
        Ok(cn.to_string())
    } else {
        Err("Invalid VIN format".into())
    }
}
```

---

### 2.4 Role-Based Access Control (RBAC)

#### 2.4.1 역할 정의

| 역할 | 권한 | 대상 |
|------|------|------|
| **ADMIN** | 모든 작업 가능 | IT 관리자 |
| **WRITE** | Flag 생성/수정/삭제 (ASIL-D 제외) | SW 개발팀 |
| **READ** | Flag 조회, 평가만 가능 | QA, Tier 1 협력사 |
| **SAFETY** | ASIL-D Flag 승인 권한 | Safety 담당자 |

---

#### 2.4.2 권한 체크 미들웨어

```rust
use axum::{
    middleware::{self, Next},
    extract::State,
    http::{Request, StatusCode},
};

async fn require_role<B>(
    State(pool): State<PgPool>,
    req: Request<B>,
    next: Next<B>,
    required_role: UserRole,
) -> Result<Response, StatusCode> {
    // JWT에서 사용자 역할 추출
    let user_role = req.extensions()
        .get::<JwtClaims>()
        .map(|claims| claims.role)
        .ok_or(StatusCode::UNAUTHORIZED)?;
    
    // 권한 체크
    if user_role >= required_role {
        Ok(next.run(req).await)
    } else {
        Err(StatusCode::FORBIDDEN)
    }
}

// 라우터에 적용
let app = Router::new()
    .route("/flags", post(create_flag))
    .layer(middleware::from_fn_with_state(pool.clone(), |req, next| {
        require_role(req, next, UserRole::WRITE)
    }));
```

---

## 3. 암호화

### 3.1 전송 중 암호화 (In Transit)

#### 3.1.1 TLS 설정

**최소 버전**: TLS 1.3

**Cipher Suites** (우선순위순):
1. `TLS_AES_256_GCM_SHA384`
2. `TLS_AES_128_GCM_SHA256`
3. `TLS_CHACHA20_POLY1305_SHA256`

```rust
use rustls::{ServerConfig, version};

let config = ServerConfig::builder()
    .with_safe_default_cipher_suites()
    .with_safe_default_kx_groups()
    .with_protocol_versions(&[&version::TLS13])?
    .with_no_client_auth()
    .with_single_cert(certs, key)?;
```

---

#### 3.1.2 Certificate Pinning (차량)

```rust
// 서버 인증서 핀 (SHA-256 해시)
const SERVER_CERT_PIN: &str = "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=";

fn verify_server_cert(cert: &Certificate) -> Result<()> {
    let mut hasher = Sha256::new();
    hasher.update(&cert.0);
    let cert_hash = base64::encode(hasher.finalize());
    
    if cert_hash == SERVER_CERT_PIN {
        Ok(())
    } else {
        Err("Certificate pin mismatch".into())
    }
}
```

---

### 3.2 저장 시 암호화 (At Rest)

#### 3.2.1 데이터베이스 암호화

**PostgreSQL**: Transparent Data Encryption (TDE)
- Azure Managed 자동 활성화
- AES-256 암호화
- 키 관리: Azure Key Vault

**SQLite (차량)**: SQLite Encryption Extension (SEE)
```rust
use rusqlite::{Connection, OpenFlags};

// 암호화된 DB 열기
let conn = Connection::open_with_flags_and_vfs(
    "fleetflag.db",
    OpenFlags::SQLITE_OPEN_READ_WRITE | OpenFlags::SQLITE_OPEN_CREATE,
    "sqlcipher"
)?;

// 암호 설정 (AES-256)
conn.pragma_update(None, "key", &encryption_key)?;
conn.pragma_update(None, "cipher", "aes-256-cbc")?;
```

---

#### 3.2.2 백업 암호화

```bash
# PostgreSQL 백업 암호화 (GPG)
pg_dump fleetflag | gzip | gpg --encrypt --recipient backup@hyundai.com > backup.sql.gz.gpg

# 복구
gpg --decrypt backup.sql.gz.gpg | gunzip | psql fleetflag
```

---

### 3.3 키 관리 (Key Management)

#### 3.3.1 Azure Key Vault 통합

```rust
use azure_security_keyvault::prelude::*;

async fn get_encryption_key(key_name: &str) -> Result<Vec<u8>> {
    let credential = DefaultAzureCredential::default();
    let client = SecretClient::new(
        "https://fleetflag-keyvault.vault.azure.net",
        credential
    )?;
    
    let secret = client.get(key_name).await?;
    Ok(secret.value().as_bytes().to_vec())
}

// JWT Private Key 가져오기
let jwt_private_key = get_encryption_key("jwt-private-key").await?;

// DB 암호화 키 가져오기
let db_encryption_key = get_encryption_key("db-encryption-key").await?;
```

---

#### 3.3.2 키 로테이션

| 키 타입 | 로테이션 주기 | 방법 |
|---------|-------------|------|
| JWT Signing Key | 90일 | Azure Key Vault 자동 로테이션 |
| API Key | 1년 (수동) | 협력사와 협의 후 갱신 |
| mTLS 인증서 | 5년 | 차량 제조 시 발급 |
| DB 암호화 키 | 180일 | Azure TDE 자동 |

---

## 4. 네트워크 보안

### 4.1 DDoS 방어

**Azure DDoS Protection Standard**:
- L3/L4 자동 방어
- Always-on Traffic Monitoring
- Adaptive Real-time Tuning

**Rate Limiting** (Application Level):
```rust
use tower_governor::{Governor, GovernorConfigBuilder};

let governor_conf = Box::new(
    GovernorConfigBuilder::default()
        .per_second(100) // 초당 100 요청
        .burst_size(200) // 버스트 200
        .finish()
        .unwrap()
);

let app = Router::new()
    .route("/api/evaluate", post(evaluate_flag))
    .layer(Governor::new(governor_conf));
```

---

### 4.2 WAF (Web Application Firewall)

**Azure Application Gateway WAF**:
- OWASP Top 10 보호
- SQL Injection 차단
- XSS 차단
- Path Traversal 차단

**커스텀 규칙**:
```json
{
  "name": "BlockSuspiciousUserAgents",
  "priority": 100,
  "ruleType": "MatchRule",
  "matchConditions": [
    {
      "matchVariables": [{"variableName": "RequestHeaders", "selector": "User-Agent"}],
      "operator": "Regex",
      "matchValues": ["(curl|wget|python-requests)"],
      "negationCondition": false
    }
  ],
  "action": "Block"
}
```

---

### 4.3 IP 화이트리스트 (Tier 1 협력사)

```rust
use ipnet::IpNet;

// 허용된 IP 범위
const ALLOWED_IP_RANGES: &[&str] = &[
    "203.0.113.0/24",    // Tier1 CompanyA
    "198.51.100.0/24",   // Tier1 CompanyB
    "192.0.2.0/24",      // 현대차 내부망
];

async fn check_ip_whitelist(req: &Request) -> Result<(), StatusCode> {
    let client_ip = req.headers()
        .get("X-Forwarded-For")
        .and_then(|h| h.to_str().ok())
        .and_then(|s| s.split(',').next())
        .ok_or(StatusCode::FORBIDDEN)?;
    
    let ip: IpAddr = client_ip.parse()
        .map_err(|_| StatusCode::BAD_REQUEST)?;
    
    for range in ALLOWED_IP_RANGES {
        let net: IpNet = range.parse().unwrap();
        if net.contains(&ip) {
            return Ok(());
        }
    }
    
    Err(StatusCode::FORBIDDEN)
}
```

---

## 5. 취약점 관리

### 5.1 정적 분석 (SAST)

**Rust**:
```bash
# Clippy (린트)
cargo clippy --all-targets --all-features -- -D warnings

# Cargo Audit (의존성 취약점 스캔)
cargo audit

# Cargo Deny (라이선스 및 보안 정책)
cargo deny check
```

**TypeScript**:
```bash
# ESLint
npm run lint

# npm audit
npm audit --audit-level=moderate
```

---

### 5.2 동적 분석 (DAST)

**OWASP ZAP** (주 1회 자동 스캔):
```yaml
# .github/workflows/security-scan.yml
name: Security Scan
on:
  schedule:
    - cron: '0 2 * * 1'  # 매주 월요일 02:00

jobs:
  zap-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: ZAP Scan
        uses: zaproxy/action-full-scan@v0.4.0
        with:
          target: 'https://fleetflag-staging.azure.hyundai.com'
```

---

### 5.3 의존성 관리

**Dependabot** (GitHub):
```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "cargo"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    
  - package-ecosystem: "npm"
    directory: "/dashboard"
    schedule:
      interval: "weekly"
```

---

### 5.4 침투 테스트 (Penetration Testing)

**주기**: 분기별 (연 4회)

**범위**:
- Admin Dashboard (웹 애플리케이션)
- REST API (인증/인가)
- gRPC 엔드포인트 (차량 통신)
- Infrastructure (Azure 환경)

**도구**:
- Burp Suite Professional
- Metasploit
- Nmap

---

## 6. 컴플라이언스

### 6.1 ISO 26262 (자동차 안전 표준)

**요구사항**:
- [ ] ASIL-D Flag는 2단계 승인 필요
- [ ] 감사 로그 7년 보존
- [ ] Kill Switch < 3초 전파
- [ ] 오프라인 모드 안전한 Default Value

---

### 6.2 GDPR (EU 개인정보 보호법)

**요구사항**:
- [ ] 사용자 동의 (Consent)
- [ ] 데이터 접근 권한 (Right to Access)
- [ ] 데이터 삭제 권한 (Right to be Forgotten)
- [ ] 데이터 이동 권한 (Data Portability)
- [ ] 데이터 저장 위치: EU 내

**구현**:
```rust
// GDPR: 사용자 데이터 내보내기
async fn export_user_data(user_id: Uuid) -> Result<UserDataExport> {
    let flags = get_user_flags(user_id).await?;
    let audit_logs = get_user_audit_logs(user_id).await?;
    
    Ok(UserDataExport {
        user: get_user(user_id).await?,
        flags,
        audit_logs,
        exported_at: Utc::now(),
    })
}

// GDPR: 사용자 데이터 삭제
async fn delete_user_data(user_id: Uuid) -> Result<()> {
    // Soft delete (감사 로그는 7년 보존)
    sqlx::query!(
        "UPDATE users SET is_deleted = true, email = NULL, name = '[DELETED]' WHERE id = $1",
        user_id
    ).execute(&pool).await?;
    
    Ok(())
}
```

---

### 6.3 개인정보보호법 (한국)

**요구사항**:
- [ ] 개인정보 암호화 저장
- [ ] 접근 로그 기록
- [ ] 개인정보 보유 기간 명시
- [ ] 파기 절차 수립

---

## 7. 사고 대응

### 7.1 보안 사고 대응 절차 (Incident Response)

#### 단계별 프로세스

```
1. 탐지 (Detection)
   ├─ 알림 수신 (Azure Security Center)
   ├─ 로그 분석 (Log Analytics)
   └─ 이상 징후 확인
   
2. 분석 (Analysis)
   ├─ 영향 범위 파악
   ├─ 근본 원인 분석
   └─ 증거 수집 (포렌식)
   
3. 격리 (Containment)
   ├─ 침해 계정 비활성화
   ├─ 네트워크 차단
   └─ 서비스 격리
   
4. 제거 (Eradication)
   ├─ 악성 코드 제거
   ├─ 취약점 패치
   └─ 비밀번호/키 재발급
   
5. 복구 (Recovery)
   ├─ 시스템 복원
   ├─ 서비스 재개
   └─ 모니터링 강화
   
6. 사후 분석 (Post-Incident)
   ├─ 보고서 작성
   ├─ 재발 방지 대책
   └─ 프로세스 개선
```

---

### 7.2 연락처

**보안 사고 신고**:
- 이메일: security@fleetflag.dev
- Slack: #fleetflag-security
- 전화: +82-2-XXXX-XXXX (24/7)

**책임자**:
| 역할 | 담당자 | 연락처 |
|------|--------|--------|
| CISO | - | - |
| Security Engineer | - | - |
| Incident Response Lead | - | - |

---

## 8. 보안 체크리스트

### 8.1 개발 단계

- [ ] 모든 외부 입력 검증
- [ ] SQL Injection 방어 (Parameterized Queries)
- [ ] XSS 방어 (CSP, 입력 이스케이프)
- [ ] CSRF 방어 (CSRF Token)
- [ ] 민감 정보 로그 제외
- [ ] 에러 메시지에 민감 정보 포함 금지
- [ ] Secrets를 코드에 하드코딩 금지

---

### 8.2 배포 단계

- [ ] HTTPS/TLS 1.3 적용
- [ ] mTLS 설정 (차량)
- [ ] Rate Limiting 설정
- [ ] WAF 규칙 활성화
- [ ] DDoS Protection 활성화
- [ ] 암호화 키 Key Vault에 저장
- [ ] 감사 로그 활성화
- [ ] 백업 암호화 확인

---

### 8.3 운영 단계

- [ ] 취약점 스캔 (주 1회)
- [ ] 침투 테스트 (분기 1회)
- [ ] 감사 로그 검토 (주 1회)
- [ ] 키 로테이션 (90일마다)
- [ ] 의존성 업데이트 (주 1회)
- [ ] 보안 패치 적용 (7일 이내)
- [ ] 사고 대응 훈련 (반기 1회)

---

## 9. 변경 이력

| 버전 | 날짜 | 작성자 | 변경 내용 |
|------|------|--------|-----------|
| v1.0 | 2026-02-01 | FleetFlag Team | 초안 작성 |

---

**문서 끝**
