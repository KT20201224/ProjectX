# 🏗️ Project X: 시스템 아키텍처 및 게임 로직 설계도 (System Architecture)

**버전:** v1.0
**기반 문서:** Project X 기획 상세 문서 (Draft v1.2) - AI-Driven Hardcore Dark Fantasy RPG

---

## 1. 전체 시스템 개요 (System Overview)

본 시스템은 **Minecraft (Fabric)** 클라이언트를 기반으로, 강력한 AI 백엔드 서버(**FastAPI + Python**)가 게임 내 주요 로직(NPC 상호작용, 스탯 계산, 하드코어 보스 생성, 아이템 파밍)을 실시간으로 관장하는 구조입니다.

특히 **RAG (장기 기억 검색)**와 **CAG (게임 지식 기반 생성)**, 그리고 유저 경험의 단절을 막기 위한 **실시간 스트리밍(WebSocket / SSE)** 통신이 이 아키텍처의 핵심입니다.

---

## 2. 기술 스택 (Technology Stack)

| 계층 (Layer) | 주요 기술 (Technology) | 역할 (Role) |
| :--- | :--- | :--- |
| **Client** | Minecraft Java Edition, Fabric Mod Loader | 게임 렌더링, 플레이어 조작, 패킷 수신 및 인게임 UI/이펙트 재생 |
| **Backend API** | Python 3.10+, FastAPI, Uvicorn | 핵심 게임 로직 처리, AI 모델 연동 허브, WebSocket 스트리밍 세션 관리 |
| **AI Engine** | LangChain / LlamaIndex, OpenAI GPT-4o (혹은 Local LLM) | 프롬프트 체이닝, RAG/CAG 파이프라인 관리, 텍스트 생성 및 성향 분석 |
| **Database** | MongoDB (NoSQL) | 플레이어 스탯 백업, 인벤토리, 상점 거래 내역, 사망 로그 아카이브 |
| **Vector DB** | Milvus / Qdrant / Pinecone | (RAG 용) 플레이어 채팅 로그 벡터 임베딩, (CAG 용) 게임 세계관 지식 저장 |
| **Cache & MQ** | Redis | 실시간 전투 데이터(숙련도, MP) 캐싱, 세션 및 동기화 상태 고속 임시 저장 |

---

## 3. 핵심 아키텍처 구조 (Core Architecture Diagram)

```mermaid
graph TD
    %% -- Client Layer --
    subgraph Client [Minecraft Client (Fabric)]
        M_Chat[채팅 및 UI 입력]
        M_Event[전투/사망 이벤트 감지]
        M_Render[텍스트 스트리밍 렌더 & 파티클]
    end

    %% -- API Gateway & Logic Layer (FastAPI) --
    subgraph Backend [FastAPI Server]
        W_Server[WebSocket / SSE Manager]
        A_Guard[프롬프트 인젝션 방어기]
        
        Game_Logic[게임 로직 매니저]
        Game_Logic --> |"상태/경험치/스탯 계산"| Calculator[스탯 & 경제 엔진]
        
        AI_Router[AI 요청 라우터]
    end

    %% -- Data Layer --
    subgraph DataBase [Data Storage]
        Mongo[(MongoDB)]
        Redis[(Redis Cache)]
        VectorDB[(VectorDB - RAG/CAG)]
    end

    %% -- AI / LLM Layer --
    subgraph AI_Layer [AI Engine]
        LangC[LangChain Pipeline]
        LLM((LLM Model))
    end

    %% -- Data Flow --
    M_Chat <-->|WebSocket: Stream| W_Server
    M_Event -->|REST API (POST)| Game_Logic
    
    W_Server --> A_Guard
    A_Guard --> AI_Router
    
    Game_Logic <--> Redis
    Redis .-> |Bulk Update| Mongo
    
    AI_Router --> |Context Query| VectorDB
    VectorDB --> |Retrieved Docs| LangC
    LangC --> LLM
    LLM --> |Stream chunks| W_Server
    
    Game_Logic --> |DB Query| Mongo
```

---

## 4. 모듈별 상세 아키텍처 및 데이터 흐름

### 4.1 실시간 통신 및 스트리밍 모듈 (WebSocket / SSE)
- **목적:** AI 챗봇(아리아, 상인 NPC)의 긴 응답 생성 시간(1~3초) 동안 유저 이탈 방지(몰입감 유지).
- **흐름:**
  1. 클라이언트(Java)에서 유저 채팅 입력 시 WebSocket을 통해 FastAPI로 패킷 전송.
  2. FastAPI는 즉시 "데이터 스캐닝 중..." 등의 대기 이펙트 명령 반환.
  3. AI 파이프라인(LLM) 구동 후 텍스트 청크(Token)가 생산되는 즉시 WebSocket을 통해 단어 단위로 밀어냄 (Push).
  4. 클라이언트는 수신받은 텍스트를 인게임 채팅창이나 홀로그램 UI 상에 타이핑 효과로 렌더링.

