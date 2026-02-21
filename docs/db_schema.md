# 🗄️ Project X: 데이터베이스 및 시스템 스키마 설계서 (DB & VectorDB Schema)

**버전:** v1.0
**기반 문서:** system_architecture.md, api_spec.md

본 문서는 **"AI-Driven Hardcore Dark Fantasy RPG"**의 방대한 플레이어 스탯, 하드코어 사망 로그, 인벤토리 데이터를 안전하게 영구 저장하기 위한 **MongoDB(NoSQL) 설계**와, AI NPC(아리아, 보스)가 과거의 기억이나 게임의 지식을 실시간으로 검색해올 때 필수적인 **Vector Database(RAG/CAG 용) 임베딩 구조**를 구체화한 개발 명세서입니다.

---

## 1. MongoDB 컬렉션 스키마 (Persistent Data Layer)

RPG 게임 특성상 유저의 상태 스키마가 복잡하고 자주 추가되므로 관계형 DB(RDBMS)보다는 JSON 구조의 **MongoDB**를 채택합니다.

### 1.1 `users` (플레이어 핵심 프로필 및 상태)
가장 조회가 잦으며, 로그인 시 클라이언트로 동기화되는 핵심 게임 데이터입니다.

```json
{
  "_id": "ObjectId",
  "uuid": "String (Minecraft Unique ID, Index)",
  "nickname": "String",
  "created_at": "ISODate",
  "last_login": "ISODate",

  "soul_profile": {
    "sync_rate": "Double (0.0 ~ 1.0) - 아리아와의 동기화 지수",
    "alignment": {
      "destruction": "Double",
      "protection": "Double",
      "observation": "Double"
      // 생성 시 유저의 "첫 대답"이 각 성향 점수로 환산되어 저장됨
    }
  },

  "stats": {
    "level": "Int",
    "mastery_points_pool": "Int (총 획득한 숙련도 경험치 지수)",
    "attributes": {
      "str": "Int", "vit": "Int", "int": "Int", "dex": "Int", "luk": "Int"
    }
  },

  "currency": {
    "aether_shards": "Int (재화)"
  }
}
```
**[인덱싱 전략]**
- `uuid` (Unique, 1): 유저 로그인 및 빠른 단일 조회를 최우선 보장.

### 1.2 `death_logs` (하드코어 사망 기록 아카이브)
사망 시 발생하는 로그. 나중에 아리아가 유저를 위로할 때나, RAG 정보로 참조할 목적으로 모두 영구 누적합니다.

```json
{
  "_id": "ObjectId",
  "player_uuid": "String (Index)",
  "death_timestamp": "ISODate",
  "location": { "x": "Double", "y": "Double", "z": "Double" },
  "cause_of_death": "String (ex: ancient_guardian_beam)",
  
  "penalty": {
    "lost_sync_rate": "Double (차감된 동기화 지수)",
    "is_recovered": "Boolean (시체 회수 여부)"
  },
  
  // AI 연동 필드
  "aria_comment": "String (사망 원인을 분석해 생성한 아리아의 위로/공략 텍스트)"
}
```
**[인덱싱 전략]**
- `player_uuid` (1), `death_timestamp` (-1): 특정 유저의 가장 최근 죽음 원인을 RAG로 빠르게 파싱하기 위한 복합 인덱스.

### 1.3 `items` (생성형 전리품 및 인벤토리 관리)
특히 신화급(Mythic) 전리품은 AI가 실시간으로 이름과 텍스트를 부여하기 때문에 상세 객체로 개별 관리됩니다.

```json
{
  "_id": "ObjectId",
  "item_uuid": "String (Unique)",
  "owner_uuid": "String (Index)",
  "rarity": "String (enum: Common, Rare, Legendary, Mythic)",
  "base_type": "String (ex: SHIELD, SWORD)",
  
  // AI 생성 메타데이터 (Mythic 등급 한정)
  "ai_generated": {
    "display_name": "String (예: 무너진 서약을 베는 칼날)",
    "flavor_text": "String (생성형 텍스트 역사)",
    "dominant_stat_bonus": "String (이 아이템이 올려주는 주력 스탯: STR)"
  }
}
```

---

## 2. Vector Database 스키마 (RAG / CAG AI Pipeline)

게임 세계관 정보와 유저의 과거 대화 기억을 "유사도 기준"으로 빠르게 검색하기 위해 사용합니다. (Milvus, Pinecone, or Qdrant 권장)

### 2.1 `lore_knowledge_base` (CAG: 시스템 공통 지식)
유저가 NPC에게 "저 몬스터는 뭐야?", "에테르는 어떻게 모아?" 라고 물었을 때, 시스템이 참고할 공략집의 임베딩 보관소입니다. (게임 업데이트 시마다 정적 데이터를 Vector 화)

| 필드명 | 타입 | 설명 |
| :--- | :--- | :--- |
| `id` | Int | 고유 식별자 |
| `topic_category` | String | 카테고리 (ex: `monster`, `mechanic`, `lore`) |
| `content_text` | String | 실제 참고할 원문 데이터 ("가디언은 붉은 코어 발동 시 회피해야 한다...") |
| `content_vector` | FloatVector[1536] | OpenAI `text-embedding-3-small` 등으로 변환된 1536차원 벡터 구조 |

**[검색(Query) 흐름]**
유저의 메타 질문 $\rightarrow$ 임베딩화 $\rightarrow$ Vector DB에서 가장 유사도가 높은 `content_text` Top 3 도출 $\rightarrow$ 아리아/NPC 프롬프트의 **`<System Knowledge>`** 블록에 주입.

### 2.2 `player_memories` (RAG: 유저 대화 기억 창고)
최종 보스가 _"네 예전 맹세는 거짓이었다!"_ 라고 도발하거나, 아리아가 _"처음 만났던 날을 기억하나요?"_ 라고 상황을 연출할 때 쓰는, **플레이어 개인별 채팅/행동 로그**입니다.

| 필드명 | 타입 | 설명 |
| :--- | :--- | :--- |
| `id` | Int | 고유 식별자 |
| `player_uuid` | String | 반드시 유저 UUID로 메타데이터 필터링을 걸어야 함. (다른 유저의 기억 접근 방지) |
| `timestamp` | Float | 로그 발생 시간 (최신순 가중치 환산 위함) |
| `context_type` | String | 당시 상황 (ex: `first_awakening`, `npc_negotiation`, `boss_encounter`) |
| `memory_text` | String | 유저의 말 ("나는 무너진 것들을 지키기 위해 검을 들었다.") |
| `memory_vector` | FloatVector[1536] | 대화 내용을 벡터화 (유저의 현재 발언/액션과 대조하기 위함) |
| `alignment_state`| Object | 그 말을 했을 당시 유저의 주요 성향 (Protection 80% 등) |

**[검색(Query) 흐름]**
1. 테네브리스 보스 스폰! (혹은 아리아 100일 차 대화 시점)
2. 백엔드가 `player_uuid` + "다짐, 선서, 목표" 라는 키워드로 Vector DB 질의.
3. 당시의 `memory_text`(지키겠다는 말)와 현재의 스탯(Destruction이 높음)을 비교하는 **모순점 비교 프롬프트**를 동적으로 생성하여 보스 대사 출력.
