# 인벤토리 / 드래그앤드롭 작업 인수인계 문서

> 목적: 인벤토리 시스템 구현 중 **드래그앤드롭 클릭 감지**가 안 되는 문제를 다른 AI/개발자가 이어서 해결하기 위한 전체 복기.
> 작성 시점 커밋: `c170fcc` (main). 이후 디버깅 변경분은 미커밋 상태일 수 있음.

---

## 1. 원래 요구사항

1. **Q키 → 인벤토리 UI 표시, Q 다시 → 닫힘** (2단계 토글). 하단의 작은 **EquipBar**는 항상 표시(닫아도 남음).
2. **EquipBar는 인벤토리 첫 번째 슬롯(0번) 아이템을 실시간 미러링**. 아이템 획득/사용/이동 시 자동 갱신.
3. 인벤토리를 열면 **커서로 아이템을 슬롯 간 이동**(드래그앤드롭)할 수 있어야 함. 순서 재배치.
4. **부적(오마모리)을 인벤토리 아이템으로** 만들어서:
   - 인벤토리/EquipBar에 표시됨
   - `1`키로 **손에 꺼내기 / 도로 치우기** 토글 (아이템은 인벤토리에 계속 있음, 소모 아님)
   - 현재 부적은 `BP_FirstPersonCharacter` 안의 자체 시스템(`OmamoriMesh` StaticMesh + `bIsOmamoriEquipped` bool + `EquipOmamori`/`HolsterOmamori` 이벤트)
5. 풀 인벤토리를 열 때: **UI+게임 입력 모드 + 마우스 커서 표시**, 캐릭터 이동/시점은 그대로.

---

## 2. 프로젝트 환경

- **UE 5.8.2** (Compatible CL 55116800). **블루프린트 전용** 프로젝트 (C++ `Source/` 없음).
- 플러그인:
  - `NarrativeInventory` v2.0.6 — `Plugins/Marketplace/NarrativeInventory/` (마켓 플러그인, Win64 프리빌트 DLL 포함, 엔진과 BuildId 일치)
  - `CommonUI` (엔진 플러그인, **필수** — `NarrativeInventoryEditor` 모듈이 의존. 끄면 에디터가 안 열림)
  - `GameplayStateTree`, `ModelContextProtocol`, `ModelingToolsEditorMode`
- **NarrativeInventory 번들 UI 위젯은 사용 불가**:
  - `WBP_InventoryWidget`, `WBP_Item`, `W_NarrativeMenu_Inventory`, `WBP_EquippedItem` 등은 `WBP_NarrativeActivatableWidget`(별도 플러그인 `NarrativeCommonUI`, **미설치**)를 부모로 해서 로드 실패.
  - 로그: `LogLinker: VerifyImport: Failed to find script package for import object 'Package /Script/NarrativeCommonUI'`
  - → **UI를 전부 커스텀 UMG로 직접 구현.** 데이터/로직 레이어(`UNarrativeInventoryComponent`, `UNarrativeItem`)는 정상 동작.
- 맵: `Content/Maps/Test_LVStage1`. GameMode: `BP_FirstPersonGameMode_C`. 캐릭터: `BP_FirstPersonCharacter`. 기본 PlayerController(커스텀 없음).
- 캐릭터 컴포넌트 계층: `CapsuleComponent`(root) → `Mesh (CharacterMesh0)` → `FirstPersonMesh` → `FirstPersonCamera`, `OmamoriMesh`.

---

## 3. 만든 에셋

| 에셋 | 경로 | 내용 |
|---|---|---|
| `IA_Inventory` | `Content/Input/Actions/` | Digital(bool). `IMC_Default`에 **Q**키 매핑 |
| `BI_Omamori` | `Content/Features/Inventory/Data/` | `NarrativeItem` 서브클래스 BP. DisplayName="부적", Thumbnail=`T_Omamori_BC` (Soft Object), Stackable off |
| `WBP_InventoryHUD` | `Content/Features/Inventory/UI/` | 컨테이너 (부모 User Widget) |
| `WBP_Inventory` | `Content/Features/Inventory/UI/` | 풀 인벤토리 패널 (부모 User Widget) |
| `WBP_InvSlot` | `Content/Features/Inventory/UI/` | 슬롯 한 칸 (부모 User Widget) |
| `WBP_EquipBar` | `Content/Features/Inventory/UI/` | 하단 바 (부모 User Widget) |

