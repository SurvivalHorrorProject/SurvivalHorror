# SurvivalHorror

> 지하철역을 배경으로 한 1인칭 서바이벌 호러게임
<img width="720" height="433" alt="ThumbNail" src="https://github.com/user-attachments/assets/c9eb2b25-2cd9-4f07-a84d-e71feb2e8189" />



[![YouTube](https://img.shields.io/badge/YouTube-Watch%20Demo-red?logo=youtube&logoColor=white)](https://youtu.be/nd54T2FFMu4)

| | |
|---|---|
| **엔진** | Unreal Engine 5.8 (블루프린트 전용, C++ `Source/` 없음) |
| **장르** | 1인칭 서바이벌 호러 |
| **개발 기간** | 2026.08.24 ~ 08.31 |

---

### 🎨 Level & System Design Team
| <center>마서현 (Level · Environment)</center> | <center>손재윤 (System · Gameplay)</center> |
| -------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| <center><img width="150px" src="https://avatars.githubusercontent.com/your-github-id-seohyun" /></center> | <center><img width="150px" src="https://avatars.githubusercontent.com/huggywuggy1289" /></center> |
| [@SeoHyeon-MA](https://github.com/SeoHyeon-MA) | [@huggywuggy1289](https://github.com/huggywuggy1289) |

---

### 📝 주요 담당 작업

| 마서현 (Level · Environment) | 손재윤 (System · Gameplay) |
| --- | --- |
| 프로젝트 초기 구성 | 몬스터 AI 전방위 감지 및 추격 Behavior Tree |
| 지하철역 맵 전체 제작 | **Behavior Tree 타겟 유효성 예외처리** |
| 소품 배치 및 벽 블로킹 박스 | 몬스터 사망 처리 및 카드키 소환 배선 |
| 바닥 및 소품 환경 메테리얼 | **플레이어 사망 판정 및 게임오버 재시도 화면** |
| 아이템 회전 메테리얼 | NarrativeInventory 도입 및 12칸 슬롯 영속 렌더 |
| 카드 메테리얼 | 슬롯 드래그앤드롭 이동 |
| 자판기 모델 배치 | 부적 장착 및 사용 1회 소모 로직 |
| 문과 카드키 상호작용 블루프린트 | 코인 픽업, 카운트 및 자판기 상점 |
| 플레이어 캐릭터 디자인 세팅 | **코인 완료 안내 UI** |
| **카메라 쉐이킹** | 플래시라이트 |
| **엔딩 시퀀스 블루프린트** | 플레이어 이동, 대시, 스프린트 키 밸런스 |
| **괴물 사망 페이드아웃 연출** | **대시 및 스프린트 키 밸런스 조정** |
| **협업용 오디오 에셋 추가** | 발소리, 배경 앰비언스 BGM, 부적 Aura 사운드 |
| | **오프닝 페이드아웃 연출** |
| | Git LFS 파일 락 및 패키지 관리 |

---

### 🔗 접점
- **자판기** → 서현: 모델 및 배치 · 재윤: 작동 로직
- **몬스터** → 재윤: AI 전반 및 사망 판정 · 서현: 사망 연출 및 카드키 소환

---

### 🆕 새로 추가된 부분
- **마서현** → 카메라 쉐이킹 · 엔딩 연출 · 시퀀스 블루프린트 · 괴물 사망 페이드아웃 · 오디오 에셋 추가
- **손재윤** → 오프닝 연출 · 코인 완료 안내 UI · 플레이어 사망 판정 및 게임오버 재시도 · 대시/스프린트 키 · Behavior Tree 예외처리 · 패키지 정리 · 오디오 에셋 추가


---

## 프로젝트 구조

```
Content/
├─ FirstPerson/
│  ├─ Blueprints/            BP_FirstPersonCharacter · BP_FirstPersonGameMode · BP_FirstPersonPlayerController · BP_FirstPersonCameraManager
│  └─ Anims/                 1인칭 손 애니메이션
├─ Features/
│  ├─ EnemyAI/
│  │  ├─ Blueprints/         BP_Whisper_Character · BP_WhisperAIController
│  │  ├─ AI/                 BT_Whisper · BB_Whisper · BTTask_FindRandomPatrolPoint
│  │  └─ Anims/               ABP_Whisper · BS_Whisper_Locomotion · Anim_Whisper_Attack3_Montage
│  ├─ Inventory/
│  │  ├─ UI/                 WBP_InventoryHUD · WBP_Inventory · WBP_InvSlot · WBP_EquipBar · WBP_Shop
│  │  └─ Data/                BI_Omamori · BI_CardKey
│  ├─ Shop/                  BP_VendingMachine
│  ├─ Coin/                  BP_Coin · WBP_CoinCompleteNotice
│  ├─ Card/                  BP_CardKeyPickup
│  ├─ CardDoor/               BP_CardDoor · WBP_CardDoorPrompt
│  ├─ Opening/                WBP_Opening
│  ├─ GameOver/               WBP_GameOver
│  ├─ Ending/                 BP_EndingTrigger · WBP_Ending
│  ├─ CameraShake/            BP_CS_HeadBob
│  ├─ Puzzle/                 Blueprints · Meshes 스켈레톤
│  └─ Audio/                  Footsteps · Music · Ambience · SFX · UI · _Mix (SC_* SoundClass · SMix_Main)
├─ Whisper/                   몬스터 스켈레탈 메시 · 애니메이션 시퀀스 · 머티리얼 · 텍스처 · Maps/Whisper
├─ SourceArt/                 지하철 환경 아트 (Environment/Props · Items · Materials · Niagara · Characters)
├─ Font/                      Font_JeongseonArirang 오프닝·알림 UI용
├─ Input/                     Actions · Touch (Enhanced Input)
├─ Characters/Mannequins/     템플릿 기본 마네킹
├─ LevelPrototyping/          템플릿 프로토타입 메시·머티리얼
└─ Maps/                      L_SubwayStation (메인) · Test_EnemyAI (AI 테스트) · Stage1 · Test_LVStage1 (레거시)
docs/INVENTORY_HANDOFF.md     인벤토리 드래그앤드롭 디버깅 인수인계
```

---

## 빌드 / 실행

1. **Unreal Engine 5.8.2** 필요
2. Git LFS 설치 후 클론 (`git lfs install` → `git clone`) — 안 하면 `.uasset`이 포인터 파일로 받아짐
3. `SurvivalHorror.uproject` 실행 → 메인 레벨 `L_SubwayStation`
4. 마켓 플러그인 `NarrativeInventory`는 `Plugins/Marketplace/`에 포함 (엔진 버전 일치 필요)

### 조작

| 키 | 동작 |
|---|---|
| WASD / 마우스 | 이동 · 시점 |
| W 더블탭 | 대시 |
| Q | 인벤토리 열기 / 닫기 |
| 1 | 부적 꺼내기 / 치우기 |
| E | 상호작용 (문, 몬스터 퇴치) |
