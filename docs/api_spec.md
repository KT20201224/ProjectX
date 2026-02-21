# 🔌 Project X: API 통신 명세서 (API Specification)

**버전:** v1.0
**기반 문서:** Project X 시스템 아키텍처 및 로드맵 문서 시리즈

본 문서는 **Minecraft (Fabric) 클라이언트**와 **FastAPI 백엔드 서버** 간의 실시간 비동기 통신 규격을 정의합니다. 일반적인 상태 동기화 및 이벤트 로그 수집을 위한 **REST API (JSON)**와, AI와의 끊김 없는 대화 경험(스트리밍 타이핑)을 위한 **WebSocket** 프로토콜로 나뉩니다.

---

## 1. 전역 헤더 및 인증 (Global Headers & Auth)

모든 REST API 요청 및 WebSocket 연결 시에는 마인크래프트 유저를 식별하기 위한 **UUID**가 필수로 포함되어야 합니다.

- `X-Player-UUID`: 마인크래프트 플레이어 고유 UUID 문자열.
- `Content-Type`: `application/json`

---

## 2. REST API 엔드포인트 (Event & State Sync)

### 2.1 플레이어 접속 및 초기화 `[POST /v1/player/login]`
- **목적:** 플레이어가 서버에 접속(로그인) 시 마인크래프트 클라이언트가 백엔드에 초기 스탯과 과거 저장된 데이터를 요청함.
- **Request payload:**
  ```json
  {
    "uuid": "550e8400-e29b-41d4-a716-446655440000",
    "nickname": "Player_Aria"
  }
  ```
- **Response payload (200 OK):**
  ```json
  {
    "status": "success",
    "soul_profile": {
      "sync_rate": 0.15,
      "alignment": "protection"
    },
    "stats": {
      "level": 1, 
      "max_hp": 120, 
      "str": 10, "vit": 15, "int": 10
    }
  }
  ```

### 2.2 하드코어 사망 이벤트 전송 `[POST /v1/event/death]`
- **목적:** 바닐라 마인크래프트에서 플레이어가 사망 시, 백엔드에 페널티 연산(동기화 지수 감점)을 요청하고 아리아의 위로 메시지 ID를 받아옴.
- **Request payload:**
  ```json
  {
    "cause_of_death": "ancient_guardian_beam",
    "location": {"x": 120, "y": 64, "z": -450},
    "held_item": "rusty_sword"
  }
  ```
- **Response payload (200 OK):**
  ```json
  {
    "status": "penalty_applied",
    "penalty_details": {
      "lost_sync_rate": 0.05,
      "current_sync_rate": 0.10
    },
    "recovery_message": "가디언의 코어가 붉어질 땐 회피에 집중했어야 했어요. 영혼의 조각을 찾아오세요."
  }
  ```

### 2.3 액션(전투)에 따른 숙련도 누적 `[POST /v1/combat/action]`
- **목적:** 마인크래프트 내에서 사용자가 특정 행동(검 휘두르기, 마법 사용, 회피 등)을 할 때 백엔드 캐시(Redis)에 MP(Mastery Points)를 누적시키는 논-블로킹(Non-blocking) 통신. (Fabric 측에서는 10~20틱 주기로 배치 전송 권장)
- **Request payload:**
  ```json
  {
    "combat_actions": [
      { "type": "sword_swing", "count": 15 },
      { "type": "guard_block", "count": 3 }
    ],
    "timestamp": 1690000000
  }
  ```
- **Response payload (202 Accepted):** (처리 속도를 위해 단순 수신 확인용 리턴)
  ```json
  { "status": "queued" }
  ```

---

## 3. WebSocket 이벤트 명세 (Streaming AI Chat)

### 3.1 아리아와의 각성/가이드 대화 `[WS /v1/stream/aria]`

- **연결 시점:** 플레이어가 튜토리얼 지역 진입, 혹은 아리아 NPC를 클릭하여 대화를 걸었을 때 연결(Connect).
- **Client (Fabric) → Server `send` (유저 입력):**
  ```json
  {
    "event": "user_message",
    "data": {
      "context": "tutorial_first_question",
      "message": "부서진 세계의 파편이라도 지키겠다."
    }
  }
  ```

- **Server → Client `receive` (타이핑 스트리밍):**  
  (LLM이 단어를 생성하는 0.1초마다 끊어서 패킷 수신)
  ```json
  {"event": "stream_chunk", "data": {"chunk": "당"}}
  {"event": "stream_chunk", "data": {"chunk": "신의 "}}
  {"event": "stream_chunk", "data": {"chunk": "눈동자에서 "}}
  // ... (클라이언트는 이 chunk들을 UI 버퍼에 append하며 타이핑 사운드 재생)
  ```

- **Server → Client `receive` (스트리밍 종료 및 상태 반영):**
  (문장 생성이 완전히 끝나면, 최종 분석된 성향 점수와 이펙트를 트리거하는 JSON 송신)
  ```json
  {
    "event": "stream_end",
    "data": {
      "full_message": "당신의 눈동자에서 수호의 의지를 보았습니다...",
      "ai_analysis": {
        "alignment_decision": "protection",
        "score_granted": 0.8
      },
      "client_effect": "spawn_particle_blue_aura",
      "stat_changes": {"vit": "+5"}
    }
  }
  ```

### 3.2 상인 NPC 흥정 채널 `[WS /v1/stream/merchant]`
- **목적:** 상인과 물건(아이템)을 걸고 흥정할 때 프롬프트 방어를 적용한 AI 분석 채널.
- **Client → Server `send`:**
  ```json
  {
    "event": "negotiation_offer",
    "data": {
      "merchant_id": "greedy_robot_01",
      "item_id": "ancient_bolt",
      "base_price": 100,
      "user_message": "제가 가진 에테르 파편이 이게 전부인데, 조금만 깎아주시면 안 될까요?"
    }
  }
  ```
- **Server → Client `receive` (스트리밍 및 최종가):**
  ```json
  // 중간 stream_chunk 통신 생략...
  
  {
    "event": "negotiation_result",
    "data": {
      "full_message": "흐음... 네 꼴이 말이 아니긴 하군. 특별히 깎아주마.",
      "final_price": 85,
      "is_success": true
    }
  }
  ```

### 3.3 테네브리스(보스) 룸 입장 및 실시간 압박 `[WS /v1/stream/boss_taunt]`
- **목적:** RAG 시스템을 기반으로 한 보스의 정신 공격 메시지를 전투 도중 실시간으로 뿌리는 채널.
- **Client → Server `send` (보스 조우 시 1회 발송):**
  ```json
  {
    "event": "boss_phase_start",
    "data": { "phase": 1, "player_hp_percent": 1.0 }
  }
  ```
- **Server → Client `receive` (서버 주도 비동기 Push):**
  (전투 중 서버단에서 유저의 플레이 성향이나 모순점을 파싱해 특정 타이밍에 자동으로 발송)
  ```json
  {
    "event": "boss_taunt",
    "data": {
      "taunt_message": "나를 파괴한다고? 네가 처음 아리아에게 한 맹세는 '지키겠다'고 했었지. 네 위선을 박살내주마!",
      "trigger_effect": "screen_shake_heavy"
    }
  }
  ```