---

## 4. BP_FirstPersonCharacter 변경분

### 컴포넌트
- `NarrativeInventory` (= `Narrative Inventory Component`, `UActorComponent`) — 액터 레벨에 추가. Capacity/Weight 설정.

### 변수
- `InvSlots` : `Narrative Item` (Object Reference) **배열**. **기본값 12칸(전부 None)**. ← 슬롯 모델의 핵심.
- `Items` : `Narrative Item` (Object Reference) 배열. (원래 `ReconcileSlots`의 로컬 변수 의도였으나 멤버 변수로 생성됨 — 기능상 무해)
- `InventoryHUD` : `WBP_InventoryHUD` 레퍼런스.
- `bHasOmamori`, `bIsOmamoriEquipped` : bool (기존). `OmamoriMesh` : StaticMeshComponent (기존).

### 이벤트 디스패처
- `OnSlotsChanged` (파라미터 없음) — 슬롯 배열이 바뀔 때마다 브로드캐스트.

### 함수 `ReconcileSlots` (구현 완료, 컴파일 OK)
```
Items = NarrativeInventory.GetItems()          // GetComponentByClass(NarrativeInventoryComponent) → GetItems
InvSlots = Items                                // 배열 그대로 복사 (Set 멤버 변수)
Array_Set(TargetArray=InvSlots, Index=11, Item=None, bSizeToFit=true)   // 12칸으로 패딩
OnSlotsChanged()                               // 브로드캐스트
```
- **알려진 한계**: 인벤토리가 바뀔 때마다 `InvSlots`를 **인벤토리 순서로 재구성**함 → 유저가 드래그로 재배치한 순서가 아이템 획득/사용 순간 리셋됨.
- **원래 의도한 알고리즘** (미구현, 와일드카드 타입 이슈로 포기):
  1. 사라진 아이템은 슬롯에서 비우기 (`if InvSlots[i] valid AND NOT Items.Contains(InvSlots[i]) → InvSlots[i] = None`)
  2. 새 아이템은 첫 빈 슬롯에 넣기 (`InvSlots.Find(None)` → `InvSlots[idx] = item`) — 위치 보존
  - `Array_Set`(KismetArrayLibrary) 와일드카드 `Item` 핀이 `ForEach ArrayElement`(Object)와 `InvSlots`(NarrativeItem)를 못 맞춰서 "undetermined type" 컴파일 에러 반복 발생. 해결책: `Get(a copy)`로 `Items[i]`를 꺼내 쓰면 타입이 NarrativeItem으로 확정되어 충돌 안 남 (현재 단순 버전이 이 방식).

### 함수 `MoveSlots(From: int, To: int)` (구현 완료, 컴파일 OK)
```
itemFrom = InvSlots[From]     (Get a copy)
itemTo   = InvSlots[To]       (Get a copy)
Array_Set(InvSlots, To,   itemFrom)
Array_Set(InvSlots, From, itemTo)
OnSlotsChanged()
```
- 함수명이 `MoveSlots` (복수형)임에 주의. `WBP_InvSlot.OnDrop`에서 호출됨.

### 함수 `TryOmamori` (기존 함수, 수정됨)
- 원래 첫 Branch 조건이 `bHasOmamori` (멤버 bool)였음 → **`NarrativeInventory.HasItem(BI_Omamori, 1)` 로 교체**.
- 구조: `if HasItem(BI_Omamori)` → `if bIsOmamoriEquipped` → true: `HolsterOmamori()` / false: `EquipOmamori()`.
- `EquipOmamori`/`HolsterOmamori` = `OmamoriMesh` visibility 토글하는 기존 커스텀 이벤트.

### Event Graph 연결
- **BeginPlay 체인**:
  `Set bHasOmamori` → `NarrativeInventory.GiveDefaultItems` → `NarrativeInventory.TryAddItemFromClass(BI_Omamori, 1)` → `NarrativeInventory.TryAddItemFromClass(BI_ExampleItemA, 2)` → `Create Widget(WBP_InventoryHUD, OwningPlayer=GetPlayerController)` → `Set InventoryHUD` → `AddToViewport` → **`ReconcileSlots`** → `NarrativeInventory` → `Bind Event to OnInventoryUpdated` + `Create Event(ReconcileSlots)`
