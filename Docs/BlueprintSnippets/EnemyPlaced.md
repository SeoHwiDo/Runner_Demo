# `EnemyPlaced` 적 배치 설계

기준: Unreal Engine 5.7.4, Blueprint-only
상태: 설계 갱신. Blueprint 그래프 적용과 Compile·PIE 검증은 별도 작업이다.

## 1. 목표

타일에 미리 둔 강화 표지판 2개는 초기화 시 비활성화한다. Enemy 배치 요청이 있는 타일에서만 표지판을 먼저 활성화한 뒤, Enemy를 다음 우선순위로 풀에서 대여·활성화한다.

```text
Boss → Elite → Normal
```

모든 Enemy는 서로 겹치지 않으며, 중심점 사이에 `MonsterMinCenterDistance` 이상의 간격을 둔다. 이 값에는 Enemy 충돌 범위와 플레이어 1명 통과 여유를 포함한다.

`NormalEnemyBox`는 더 이상 Enemy 배치에 사용하지 않는다.

## 2. 타일 공간 규칙

타일 진행 방향은 `TileFront → TileBack`이다.

```text
TileFront
→ 비활성 상태로 미리 배치된 강화 표지판 예약 영역
→ Boss / Elite 예약 영역
→ Normal 2레인 배치 영역
→ TileBack
```

### `BP_BaseTile` 필수 항목

```text
Tiles                       : Static Mesh Component
TileFront                   : Scene Component
TileBack                    : Scene Component
BossElitePos01              : Scene Component
BossElitePos02              : Scene Component
TileEnemies                 : BP_EnemyBase Object Reference Array
MonsterMinCenterDistance    : Float
SignReservedLength          : Float
강화 표지판 2개             : 타일에 미리 배치한 Component 또는 Child Actor
LeftSignStatus              : E_Status
LeftSignValue               : Float
RightSignStatus             : E_Status
RightSignValue              : Float
```

권장 초기값:

```text
MonsterMinCenterDistance = 300.0
```

`SignReservedLength`는 강화 표지판 2개가 차지하는 `TileFront` 쪽 길이다. 표지판의 실제 폭과 간격을 기준으로 설정한다.

### 컴포넌트 배치 조건

- 강화 표지판은 `TileFront`부터 `SignReservedLength` 안에 미리 배치한다.
- `BossElitePos01`, `BossElitePos02`는 표지판 예약 영역 뒤에 배치한다.
- 두 Boss/Elite 위치와 Normal 시작 위치의 2D 거리는 모두 `MonsterMinCenterDistance` 이상이어야 한다.
- `BossElitePos01`과 `BossElitePos02`의 2D 거리도 `MonsterMinCenterDistance` 이상이어야 한다.
- `Tiles` 폭의 절반은 `MonsterMinCenterDistance` 이상이어야 같은 행의 Normal 2레인이 서로 겹치지 않는다.

## 3. 호출 순서

타일 재사용 시 실행 순서를 고정한다.

```text
기존 TileEnemies 반환
→ TileEnemies Clear
→ 타일 새 위치로 이동
→ BP_BaseTile 초기화: 강화 표지판 2개 비활성화
→ CalculateStage
→ EnemyPlaced(TileRef)
```

`BP_TileManager`는 표지판을 Spawn하거나 Destroy하지 않는다. 표지판은 `BP_BaseTile`에 미리 배치한다.

`BP_BaseTile` 초기화는 표지판 2개를 모두 비활성화한다. 표지판 비활성화에는 최소한 표시와 Collision 비활성화가 포함된다.

Enemy가 활성화에 성공하면 즉시 `TileEnemies`에 `Add Unique`한다. 다음 우선순위 Enemy는 이 배열에 등록된 위치를 기준으로 여유 공간을 검사한다.

## 4. `EnemyPlaced`

현재 에셋의 함수명 `PlacedEnemyTypeIOnTile`과 `SeclectSpawnTransform`은 오탈자가 있어도 이번 제출 범위에서는 이름을 유지한다.

유효한 Pattern과 `SpawnCount > 0`이 확인된 타일에서만, 첫 `PlacedEnemyTypeIOnTile` 호출 전에 타일에 미리 둔 강화 표지판 2개를 활성화한다.

