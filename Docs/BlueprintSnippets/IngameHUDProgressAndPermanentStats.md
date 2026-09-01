# 인게임 HUD 진행도·현재 Stat 표시와 영구 보너스 연결

기준 엔진: Unreal Engine 5.7.4, Blueprint-only
대상: `WBP_PlayerHUD`, `WBP_HUDStatRow`, `BP_RunnerPlayerController`, `BP_LevelManager`, `BP_RunnerPlayer`, `BP_RunnerGameInstance`

## 적용 상태

- 이 문서는 인게임 HUD에 골 지점까지의 ProgressBar와 현재 Stat 값을 표시하는 방법, `BP_RunnerGameInstance`의 영구 보너스 Stat을 `BP_RunnerPlayer`로 가져오는 방법을 기록한다.
- 현재 저장된 Blueprint의 컴파일과 PIE 결과를 완료로 판정하는 문서가 아니다. 아래 연결을 적용한 뒤 UE 5.7.4에서 별도로 검증해야 한다.
- 문서의 변수명과 실제 프로젝트 변수명이 다르면 타입과 역할이 같은 기존 변수를 연결한다.

## 1. 인게임 HUD에 골 지점까지의 ProgressBar 표시

### 1.1 필요한 값

`BP_LevelManager`에서 HUD로 아래 값을 전달한다.

| 값 | 타입 | 설명 |
|---|---|---|
| `CurrentStage` | Integer | 현재 Stage. 이 문서에서는 1부터 시작한다고 가정한다. |
| `PassedTilesInStage` | Integer | 현재 Stage에서 통과한 타일 수 |
| `TilesPerStage` | Integer | Stage당 타일 수. 현재 기준은 10 |
| `TotalStages` | Integer | 전체 Stage 수. 현재 기준은 3 |

전체 골 지점까지의 진행률은 다음과 같이 계산한다.

```text
TotalTiles = TotalStages * TilesPerStage
PassedTotal = ((CurrentStage - 1) * TilesPerStage) + PassedTilesInStage
Percent = Clamp(Float(PassedTotal) / Float(Max(TotalTiles, 1)), 0.0, 1.0)
```

`CurrentStage`가 0부터 시작하는 프로젝트라면 `CurrentStage - 1`을 사용하지 않는다.

### 1.2 `WBP_PlayerHUD` 위젯 구성

최소 구성은 아래와 같다.

```text
WBP_PlayerHUD
└─ CanvasPanel_Root
   └─ SafeZone_Root
      └─ VerticalBox_HUD
         ├─ Text_Stage
         ├─ Text_GoalProgress
         ├─ ProgressBar_Goal
         └─ VerticalBox_Stats
```

`ProgressBar_Goal.Percent`에는 타일 수 자체가 아니라 위 계산의 `Percent`를 넣는다. ProgressBar의 Percent 범위는 `0.0`부터 `1.0`이다.

`WBP_PlayerHUD`에 함수 또는 Custom Event를 만든다.

```text
UpdateGoalProgress
Inputs
  CurrentStage       : Integer
  PassedTilesInStage : Integer
  TilesPerStage      : Integer
  TotalStages        : Integer
```

노드 연결:

```text
UpdateGoalProgress
→ TotalTiles 계산
→ PassedTotal 계산
→ Integer 값을 Float로 변환
→ PassedTotal / Max(TotalTiles, 1)
→ Clamp (Float, 0.0, 1.0)
→ ProgressBar_Goal.SetPercent
```

표시 Text 예시:

```text
Text_Stage        = "Stage {CurrentStage} / {TotalStages}"
Text_GoalProgress = "{PassedTotal} / {TotalTiles}"
```

Integer끼리 먼저 나누면 진행률이 중간 구간에서 0으로 남을 수 있다. 나누기 전에 두 값을 Float로 변환한다.

### 1.3 갱신 시점

ProgressBar는 Tick에서 계산하지 않는다. 타일 통과 수가 실제로 변경되는 지점에서 한 번만 HUD를 갱신한다.

최소 연결:

```text
BP_LevelManager
타일 통과 처리
→ PassedTilesInStage 증가
→ BP_RunnerPlayerController의 HUD 참조 확인
→ WBP_PlayerHUD.UpdateGoalProgress 호출
```

권장 연결은 `BP_LevelManager`에 `OnStageProgressChanged` Event Dispatcher를 만들고 `BP_RunnerPlayerController`가 HUD 생성 후 한 번 Bind하는 방식이다.

## 2. 인게임 HUD에 현재 Stat 값 표시

HUD는 `GameInstance.PermanentBonusStats`를 직접 표시하지 않는다. `BP_RunnerPlayer.UpdateStats`가 계산한 최종 결과인 `CurrentStats.StatusValue`를 표시한다.

최종 값 기준:

```text
CurrentStats = BaseStats + RunBoundStats + PermanentBonusStats
```

### 2.1 `WBP_HUDStatRow` 구성