- `EnhancedInputAction IA_Inventory (Started)` → `InventoryHUD` → `ToggleInventory`
  - ※ 처음에 `Triggered`로 했다가 연타 문제로 `Started`로 바꿈.
- `EnhancedInputAction IA_DrawOmamori (Triggered)` → `TryOmamori` (`1`키). `IA_DrawOmamori`는 이미 `IMC_Default`에 1키로 매핑돼 있던 기존 액션.

---

## 5. WBP_InventoryHUD

### 계층 (Designer)
- 루트 `Canvas Panel`
  - `WBP_EquipBar` 인스턴스 — 하단중앙 앵커, **Size To Content** 체크, Position (0, -40), Alignment (0.5, 1.0). Visibility: `Not Hit-Testable (Self & All Children)` *(디버깅 중 설정)*
  - `InventoryPanel` = `WBP_Inventory` 인스턴스 — 중앙 앵커, Size **1000 x 600**, 초기 Visibility **Collapsed**, **Is Variable** 체크

### 변수
- `bInventoryOpen` : bool
- `InventoryPanel` : `WBP_Inventory` 레퍼런스 (위 배치 위젯이 Is Variable로 잡힌 것)

### 함수 `SetInventoryOpen(bOpen: bool)`
```
Set bInventoryOpen = bOpen
Branch(bOpen):
  True:
    InventoryPanel.SetVisibility(Visible)
    GetOwningPlayer → SetInputMode_GameAndUIEx    // bHideCursorDuringCapture = false (디버깅 중)
    Set PlayerController.bShowMouseCursor = true
  False:
    InventoryPanel.SetVisibility(Collapsed)
    GetOwningPlayer → SetInputMode_GameOnly
    Set PlayerController.bShowMouseCursor = false
```
- ⚠️ **SetVisibility 두 개 다 Target이 `InventoryPanel`**. self 아님. (처음에 self로 돼 있어서 HUD 전체가 껐다 켜지는 버그 있었음 → 고침)

### 함수 `ToggleInventory()`
```
SetInventoryOpen(bOpen = NOT bInventoryOpen)
```

---

## 6. WBP_Inventory

### 계층
- 루트 `Border_39` — Visibility `Not Hit-Testable (Self Only)` *(디버깅 중)*
  - `ItemContainer` = `Wrap Box` — **Is Variable**, Visibility `Not Hit-Testable (Self Only)` *(디버깅 중)*

### 커스텀 이벤트 `Refresh`
```
ItemContainer.ClearChildren
GetOwningPlayerPawn → Cast to BP_FirstPersonCharacter → Get InvSlots
For Loop (First Index 0, Last Index 11):       // ← 처음에 Last Index가 비어서(=0) 1칸만 렌더됐던 버그. 반드시 11.
  slotItem = InvSlots[LoopIndex]   (Get a copy)
  w = Create Widget(class=WBP_InvSlot, Item=slotItem, Slot Index=LoopIndex)
  ItemContainer.AddChild(w)
```

### Event Construct
```
Refresh()
GetOwningPlayerPawn → Cast to BP_FirstPersonCharacter
  → Bind Event to OnSlotsChanged  +  Create Event(Refresh)
```
- 즉 캐릭터의 `OnSlotsChanged` 브로드캐스트마다 그리드 전체 재생성.

---

## 7. WBP_InvSlot  ← **여기가 막힌 부분**

### 계층
- 루트 `Icon` (`Image`, 이름 `Icon`) — Visibility `Visible`로 설정됨.
  - (별도 배경/Border 없음. 루트가 곧 Image.)

### 변수
- `Item` : `Narrative Item` (Object Reference). **Instance Editable + Expose on Spawn**.
- `SlotIndex` : int. **Instance Editable + Expose on Spawn**.

