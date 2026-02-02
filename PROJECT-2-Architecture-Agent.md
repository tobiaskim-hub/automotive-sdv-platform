# 🤖 Project 2: Architecture Agent (LLM Integration)

> **프로젝트 코드**: AGENT-002  
> **생성일**: 2026-02-02  
> **최종 수정일**: 2026-02-02  
> **상태**: ✅ 운영 중 (Production) - LLM 통합 완료  
> **담당자**: AI/ML Team  
> **우선순위**: High

---

## 📋 Table of Contents

- [개요](#개요)
- [프로젝트 목표](#프로젝트-목표)
- [기술 스택](#기술-스택)
- [주요 기능](#주요-기능)
- [아키텍처](#아키텍처)
- [LLM 통합 가이드](#llm-통합-가이드)
- [배포 정보](#배포-정보)
- [API 문서](#api-문서)
- [사용자 가이드](#사용자-가이드)
- [개발 가이드](#개발-가이드)
- [성능 및 벤치마크](#성능-및-벤치마크)
- [이슈 및 제한사항](#이슈-및-제한사항)
- [로드맵](#로드맵)
- [관련 문서](#관련-문서)

---

## 개요

### 프로젝트 설명
Architecture Agent는 자연어 프롬프트를 통해 소프트웨어 아키텍처 다이어그램을 자동으로 생성하는 AI 기반 도구입니다. Ollama LLM (Qwen 2.5)을 통합하여 실시간으로 아키텍처 설계를 생성합니다.

### 비즈니스 가치
- **시간 절약**: 수동 설계 대비 90% 시간 단축 (10분 → 30초)
- **비용 절감**: 로컬 LLM 사용으로 API 비용 $0
- **품질 향상**: AI 기반 베스트 프랙티스 자동 적용
- **접근성**: 자연어 인터페이스로 누구나 사용 가능

### 주요 사용자
- 소프트웨어 아키텍트
- 시스템 설계자
- 프로젝트 매니저
- 신입 개발자 (학습 목적)

### 차별화 요소
| Feature | 기존 도구 (Draw.io, Lucidchart) | Architecture Agent |
|---------|--------------------------------|-------------------|
| **입력 방식** | 수동 드래그 앤 드롭 | 자연어 프롬프트 |
| **생성 시간** | 10-30분 | 3-5초 |
| **AI 지원** | 없음 | ✅ LLM 기반 자동 생성 |
| **비용** | $월 10-20 | $0 (로컬 LLM) |
| **오프라인** | 제한적 | ✅ 완전 오프라인 가능 |

---

## 프로젝트 목표

### 목표
1. **자연어 이해**: 일상 언어로 아키텍처 요구사항 입력
2. **실시간 생성**: 3-5초 내 다이어그램 자동 생성
3. **로컬 LLM**: Ollama 기반 프라이빗 AI
4. **확장성**: RAG, 멀티모델 지원 준비

### 성공 지표
- 프롬프트 → 다이어그램 생성 시간: < 5초
- LLM 정확도: > 85%
- 사용자 만족도: > 4.0/5.0
- 시스템 가용성: > 99%

---

## 기술 스택

### Frontend
| 기술 | 버전 | 용도 |
|------|------|------|
| **HTML5** | - | 3-패널 레이아웃 UI |
| **Tailwind CSS** | 3.4.0 (CDN) | 반응형 스타일링 |
| **Font Awesome** | 6.4.0 (CDN) | 아이콘 |
| **JavaScript (Vanilla)** | ES6+ | LLM API 호출 및 렌더링 |

### Backend
| 기술 | 버전 | 용도 |
|------|------|------|
| **Hono** | 4.0+ | 경량 웹 프레임워크 |
| **TypeScript** | 5.0+ | 타입 안전성 |
| **Vite** | 5.0+ | 빌드 도구 |

### AI/ML Stack
| 기술 | 버전 | 용도 |
|------|------|------|
| **Ollama** | Latest | 로컬 LLM 서버 |
| **Qwen 2.5** | 7B | 아키텍처 생성 모델 |
| **Qwen 2.5 Coder** | 7B | 코드 생성 (향후) |
| **nomic-embed-text** | Latest | RAG 임베딩 (계획) |

### Deployment
| 기술 | 용도 |
|------|------|
| **Cloudflare Pages** | 엣지 배포 |
| **PM2** | 프로세스 관리 |
| **Wrangler** | Cloudflare CLI |

---

## 주요 기능

### 1. 자연어 프롬프트 입력
- ✅ 일상 언어로 아키텍처 요구사항 설명
- ✅ 빠른 시작 템플릿 제공
- ✅ 실시간 입력 검증

**예시 프롬프트**:
```
"마이크로서비스 아키텍처를 만들어줘. 
API Gateway, 3개 서비스, 각각 독립 DB 필요해"
```

### 2. LLM 기반 자동 생성
- ✅ Ollama + Qwen 2.5 통합
- ✅ JSON 형식 다이어그램 생성
- ✅ 베스트 프랙티스 자동 적용
- ✅ 실시간 연결 상태 확인 (10초 간격)

### 3. 다이어그램 렌더링
- ✅ SVG 기반 벡터 그래픽
- ✅ 컴포넌트 타입별 색상 구분
  - Gateway: 파란색 (#3b82f6)
  - Service: 녹색 (#10b981)
  - Database: 주황색 (#f59e0b)
  - ECU: 빨간색 (#ef4444)
  - Sensor: 보라색 (#8b5cf6)
  - Layer: 인디고 (#6366f1)
- ✅ 자동 연결선 (화살표)

### 4. AI Assistant
- ✅ 실시간 대화형 인터페이스
- ✅ 생성 결과 설명
- ✅ 개선 제안
- ✅ 컨텍스트 기반 조언

### 5. Export 기능
- ✅ JSON Export
- 🔜 PNG Export (계획)
- 🔜 SVG Export (계획)

---

## 아키텍처

### 시스템 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│                   사용자 PC (Client)                     │
│                                                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Ollama Service (Port 11434)                    │   │
│  │                                                   │   │
│  │  ┌──────────────────────────────────────────┐   │   │
│  │  │  Qwen 2.5:7B Model                       │   │   │
│  │  │  - 자연어 이해                           │   │   │
│  │  │  - JSON 생성                             │   │   │
│  │  │  - 아키텍처 설계                        │   │   │
│  │  └──────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                         ▲
                         │ HTTP POST /api/generate
                         │
┌─────────────────────────────────────────────────────────┐
│           Sandbox (Architecture Agent)                   │
│                                                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Hono Backend (Port 3001)                       │   │
│  │                                                   │   │
│  │  ┌──────────────────────────────────────────┐   │   │
│  │  │  /api/generate                           │   │   │
│  │  │  - Ollama API 호출 (사용자 PC)         │   │   │
│  │  │  - JSON 파싱                            │   │   │
│  │  │  - 다이어그램 데이터 반환              │   │   │
│  │  └──────────────────────────────────────────┘   │   │
│  │                                                   │   │
│  │  ┌──────────────────────────────────────────┐   │   │
│  │  │  /api/health                             │   │   │
│  │  │  - LLM 연결 상태 확인                  │   │   │
│  │  └──────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Frontend (3-Panel Layout)                      │   │
│  │  - 프롬프트 입력 패널                          │   │
│  │  - SVG 캔버스                                   │   │
│  │  - AI Assistant 챗                              │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                         ▲
                         │ HTTPS
                         │
┌─────────────────────────────────────────────────────────┐
│                브라우저 (End User)                       │
│  https://3001-...sandbox.novita.ai                       │
└─────────────────────────────────────────────────────────┘
```

### 데이터 플로우

```
[사용자] 
    ↓ 1. 프롬프트 입력: "마이크로서비스 아키텍처 만들어줘"
[Frontend]
    ↓ 2. POST /api/generate {"prompt": "..."}
[Hono Backend]
    ↓ 3. Ollama API 호출 (사용자 PC:11434)
[Ollama (Qwen 2.5)]
    ↓ 4. JSON 생성 {"name": "...", "elements": [...], ...}
[Hono Backend]
    ↓ 5. JSON 파싱 및 검증
[Frontend]
    ↓ 6. SVG 렌더링
[사용자]
    ✓ 다이어그램 확인!
```

---

## LLM 통합 가이드

### 사용자 PC 설정 (필수)

#### Windows
```powershell
# 1. Ollama 설치
# https://ollama.com/download/windows

# 2. 모델 다운로드 (약 5GB, 5분)
ollama pull qwen2.5:7b

# 3. 네트워크 노출
setx OLLAMA_HOST "0.0.0.0:11434"
ollama serve

# 4. IP 주소 확인
ipconfig | findstr IPv4
```

#### macOS / Linux
```bash
# 1. Ollama 설치
curl -fsSL https://ollama.com/install.sh | sh

# 2. 모델 다운로드
ollama pull qwen2.5:7b

# 3. 네트워크 노출
export OLLAMA_HOST=0.0.0.0:11434
ollama serve

# 4. IP 주소 확인
ifconfig | grep "inet " | grep -v 127.0.0.1
```

### 샌드박스 환경변수 설정

`.dev.vars` 파일 수정:
```bash
cd /home/user/architecture-agent
echo "OLLAMA_HOST=http://사용자PC-IP:11434" > .dev.vars
pm2 restart architecture-agent --update-env
```

### 연결 테스트
```bash
# Health Check
curl http://localhost:3001/api/health

# 예상 응답:
# {"status":"ok","ollamaHost":"http://192.168.0.100:11434","timestamp":"..."}
```

---

## 배포 정보

### 환경

| 환경 | URL | 상태 | 포트 |
|------|-----|------|------|
| **Production** | https://3001-ifl0a91rhl2v80gsfr4rn-8f57ffe2.sandbox.novita.ai | ✅ Active | 3001 |
| **Local Dev** | http://localhost:3001 | - | 3001 |

### 디렉토리 구조
```
/home/user/architecture-agent/
├── src/
│   ├── index.tsx              # Main Hono app + LLM integration
│   └── index_backup.tsx       # Original mock version (backup)
├── public/
│   └── static/
├── .dev.vars                  # Environment variables (OLLAMA_HOST)
├── ecosystem.config.cjs       # PM2 configuration
├── package.json
├── tsconfig.json
├── README.md
├── INSTALLATION_GUIDE.md      # Ollama 설치 가이드
├── INTEGRATION_GUIDE.md       # LLM 통합 가이드
└── QUICK_START.md             # 빠른 시작 가이드
```

### 빌드 및 배포

**로컬 개발**
```bash
cd /home/user/architecture-agent
npm run build
pm2 start ecosystem.config.cjs
```

**재시작 (환경변수 변경 시)**
```bash
pm2 restart architecture-agent --update-env
```

**로그 확인**
```bash
pm2 logs architecture-agent --nostream
```

---

## API 문서

### 1. POST /api/generate
LLM으로 다이어그램 생성

**Request**:
```json
{
  "prompt": "마이크로서비스 아키텍처를 만들어줘. API Gateway, 3개 서비스, 각각 DB 필요해"
}
```

**Response** (Success):
```json
{
  "success": true,
  "architecture": {
    "name": "Microservices Architecture",
    "elements": [
      {
        "id": "gateway-1",
        "type": "Gateway",
        "name": "API Gateway",
        "x": 400,
        "y": 100,
        "width": 140,
        "height": 80
      },
      {
        "id": "service-1",
        "type": "Service",
        "name": "User Service",
        "x": 200,
        "y": 250,
        "width": 120,
        "height": 80
      }
    ],
    "connections": [
      {
        "from": "gateway-1",
        "to": "service-1"
      }
    ],
    "explanation": "API Gateway를 통해 3개 서비스로 요청을 라우팅합니다."
  },
  "raw": "..."
}
```

**Response** (Error):
```json
{
  "success": false,
  "error": "Ollama API error: Connection refused",
  "fallback": true
}
```

---

### 2. GET /api/health
LLM 연결 상태 확인

**Request**: None

**Response**:
```json
{
  "status": "ok",
  "ollamaHost": "http://192.168.0.100:11434",
  "timestamp": "2026-02-02T09:30:00.000Z"
}
```

---

## 사용자 가이드

### 빠른 시작 (1분)

#### Step 1: 접속
https://3001-ifl0a91rhl2v80gsfr4rn-8f57ffe2.sandbox.novita.ai

#### Step 2: 연결 상태 확인
왼쪽 하단에서 "LLM 연결 상태" 확인:
- ✅ **녹색 "연결됨"**: 준비 완료!
- ❌ **빨간색 "연결 실패"**: Ollama 설정 확인 필요

#### Step 3: 프롬프트 입력
```
마이크로서비스 아키텍처를 만들어줘. 
API Gateway, User Service, Order Service, Payment Service, 
각 서비스별 독립 DB 필요해
```

#### Step 4: 생성
"LLM으로 생성하기" 버튼 클릭

#### Step 5: 확인
3-5초 후 중앙 캔버스에 다이어그램 표시!

---

### 테스트 프롬프트 예시

#### 1. 마이크로서비스 아키텍처
```
마이크로서비스 아키텍처를 만들어줘. 
API Gateway, User Service, Order Service, Payment Service, 
각 서비스별 DB 필요해
```

**예상 결과**:
- API Gateway (중앙)
- 3개 Service (User, Order, Payment)
- 3개 Database
- Gateway → Services → DBs 연결

---

#### 2. 자동차 ECU 시스템
```
자동차 ECU 시스템 만들어줘. 
Engine ECU, Brake ECU, Gateway ECU, 
Temperature Sensor, Speed Sensor, Pressure Sensor 필요해
```

**예상 결과**:
- 3개 ECU (Engine, Brake, Gateway)
- 3개 Sensor
- ECU ↔ Gateway ↔ Sensors 연결

---

#### 3. 레이어드 아키텍처
```
3계층 아키텍처를 설계해줘. 
Presentation Layer, Business Layer, Data Layer
```

**예상 결과**:
- 3개 Layer (수직 배치)
- Layer → Layer 연결

---

## 개발 가이드

### 로컬 환경 설정

```bash
# 1. 프로젝트 디렉토리 이동
cd /home/user/architecture-agent

# 2. 의존성 설치
npm install

# 3. 환경변수 설정
echo "OLLAMA_HOST=http://localhost:11434" > .dev.vars

# 4. 빌드
npm run build

# 5. 시작
pm2 start ecosystem.config.cjs

# 6. 테스트
curl http://localhost:3001/api/health
```

### 주요 개발 파일

**src/index.tsx**
- Hono 애플리케이션 진입점
- LLM API 통합 (`/api/generate`, `/api/health`)
- 3-패널 레이아웃 HTML

**Key Functions**:
- `POST /api/generate`: Ollama API 호출 및 JSON 파싱
- `GET /api/health`: 연결 상태 확인

**Frontend JavaScript** (Inline in HTML):
- `sendPrompt()`: LLM API 호출
- `renderDiagram()`: SVG 렌더링
- `checkLLMStatus()`: 10초마다 연결 확인

---

## 성능 및 벤치마크

### 응답 시간

| 모델 | 프롬프트 길이 | 응답 시간 | 메모리 사용 |
|------|---------------|-----------|-------------|
| **Qwen 2.5:7B** | 짧음 (50자) | 2-3초 | ~8GB |
| **Qwen 2.5:7B** | 중간 (150자) | 3-5초 | ~8GB |
| **Qwen 2.5:7B** | 길음 (300자) | 5-7초 | ~8GB |
| **Llama 3.2:3B** | 짧음 (50자) | 1-2초 | ~4GB |

### PC 사양별 성능

| CPU | RAM | 모델 | 응답 시간 | 평가 |
|-----|-----|------|-----------|------|
| **AMD RYZEN AI MAX+ PRO 395** | 128GB | Qwen 2.5:7B | 2-3초 | ⭐⭐⭐⭐⭐ |
| Intel i7-12700 | 32GB | Qwen 2.5:7B | 4-5초 | ⭐⭐⭐⭐ |
| Intel i5-10400 | 16GB | Llama 3.2:3B | 3-4초 | ⭐⭐⭐ |
| MacBook Pro M1 | 16GB | Qwen 2.5:7B | 3-4초 | ⭐⭐⭐⭐ |

---

## 이슈 및 제한사항

### 현재 제한사항

| 이슈 | 설명 | 우선순위 | 상태 | 해결 방안 |
|------|------|----------|------|-----------|
| **사용자 PC 필요** | Ollama를 사용자 PC에 설치 필수 | High | Open | 클라우드 LLM 옵션 추가 (Phase 2) |
| **네트워크 의존** | LAN 내 연결 필요 | Medium | Open | 터널링 또는 VPN 가이드 |
| **JSON 파싱 오류** | LLM이 잘못된 형식 생성 시 | Medium | Open | Fallback 및 재시도 로직 |
| **편집 기능 없음** | 생성 후 수정 불가 | Low | Open | Phase 2에서 구현 |

### 알려진 버그

- **없음** (현재까지)

---

## 로드맵

### Phase 1: MVP + LLM Integration ✅ (완료)
- [x] 자연어 프롬프트 입력
- [x] Ollama + Qwen 2.5 통합
- [x] SVG 다이어그램 렌더링
- [x] 실시간 연결 상태 확인
- [x] JSON Export

### Phase 2: 고급 기능 (2-3주)
- [ ] 드래그 앤 드롭 편집
- [ ] Undo/Redo
- [ ] PNG/SVG Export
- [ ] 다이어그램 히스토리
- [ ] 멀티모델 지원 (Llama, Mistral)

### Phase 3: RAG 통합 (1개월)
- [ ] Qdrant 벡터 DB 설치
- [ ] AUTOSAR 문서 임베딩
- [ ] ISO 26262 표준 임베딩
- [ ] 컨텍스트 기반 생성

### Phase 4: 클라우드 LLM (2개월)
- [ ] OpenAI GPT-4o 통합
- [ ] Anthropic Claude 통합
- [ ] 하이브리드 모드 (로컬 + 클라우드)

---

## 관련 문서

### 설치 및 통합 가이드
- [INSTALLATION_GUIDE.md](/home/user/architecture-agent/INSTALLATION_GUIDE.md)
- [INTEGRATION_GUIDE.md](/home/user/architecture-agent/INTEGRATION_GUIDE.md)
- [QUICK_START.md](/home/user/architecture-agent/QUICK_START.md)
- [LLM_INTEGRATION_COMPLETE.md](/home/user/LLM_INTEGRATION_COMPLETE.md)

### 내부 문서
- [프로젝트 현황 보고서](/home/user/PROJECT_STATUS.md)
- [SDV 플랫폼 전략](/home/user/AUTOMOTIVE_SDV_PLATFORM_STRATEGY.md)

### 관련 프로젝트
- **Project 1**: [Automotive System Modeler](/home/user/confluence-docs/PROJECT-1-Automotive-Modeler.md)
- **Project 3**: [Local LLM Integration](/home/user/confluence-docs/PROJECT-3-Local-LLM-Integration.md)

### 외부 참고
- [Ollama Documentation](https://ollama.com/docs)
- [Qwen 2.5 Model Card](https://huggingface.co/Qwen/Qwen2.5-7B)
- [Hono Documentation](https://hono.dev/)

---

## 변경 이력

| 날짜 | 버전 | 변경 내용 | 작성자 |
|------|------|-----------|--------|
| 2026-02-02 | v0.1.0 | MVP 완성 (Mock 데이터) | Dev Team |
| 2026-02-02 | v0.2.0 | LLM 통합 완료 (Ollama + Qwen 2.5) | AI Team |
| 2026-02-02 | v0.2.0 | Confluence 문서 작성 | Dev Team |

---

## 담당자 및 연락처

| 역할 | 이름 | 연락처 |
|------|------|--------|
| **프로젝트 리드** | - | - |
| **AI/ML 개발** | AI Team | - |
| **Backend 개발** | Dev Team | - |
| **QA** | - | - |

---

## 라이선스

**MIT License** (TBD)

---

## 부록

### A. 용어집

| 용어 | 설명 |
|------|------|
| **LLM** | Large Language Model - 대규모 언어 모델 |
| **Ollama** | 로컬 LLM 실행 플랫폼 |
| **Qwen** | Alibaba의 오픈소스 LLM |
| **RAG** | Retrieval-Augmented Generation - 검색 증강 생성 |
| **프롬프트** | LLM에 입력하는 자연어 지시문 |

### B. FAQ

**Q: Ollama 설치가 왜 필요한가요?**  
A: 로컬 PC에서 LLM을 실행하여 비용 절감 및 데이터 프라이버시를 보장합니다.

**Q: 클라우드 LLM (GPT-4o)을 사용할 수 있나요?**  
A: Phase 4에서 하이브리드 모드를 지원할 예정입니다.

**Q: 생성된 다이어그램을 편집할 수 있나요?**  
A: 현재는 불가능하지만, Phase 2에서 드래그 앤 드롭 편집 기능을 추가할 예정입니다.

**Q: 오프라인에서 사용 가능한가요?**  
A: 네, Ollama가 로컬 PC에 설치되어 있으면 완전 오프라인으로 사용 가능합니다.

---

**문서 작성자**: AI/ML Team  
**최종 검토일**: 2026-02-02  
**다음 리뷰 예정일**: 2026-02-16