최소 계층:

```text
WBP_HUDStatRow
└─ HorizontalBox_Row
   ├─ Text_StatName
   ├─ Spacer_Value
   └─ Text_StatValue
```

변수:

| 변수 | 타입 | 설정 |
|---|---|---|
| `StatType` | `E_Stats` | Instance Editable, Expose on Spawn |
| `StatName` | Text | Instance Editable, Expose on Spawn |
| `StatValue` | Float | Instance Editable, Expose on Spawn |

`SetStatData` 함수 또는 Custom Event를 만든다.

```text
SetStatData(StatType, StatName, StatValue)
→ Text_StatName.SetText(StatName)
→ StatValue를 Text로 변환
→ Text_StatValue.SetText
```

`ArrowCount`처럼 정수로 표시해야 하는 항목은 `Round` 또는 `Truncate` 후 Integer Text로 변환한다. 나머지 값은 소수 자릿수를 0~2자리로 제한한다.

### 2.2 Row 생성과 값 연결

`WBP_PlayerHUD`의 `VerticalBox_Stats`에서 표시할 `E_Stats` 목록을 순회한다.

```text
DisplayedStats For Each Loop
→ Create Widget (WBP_HUDStatRow)
→ StatType = Array Element
→ BP_RunnerPlayer.CurrentStats.StatusValue.Find(Array Element)
→ Found Branch
   True  → 찾은 값을 StatValue로 설정
   False → 0.0을 StatValue로 설정
→ VerticalBox_Stats.AddChild
```

현재 프로젝트의 표시 후보:

```text
ArrowDamage
ArrowRange
ArrowSpeed
ArrowCount
HorizonSpeed
```

Widget을 다시 초기화할 때는 `VerticalBox_Stats.ClearChildren`을 먼저 호출해 Row가 중복 생성되지 않게 한다.

### 2.3 Stat 갱신 시점

`BP_RunnerPlayer.UpdateStats`가 `CurrentStats` 계산을 완료한 뒤 HUD를 갱신한다.

```text
BP_RunnerPlayer.UpdateStats
→ For Each Loop Completed
→ Set CurrentStats
→ OnStatsUpdated Call
```

`BP_RunnerPlayerController` 또는 `WBP_PlayerHUD`는 `OnStatsUpdated`에 한 번 Bind하고, 이벤트가 호출되면 각 Row의 값을 다시 설정한다. UMG Property Binding이나 Tick에서 Map을 계속 조회하지 않는다.

## 3. `GameInstance.PermanentBonusStats`를 가져오지 못하는 문제

### 3.1 첨부 그래프에서 확인된 원인

첨부된 `BP_RunnerPlayer.UpdateStats` 그래프에는 다음 문제가 있다.

1. `Get Game Instance`는 `Cast To BP_RunnerGameInstance.Object`에 연결돼 있지만 Cast의 흰색 실행 입력 핀이 연결되지 않았다.
2. 함수 Entry의 실행선은 Cast가 아니라 바로 `For Each Loop`로 들어간다. 따라서 Cast 성공 뒤의 `Set PlayerPermanentBonusStats`가 실행되지 않는다.
3. `Set PlayerPermanentBonusStats`의 값 입력 핀이 비어 있다.
4. `BP_RunnerGameInstance`의 화면상 변수 `PermentBonusStat`은 `E_Stats → Float Map`이고, 플레이어의 `PlayerPermanentBonusStats`는 `ST_PlayerStats` Struct다. Map을 Struct 변수에 바로 연결할 수 없다.
5. `BP_RunnerGameInstance.PermentBonusStat`의 `Private`가 체크돼 있다. 다른 Blueprint에서 변수 Get 노드를 직접 사용하려면 `Private`를 해제하거나 GameInstance에 Getter 함수를 만들어야 한다.

위 상태에서는 `PlayerPermanentBonusStats.StatusValue`가 기본값 또는 빈 Map으로 남으므로 `Find` 결과가 전부 0처럼 보일 수 있다.

### 3.2 GameInstance 클래스 설정 확인

먼저 아래 설정을 확인한다.

```text
Project Settings
→ Maps & Modes
→ Game Instance Class
→ BP_RunnerGameInstance
```

현재 `Config/DefaultEngine.ini`에는 다음 클래스가 설정돼 있다.

```text
/Game/GameFrameWork/BP_RunnerGameInstance.BP_RunnerGameInstance_C
```

PIE에서 Cast가 실패한다면 실행 중인 GameInstance 클래스가 이 설정과 일치하는지 다시 확인한다.

### 3.3 최소 수정 방법

`BP_RunnerGameInstance`의 실제 변수명은 화면상 `PermentBonusStat`으로 보인다. 문서에서는 의미가 분명하도록 `PermanentBonusStats`라고 부른다.

방법 A: 변수의 `Private`를 해제한다.

```text
BP_RunnerGameInstance
→ PermanentBonusStats 선택
→ Details
→ Private 체크 해제
```