### Event Construct
```
Set Visibility(self, Visible)        // 디버깅 중 추가
IsValid(Item)?:
  Valid:
    Item.Thumbnail (Soft Object Ptr) → Load Asset → Cast to Texture2D → Icon.SetBrushFromTexture
  Not Valid:
    Icon.SetVisibility(Collapsed)     // ← 빈 슬롯은 Icon이 0크기가 됨 → 시각적으로 안 보임(그래서 "12칸인데 2칸처럼 보임")
```

### 오버라이드 함수 (UMG가 자동 호출하는 이벤트 — 별도 배선 불필요)

**`OnMouseButtonDown`** (반환형 Event Reply)
```
[Print String "MD"]  →  DetectDragIfPressed(PointerEvent = MouseEvent(FunctionEntry),
                                            DragKey = LeftMouseButton)
DetectDragIfPressed.ReturnValue  →  Return Node.Return Value
```

**`OnDragDetected`** (반환형 Drag Drop Operation)
```
CreateDragDropOperation(Class = DragDropOperation)
  → Set Payload(op, Payload = self)
op.ReturnValue  →  Return Node.Operation
```

**`OnDrop`** (반환형 bool)
```
[Print String "DROP"]  →  Operation.Payload  →  Cast to WBP_InvSlot  →  Get SlotIndex  (= source index)
GetOwningPlayerPawn → Cast to BP_FirstPersonCharacter → MoveSlots(From = source SlotIndex, To = self.SlotIndex)
Return Node.Return Value = true
```

---

## 8. WBP_EquipBar

### 계층
- 루트 `Horizontal Box` (원래 Canvas Panel → **Replace With Horizontal Box**. Canvas Panel 루트는 desired size 0이라 부모 위젯 안에서 찌부됨)
  - `SlotContainer` = `Horizontal Box`, Is Variable

### 커스텀 이벤트 `Refresh`
```
SlotContainer.ClearChildren
GetOwningPlayerPawn → GetComponentByClass(NarrativeInventoryComponent) → GetItems
Length(GetItems) > 0 ?:
  True:
    Items → Get(a copy) index 0
    w = Create Widget(WBP_InvSlot, Item = element0)
    SlotContainer.AddChild(w)
```

### Event Construct
```
Refresh()
GetOwningPlayerPawn → GetComponentByClass(NarrativeInventoryComponent)
  → Bind Event to OnInventoryUpdated  +  Create Event(Refresh)
```
- ⚠️ **미완성(F단계)**: EquipBar가 아직 `GetItems()[0]`을 읽음. 요구사항대로 하려면 `InvSlots[0]`을 읽고 `OnSlotsChanged`에 바인딩해야 함 (드래그 재배치가 EquipBar에 반영되도록).

---

## 9. 현재 동작하는 것 ✅

- Q → InventoryPanel 표시(아이템 아이콘 동적 로딩), Q → 닫힘, EquipBar는 계속 표시.
- EquipBar에 인벤토리 0번 아이템(부적) 아이콘 미러링.
- 부적이 인벤토리에 들어감. `1`키로 `EquipOmamori`/`HolsterOmamori` 토글 (부적은 인벤토리 유지). `HasItem` 게이트 동작.
- 그리드: `InvSlots` 12칸을 `WBP_InvSlot`으로 렌더. 부적 + 예제아이템(`BI_ExampleItemA`) 2개 아이콘 표시. 빈 슬롯 10개는 Icon Collapsed라 안 보임.
- `ReconcileSlots`, `MoveSlots` 컴파일 통과.
- 인벤토리 열면 마우스 커서 표시됨(유저 육안 확인).

커밋: `575d7d5` (슬롯 모델 + 12칸 렌더 기반), `c170fcc` ("인벤토리 12칸 슬롯 렌더 정상화" — For Loop 범위 수정).

---

## 10. 막힌 문제 ❌ — 드래그앤드롭 클릭 감지

**증상**: `WBP_InvSlot`의 `OnMouseButtonDown` 오버라이드가 **아예 호출되지 않음**. 맨 앞에 넣은 `Print String "MD"`가 슬롯을 클릭해도 안 뜸.

→ `DetectDragIfPressed` 실행 안 됨 → `OnDragDetected` 안 불림 → 드래그 시작 자체가 안 일어남 → `OnDrop`도 안 불림 → 아이템 위치 안 바뀜.

**Output Log에 관련 에러 없음.**

---

## 11. 시도했으나 실패한 것

