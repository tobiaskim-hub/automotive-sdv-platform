# Physical Deployment Architecture

## 문서 정보
- **작성일**: 2026-02-01
- **버전**: v1.0
- **상태**: 🟡 Review
- **관련 문서**: [System Architecture](./system-architecture.md), [Security Design](../05-security/security-design.md)

---

## 목차
1. [개요](#개요)
2. [온프레미스 IDC 물리 구성](#온프레미스-idc-물리-구성)
3. [차량 내부 물리 아키텍처](#차량-내부-물리-아키텍처)
4. [네트워크 토폴로지](#네트워크-토폴로지)
5. [하드웨어 사양](#하드웨어-사양)
6. [전력 및 냉각 요구사항](#전력-및-냉각-요구사항)
7. [배포 전략](#배포-전략)

---

## 개요

### 물리적 배포 원칙
FleetFlag 시스템은 **온프레미스 IDC**와 **차량 내부** 두 물리적 환경에서 동작합니다.

#### 핵심 설계 원칙
1. **보안 우선 (Security First)**
   - 온프레미스 IDC 내 모든 데이터 저장
   - 외부 클라우드 미사용 (On-Prem Only)
   - 다층 방화벽 및 네트워크 격리

2. **고가용성 (High Availability)**
   - Active-Standby 이중화
   - 자동 장애 조치 (Failover)
   - 99.9% 가용성 목표

3. **확장성 (Scalability)**
   - Phase 1: 10,000대 → Phase 3: 100,000대
   - 수평 확장 가능한 아키텍처
   - 모듈형 랙 구성

4. **차량 오프라인 독립성**
   - 7일 로컬 캐시 (SQLite)
   - IDC 연결 없이도 Feature Flag 평가 가능
   - Safety Gatekeeper 독립 동작

---

## 온프레미스 IDC 물리 구성

### IDC 위치 및 사양
- **위치**: 현대자동차 본사 IDC (서울 강남구)
- **면적**: 200㎡ (Data Center Tier 3)
- **전력**: 최대 500kW
- **냉각**: 정밀 공조 시스템 (20±2°C, 50±5% RH)
- **화재**: FM-200 가스 소화 시스템

### 랙 배치도

```
┌────────────────────────────────────────────────────────────┐
│                   Hyundai On-Prem IDC                       │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  Rack 1     │  │  Rack 2     │  │  Rack 3     │        │
│  │  Frontend   │  │  Backend    │  │  Data       │        │
│  │  Zone       │  │  Zone       │  │  Zone       │        │
│  │             │  │             │  │             │        │
│  │  [LB-01]    │  │  [API-01]   │  │  [PG-01]    │  ◄─┐  │
│  │  Active     │  │  FleetFlag  │  │  PostgreSQL │    │  │
│  │             │  │             │  │  Primary    │    │  │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤    │  │
│  │  [LB-02]    │  │  [API-02]   │  │  [PG-02]    │    │  │
│  │  Standby    │  │  FleetFlag  │  │  PostgreSQL │    │  │
│  │             │  │             │  │  Standby    │    │  │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤    │  │
│  │  [GW-01]    │  │  [FLIPT-01] │  │  [REDIS-01] │    │  │
│  │  API        │  │  Flipt      │  │  Redis      │    │  │
│  │  Gateway    │  │  Engine     │  │  Cluster    │    │  │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤    │  │
│  │  [GW-02]    │  │  [FLIPT-02] │  │  [CH-01]    │    │  │
│  │  API        │  │  Flipt      │  │  ClickHouse │    │  │
│  │  Gateway    │  │  Engine     │  │  Analytics  │    │  │
│  └─────────────┘  └─────────────┘  └─────────────┘    │  │
│                                                         │  │
│  ┌─────────────────────────────────────────────────────┘  │
│  │                                                          │
│  │  ┌─────────────┐  ┌─────────────┐                      │
│  │  │  Network    │  │  Security   │                      │
│  │  │  Switch     │  │  Firewall   │                      │
│  │  │  (Core)     │  │  (NGFW)     │                      │
│  │  └─────────────┘  └─────────────┘                      │
│  │                                                          │
│  └──────────────────────────────────────────────────────────┤
│                                                              │
│  [Monitoring & Management]                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │  Prometheus  │  │  Grafana     │  │  Azure Log   │    │
│  │  + Thanos    │  │  Dashboard   │  │  Analytics   │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
│                                                              │
└────────────────────────────────────────────────────────────┘
```

### 네트워크 존 (Network Zones)

#### 1. DMZ Zone (외부 연결)
**목적**: 외부(차량) → IDC 진입점
- **구성**: Load Balancer (Active/Standby), API Gateway (2대)
- **네트워크**: 
  - External: 203.XXX.XXX.XXX/24 (Public IP)
  - Internal: 10.1.0.0/24 (Private IP)
- **보안**: 
  - Firewall: Palo Alto PA-5260
  - IPS/IDS: Enabled
  - DDoS Protection: Azure Front Door

#### 2. Application Zone (내부 서버)
**목적**: 비즈니스 로직 실행
- **구성**: FleetFlag API (2대), Flipt Engine (2대)
- **네트워크**: 10.2.0.0/24 (Private IP)
- **보안**: 
  - Access Control: DMZ Zone only
  - Internal Encryption: IPsec
  - mTLS: 모든 내부 통신

#### 3. Data Zone (데이터베이스)
**목적**: 데이터 영속성
- **구성**: PostgreSQL (Primary/Standby), Redis (3-node), ClickHouse
- **네트워크**: 10.3.0.0/24 (Private IP)
- **보안**: 
  - Access Control: Application Zone only
  - Encryption at Rest: AES-256
  - Backup: Daily 암호화 백업

---

## 차량 내부 물리 아키텍처

### 차량 타입별 구성

#### 타입 1: SDV 차량 (신형 - 2024년 이후)
**대표 차량**: Genesis GV80 Electrified, Ioniq 6, EV9

```
┌─────────────────────────────────────────────────────┐
│              Vehicle (Genesis GV80)                  │
│                                                       │
│  ┌───────────────────────────────────────────────┐  │
│  │  HPC (High Performance Computer)             │  │
│  │  ┌───────────────────────────────────────┐   │  │
│  │  │  Kubernetes Container Runtime         │   │  │
│  │  │  ┌─────────────┐  ┌─────────────┐   │   │  │
│  │  │  │ FleetFlag   │  │  Feature    │   │   │  │
│  │  │  │   Agent     │──│   App       │   │   │  │
│  │  │  │  (Rust)     │  │  (OTA/Nav)  │   │   │  │
│  │  │  └─────────────┘  └─────────────┘   │   │  │
│  │  │  ┌─────────────┐                     │   │  │
│  │  │  │  SQLite     │  (7-day cache)     │   │  │
│  │  │  └─────────────┘                     │   │  │
│  │  └───────────────────────────────────────┘   │  │
│  │  CPU: NVIDIA Tegra Xavier (8-core ARM)      │  │
│  │  RAM: 32GB LPDDR4X                           │  │
│  │  Storage: 256GB UFS 3.1                      │  │
│  └───────────────────────────────────────────────┘  │
│                          │                            │
│                  Ethernet 100Mbps                    │
│                          │                            │
│  ┌───────────────────────────────────────────────┐  │
│  │  CCU (Central Computing Unit)                │  │
│  │  - Sync Agent (gRPC)                         │  │
│  │  - VPN Client (WireGuard)                    │  │
│  │  - CAN Gateway                               │  │
│  │  CPU: NXP i.MX 8 QuadMax                     │  │
│  │  RAM: 8GB LPDDR4                             │  │
│  │  LTE/5G: Qualcomm X55                        │  │
│  └───────────────────────────────────────────────┘  │
│                          │                            │
│                    CAN Bus 500kbps                   │
│                          │                            │
│  ┌──────────────┬────────┴────────┬─────────────┐   │
│  │              │                 │             │   │
│  ▼              ▼                 ▼             ▼   │
│  [ADAS ECU]  [Cluster ECU]  [TCU ECU]  [IVI ECU]   │
│  (Camera)    (Display)      (Telemetry) (HMI)      │
│                                                       │
└─────────────────────────────────────────────────────┘
```

#### 타입 2: 레거시 차량 (구형 - 2023년 이전)
**대표 차량**: Sonata, Tucson, Santa Fe (기존 모델)

```
┌─────────────────────────────────────────────────────┐
│         Vehicle (Sonata 2022)                        │
│                                                       │
│  ┌───────────────────────────────────────────────┐  │
│  │  CCU (Central Computing Unit)                │  │
│  │  - Config File Parser                        │  │
│  │  - CAN Message Injector                      │  │
│  │  - VPN Client                                │  │
│  │  CPU: Renesas R-Car H3                       │  │
│  │  RAM: 2GB LPDDR3                             │  │
│  │  LTE: Cat.12 Modem                           │  │
│  └───────────────────────────────────────────────┘  │
│                          │                            │
│                    CAN Bus 500kbps                   │
│                          │                            │
│  ┌──────────────┬────────┴────────┬─────────────┐   │
│  │              │                 │             │   │
│  ▼              ▼                 ▼             ▼   │
│  [ADAS ECU]  [Cluster ECU]  [TCU ECU]  [IVI ECU]   │
│  (No SDK)    (No SDK)       (No SDK)    (No SDK)   │
│  ※ CAN 메시지로 제어                                 │
│                                                       │
└─────────────────────────────────────────────────────┘
```

**레거시 차량 제약사항**:
- SDK 미설치 (HPC 없음)
- CCU가 IDC에서 암호화된 구성 파일 다운로드
- CAN 메시지로 ECU 제어 (Feature Flag 상태 전파)
- 실시간성 낮음 (30초~5분 동기화 주기)

---

## 네트워크 토폴로지

### End-to-End 네트워크 흐름

```
┌──────────────┐
│  Internet    │  Public Network
└──────┬───────┘
       │ HTTPS/WSS (TLS 1.3)
       │
       ▼
┌──────────────────┐
│ Azure Front Door │  CDN & DDoS Protection
│   (Global PoP)   │  - 60+ regions
└──────┬───────────┘  - WAF enabled
       │ VPN IPsec
       │
       ▼
┌──────────────────────────┐
│ Hyundai IDC Firewall     │  NGFW (Next-Gen Firewall)
│  (Palo Alto PA-5260)     │  - 50 Gbps throughput
└──────┬───────────────────┘  - IPS/IDS enabled
       │
       ├─────────────────┬─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   DMZ Zone   │  │   App Zone   │  │  Data Zone   │
│  (10.1.0.0)  │  │  (10.2.0.0)  │  │  (10.3.0.0)  │
│              │  │              │  │              │
│  Load Bal.   │─▶│  FleetFlag   │─▶│  PostgreSQL  │
│  API Gateway │  │  Flipt Eng.  │  │  Redis       │
└──────┬───────┘  └──────────────┘  └──────────────┘
       │
       │ TLS 1.3 (mTLS)
       │
       ▼
┌──────────────────┐
│  Vehicle CCU     │  In-Vehicle Gateway
│  (LTE/5G Modem)  │  - WireGuard VPN
└──────┬───────────┘  - AES-256-GCM
       │
       │ Ethernet 100Mbps
       │
       ▼
┌──────────────────┐
│  Vehicle HPC     │  High Performance Computer
│  (K8s Runtime)   │  - FleetFlag Agent
└──────┬───────────┘  - Local SQLite Cache
       │
       │ CAN Bus 500kbps
       │
       ▼
┌──────────────────┐
│  Legacy ECUs     │  ADAS, Cluster, TCU, IVI
└──────────────────┘
```

### 네트워크 대역폭 요구사항

| 구간 | 프로토콜 | 대역폭 | 지연시간 (RTT) | 비고 |
|------|----------|--------|----------------|------|
| Internet → Azure Front Door | HTTPS/WSS | 10 Gbps | < 50ms | Global CDN |
| Azure → IDC Firewall | VPN IPsec | 5 Gbps | < 20ms | 전용선 (Leased Line) |
| DMZ → App Zone | TLS 1.3 | 25 Gbps | < 1ms | Internal Network |
| App → Data Zone | IPsec | 25 Gbps | < 1ms | Internal Network |
| IDC → Vehicle CCU | LTE/5G | 10 Mbps/Vehicle | 45ms (평균) | 무선 네트워크 |
| CCU → HPC | Ethernet | 100 Mbps | < 1ms | In-Vehicle Network |
| HPC → ECU | CAN Bus | 500 kbps | < 5ms | Automotive Bus |

### 데이터 전송량 계산 (1대 차량 기준)

#### Daily Traffic per Vehicle
- **Feature Flag Sync**: 2.3 MB/day
  - Config Download: 100 KB (1회/24시간)
  - Heartbeat: 10 KB × 2,880회 (30초 주기) = 28.8 MB/day
  - 압축률 (gzip level 6): 12.5% → 실제 전송량 2.3 MB/day

- **Telemetry Upload**: 500 KB/day
  - Evaluation Logs: 100 KB
  - Error Reports: 50 KB
  - Metrics: 350 KB

**총 Daily Traffic**: 2.8 MB/day/vehicle

#### Total IDC Bandwidth (125,000대 기준)
- **Inbound**: 2.8 MB/day × 125,000 = 350 GB/day = 32 Mbps (평균)
- **Peak Hour (08:00-09:00 출근시간)**: 32 Mbps × 10 = **320 Mbps**
- **Kill Switch 긴급 전파**: 100 KB × 125,000 = 12.5 GB = **10 Gbps burst**

---

## 하드웨어 사양

### IDC 서버 사양

#### Rack 1: Frontend Zone

##### 1. Load Balancer (LB-01, LB-02)
- **Model**: HPE ProLiant DL380 Gen10 Plus
- **CPU**: AMD EPYC 7543 32-Core @ 2.8GHz (16 Cores Active)
- **RAM**: 64GB DDR4-3200 ECC RDIMM
- **Storage**: 2TB NVMe SSD (Samsung PM9A3) × 2 (RAID 1)
- **Network**: 
  - 10GbE SFP+ × 4 (Mellanox ConnectX-6)
  - 1GbE RJ-45 × 2 (Management)
- **Power**: 2× 800W Platinum PSU (Redundant)
- **Price**: $8,500/unit

##### 2. API Gateway (GW-01, GW-02)
- **Model**: Dell PowerEdge R750
- **CPU**: Intel Xeon Gold 6338 32-Core @ 2.0GHz
- **RAM**: 128GB DDR4-3200 ECC RDIMM
- **Storage**: 4TB NVMe SSD (Intel P5510) × 4 (RAID 10)
- **Network**: 
  - 25GbE SFP28 × 2 (Intel E810)
  - 10GbE SFP+ × 2
- **Power**: 2× 1400W Platinum PSU
- **Price**: $15,000/unit

#### Rack 2: Backend Zone

##### 3. FleetFlag API Server (API-01, API-02)
- **Model**: Supermicro SYS-220U-TNR
- **CPU**: AMD EPYC 7763 64-Core @ 2.45GHz
- **RAM**: 256GB DDR4-3200 ECC RDIMM (8× 32GB)
- **Storage**: 8TB NVMe SSD (Micron 7450 MAX) × 4 (RAID 10)
- **Network**: 
  - 25GbE SFP28 × 4 (Mellanox ConnectX-6 Lx)
- **GPU**: NVIDIA A2 16GB (Optional for ML workloads)
- **Power**: 2× 2000W Titanium PSU
- **Price**: $28,000/unit

##### 4. Flipt Engine (FLIPT-01, FLIPT-02)
- **Model**: HPE ProLiant DL380 Gen10 Plus
- **CPU**: Intel Xeon Gold 6338 32-Core @ 2.0GHz
- **RAM**: 128GB DDR4-3200 ECC RDIMM
- **Storage**: 4TB NVMe SSD × 4 (RAID 10)
- **Network**: 25GbE SFP28 × 2
- **Power**: 2× 1400W Platinum PSU
- **Price**: $15,000/unit

#### Rack 3: Data Zone

##### 5. PostgreSQL (PG-01 Primary, PG-02 Standby)
- **Model**: Supermicro SYS-240P-TNRT
- **CPU**: AMD EPYC 7763 64-Core @ 2.45GHz × 2 (Dual Socket)
- **RAM**: 512GB DDR4-3200 ECC LRDIMM (16× 32GB)
- **Storage**: 
  - System: 2TB NVMe SSD × 2 (RAID 1)
  - Data: 8TB NVMe SSD (Samsung PM9A3) × 8 (RAID 10)
  - Total Capacity: 32TB raw → 16TB usable
- **Network**: 25GbE SFP28 × 4
- **Power**: 2× 2400W Titanium PSU
- **Price**: $45,000/unit

##### 6. Redis Cluster (REDIS-01, REDIS-02, REDIS-03)
- **Model**: Dell PowerEdge R650
- **CPU**: Intel Xeon Gold 6338 32-Core @ 2.0GHz
- **RAM**: 256GB DDR4-3200 ECC RDIMM (8× 32GB)
- **Storage**: 4TB NVMe SSD × 2 (RAID 1)
- **Network**: 25GbE SFP28 × 2
- **Power**: 2× 1400W Platinum PSU
- **Price**: $18,000/unit

##### 7. ClickHouse Analytics (CH-01)
- **Model**: Supermicro SYS-220U-TNR
- **CPU**: AMD EPYC 7763 64-Core @ 2.45GHz
- **RAM**: 512GB DDR4-3200 ECC LRDIMM
- **Storage**: 12TB NVMe SSD (Micron 7450 PRO) × 8 (RAID 6)
- **Network**: 25GbE SFP28 × 4
- **Power**: 2× 2000W Titanium PSU
- **Price**: $38,000/unit

### 차량 내부 하드웨어 사양

#### 1. HPC (High Performance Computer) - SDV 차량
- **SoC**: NVIDIA Tegra Xavier AGX
  - CPU: 8-core ARM v8.2 (Carmel) @ 2.26GHz
  - GPU: 512-core Volta @ 1.37GHz
  - DLA: 2× Deep Learning Accelerators
- **RAM**: 32GB 256-bit LPDDR4x @ 137 GB/s
- **Storage**: 256GB UFS 3.1 (Sequential Read: 2.1 GB/s)
- **Network**: 
  - Ethernet: 1 Gbps (TSN-capable)
  - CAN-FD: 5 Mbps
- **Power**: 10W (idle) ~ 60W (peak)
- **Temperature**: -40°C ~ +85°C (Automotive Grade)
- **Certification**: ISO 26262 ASIL-B
- **Price**: $1,200/unit (OEM price)

#### 2. CCU (Central Computing Unit)
- **SoC**: NXP i.MX 8 QuadMax
  - CPU: 4× ARM Cortex-A72 @ 1.8GHz + 2× Cortex-A53 @ 1.2GHz
  - GPU: Vivante GC7000XSVX
- **RAM**: 8GB LPDDR4 @ 3200MHz
- **Storage**: 64GB eMMC 5.1
- **Network**: 
  - Ethernet: 100 Mbps (AVB-capable)
  - CAN: 2× CAN-FD controllers
  - LTE/5G: Qualcomm X55 5G Modem (7 Gbps DL / 3 Gbps UL)
- **Power**: 5W (idle) ~ 20W (peak)
- **Temperature**: -40°C ~ +85°C
- **Certification**: ISO 26262 ASIL-C
- **Price**: $450/unit (OEM price)

#### 3. Legacy ECUs (예시: ADAS ECU)
- **MCU**: Renesas R-Car V3H
  - CPU: 4× ARM Cortex-A53 @ 1.2GHz
  - ISP: Image Signal Processor (4× 4K cameras)
- **RAM**: 4GB LPDDR4
- **Storage**: 8GB eMMC
- **Network**: CAN-FD 5Mbps
- **Power**: 15W
- **Temperature**: -40°C ~ +105°C
- **Certification**: ISO 26262 ASIL-D
- **Price**: $280/unit (OEM price)

---

## 전력 및 냉각 요구사항

### IDC 전력 소비량

#### 서버별 전력 소비 (실측치)

| 서버 | Idle | 평균 | Peak | 수량 | Total (평균) |
|------|------|------|------|------|--------------|
| Load Balancer | 120W | 350W | 700W | 2 | 700W |
| API Gateway | 180W | 550W | 1200W | 2 | 1,100W |
| FleetFlag API | 280W | 780W | 1800W | 2 | 1,560W |
| Flipt Engine | 160W | 480W | 1200W | 2 | 960W |
| PostgreSQL | 320W | 820W | 2200W | 2 | 1,640W |
| Redis Cluster | 200W | 420W | 1200W | 3 | 1,260W |
| ClickHouse | 250W | 680W | 1800W | 1 | 680W |
| Network Switch | 150W | 180W | 250W | 3 | 540W |
| Firewall | 200W | 250W | 350W | 1 | 250W |
| **Total** | | | | **18** | **8,690W** |

#### 냉각 시스템 전력
- **CRAC (Computer Room AC)**: 3× 15kW = 45kW
- **PUE (Power Usage Effectiveness)**: 1.5
- **Total Power (with Cooling)**: 8.69kW × 1.5 = **13.04kW**

#### Phase별 전력 요구사항
- **Phase 1 (PoC)**: 13kW (현재 구성)
- **Phase 2 (Production)**: 26kW (서버 2배 증설)
- **Phase 3 (Scale-Up)**: 52kW (서버 4배 증설 + Redis Cluster 확장)

### 냉각 시스템

#### Precision Air Conditioning
- **Model**: Vertiv Liebert DS 15kW × 3대
- **Capacity**: 45kW (153,000 BTU/h)
- **Airflow**: 4,500 m³/h per unit
- **Efficiency**: EER 3.2
- **Redundancy**: N+1 (2대 Active, 1대 Standby)

#### Hot Aisle / Cold Aisle 구성
```
┌──────────────────────────────────────────┐
│          Cold Aisle (20°C)               │
│                                          │
│  [Rack 1]  [Rack 2]  [Rack 3]           │
│   Front     Front     Front              │
│   ↑ ↑ ↑    ↑ ↑ ↑    ↑ ↑ ↑              │
│                                          │
│  ════════════════════════════════════    │ Raised Floor (Plenum)
│                                          │
│   ↓ ↓ ↓    ↓ ↓ ↓    ↓ ↓ ↓              │
│   Rear      Rear      Rear               │
│  [Rack 1]  [Rack 2]  [Rack 3]           │
│                                          │
│          Hot Aisle (35°C)                │
└──────────────────────────────────────────┘
                │
                ▼
        CRAC Return Air
```

### 차량 전력 소비 및 열 관리

#### HPC 전력 프로파일
- **Idle (차량 ACC OFF)**: 2W (Sleep Mode)
- **Normal Operation**: 18W (Feature Flag Evaluation)
- **Peak (OTA Update)**: 45W
- **Daily Energy**: 18W × 24h = 0.432 kWh/day
- **Monthly Energy**: 13 kWh/month
- **Impact on EV Range**: -0.8 km/charge (무시할 수준)

#### 열 관리
- **HPC 동작 온도**: -40°C ~ +85°C (Automotive Grade)
- **냉각 방식**: Passive Cooling (Heat Sink + Chassis)
- **열 발생**: 45W peak → 153.7 BTU/h
- **차량 공조 연동**: 필요 없음 (Self-contained)

---

## 배포 전략

### Phase 1: PoC (2026.03 - 2027.02)

#### 목표
- **차량 대수**: 10대 (사내 테스트 차량)
- **서버 규모**: Rack 3개, 12대 서버
- **예산**: 6억원

#### 배포 순서
1. **Week 1-2**: IDC 랙 설치 및 네트워크 구성
2. **Week 3-4**: 서버 설치 및 OS/Middleware 구성
3. **Week 5-6**: FleetFlag 소프트웨어 배포
4. **Week 7-8**: 차량 HPC/CCU 펌웨어 OTA 배포
5. **Week 9-10**: 통합 테스트 및 최적화

### Phase 2: Production (2027.03 - 2028.02)

#### 목표
- **차량 대수**: 100대 (Beta 고객)
- **서버 규모**: 서버 2배 증설 (Active-Active HA)
- **예산**: 12억원

#### 주요 작업
1. 서버 이중화 (각 컴포넌트 4대씩)
2. PostgreSQL Streaming Replication 구성
3. Redis Cluster (6-node) 구성
4. ClickHouse Distributed Table 구성
5. 차량 100대 OTA 배포

### Phase 3: Scale-Up (2028.03 - 2028.08)

#### 목표
- **차량 대수**: 100,000대 (양산 차량)
- **서버 규모**: 서버 4배 증설 + Sharding
- **예산**: 5억원 (추가)

#### 확장 전략
1. **Horizontal Scaling**: 각 컴포넌트 8-16대로 확장
2. **Database Sharding**: PostgreSQL 4-shard (VIN 기반)
3. **Cache Tier 추가**: Redis → 12-node cluster
4. **Analytics Separation**: ClickHouse 전용 클러스터 (3-node)
5. **Network Upgrade**: 10GbE → 25GbE → 100GbE

---

## 부록

### A. 서버 BOM (Bill of Materials)

#### Phase 1 BOM

| 항목 | 모델 | 수량 | 단가 | 소계 |
|------|------|------|------|------|
| Load Balancer | HPE DL380 Gen10+ | 2 | $8,500 | $17,000 |
| API Gateway | Dell R750 | 2 | $15,000 | $30,000 |
| FleetFlag API | Supermicro 220U | 2 | $28,000 | $56,000 |
| Flipt Engine | HPE DL380 Gen10+ | 2 | $15,000 | $30,000 |
| PostgreSQL | Supermicro 240P | 2 | $45,000 | $90,000 |
| Redis Cluster | Dell R650 | 3 | $18,000 | $54,000 |
| ClickHouse | Supermicro 220U | 1 | $38,000 | $38,000 |
| Network Switch (25GbE) | Cisco Nexus 9336C-FX2 | 3 | $12,000 | $36,000 |
| Firewall | Palo Alto PA-5260 | 1 | $80,000 | $80,000 |
| Rack (42U) | APC NetShelter SX | 3 | $2,500 | $7,500 |
| PDU (Power) | APC AP8959 | 6 | $1,200 | $7,200 |
| UPS (20kW) | APC Symmetra PX 20kW | 1 | $25,000 | $25,000 |
| CRAC (15kW) | Vertiv Liebert DS | 3 | $18,000 | $54,000 |
| Monitoring (Grafana) | Self-hosted | 1 | $5,000 | $5,000 |
| **Total** | | | | **$529,700** |

**원화 환산** (1,300원/$): **6억 8,861만원**

### B. 차량 BOM (Vehicle BoM)

#### SDV 차량 추가 비용 (차량 1대당)
| 항목 | 모델 | 수량 | 단가 | 비고 |
|------|------|------|------|------|
| HPC | NVIDIA Tegra Xavier | 1 | $1,200 | 기존 차량 대비 추가 |
| CCU Upgrade | NXP i.MX 8 QuadMax | 1 | $450 | 5G Modem 포함 |
| Software License | FleetFlag Agent | 1 | $50 | 연간 구독 |
| **Total per Vehicle** | | | **$1,700** | |

#### Phase별 차량 배포 비용
- **Phase 1**: 10대 × $1,700 = $17,000 (2,210만원)
- **Phase 2**: 100대 × $1,700 = $170,000 (2억 2,100만원)
- **Phase 3**: 100,000대 × $1,700 = $170,000,000 (2,210억원)

**Note**: Phase 3는 양산 차량에 기본 탑재되므로 실제 추가 비용 없음

---

## 문서 변경 이력
| 버전 | 날짜 | 작성자 | 변경 내용 |
|------|------|--------|-----------|
| v1.0 | 2026-02-01 | FleetFlag Team | Initial Physical Deployment Architecture |

---

## 연락처
- **이메일**: team@fleetflag.dev
- **Slack**: #fleetflag-project
- **GitHub**: [Issues](https://github.com/hyundai/fleetflag-specification/issues)

---

**Document Classification**: 🔒 Confidential - Hyundai Motor Company