```text
유효한 Enemy 요청 없음
→ 초기화 상태의 표지판 비활성화 유지
→ Success = false

유효한 Enemy 요청 있음
→ PlaceUpgradeSigns(TileRef)
→ Boss → Elite → Normal 순서의 Enemy 배치 시작
```

풀 고갈로 Enemy 대여가 실패해도, 해당 타일은 Enemy 배치 요청이 있었으므로 활성화한 표지판은 유지한다. 표지판 활성화 기준은 실제 Enemy 활성화 성공 여부가 아니라 유효한 Enemy 배치 요청 여부다.

## 5. `PlaceUpgradeSigns`

`BP_LevelManager`에 구현한다. 표지판을 Spawn하지 않고, `TileRef`의 `LeftSign`, `RightSign`과 해당 강화 데이터를 설정한다.

### 고정 강화값

```text
HorizonSpeed = +100.0
ArrowRange   = +200.0
ArrowSpeed   = +150.0
ArrowCount   = +1.0
```

### 실행 흐름

```text
PlaceUpgradeSigns(TileRef)
→ E_Status 배열 생성
  [HorizonSpeed, ArrowRange, ArrowSpeed, ArrowCount]
→ Shuffle
→ LeftStatus  = Array[0]
→ RightStatus = Array[1]
→ GetFixedStatusValue(LeftStatus)
→ GetFixedStatusValue(RightStatus)
→ TileRef.LeftSignStatus  설정
→ TileRef.LeftSignValue   설정
→ TileRef.RightSignStatus 설정
→ TileRef.RightSignValue  설정
→ LeftSign / RightSign 표시와 Collision 활성화
```

`Shuffle` 후 앞의 두 원소를 사용하므로 두 표지판에는 서로 다른 Status가 들어간다.

### `GetFixedStatusValue`

`BP_LevelManager`의 Pure 함수로 구현한다.

```text
Input: Status : E_Status
Output: Value : Float

Switch on E_Status
  HorizonSpeed → 100.0
  ArrowRange   → 200.0
  ArrowSpeed   → 150.0
  ArrowCount   → 1.0
```

```text
Early 타일
→ PlacedEnemyTypeIOnTile(TileRef, Normal, NormalCount)

Mid 타일
→ PlacedEnemyTypeIOnTile(TileRef, Elite, MidEliteCount)
→ PlacedEnemyTypeIOnTile(TileRef, Normal, NormalCount)

Boss 타일
→ PlacedEnemyTypeIOnTile(TileRef, Boss, BossCount)
```

같은 타일에 세 Type을 모두 요청하는 경우에도 호출 순서는 항상 아래와 같다.

```text
Boss → Elite → Normal
```

`Mid` 구간의 기존 `Normal → Elite` 호출 순서는 `Elite → Normal`으로 변경한다.

## 6. `PlacedEnemyTypeIOnTile`

```text
AcquireEnemy
→ SeclectSpawnTransform
→ ActivateFromPool(SpawnTransform, TileRef)
→ 성공한 Enemy만 TileRef.TileEnemies Add Unique
```

`SeclectSpawnTransform`이 실패하면 대여한 Enemy를 `ReturnEnemy`로 반환하고 해당 요청을 생략한다. 풀 고갈 시 새 Actor를 Spawn하거나 활성 Actor를 강제 회수하지 않는다.

## 7. `SeclectSpawnTransform`

### 함수 시그니처

```text
Input
  TileRef     : BP_BaseTile Object Reference
  EnemyType   : E_EnemyType
  SpawnIndex  : Integer
  SpawnCount  : Integer

Output
  SpawnTransform   : Transform
  transformSuccess : Boolean
```

`SpawnCount`는 위치 계산에 사용하지 않는다. Normal 위치는 `SpawnIndex`를 이용한 고정 슬롯으로 결정한다.

### 6.1 Boss / Elite

```text
SpawnIndex % 2
  0 → BossElitePos01.GetComponentTransform
  1 → BossElitePos02.GetComponentTransform
```

선택한 Transform 후보와 `TileEnemies`의 모든 유효 Enemy 위치를 2D 거리로 비교한다.

```text
Distance2D(Candidate, ExistingEnemy) >= MonsterMinCenterDistance
```