| # | 시도 | 결과 |
|---|---|---|
| 1 | `WBP_InvSlot` 루트/`Icon` Visibility = `Visible` | 이미 Visible이었음. 변화 없음 |
| 2 | `WBP_InvSlot` Event Construct에 `Set Visibility(self, Visible)` 추가 | MD 안 뜸 |
| 3 | `WBP_Inventory`의 `Border_39`, `ItemContainer`(WrapBox) → `Not Hit-Testable (Self Only)` | MD 안 뜸 |
| 4 | `WBP_EquipBar` 인스턴스 → `Not Hit-Testable (Self & All Children)` | MD 안 뜸 |
| 5 | `SetInventoryOpen` 여는 쪽 입력모드 → `Set Input Mode UI Only`로 교체 | MD 안 뜸 + Q키로 못 닫음(캐릭터 입력 차단). 되돌림 |
| 6 | `Set Input Mode Game And UI`의 `bHideCursorDuringCapture` = false | MD 안 뜸 |
| 7 | Project Settings → **Game Viewport Client Class = `CommonGameViewportClient`** | 로그의 `Using CommonUI without a CommonGameViewportClient` **에러는 사라짐**. but MD 여전히 안 뜸 |
| 8 | `CommonUI` 플러그인 비활성화 시도 | `NarrativeInventoryEditor` 로드 실패로 **에디터 안 열림**. `.uproject`에 CommonUI 되살려서 복구 (필수 의존성) |

---

## 12. 로그 단서 (PIE 시작 시)

```
LogUIActionRouter: Display: Found 0 derived classes when attemping to create action router (CommonUIActionRouterBase)
```
- ↑ Game Viewport Client Class를 `CommonGameViewportClient`로 설정한 **후에도** 이 줄은 남음. CommonUI 액션 라우터가 파생 클래스를 못 찾아 base로 fallback 중.

```
LogViewport: Display: Viewport MouseCaptureMode Changed, CapturePermanently_IncludingInitialMouseDown -> CaptureDuringMouseDown
LogViewport: Display: Player bShowMouseCursor Changed, False -> True
```
- ↑ 커서/입력모드 세팅은 적용됨.

```
LogClass: Warning: Array Inner Type mismatch in Stats - Previous (StrProperty) Current(StructProperty(NarrativeItemStat(/Script/NarrativeInventory))) in package: /NarrativeInventory/Misc/ExampleItems/BI_ExampleItemA
```
- ↑ 플러그인 예제 아이템 경고. 무관해 보임.

`BI_ExampleItemA` 표시명이 로그상 "Cube (Example Item)".

---

## 13. 미해결 가설 / 다음 시도 후보 (우선순위 순)

1. **★ Widget Reflector로 실제 히트테스트 확인** — PIE 중 `Ctrl+Shift+W` → **"Pick Hit-Testable Widgets"** 체크 → **"Pick Painted Widget"** → 슬롯 아이콘 위에 커서. 커서 밑 위젯 트리를 보면:
   - `WBP_InvSlot` / `Icon`이 나오면 → 히트테스트는 되는 것. `OnMouseButtonDown` 오버라이드가 안 불리는 다른 이유를 찾아야 함(입력 라우팅).
   - 다른 위젯(HUD Canvas, EquipBar, 뭔가 전체 덮는 것)이 나오면 → **그게 클릭을 먹는 범인**. 그 위젯 Visibility를 `Not Hit-Testable`로.
   - 아무것도 안 나오면(게임 뷰포트) → 슬롯이 hit-testable하지 않거나 크기 0.

2. **CommonUI 입력 라우팅 간섭** — CommonUI가 켜져 있고(필수) `CommonGameViewportClient` 설정했는데도 액션 라우터가 half-configured(`Found 0 derived classes`). CommonUI는 원칙적으로 마우스 포인터 입력엔 관여 안 하지만, `CommonGameViewportClient`가 Slate 입력 전처리를 하므로 영향 가능. 시도:
   - Project Settings → `CommonUI Editor` / `Common Input Settings` 기본값 세팅 확인
   - 또는 `WBP_InventoryHUD`를 `CommonActivatableWidget` 상속으로 만들고 `CommonActivatableWidgetStack`에 push (큰 리팩터)
   - 또는 CommonUI가 정말 마우스에 무관함을 확인하고 다른 원인으로.

