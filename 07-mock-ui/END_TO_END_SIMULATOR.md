# End-to-End System Flow Simulator

## 📋 개요

FleetFlag 시스템의 **전체 동작 흐름을 시각적으로 시뮬레이션**하는 인터랙티브 도구입니다. Admin Console에서 Feature Flag를 생성한 후, 서버를 거쳐 차량에 배포되고 최종적으로 분석 데이터가 수집되기까지의 **End-to-End 프로세스**를 실시간으로 확인할 수 있습니다.

## 🎯 목적

1. **시스템 이해도 향상**: 전체 아키텍처와 데이터 흐름을 직관적으로 파악
2. **교육 및 데모**: Stakeholder에게 시스템 동작 방식 시연
3. **문제 해결**: 각 단계별 데이터 페이로드 확인으로 디버깅 지원
4. **성능 예측**: 예상 지연시간 및 처리량 시뮬레이션

## 🎬 시뮬레이션 시나리오 (3가지)

### 1. Feature 생성 및 배포 (기본 시나리오)
**흐름**: Admin Console → API Server → Flipt Engine → PostgreSQL → Redis → Vehicle Agent → SDK → ClickHouse

**주요 단계**:
1. **Admin 요청**: 이성룡 부장이 "음성인식 AI (GPT-4o)" Feature 생성
2. **API 검증**: ASIL-B 등급 검증 및 BOM 의존성 체크
3. **Flipt 등록**: OpenFeature 표준으로 Flag 등록
4. **DB 저장**: PostgreSQL에 메타데이터 저장
5. **캐시 갱신**: Redis Pub/Sub로 5,243명의 구독자에게 알림
6. **차량 동기화**: 1,250대 차량이 Delta Sync 수행 (143ms)
7. **SDK 평가**: 로컬 캐시에서 2.1ms 만에 평가 완료
8. **분석 수집**: ClickHouse에 4,523건의 평가 이벤트 배치 저장

**예상 총 소요시간**: 5초  
**평균 지연시간**: 2.1ms (SDK 평가)  
**캐시 적중률**: 98%

---

### 2. Kill Switch 긴급 발동
**흐름**: Admin Console → API Server → PostgreSQL → Redis Pub/Sub → WebSocket → Vehicle Agent → SDK → Slack

**주요 단계**:
1. **Admin 요청**: 김철수 CTO가 "자율주행 L3" Feature에 Kill Switch 발동
   - 사유: Mobileye EyeQ5 센서 펌웨어 결함
   - 2FA 인증 완료
2. **API 처리**: Emergency 우선순위로 즉시 처리 (Validation 생략)
3. **DB 기록**: Kill Switch 상태 및 감사 로그 기록
4. **Redis 브로드캐스트**: CRITICAL 우선순위로 12,847명의 구독자에게 알림
5. **WebSocket 푸시**: 2.8초 만에 12,800대 차량에 전파 (47대 오프라인)
6. **SDK 강제 비활성화**: 로컬 캐시 무시하고 즉시 false로 설정
7. **Slack 알림**: #safety-critical 채널에 P1 알림 발송
8. **감사 로그**: 전파 시간 2시간 43분 (목표 3시간 이내 달성)

**예상 총 소요시간**: 3초  
**전파 완료율**: 99.6% (12,800 / 12,847)  
**오프라인 차량**: 47대 (재연결 시 자동 적용)

---

### 3. A/B 테스트 실행
**흐름**: Admin Console → API Server → PostgreSQL → Vehicle Agent → SDK → ClickHouse (14일 후 결과 분석)

**주요 단계**:
1. **Admin 요청**: 박지현 책임이 "음성인식 엔진 비교" 실험 시작
   - Control: 기존 엔진
   - Variant A: GPT-4o
   - Variant B: Whisper
   - 트래픽 분할: 33% / 33% / 34%
2. **Sticky Session 설정**: VIN 기반 해싱으로 차량별 고정 그룹 할당
3. **DB 저장**: 실험 설정 및 기간 저장 (14일)
4. **차량 할당**: 10,234대를 3개 그룹으로 균등 배분
5. **SDK 평가**: 각 차량은 할당된 그룹의 Flag 평가
6. **메트릭 수집**: 14일간 2,847,231건의 평가 이벤트 수집
7. **결과 분석**:
   - Control: 인식률 94.2%, 만족도 3.8
   - **Variant A (GPT-4o)**: 인식률 98.7%, 만족도 4.6 ✅ **승자**
   - Variant B (Whisper): 인식률 97.1%, 만족도 4.2
   - 통계적 유의성: p-value < 0.001, 95% 신뢰도

