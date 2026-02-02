# 📚 Confluence Documentation Index

> **프로젝트 포트폴리오**  
> **최종 업데이트**: 2026-02-02  
> **관리 팀**: Development & Strategy Team

---

## 🎯 Overview

이 공간은 자동차 소프트웨어 개발을 위한 AI 기반 도구 및 플랫폼 프로젝트들의 중앙 문서 저장소입니다.

### 프로젝트 현황

| 프로젝트 | 코드 | 상태 | 진행률 | 담당 팀 | URL |
|---------|------|------|--------|---------|-----|
| **Automotive System Modeler** | WEBAPP-001 | ✅ 운영 중 | 100% | Dev Team | [바로가기](https://3000-ifl0a91rhl2v80gsfr4rn-8f57ffe2.sandbox.novita.ai) |
| **Architecture Agent** | AGENT-002 | ✅ 운영 중 | 100% | AI/ML Team | [바로가기](https://3001-ifl0a91rhl2v80gsfr4rn-8f57ffe2.sandbox.novita.ai) |
| **SDV Platform Strategy** | SDV-003 | 📋 계획 | 5% | Strategy Team | - |

---

## 📖 프로젝트 문서

### 1. 🚗 Automotive System Modeler
**프로젝트 코드**: WEBAPP-001  
**상태**: ✅ Production  
**포트**: 3000

**개요**:  
웹 기반 UML/SysML 다이어그램 편집기로, 자동차 시스템 아키텍처를 드래그 앤 드롭으로 설계하는 도구입니다.

**주요 기능**:
- 드래그 앤 드롭 컴포넌트 배치
- ECU, 센서, 액추에이터 간 신호 연결
- JSON Export/Import
- 실시간 편집

**문서**:
- 📄 [전체 문서 보기](PROJECT-1-Automotive-Modeler.md)
- 🔗 [라이브 데모](https://3000-ifl0a91rhl2v80gsfr4rn-8f57ffe2.sandbox.novita.ai)
- 📊 [API 문서](#api-문서-webapp-001)
- 👨‍💻 [개발 가이드](#개발-가이드-webapp-001)

**기술 스택**:
- Frontend: HTML5, Tailwind CSS, Vanilla JS
- Backend: Hono, TypeScript
- Deployment: Cloudflare Pages, PM2

---

### 2. 🤖 Architecture Agent (LLM Integration)
**프로젝트 코드**: AGENT-002  
**상태**: ✅ Production  
**포트**: 3001

**개요**:  
자연어 프롬프트로 소프트웨어 아키텍처 다이어그램을 자동 생성하는 AI 기반 도구입니다. Ollama LLM (Qwen 2.5) 통합.

**주요 기능**:
- 자연어 프롬프트 입력
- LLM 기반 자동 다이어그램 생성 (3-5초)
- 실시간 연결 상태 확인
- AI Assistant 대화형 인터페이스
- JSON Export

**문서**:
- 📄 [전체 문서 보기](PROJECT-2-Architecture-Agent.md)
- 🔗 [라이브 데모](https://3001-ifl0a91rhl2v80gsfr4rn-8f57ffe2.sandbox.novita.ai)
- 🛠️ [LLM 설치 가이드](#llm-설치-가이드)
- 📊 [API 문서](#api-문서-agent-002)
- 🔬 [성능 벤치마크](#성능-벤치마크)

**기술 스택**:
- Frontend: HTML5, Tailwind CSS, Vanilla JS
- Backend: Hono, TypeScript
- AI: Ollama, Qwen 2.5:7B
- Deployment: Cloudflare Pages, PM2

---

### 3. 🚀 Automotive SDV Platform Strategy
**프로젝트 코드**: SDV-003  
**상태**: 📋 Planning  
**진행률**: 5%

**개요**:  
자동차 소프트웨어 개발을 위한 전용 AI 플랫폼 전략. Genspark와 유사한 수직 통합 생태계 구축.

**주요 전략**:
- **AutoBox**: 온디바이스 LLM (Mac mini 기반)
- **AutoDev Cloud**: 웹 기반 SaaS
- **AutoStudio**: VS Code 포크 (SDV 전용)
- **AutoSearch**: Chromium 포크 (브라우저)

**문서**:
- 📄 [전체 전략 문서 보기](PROJECT-3-SDV-Platform-Strategy.md)
- 📈 [시장 분석](#시장-분석)
- 🗺️ [로드맵](#로드맵-year-1-3)
- 💰 [투자 계획](#투자-계획)
- ⚠️ [위험 분석](#위험-분석)

**타임라인**:
- **2026 Q1-Q4**: AutoBox v1.0 출시
- **2027**: AutoDev Cloud 출시
- **2028+**: 전체 생태계 통합

---

## 🔗 빠른 링크

### 운영 URL
- **Automotive Modeler**: https://3000-ifl0a91rhl2v80gsfr4rn-8f57ffe2.sandbox.novita.ai
- **Architecture Agent**: https://3001-ifl0a91rhl2v80gsfr4rn-8f57ffe2.sandbox.novita.ai

### Git 저장소
- **Automotive Modeler**: `/home/user/webapp/`
- **Architecture Agent**: `/home/user/architecture-agent/`
- **SDV Strategy**: `/home/user/local-llm-integration/`

### 주요 문서
- [프로젝트 현황 보고서](/home/user/PROJECT_STATUS.md)
- [프로젝트 분리 보고서](/home/user/PROJECT_SEPARATION_REPORT.md)
- [LLM 통합 완료 가이드](/home/user/LLM_INTEGRATION_COMPLETE.md)
- [SDV 플랫폼 전략](/home/user/AUTOMOTIVE_SDV_PLATFORM_STRATEGY.md)

---

## 📊 프로젝트 통계

### 개발 현황

| 항목 | Project 1 | Project 2 | Project 3 |
|------|-----------|-----------|-----------|
| **코드 라인** | ~1,500 | ~1,200 | N/A |
| **파일 수** | 12 | 14 | 5 (문서) |
| **Git 커밋** | 5 | 2 | 1 |
| **배포 상태** | ✅ Active | ✅ Active | 📋 Planning |

### 사용자 통계 (예상)

| 항목 | Project 1 | Project 2 |
|------|-----------|-----------|
| **일일 사용자** | - | - |
| **다이어그램 생성** | - | - |
| **평균 응답 시간** | <200ms | 3-5초 (LLM) |

---

## 🛠️ 개발 환경

### 필수 도구
- **Node.js**: v18+
- **npm**: v9+
- **PM2**: Latest
- **Ollama**: Latest (Project 2)
- **Git**: Latest

### 샌드박스 환경
- **Home Directory**: `/home/user/`
- **Project 1 Path**: `/home/user/webapp/`
- **Project 2 Path**: `/home/user/architecture-agent/`
- **Docs Path**: `/home/user/confluence-docs/`

### 포트 할당
- **3000**: Automotive System Modeler
- **3001**: Architecture Agent
- **11434**: Ollama LLM (사용자 PC)

---

## 📋 API 문서 (WEBAPP-001)

### GET /
메인 페이지 반환

### GET /api/model
현재 모델 조회

**Response**:
```json
{
  "elements": [...],
  "connections": [...]
}
```

### POST /api/model
모델 저장

### POST /api/model/reset
모델 초기화

**자세한 내용**: [Project 1 API 문서](PROJECT-1-Automotive-Modeler.md#api-문서)

---

## 📋 API 문서 (AGENT-002)

### POST /api/generate
LLM으로 다이어그램 생성

**Request**:
```json
{
  "prompt": "마이크로서비스 아키텍처 만들어줘"
}
```

**Response**:
```json
{
  "success": true,
  "architecture": {
    "name": "...",
    "elements": [...],
    "connections": [...],
    "explanation": "..."
  }
}
```

### GET /api/health
LLM 연결 상태 확인

**자세한 내용**: [Project 2 API 문서](PROJECT-2-Architecture-Agent.md#api-문서)

---

## 🔧 LLM 설치 가이드

### Windows
```powershell
# 1. Ollama 다운로드
# https://ollama.com/download/windows

# 2. 모델 다운로드
ollama pull qwen2.5:7b

# 3. 네트워크 노출
setx OLLAMA_HOST "0.0.0.0:11434"
ollama serve
```

### macOS / Linux
```bash
# 1. Ollama 설치
curl -fsSL https://ollama.com/install.sh | sh

# 2. 모델 다운로드
ollama pull qwen2.5:7b

# 3. 네트워크 노출
export OLLAMA_HOST=0.0.0.0:11434
ollama serve
```

**자세한 내용**: [LLM 통합 가이드](/home/user/architecture-agent/INTEGRATION_GUIDE.md)

---

## 📈 성능 벤치마크

### Architecture Agent (LLM)

| 모델 | 응답 시간 | 메모리 | 품질 |
|------|-----------|--------|------|
| **Qwen 2.5:7B** | 2-3초 | ~8GB | ⭐⭐⭐⭐⭐ |
| **Llama 3.2:3B** | 1-2초 | ~4GB | ⭐⭐⭐⭐ |
| **Mistral:7B** | 4-6초 | ~8GB | ⭐⭐⭐⭐ |

**자세한 내용**: [성능 벤치마크](PROJECT-2-Architecture-Agent.md#성능-및-벤치마크)

---

## 🗺️ 로드맵 (Year 1-3)

### 2026 Q1
- [x] 전략 수립
- [x] Project 1 MVP 완성
- [x] Project 2 LLM 통합
- [ ] AutoBox 프로토타입

### 2026 Q2-Q4
- [ ] AutoBox v1.0 출시
- [ ] 초기 고객 확보 (50 기업)
- [ ] VS Code / Chromium 포크 시작

### 2027
- [ ] AutoDev Cloud 출시
- [ ] SaaS 모델 런칭
- [ ] 매출 $2M 달성

### 2028+
- [ ] 전체 생태계 통합
- [ ] 글로벌 확장
- [ ] 매출 $10M+ 달성

**자세한 내용**: [전체 로드맵](PROJECT-3-SDV-Platform-Strategy.md#로드맵)

---

## 💰 투자 계획

### Year 1 예산: $550K

| 항목 | 비용 |
|------|------|
| 팀 (5명, 6개월) | $300K |
| 하드웨어 (Mac mini 50대) | $70K |
| 인프라 (클라우드) | $30K |
| 라이선스 검토 | $20K |
| 문서 구매 (AUTOSAR, ISO) | $10K |
| 마케팅 | $50K |
| 기타 운영비 | $70K |

### 수익 예상

| 연도 | ARR |
|------|-----|
| 2026 | $500K |
| 2027 | $2M |
| 2028 | $10M |

**자세한 내용**: [투자 계획](PROJECT-3-SDV-Platform-Strategy.md#투자-계획)

---

## ⚠️ 위험 분석

| 위험 | 확률 | 영향 | 완화 전략 |
|------|------|------|-----------|
| 시장 수용 부족 | Medium | High | MVP 빠른 출시 |
| 기술적 난이도 | Medium | Medium | 오픈소스 활용 |
| 경쟁사 진입 | High | Medium | 빠른 시장 선점 |
| 규제 변화 | Low | High | 표준 준수 |

**자세한 내용**: [위험 분석](PROJECT-3-SDV-Platform-Strategy.md#위험-분석)

---

## 👥 팀 구성

| 역할 | 인원 | 담당 프로젝트 |
|------|------|---------------|
| **프로젝트 리드** | 1 | 전체 |
| **Frontend 개발** | 2 | Project 1, 2 |
| **Backend 개발** | 1 | Project 1, 2 |
| **AI/ML 엔지니어** | 2 | Project 2 |
| **전략/비즈니스** | 1 | Project 3 |
| **QA** | 1 | 전체 |

---

## 📞 문의 및 지원

### 내부 문의
- **기술 문의**: Dev Team
- **전략 문의**: Strategy Team
- **버그 리포트**: QA Team

### 외부 참고
- [AUTOSAR Official](https://www.autosar.org/)
- [ISO 26262 Standard](https://www.iso.org/standard/68383.html)
- [Ollama Documentation](https://ollama.com/docs)
- [Hono Documentation](https://hono.dev/)

---

## 📝 변경 이력

| 날짜 | 변경 내용 | 작성자 |
|------|-----------|--------|
| 2026-02-02 | Confluence 문서 공간 생성 | Dev Team |
| 2026-02-02 | Project 1-3 문서 작성 | Dev Team |
| 2026-02-02 | 인덱스 페이지 생성 | Dev Team |

---

## 🏷️ 태그

`automotive` `sdv` `ai` `llm` `architecture` `autosar` `iso26262` `hono` `cloudflare` `ollama` `qwen`

---

**문서 관리자**: Development Team  
**최종 업데이트**: 2026-02-02  
**다음 리뷰 예정일**: 2026-02-16 (격주 리뷰)

---

## 🎯 다음 액션 아이템

- [ ] Ollama 설치 (사용자 PC)
- [ ] AutoBox 프로토타입 개발 시작
- [ ] 초기 고객 인터뷰 (5명)
- [ ] 월례 팀 미팅 (2026-02-15)