3. **`Icon` Image 크기 0 의심** — 루트가 `Image`인데 `SetBrushFromTexture`의 `bMatchSize=false`면 브러시 크기가 기본값(작거나 0). UserWidget이 hit-testable하려면 자식이 실제 크기를 가져야 함.
   - `WBP_InvSlot`을 **`Size Box`(WidthOverride/HeightOverride 64x64 등) → `Border` → `Image`** 구조로 재구성. **`Button`은 쓰지 말 것** (Button이 `OnMouseButtonDown`을 Handled로 소비해서 UserWidget 오버라이드가 안 불림).
   - 빈 슬롯도 `Border` 배경은 남기고 `Icon`만 Collapsed → 빈 칸에도 드롭 가능해짐.

4. **상위 위젯이 슬롯을 덮음** — `WBP_InventoryHUD` 루트 Canvas는 기본 `SelfHitTestInvisible`이지만, `WBP_Inventory`의 `Border_39`가 `Visible`이고 WrapBox보다 앞에 그려지면 클릭을 먹을 수 있음(현재 `Not Hit-Testable (Self Only)`로 바꿔놓긴 함). InventoryPanel(`WBP_Inventory`) 인스턴스 자체의 Visibility도 확인 (UserWidget 기본 `SelfHitTestInvisible` — 자식은 hit되므로 OK여야 함).

5. **`SetInputMode_GameAndUIEx`의 `InWidgetToFocus`** 를 `InventoryPanel`로 지정 안 됨 → 지정해보기.

6. **PIE 창 종류** — "Selected Viewport"에서 실행 중. 별도 창(Standalone/New Editor Window)에서 테스트, `Shift+F1`(마우스 컨트롤 토글) 상태 확인.

7. **간단 검증**: `WBP_InvSlot`에 임시로 `Button`을 루트로 넣고 `OnClicked`에 Print — 클릭이 먹히는지만 확인(먹히면 히트테스트는 되는 것 → 오버라이드/입력라우팅 문제로 좁혀짐). 확인 후 Button 제거.

---

## 14. 참고: 세션 중 겪은 다른 함정들 (이미 해결)

- `WBP_InventoryWidget`(플러그인) 은 `NarrativeCommonUI` 미설치로 로드 실패 → 커스텀 UI로 전환.
- `Content Browser`에서 플러그인 콘텐츠 안 보임 = "Show Plugin Content" 꺼짐.
- `WBP_EquipBar` 루트가 Canvas Panel이라 부모 안에서 0크기 → Horizontal Box로 Replace.
- `WBP_Inventory` For Loop `Last Index` 미설정(=0) → 1칸만 렌더. **0~11로 설정 필수.**
- `ReconcileSlots`의 `Array_Set` 와일드카드 "undetermined type" 반복 → `ForEach ArrayElement`(Object) 대신 `Items[i]`(Get a copy, NarrativeItem)를 쓰면 해결. 최종본은 단순화(`InvSlots = Items` + pad).
- `InvSlots`/`Items` 변수를 처음에 `Object` 배열로 만들어서 타입 미결정 → `Narrative Item` 배열로 변경해야 `Array_Set` 해결.
- BP 노드 그래프는 **클립보드 텍스트(`Begin Object ... End Object`) 복붙**으로 대량 이식 가능 (세션에서 `ReconcileSlots` 본문을 이 방식으로 교체함).

---

## 15. 빠른 재현 절차

1. `Test_LVStage1` 열고 PIE.
2. 시작 시 로그: `Adding 부적...`, `Adding Cube (Example Item)...` (아이템 2개 지급 확인).
3. **Q** → 화면 중앙에 반투명 패널 + 아이콘 2개, 하단에 부적 아이콘 1개, 마우스 커서 표시.
4. 슬롯 아이콘을 클릭/드래그 → **아무 반응 없음** (Print "MD"/"DRAG"/"DROP" 안 뜸, 위치 안 바뀜).
5. **Q** → 닫힘.
6. **1** → 부적이 손(카메라 앞)에 나타남. **1** → 사라짐 (부적은 인벤토리/EquipBar에 계속 있음).