방법 B: `Private`를 유지하고 Getter 함수를 만든다. 이 방법을 권장한다.

```text
Function: GetPermanentBonusStats
Output: PermanentBonusStats (Map<E_Stats, Float>)

Get PermanentBonusStats
→ Return Node.PermanentBonusStats
```

### 3.4 `BP_RunnerPlayer.UpdateStats` 실행선 수정

함수 Entry가 먼저 Cast를 실행하고, 데이터 복사가 끝난 다음 Stat 합산 Loop를 실행해야 한다.

```text
UpdateStats Entry.Exec
→ Get Game Instance
→ Cast To BP_RunnerGameInstance.Exec
→ Cast Succeeded
→ GetPermanentBonusStats
→ PlayerPermanentBonusStats 갱신
→ For Each Loop
```

`PlayerPermanentBonusStats`가 `ST_PlayerStats`라면 Map을 Struct의 `StatusValue` 멤버에 넣는다.

```text
Get PlayerPermanentBonusStats
→ Set Members in ST_PlayerStats
   StatusValue = GameInstance.GetPermanentBonusStats 출력 Map
→ Set PlayerPermanentBonusStats
```

실행선:

```text
Cast Succeeded
→ Set PlayerPermanentBonusStats
→ For Each Loop
```

Cast 실패 경로에는 Print String을 임시로 연결해 설정 오류를 확인하고, 문제를 해결한 뒤 디버그 출력은 제거한다.

### 3.5 Stat 합산 연결

`For Each Loop.Array Element`인 `E_Stats` 값을 세 Map의 `Find.Key`에 동일하게 연결한다.

```text
BaseStats.StatusValue.Find(Array Element)
RunBoundStats.StatusValue.Find(Array Element)
PlayerPermanentBonusStats.StatusValue.Find(Array Element)
```

세 값을 더한 결과를 새 Map에 넣고, Loop 완료 후 `CurrentStats.StatusValue`로 설정한다.

```text
Base Value + Run Value + Permanent Value
→ Current Status Map.Add
   Key   = Array Element
   Value = 합산값

For Each Loop.Completed
→ Set Members in ST_PlayerStats
   StatusValue = Current Status Map
→ Set CurrentStats
→ HUD 갱신
```

Map의 `Find`에서 `Found = false`이면 해당 항목의 값은 0으로 처리한다. `StatsList`의 Enum 값과 GameInstance Map의 Key가 일치해야 한다.

### 3.6 레벨을 다시 들어갈 때

`GameInstance`는 레벨 전환 후에도 유지되지만 `BP_RunnerPlayer`는 인게임 맵에 들어올 때 새로 생성된다. 플레이어 참조나 복사한 Map이 자동으로 유지된다고 가정하면 안 된다.

```text
BP_RunnerPlayer.Event BeginPlay
→ InitStatus
→ GameInstance의 PermanentBonusStats 가져오기
→ UpdateStats
```

상점 구매 후 인게임으로 새로 들어올 때도 위 흐름이 다시 실행돼야 한다. 같은 인게임 세션에서 영구 강화가 바뀌는 구조라면 구매 성공 후 `UpdateStats`를 다시 호출한다.

## 4. UE 5.7.4 검증 순서

1. `BP_RunnerGameInstance`, `BP_RunnerPlayer`, `BP_LevelManager`, `BP_RunnerPlayerController`, `WBP_PlayerHUD`, `WBP_HUDStatRow`를 모두 Compile한다.
2. 메인 메뉴에서 인게임 맵으로 들어가 HUD가 한 번만 생성되는지 확인한다.
3. 타일을 통과할 때 ProgressBar가 0~1 사이에서 증가하고 Text가 `PassedTotal / TotalTiles`로 바뀌는지 확인한다.
4. `CurrentStats.StatusValue`에 있는 각 항목이 HUD Row에 표시되는지 확인한다.
5. GameInstance의 영구 보너스 Map에서 한 항목을 0이 아닌 값으로 설정하고 인게임에 들어간다.
6. `CurrentStats = Base + Run + Permanent` 결과가 HUD에 표시되는지 확인한다.
7. 메인 메뉴로 나갔다가 다시 인게임에 들어가도 새 플레이어가 GameInstance 값을 다시 가져오는지 확인한다.
8. Map에 없는 Key는 오류 없이 0으로 표시되는지 확인한다.

## 5. 완료 기준

- ProgressBar가 전체 골 지점 기준으로 증가한다.
- HUD Stat Row가 `BP_RunnerPlayer.CurrentStats`의 현재 값을 표시한다.
- `BP_RunnerPlayer.UpdateStats`의 Cast와 Map 복사 흐름이 실행선에 포함된다.
- 영구 보너스가 Base·런 중 보너스와 합산된다.
- 메인 메뉴 → 인게임 재진입 후에도 같은 영구 보너스가 적용된다.
- Blueprint 컴파일 오류와 PIE 진행 불가가 없다.