**예상 총 소요기간**: 14일  
**총 평가 횟수**: 2,847,231회  
**승자**: Variant A (GPT-4o) - 인식률 +4.5%p, 만족도 +21.1%

---

## 🖥️ 화면 구성

### 1. 시나리오 선택
3개의 시나리오 중 하나를 선택:
- **Feature 생성 및 배포** (기본)
- **Kill Switch 긴급 발동**
- **A/B 테스트 실행**

### 2. 시스템 흐름도 (Interactive Diagram)
7개의 핵심 컴포넌트를 시각화:
- **Admin Console** (React Dashboard)
- **FleetFlag API** (Rust/Axum)
- **Flipt Engine** (OpenFeature)
- **PostgreSQL** (Metadata Store)
- **Redis Cluster** (Cache + Pub/Sub)
- **ClickHouse** (Analytics DB)
- **Vehicle Agent** (Rust + SQLite)
- **Feature Flag SDK** (C++/Python/Rust)

**인터랙션**:
- 각 노드가 활성화되면 **녹색 테두리 + 펄스 애니메이션**
- 화살표가 활성화되면 **녹색 점선 애니메이션** (데이터 흐름 표시)
- 노드 hover 시 **확대 효과**

### 3. 실행 단계 (Progress Timeline)
7단계 프로그레스 바:
1. Admin 요청
2. API 검증
3. DB 저장
4. 캐시 갱신
5. 차량 동기화
6. SDK 평가
7. 분석 수집

**색상 코드**:
- 회색: 대기 중
- 파란색: 진행 중 (펄스 애니메이션)
- 녹색: 완료

### 4. 실시간 로그 (Live Log Stream)
각 단계의 로그 메시지를 실시간으로 표시:
```
[14:30:22] Admin (이성룡 부장) - POST /api/v1/flags
[14:30:23] API Server - Validation passed. ASIL-B requires team lead approval.
[14:30:23] Flipt Engine - Flag registered with OpenFeature spec
[14:30:24] PostgreSQL - INSERT into flags table (ID: 1234)
[14:30:24] Redis - PUBLISH flag:update:feature-voice-ai-gpt4o
...
```

**색상 코드**:
- 파란색: 정보 (info)
- 녹색: 성공 (success)
- 빨간색: 오류 (error)

### 5. 데이터 페이로드 (JSON Viewer)
각 단계에서 전송/저장되는 실제 데이터 구조를 JSON 형식으로 표시:
```json
{
  "key": "feature-voice-ai-gpt4o",
  "name": "음성인식 AI (GPT-4o)",
  "type": "boolean",
  "asil_level": "ASIL-B",
  "default_value": false,
  "rollout_strategy": "canary",
  "target_percentage": 1
}
```

**구문 강조**:
- 키워드: 파란색
- 문자열: 노란색
- 숫자/불린: 녹색

### 6. 성능 메트릭 (Real-time Metrics)
4개의 KPI 카드:
- **평균 지연시간**: 0ms → 2.1ms
- **동기화된 차량**: 0대 → 1,250대
- **평가 횟수**: 0회 → 4,523회
- **캐시 적중률**: 0% → 98%

---

## 🎨 기술 스택

### 프론트엔드
- **HTML5 + Tailwind CSS**: 레이아웃 및 스타일링
- **SVG**: 화살표 및 애니메이션
- **Vanilla JavaScript**: 시뮬레이션 로직 (프레임워크 없음)

### 애니메이션
- **CSS Animations**: pulse, dash, slideIn
- **Keyframes**: 60fps 부드러운 애니메이션
- **Transitions**: 0.3s ease 트랜지션

### 데이터 구조
- **시나리오 객체**: 각 시나리오의 단계별 데이터 정의
- **상태 관리**: isRunning, currentStep, logIndex

---

## 🚀 사용 방법

### 1. 로컬 실행
```bash
# 웹 서버 실행
cd /home/user/fleetflag-specification/07-mock-ui
python3 -m http.server 8000

# 브라우저에서 접속
open http://localhost:8000/07-end-to-end-simulator.html
```

### 2. 시뮬레이션 실행
1. **시나리오 선택**: 3개 중 하나 선택 (기본: Feature 생성 및 배포)
2. **시작 버튼 클릭**: 녹색 "시뮬레이션 시작" 버튼
3. **자동 진행**: 각 단계가 2초 간격으로 자동 진행
4. **로그 확인**: 실시간 로그 및 데이터 페이로드 확인
5. **완료**: "✅ 시뮬레이션 완료!" 메시지

### 3. 초기화
- **초기화 버튼**: 회색 "초기화" 버튼으로 모든 상태 리셋

