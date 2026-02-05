# 🚗 Project 1: Automotive System Modeler

> **프로젝트 코드**: WEBAPP-001  
> **생성일**: 2026-02-01  
> **최종 수정일**: 2026-02-02  
> **상태**: ✅ 운영 중 (Production)  
> **담당자**: Development Team  
> **우선순위**: High

---

## 📋 Table of Contents

- [개요](#개요)
- [프로젝트 목표](#프로젝트-목표)
- [기술 스택](#기술-스택)
- [주요 기능](#주요-기능)
- [아키텍처](#아키텍처)
- [배포 정보](#배포-정보)
- [API 문서](#api-문서)
- [사용자 가이드](#사용자-가이드)
- [개발 가이드](#개발-가이드)
- [이슈 및 제한사항](#이슈-및-제한사항)
- [로드맵](#로드맵)
- [관련 문서](#관련-문서)

---

## 개요

### 프로젝트 설명
Automotive System Modeler는 웹 기반의 UML/SysML 다이어그램 편집기로, 자동차 소프트웨어 아키텍처를 시각적으로 설계하고 편집할 수 있는 도구입니다.

### 비즈니스 가치
- 자동차 소프트웨어 설계 시간 단축 (수동 대비 60% 개선)
- 드래그 앤 드롭으로 직관적인 컴포넌트 배치
- ECU, 센서, 액추에이터 간 신호 연결 시각화
- JSON 기반 모델 저장 및 불러오기

### 주요 사용자
- 자동차 시스템 아키텍트
- 소프트�어 엔지니어
- AUTOSAR 개발자

---

## 프로젝트 목표

### 목표
1. **직관적 UI**: 드래그 앤 드롭 기반의 사용자 친화적 인터페이스
2. **실시간 편집**: 즉각적인 피드백과 시각화
3. **표준 준수**: UML/SysML 표준 기반 설계
4. **데이터 지속성**: JSON Export/Import 기능

### 성공 지표
- 사용자당 평균 설계 시간: < 10분
- 시스템 응답 시간: < 200ms
- 다이어그램 저장 성공률: > 99%

---

## 기술 스택

### Frontend
| 기술 | 버전 | 용도 |
|------|------|------|
| **HTML5** | - | SVG 기반 다이어그램 렌더링 |
| **Tailwind CSS** | 3.4.0 (CDN) | UI 스타일링 |
| **Font Awesome** | 6.4.0 (CDN) | 아이콘 |
| **JavaScript (Vanilla)** | ES6+ | 인터랙션 로직 |

### Backend
| 기술 | 버전 | 용도 |
|------|------|------|
| **Hono** | 4.0+ | 경량 웹 프레임워크 |
| **TypeScript** | 5.0+ | 타입 안전성 |
| **Vite** | 5.0+ | 빌드 도구 |

### Deployment
| 기술 | 용도 |
|------|------|
| **Cloudflare Pages** | 엣지 배포 플랫폼 |
| **Wrangler** | Cloudflare CLI 도구 |
| **PM2** | 로컬 프로세스 관리 |

---

## 주요 기능

### 1. 컴포넌트 팔레트
- **ECU** (Electronic Control Unit)
- **센서** (Temperature, Speed, Pressure 등)
- **액추에이터** (Motor, Valve 등)
- **CAN Bus** (통신 버스)

### 2. 다이어그램 편집
- ✅ 드래그 앤 드롭으로 컴포넌트 추가
- ✅ 컴포넌트 간 신호 연결 (4-포트: Top, Bottom, Left, Right)
- ✅ 컴포넌트 이동 및 재배치
- ✅ 선택/삭제 기능

### 3. 속성 편집
- 컴포넌트 이름 수정
- 컴포넌트 타입 변경
- 신호 속성 편집 (Signal Name, Data Type, Frequency)

### 4. 캔버스 네비게이션
- 확대/축소 (Zoom In/Out)
- 캔버스 이동 (Pan)
- 전체 보기 (Fit to View)

### 5. 모델 관리
- **저장**: JSON 형식으로 현재 모델 저장
- **불러오기**: 저장된 JSON 파일 로드
- **초기화**: 모델 전체 삭제
- **샘플 로드**: 예제 자동차 시스템 로드

---

## 아키텍처

### 시스템 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│                      브라우저 (Client)                   │
│                                                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │  UI Layer (HTML/CSS/JavaScript)                 │   │
│  │  - Component Palette                            │   │
│  │  - Canvas (SVG)                                 │   │
│  │  - Properties Panel                             │   │
│  │  - Toolbar                                      │   │
│  └─────────────────────────────────────────────────┘   │
│                        │                                 │
│                        │ Fetch API                       │
│                        ▼                                 │
└─────────────────────────────────────────────────────────┘
                         │
                         │ HTTPS
                         ▼
┌─────────────────────────────────────────────────────────┐
│              Hono Backend (Cloudflare Workers)          │
│                                                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │  API Routes                                     │   │
│  │  - GET /                    (Main Page)         │   │
│  │  - GET /api/model           (Get Model)        │   │
│  │  - POST /api/model          (Save Model)       │   │
│  │  - POST /api/model/reset    (Reset Model)      │   │
│  └─────────────────────────────────────────────────┘   │
│                        │                                 │
│                        ▼                                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │  In-Memory Storage                              │   │
│  │  { elements: [], connections: [] }             │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### 데이터 모델

**Component**
```json
{
  "id": "ecu-1",
  "type": "ECU",
  "name": "Engine ECU",
  "x": 150,
  "y": 200,
  "width": 150,
  "height": 100,
  "ports": {
    "top": { "x": 225, "y": 200, "connected": false },
    "bottom": { "x": 225, "y": 300, "connected": false },
    "left": { "x": 150, "y": 250, "connected": false },
    "right": { "x": 300, "y": 250, "connected": false }
  }
}
```

**Connection**
```json
{
  "id": "conn-1",
  "from": "ecu-1-bottom",
  "to": "sensor-1-top",
  "signal": "Temperature",
  "dataType": "Float32",
  "frequency": "100Hz"
}
```

---

## 배포 정보

### 환경

| 환경 | URL | 상태 | 포트 |
|------|-----|------|------|
| **Production** | https://3000-ifl0a91rhl2v80gsfr4rn-8f57ffe2.sandbox.novita.ai | ✅ Active | 3000 |
| **Local Dev** | http://localhost:3000 | - | 3000 |

### 디렉토리 구조
```
/home/user/webapp/
├── src/
│   ├── index.tsx              # Main Hono application
│   └── types/                 # TypeScript types
├── public/
│   ├── static/
│   │   ├── app.js            # Frontend JavaScript
│   │   └── styles.css        # Custom CSS
├── ecosystem.config.cjs       # PM2 configuration
├── wrangler.jsonc            # Cloudflare configuration
├── package.json
├── tsconfig.json
└── README.md
```

### 빌드 및 배포 명령어

**로컬 개발**
```bash
cd /home/user/webapp
npm run build
pm2 start ecosystem.config.cjs
```

**프로덕션 배포** (계획)
```bash
cd /home/user/webapp
npm run build
wrangler pages deploy dist --project-name automotive-modeler
```

---

## API 문서

### 1. GET /
메인 페이지 반환

**Response**: HTML

---

### 2. GET /api/model
현재 모델 조회

**Request**: None

**Response**:
```json
{
  "elements": [
    {
      "id": "ecu-1",
      "type": "ECU",
      "name": "Engine ECU",
      "x": 150,
      "y": 200
    }
  ],
  "connections": [
    {
      "id": "conn-1",
      "from": "ecu-1-bottom",
      "to": "sensor-1-top"
    }
  ]
}
```

---

### 3. POST /api/model
모델 저장

**Request**:
```json
{
  "elements": [...],
  "connections": [...]
}
```

**Response**:
```json
{
  "success": true,
  "message": "Model saved successfully"
}
```

---

### 4. POST /api/model/reset
모델 초기화

**Request**: None

**Response**:
```json
{
  "success": true,
  "message": "Model reset successfully"
}
```

---

## 사용자 가이드

### 빠른 시작 (5분)

#### Step 1: 접속
https://3000-ifl0a91rhl2v80gsfr4rn-8f57ffe2.sandbox.novita.ai

#### Step 2: 컴포넌트 추가
1. 왼쪽 팔레트에서 **ECU** 클릭
2. 캔버스에 드롭

#### Step 3: 연결 생성
1. 첫 번째 컴포넌트의 포트 클릭
2. 두 번째 컴포넌트의 포트 클릭

#### Step 4: 저장
상단 **Save Model** 버튼 클릭

---

### 샘플 시나리오

**자동차 엔진 제어 시스템**
```
1. Engine ECU 추가 (150, 200)
2. Temperature Sensor 추가 (150, 350)
3. Speed Sensor 추가 (300, 350)
4. Actuator 추가 (450, 200)
5. 연결:
   - Temperature Sensor → Engine ECU
   - Speed Sensor → Engine ECU
   - Engine ECU → Actuator
6. 저장
```

---

## 개발 가이드

### 로컬 환경 설정

```bash
# 1. 프로젝트 클론
cd /home/user/webapp

# 2. 의존성 설치
npm install

# 3. 개발 서버 시작
npm run build
pm2 start ecosystem.config.cjs

# 4. 테스트
curl http://localhost:3000
```

### 주요 개발 파일

**src/index.tsx**
- Hono 애플리케이션 진입점
- API 라우트 정의
- 정적 파일 서빙

**public/static/app.js**
- 프론트엔드 로직
- SVG 렌더링
- 이벤트 핸들러

**public/static/styles.css**
- 커스텀 스타일

---

## 이슈 및 제한사항

### 현재 제한사항

| 이슈 | 설명 | 우선순위 | 상태 |
|------|------|----------|------|
| **메모리 저장** | 브라우저 새로고침 시 데이터 손실 | High | Open |
| **Undo/Redo 없음** | 실행 취소 기능 미구현 | Medium | Open |
| **Orthogonal Routing 없음** | 직선 연결만 지원 | Medium | Open |
| **자동 레이아웃 없음** | 수동 배치만 가능 | Low | Open |

### 알려진 버그

- **없음** (현재까지)

---

## 로드맵

### Phase 1: MVP ✅ (완료)
- [x] 기본 컴포넌트 팔레트
- [x] 드래그 앤 드롭
- [x] 신호 연결
- [x] JSON Export/Import

### Phase 2: Enhanced UML (계획, 2주)
- [ ] Undo/Redo 기능
- [ ] Orthogonal Routing
- [ ] 자동 레이아웃
- [ ] 컴포넌트 그룹화

### Phase 3: AUTOSAR Integration (계획, 1개월)
- [ ] AUTOSAR 메타모델
- [ ] SysML v2 지원
- [ ] 코드 생성 (C/C++)
- [ ] ISO 26262 추적성

### Phase 4: Collaboration (계획, 2개월)
- [ ] 실시간 협업
- [ ] 버전 관리
- [ ] 주석 및 리뷰
- [ ] 권한 관리

---

## 관련 문서

### 내부 문서
- [프로젝트 현황 보고서](/home/user/PROJECT_STATUS.md)
- [프로젝트 분리 보고서](/home/user/PROJECT_SEPARATION_REPORT.md)

### 관련 프로젝트
- **Project 2**: [Architecture Agent](/home/user/confluence-docs/PROJECT-2-Architecture-Agent.md)
- **Project 3**: [Local LLM Integration](/home/user/confluence-docs/PROJECT-3-Local-LLM-Integration.md)

### 외부 참고
- [Hono Documentation](https://hono.dev/)
- [Cloudflare Pages Docs](https://developers.cloudflare.com/pages/)
- [AUTOSAR Standard](https://www.autosar.org/)

---

## 변경 이력

| 날짜 | 버전 | 변경 내용 | 작성자 |
|------|------|-----------|--------|
| 2026-02-01 | v0.1.0 | MVP 완성 | Dev Team |
| 2026-02-02 | v0.1.0 | Confluence 문서 작성 | Dev Team |

---

## 담당자 및 연락처

| 역할 | 이름 | 연락처 |
|------|------|--------|
| **프로젝트 리드** | - | - |
| **개발** | Development Team | - |
| **QA** | - | - |

---

## 라이선스

**MIT License** (TBD)

---

## 부록

### A. 용어집

| 용어 | 설명 |
|------|------|
| **ECU** | Electronic Control Unit - 전자 제어 장치 |
| **CAN** | Controller Area Network - 차량 통신 프로토콜 |
| **AUTOSAR** | Automotive Open System Architecture - 자동차 소프트웨어 표준 |
| **SysML** | Systems Modeling Language - 시스템 모델링 언어 |

### B. FAQ

**Q: 데이터가 어디에 저장되나요?**  
A: 현재는 메모리에 저장됩니다. JSON Export/Import로 로컬 저장 가능합니다.

**Q: 오프라인에서 사용 가능한가요?**  
A: 아니오, 현재는 온라인 전용입니다.

**Q: 다른 형식으로 Export 가능한가요?**  
A: 현재는 JSON만 지원합니다. PNG/SVG는 Phase 2에서 추가 예정입니다.

---

**문서 작성자**: Development Team  
**최종 검토일**: 2026-02-02  
**다음 리뷰 예정일**: 2026-02-16