조건을 만족하면 `SpawnTransform`으로 반환한다. 만족하지 않으면 `transformSuccess = false`로 반환한다.

### 6.2 Normal

Normal은 `Tiles`의 Local Bounds에서 Y 폭을 구해, 타일 폭을 반으로 나눈 두 구역의 중앙에만 배치한다.

```text
Tiles.GetLocalBounds
→ LocalMin, LocalMax

LaneIndex = SpawnIndex % 2
RowIndex  = SpawnIndex / 2   // 정수 나눗셈

LaneAlpha = 0.25 + (LaneIndex * 0.5)
LocalY    = Lerp(LocalMin.Y, LocalMax.Y, LaneAlpha)
```

따라서 Normal의 Y 위치는 다음 두 값이다.

```text
LocalMin.Y + (전체 Y 폭 * 0.25)
LocalMin.Y + (전체 Y 폭 * 0.75)
```

Normal의 진행 방향 거리 계산:

```text
NormalStartDistance
  = SignReservedLength
  + (MonsterMinCenterDistance * 2)

ForwardDistance
  = NormalStartDistance
  + (RowIndex * MonsterMinCenterDistance)
```

`TileFront` 월드 위치에서 `Normalize(TileBack - TileFront)` 방향으로 `ForwardDistance`만큼 이동하고, `Tiles`의 Right Vector 방향으로 `LocalY`만큼 이동해 후보 위치를 만든다.

```text
CandidateLocation
  = TileFront.WorldLocation
  + ForwardDirection * ForwardDistance
  + Tiles.RightVector * LocalY
```

`NormalStartDistance` 앞의 두 간격은 Boss/Elite 예약 공간이다. Boss 또는 Elite가 없는 타일에서도 같은 공간을 예약해 모든 타일의 배치 규칙을 동일하게 유지한다.

후보 위치가 `TileEnemies`의 유효 Enemy와 `MonsterMinCenterDistance`보다 가까우면 다음 행을 검사한다.

```text
TryRow = RowIndex부터 타일 끝 전까지 반복
→ 같은 Lane의 CandidateLocation 계산
→ 모든 Existing Enemy와 Distance2D 검사
→ 비어 있으면 SpawnTransform 반환
```

빈 행이 없으면 `transformSuccess = false`를 반환한다.

### 6.3 Transform 공통 설정

```text
Location = 선택한 후보 월드 위치
Rotation = Tiles.GetComponentRotation
Scale    = (1, 1, 1)
```

Enemy가 플레이어 반대 방향을 보아야 하는 경우에는 `Rotation`만 별도 조정한다. 위치 계산 규칙은 변경하지 않는다.

## 8. 제외 사항

- `NormalEnemyBox` 기반 범위 추첨
- Local X 구간 분할 및 Random Float 배치
- Tick 기반 Enemy 검색
- `Get All Actors Of Class`로 타일 소유 Enemy 검색
- 풀 고갈 시 새 Enemy Spawn 또는 활성 Enemy 강제 회수

## 9. 사용자 검증 항목

```text
1. 타일 최초 생성과 재사용 초기화 시 강화 표지판 2개가 비활성화되는지 확인
2. Enemy 배치 요청이 없는 타일에서는 강화 표지판이 비활성화 상태로 유지되는지 확인
3. Enemy 배치 요청이 있는 타일에서 두 표지판의 Status가 서로 다르고 고정값이 저장되는지 확인
4. Enemy 배치 요청이 있는 타일에서는 강화 표지판이 Enemy보다 먼저 활성화되는지 확인
5. Mid 타일에서 Elite가 Normal보다 먼저 활성화되는지 확인
6. Boss / Elite / Normal이 서로 MonsterMinCenterDistance 이상 떨어지는지 확인
7. Normal이 타일 폭의 좌·우 절반 중앙 레인만 사용하는지 확인
8. Normal이 표지판 및 Boss/Elite 예약 영역 뒤부터 배치되는지 확인
9. 타일 재사용 시 이전 TileEnemies가 반환되고 새 타일 Enemy만 등록되는지 확인
10. 풀 고갈 또는 빈 슬롯에서 새 Actor Spawn 없이 요청이 생략되는지 확인
```