---

## 🔍 주요 기능

### 1. 실시간 시각화
- **노드 활성화**: 현재 처리 중인 컴포넌트 강조
- **화살표 애니메이션**: 데이터 흐름 방향 표시
- **단계별 진행**: 프로그레스 바로 전체 진행률 파악

### 2. 상세 로그
- **타임스탬프**: 각 로그에 정확한 시각 표시
- **담당자 정보**: 누가 어떤 작업을 수행했는지 표시
- **자동 스크롤**: 새 로그 추가 시 자동 하단 스크롤

### 3. 데이터 투명성
- **JSON 페이로드**: 각 단계의 실제 데이터 구조 확인
- **구문 강조**: 키워드, 문자열, 숫자 색상 구분
- **검증 결과**: Validation, BOM Check 등 확인 가능

### 4. 성능 시뮬레이션
- **지연시간 추적**: ms 단위 정확한 지연시간
- **차량 수 집계**: 동기화된 차량 실시간 카운트
- **평가 횟수**: 누적 평가 이벤트 추적
- **캐시 효율**: 캐시 적중률 실시간 업데이트

---

## 📝 시나리오별 핵심 포인트

### Feature 생성 및 배포
- **BOM 의존성 검증**: API 단계에서 자동 체크
- **Canary 배포**: 1%부터 점진적 배포
- **Delta Sync**: 변경된 Flag만 동기화 (효율성)
- **로컬 캐시**: 2.1ms 초저지연 평가

### Kill Switch 긴급 발동
- **2FA 인증**: ASIL-D 등급 보안 강화
- **Validation 생략**: Emergency 우선순위로 즉시 처리
- **WebSocket 푸시**: 2.8초 만에 12,800대 전파
- **Slack 알림**: #safety-critical 채널 실시간 알림

### A/B 테스트 실행
- **VIN 기반 Sticky Session**: 동일 차량은 고정 그룹
- **균등 배분**: 33% / 33% / 34% 정확한 트래픽 분할
- **14일 수집**: 2,847,231건의 평가 이벤트
- **통계적 유의성**: p-value < 0.001 검증

---

## 🎓 교육 활용

### 1. 신규 개발자 온보딩
- 전체 시스템 아키텍처 이해
- 각 컴포넌트의 역할 파악
- 데이터 흐름 학습

### 2. Stakeholder 데모
- CTO, Safety Team에게 시스템 동작 시연
- 비기술 팀에게 시각적으로 설명
- 승인 단계에서 설득 자료

### 3. 문제 해결 시뮬레이션
- 특정 단계에서 오류 발생 시나리오
- 롤백 프로세스 이해
- Kill Switch 긴급 대응 훈련

---

## 🔧 커스터마이징

### 새 시나리오 추가
`scenarios` 객체에 새 시나리오 추가:
```javascript
scenarios['custom-scenario'] = {
    name: '커스텀 시나리오',
    steps: [
        {
            node: 'node-admin',
            arrow: 'arrow-1',
            step: 'step-1',
            title: '첫 번째 단계',
            log: '[시간] 로그 메시지',
            payload: { key: 'value' },
            latency: 10,
            vehicles: 0,
            evaluations: 0,
            cache: 0
        }
        // ... 추가 단계
    ]
};
```

### 타이밍 조정
`await sleep(2000);` 부분을 수정하여 단계 간 대기 시간 조정 (기본: 2초)

---

## 📊 통계 및 메트릭

### 시뮬레이터 통계
- **총 시나리오 수**: 3개
- **총 단계 수**: 평균 7-8단계
- **애니메이션 프레임**: 60fps
- **로그 표시 속도**: 즉시 (< 10ms)
- **JSON 렌더링**: 구문 강조 적용

### 성능 목표
- **UI 반응속도**: < 50ms
- **애니메이션 부드러움**: 60fps 유지
- **메모리 사용량**: < 100MB
- **CPU 사용률**: < 5% (idle)

---

## 🔗 관련 문서

- [시스템 아키텍처 설계서](../02-architecture/system-architecture.md)
- [API 인터페이스 명세서](../03-interface/api-specification.md)
- [UI 와이어프레임](../06-ui-ux/ui-wireframes.md)
- [종합 개발 가이드](../DEVELOPMENT_GUIDE.md)

---

## 📞 문의

- **기술 문의**: fleetflag-dev@hyundai.com
- **시뮬레이터 개선 제안**: GitHub Issues

---

**최종 업데이트**: 2026-02-01  
**버전**: v1.0.0  
**작성자**: FleetFlag 개발팀
