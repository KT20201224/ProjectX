# 🗺️ Project X: 개발 로드맵 및 마일스톤 (Development Roadmap)

**버전:** v1.0
**기반 문서:** Project X 시스템 아키텍처 및 기획 문서 시리즈

본 문서는 **"AI-Driven Hardcore Dark Fantasy RPG" (Project X)**의 성공적인 개발을 위한 단계별 실행 계획(로드맵)입니다. 백엔드 시스템 구축부터 마인크래프트 클라이언트(Fabric) 통신, 그리고 고도화된 AI 파이프라인(RAG, CAG, Streaming) 연동까지 점진적으로 개발하고 테스트할 수 있도록 설계되었습니다.

---

## 📅 전체 배포 일정 마일스톤 (Milestones)

- **[Milestone 1] Core Backbone (M1):** FastAPI 서버와 초기 AI 텍스트 생성 테스트 완료
- **[Milestone 2] Data & Interaction (M2):** MongoDB/Redis 데이터 연동 및 WebSocket 스트리밍 기반 실시간 채팅 구현
- **[Milestone 3] AI Pipeline & RPG Logic (M3):** RAG(기억), CAG(지식) 파이프라인 및 성향/숙련도 계산 로직 통합
- **[Milestone 4] Minecraft Integration (M4):** Java Fabric 클라이언트 개발 및 백엔드 완벽 동기화 (알파 테스트)
- **[Milestone 5] Final Boss & Endings (M5):** 보스 '테네브리스' AI 고도화 및 엔딩 분기 구현 (베타 배포)

---

## 🛠️ 단계별 세부 개발 스크린트 (Sprints)

### 🟢 [Sprint 1] 백엔드 및 AI 기초 공사 (Core Backbone)
**목표:** FastAPI 아키텍처를 세우고 가장 단순한 형태의 AI 텍스트 생성 파이프라인(Chat Endpoint) 구축.

- **Task 1.1:** Python 환경 세팅 (FastAPI, Uvicorn, LangChain 설치)
- **Task 1.2:** 프로젝트 기본 디렉터리 구조 설계 (routers, models, services, core)
- **Task 1.3:** 단일 `POST` 엔드포인트 `/v1/chat/test` 생성 및 OpenAI(또는 로컬 LLM) API 연결
- **Task 1.4:** '아리아'의 기본 페르소나 시스템 프롬프트(System Prompt) 작성 및 단순 대화 테스트
- **Task 1.5:** LLM 응답을 파싱하여 **[텍스트 + JSON(성향 분석 점수)]**로 안전하게 분리 추출하는 로직 구현

---

### 🟡 [Sprint 2] 실시간 스트리밍 및 상태 관리 (Streaming & Data)
**목표:** RPG 코어 구조인 DB를 연동하고, 유저-AI 간의 대화를 실시간 WebSocket으로 렌더링.

- **Task 2.1:** MongoDB 환경 세팅 및 ODM(Beanie 또는 Motor) 연동
- **Task 2.2:** `users`, `stats` Collection 초기 스키마 구현 및 CRUD API 개발
- **Task 2.3:** Redis 연동 및 유저 세션(UUID) 기반 접속/위치 상태 캐싱 구현
- **Task 2.4:** WebSocket (`/v1/stream/aria`) 엔드포인트 개설
- **Task 2.5:** LLM의 생성 텍스트를 청크 단위로 클라이언트로 Push하는 **스트리밍 파이프라인** 적용 및 에러 핸들링 로직

---

### 🟠 [Sprint 3] 하드코어 시스템 및 고도화 파이프라인 (RAG & RPG Logic)
**목표:** 지식 검색(CAG)과 기억 검색(RAG) 파이프라인 완성, 게임의 본질인 데스 패널티 로직 추가.