### 4.2 AI 파이프라인: RAG & CAG 연동
단순 LLM 호출이 아닌, 게임 문맥상 정확한 정보를 부여하는 체인(Chain).
- **CAG (Context-Augmented Generation) 파이프라인:**
  - `VectorDB`의 'Knowledge_Base' 콜렉션에 보스 패턴, 아이템 조합, 상인 성격 등의 가이드 문서를 사전 임베딩.
  - 유저 질문(예: "가디언 약점?") 시 가장 유사한 지식을 찾아 LLM의 시스템 프롬프트(System Prompt)에 주입.
- **RAG (장기 기억 / 유저 특화 생태계) 파이프라인:**
  - `VectorDB`의 'User_Memory' 콜렉션에 플레이어가 과거에 남겼던 튜토리얼 답변, 대화 로그를 임베딩 해 둠.
  - "테네브리스(보스)" 스폰이나 NPC 각성 면접 시, 유저의 과거 명언(?)과 모순되는 점을 찾아내 정신 공격 프롬프트에 활용.

### 4.3 데이터베이스 동기화 및 캐싱 (Redis & MongoDB)
RPG의 핵심인 잦은 쓰기 작업(Write)의 분산 처리 아키텍처.
- **Redis (In-Memory Cache):**
  - 유저의 실시간 위치, 현재 전투 중 쌓이는 'Mastery Points(숙련도)', 잡몹 처치에 따른 파편(에테르) 획득량 등을 메모리에 누적 보관.
- **MongoDB (Persistent Storage):**
  - Redis에 쌓인 데이터를 **1분(혹은 전투 종료 시) 단위로 벌크 업데이트(Bulk Update)**.
  - 불변/영구 보존이 필요한 데이터(전리품 습득, 보스 처치, 하드코어 사망 로그 등)는 Redis를 거치지 않고 직접 MongoDB에 동기화.

### 4.4 프롬프트 보안 및 가드레일 (Guardrails)
- **목적:** AI를 세뇌하여 무료 아이템을 뜯어내거나, 설정 충돌(예: "나는 OpenAI 모델이다" 등의 발화)을 일으키는 프롬프트 인젝션 원천 방어.
- **구조 (Pre-computation / Output Validation):**
  - **입력 단:** 유저 메시지는 항상 고정된 시스템 지시문 하단에 'User Input:' 형태로 제한적으로 주입됨.
  - **출력 단 (상인 협상 로직):** LLM은 절대로 "최종 가격($)"을 리턴하지 못함. 서버(FastAPI)는 오직 LLM이 판정한 **"할인 비율 (0.00 ~ 0.20)"** JSON만 신뢰하며, 최종 가격 결제 로직은 서버단의 Python 엔진에서 수학적으로 계산하여 검증 됨.

---

## 5. API 명세서 초안 (API Draft Protocol)

### 5.1 REST API (이벤트/데이터 처리)
| Endpoint | Method | 설명 (Description) | Payload 예시 |
| :--- | :---: | :--- | :--- |
| `/v1/player/login` | `POST` | 게임 접속 시 DB 스탯 동기화 | `{"uuid": "..."}` |
| `/v1/combat/action` | `POST` | 검 휘두름, 마법 사용 등 숙련도(MP) 누적 | `{"action_type": "sword_hit", "amount": 1}` |
| `/v1/event/death` | `POST` | 하드코어 사망 처리 및 로그 수집 | `{"cause": "fall", "location": [...]}` |

### 5.2 WebSocket API (실시간 상호작용)
| Channel / Event | 설명 (Description) |
| :--- | :--- |
| `ws://host/v1/stream/aria` | 아리아와의 대화 및 동기화 각성 면접 스트림 채널 |
| `ws://host/v1/stream/merchant` | 상인 NPC와의 아이템 흥정 채팅 채널 (종료 지점에서 JSON(할인율) 리턴) |
| `ws://host/v1/stream/boss_taunt` | 테네브리스 보스룸 돌입 시 보스의 실시간 음성/채팅 정신 공격 스트리밍 |

---

## 6. 개발 로드맵 단계별 목표 (Development Tiers)

1. **Tier 1 (Core Infrastructure & MVP):**
   - FastAPI 뼈대 구축 및 MongoDB 기본 Collection(users, stats, history) 셋업.
   - 단일 엔드포인트에서 LLM(ChatGPT 등) API 테스트. (성향/점수 분석 JSON 파싱 안정화)
   - Fabric 클라이언트에서 HTTP Post로 대화 메시지 로그 전송 테스트.

2. **Tier 2 (AI Pipeline Optimization):**
   - WebSocket을 연동하여 텍스트 스트리밍 환경 구성. 클라이언트 타이핑 연출 적용.
   - Vector DB 연결 후 게임 시나리오(CAG) 문서 임베딩 테스트. 유저 질문에 힌트 리턴 구동.
   - 프롬프트 가드레일 적용 및 '상인 협상 프롬프트' 난이도 조절.

3. **Tier 3 (RPG & Hardcore Logic):**
   - Redis를 도입하여 마인크래프트의 실시간 전투 행동 횟수 캐싱 및 숙련도 계산 적용.
   - 하드코어 사망 시 서버 로직에 의한 스탯 저하, 및 사망 로그를 읽는 아리아의 위로(RAG) 기능.
   - 0.1% 확률의 몹처치 감지 훅 생성 및 "레전더리 유물 작명(Generative Loot)" 연동 구축.
