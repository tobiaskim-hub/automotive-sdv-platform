# Confluence Documentation

이 디렉토리는 모든 프로젝트의 Confluence 스타일 문서를 포함합니다.

## 📚 문서 구조

```
confluence-docs/
├── INDEX.md                              # 메인 인덱스 (시작 페이지)
├── PROJECT-1-Automotive-Modeler.md       # Project 1 상세 문서
├── PROJECT-2-Architecture-Agent.md       # Project 2 상세 문서
├── PROJECT-3-SDV-Platform-Strategy.md    # Project 3 전략 문서
└── README.md                             # 이 파일
```

## 🚀 빠른 시작

### 메인 인덱스 보기
[INDEX.md](INDEX.md) - 모든 프로젝트의 개요 및 빠른 링크

### 프로젝트별 문서

1. **[Project 1: Automotive System Modeler](PROJECT-1-Automotive-Modeler.md)**
   - 웹 기반 UML/SysML 다이어그램 편집기
   - 드래그 앤 드롭 컴포넌트 배치
   - 상태: ✅ Production
   - 포트: 3000

2. **[Project 2: Architecture Agent](PROJECT-2-Architecture-Agent.md)**
   - LLM 기반 아키텍처 자동 생성 도구
   - Ollama + Qwen 2.5 통합
   - 상태: ✅ Production
   - 포트: 3001

3. **[Project 3: SDV Platform Strategy](PROJECT-3-SDV-Platform-Strategy.md)**
   - 자동차 소프트웨어 개발 플랫폼 전략
   - AutoBox + AutoDev Cloud + 생태계
   - 상태: 📋 Planning

## 📖 문서 스타일 가이드

### Confluence 스타일 특징
- ✅ 표준화된 섹션 구조
- ✅ Table of Contents
- ✅ 상태 배지 및 메타데이터
- ✅ 표를 활용한 정보 정리
- ✅ 코드 블록 및 예시
- ✅ 관련 문서 링크
- ✅ 변경 이력 추적

### 필수 섹션
모든 프로젝트 문서는 다음 섹션을 포함합니다:

1. **개요** (Overview)
2. **프로젝트 목표** (Goals & Objectives)
3. **기술 스택** (Tech Stack)
4. **주요 기능** (Key Features)
5. **아키텍처** (Architecture)
6. **배포 정보** (Deployment)
7. **API 문서** (API Documentation)
8. **사용자 가이드** (User Guide)
9. **개발 가이드** (Developer Guide)
10. **이슈 및 제한사항** (Issues & Limitations)
11. **로드맵** (Roadmap)
12. **관련 문서** (Related Documents)

## 🔄 문서 업데이트

### 업데이트 주기
- **주간**: 진행 상황 업데이트
- **격주**: 전체 리뷰 및 검증
- **월간**: 로드맵 및 전략 재검토

### 업데이트 프로세스
1. 해당 프로젝트 문서 수정
2. 변경 이력 섹션에 기록
3. INDEX.md의 "최종 업데이트" 날짜 갱신
4. Git 커밋 (메시지: "docs: Update PROJECT-X documentation")

## 📝 문서 템플릿

새 프로젝트 문서를 작성할 때는 기존 문서를 템플릿으로 사용하세요:

```bash
# 새 프로젝트 문서 생성 예시
cp PROJECT-1-Automotive-Modeler.md PROJECT-4-New-Project.md
```

그 후 다음 필드를 수정:
- 프로젝트 코드
- 프로젝트 이름
- 상태
- 담당자
- 모든 섹션 내용

## 🔗 외부 시스템 통합

### Confluence로 Import 방법

#### Option 1: Markdown Import (추천)
1. Confluence Space 생성
2. "..." 메뉴 → "Import"
3. "Markdown" 선택
4. 파일 업로드

#### Option 2: 수동 복사 & 붙여넣기
1. GitHub에서 Markdown 미리보기
2. 내용 복사
3. Confluence 페이지 생성
4. 붙여넣기

#### Option 3: Confluence Cloud API
```bash
# API를 통한 자동 업로드 (예시)
curl -u user@example.com:api_token \
  -X POST \
  -H "Content-Type: application/json" \
  -d @confluence-page.json \
  https://your-domain.atlassian.net/wiki/rest/api/content
```

## 🎯 사용 예시

### 프로젝트 현황 확인
```bash
# INDEX.md에서 전체 프로젝트 현황 확인
cat /home/user/confluence-docs/INDEX.md
```

### 특정 프로젝트 문서 읽기
```bash
# Project 2 문서 읽기
cat /home/user/confluence-docs/PROJECT-2-Architecture-Agent.md
```

### 문서 검색
```bash
# "LLM" 키워드로 검색
grep -r "LLM" /home/user/confluence-docs/
```

## 📊 문서 통계

| 문서 | 크기 | 섹션 수 | 마지막 업데이트 |
|------|------|---------|-----------------|
| INDEX.md | ~7.5KB | 15 | 2026-02-02 |
| PROJECT-1 | ~9.4KB | 12 | 2026-02-02 |
| PROJECT-2 | ~14.2KB | 12 | 2026-02-02 |
| PROJECT-3 | ~10.3KB | 12 | 2026-02-02 |
| **Total** | ~41.4KB | 51 | - |

## 🤝 기여 가이드

### 문서 개선 제안
1. 이슈 생성 또는 PR 제출
2. 변경 사항 명확히 설명
3. 관련 섹션 링크 포함

### 문서 품질 기준
- ✅ 명확하고 간결한 문장
- ✅ 실행 가능한 예시 코드
- ✅ 최신 정보 유지
- ✅ 링크 유효성 확인
- ✅ 스크린샷 또는 다이어그램 포함 (가능 시)

## 📞 문의

문서 관련 문의:
- 기술 문서: Dev Team
- 전략 문서: Strategy Team
- 오타/오류: Anyone (PR 환영!)

---

**Created**: 2026-02-02  
**Last Updated**: 2026-02-02  
**Version**: 1.0