- **Task 3.1:** Vector Database(Milvus 또는 Pinecone 등) 연동
- **Task 3.2:** _(CAG)_ 게임 내 몬스터 도감, 상인 성격, 세계관 등 '로어 북(Lore)' 텍스트의 임베딩 파이프라인 구축
- **Task 3.3:** _(RAG)_ 첫 만남 대화 로그, 각성 면접 등 유저의 채팅 로그 벡터화 및 저장 로직 구현
- **Task 3.4:** 채팅 요청 시, VectorDB에서 관련 지식/기억을 검색해 프롬프트에 주입하는 **RAG/CAG 체인 조합(Chain Routing)** 완성
- **Task 3.5:** 전투 액션(칼질 등) 수신 시 Redis에 `Mastery Points (MP)` 캐싱 및 1분 벌크 업데이트 스케줄러(Celery/APScheduler) 작성
- **Task 3.6:** [하드코어 로직] 사망 패널티 기록 및 동기화 지수 차감 연산 

---

### 🔵 [Sprint 4] 마인크래프트 클라이언트 통합 (Fabric Integration)
**목표:** 준비된 백엔드 시스템과 실제 게임(Minecraft)을 동기화하여 알파 빌드 생성.

- **Task 4.1:** Fabric 모드 템플릿 환경 구성 및 JSON 통신, WebSocket 라이브러리 연동
- **Task 4.2:** 커스텀 UI/GUI 오버레이 개발 (AI 채팅 스트리밍 텍스트가 표시될 반투명 인게임 UI)
- **Task 4.3:** 몬스터 데미지, 플레이어 사망 등 바닐라 이벤트 훅 가로채기(Mixin) 및 FastAPI로 전송
- **Task 4.4:** 서버로부터 스탯 업데이트(STR, DEF 등) JSON 패킷 수신 시 마인크래프트 내부 Attributes 강제 동기화 
- **Task 4.5:** 상인 AI NPC 렌더링 및 클릭 시 흥정 전용 GUI 팝업 개발

---

### 🟣 [Sprint 5] 신화 템 생성 및 보스 시스템 (The Endgame)
**목표:** RAG 성능을 극대화한 최종 보스 페이즈 연출 및 AI 생성형 전리품 보상 시스템 구축.

- **Task 5.1:** 0.1% 확률 'Mythic' 등급 무기 드랍 시 즉석 이름 및 플레이버 텍스트 생성 프롬프트 로직 적용
- **Task 5.2:** 백엔드에서 생성된 난수급 수치(공격력, 옵션)를 마인크래프트 NBT 데이터로 즉시 파싱 및 유저에게 지급 연동
- **Task 5.3:** '테네브리스' 보스 공격 패턴 매니저(AI Data Mirroring 로직) 구현 (유저 스탯 비례 스케일링)
- **Task 5.4:** 보스전 도중 유저의 과거 모순을 담은 '정신 공격 대사' 실시간 생성 및 발송 스트림 (RAG 극대화 연출)
- **Task 5.5:** 최종 동기화 지수에 따른 3가지 멀티 엔딩 컷신 및 로직 배포

---

## 🔧 위험 요소 및 관리 방안 (Risk Management)

1. **지연(Latency) 이슈:** 언어모델 속도로 인한 인게임 끊김 
   - *대응:* LLM 파이프라인에서 Time-to-First-Token(초기 응답시간) 최적화, Fabric 클라이언트 단의 비동기 마스킹(파티클 연출) 적용 필수.
2. **프롬프트 인젝션:** 상인 NPC 무료 제공 어뷰징 
   - *대응:* 텍스트 파싱을 서버에서 전담, 서버측 할인율 Clamp(0~20%) 하드코딩 필트링 도입. (Sprint 1/2에서 엄격한 단위 테스트 진행)
3. **토큰(Token) 비용:** 과도한 채팅으로 인한 API 요금 폭탄
   - *대응:* RAG 검색 시 유사도(Similarity) 허들을 높여 불필요한 컨텍스트 주입 제거. 단순 상호작용은 Local LLM (Llama 3 등)으로 모델 경량화 우회 구조 설계.

---
*(※ 위 로드맵은 개발 진행 상황과 API 파이프라인 안정화에 따라 탄력적으로 조정됩니다.)*
