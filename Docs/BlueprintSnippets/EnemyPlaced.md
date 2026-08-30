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
- `LeftSign`과 `RightSign`의 월드 Y 좌표 차이는 `MonsterMinCenterDistance` 이상이어야 같은 행의 Normal 2레인이 서로 겹치지 않는다.

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

## 7. `SeclectSpawnTransform` Blueprint 노드 작성

현재 에셋 함수명은 오탈자를 포함한 `SeclectSpawnTransform`이다. 함수명은 이번 작업에서 변경하지 않는다.

이 함수는 **후보 위치 계산과 기존 Enemy 간격 검사만** 수행한다. Enemy 대여·반환과 `ActivateFromPool`은 `PlacedEnemyTypeIOnTile`에 남긴다.

### 7.1 함수 시그니처

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

`SpawnCount`는 이 함수에서 사용하지 않는다. Normal의 레인과 행은 `SpawnIndex`로만 결정한다.

### 7.2 로컬 변수

함수 Details의 **로컬 변수**에 아래 항목을 만든다.

| 변수 | Type | 기본값 | 용도 |
|---|---|---:|---|
| `CandidateLocation` | Vector | `(0, 0, 0)` | 현재 검사 중인 후보 위치 |
| `CandidateTransform` | Transform | Identity | Boss/Elite 후보 Transform |
| `bHasOverlap` | Boolean | `false` | 현재 후보가 기존 Enemy와 겹치는지 여부 |
| `bFoundTransform` | Boolean | `false` | 빈 슬롯을 찾았는지 여부 |
| `LaneIndex` | Integer | `0` | Normal 좌·우 레인 인덱스 |
| `StartRowIndex` | Integer | `0` | `SpawnIndex`가 지정한 첫 행 |
| `MaxRowIndex` | Integer | `0` | 타일 안에서 검사 가능한 마지막 행 |
| `SignWorldY` | Float | `0.0` | 선택한 표지판과 동일한 Normal 월드 Y 위치 |
| `NormalStartDistance` | Float | `0.0` | 표지판·Boss/Elite 예약 영역 뒤 시작 거리 |

`bFoundTransform`과 `bHasOverlap`은 함수 시작 시 `false`로 설정한다. `SpawnTransform`과 `transformSuccess`는 로컬 변수가 아니라 `Function Result`의 출력 핀이므로 중간에 별도 `Set` 노드를 만들지 않는다. 현재 존재하는 `Function Result` 노드 하나를 그대로 사용한다.

### 7.3 전체 실행 구조

현재 만들어 둔 `TileEnemys → For Each Loop With Break → Distance 2D → < (Float)` 블록은 삭제하지 않는다. 이 블록의 앞에 후보 Transform 계산을 추가하고, 뒤에 후보 채택과 외부 반복 종료를 추가한다.

전체 실행선은 아래 순서다.

```text
Function Entry
→ Set bFoundTransform = false
→ Set bHasOverlap = false
→ Set LaneIndex
→ Set StartRowIndex
→ Set SignWorldY
→ Set NormalStartDistance
→ Set MaxRowIndex
→ For Loop With Break                    // 후보 반복
    First Index = Select(Integer)
      EnemyType == Normal : StartRowIndex
      EnemyType != Normal : SpawnIndex
    Last Index = Select(Integer)
      EnemyType == Normal : MaxRowIndex
      EnemyType != Normal : 1

For Loop With Break.Loop Body
→ BossEliteCandidateTransform 계산
→ NormalCandidateTransform 계산
→ Select(Transform)
    EnemyType == Normal : NormalCandidateTransform
    EnemyType != Normal : BossEliteCandidateTransform
→ Set CandidateTransform
→ Break Transform.Location
→ Set CandidateLocation
→ Set bHasOverlap = false
→ For Each Loop With Break               // TileEnemys 간격 검사
    Distance 2D < MonsterMinCenterDistance
    → Set bHasOverlap = true
    → For Each Loop With Break.Break

For Each Loop With Break.Completed
→ NOT Boolean(bHasOverlap)
→ Branch
    True
    → Set bFoundTransform = true
    → For Loop With Break.Break

    False
    → 실행 핀을 연결하지 않음
    → 현재 Loop Body가 끝나면서 다음 후보 Index 검사

For Loop With Break.Completed
→ Function Result
    SpawnTransform = Select(Transform)
      bFoundTransform = true  : CandidateTransform
      bFoundTransform = false : Identity Transform
    transformSuccess = bFoundTransform
```

후보 반복 범위는 다음과 같다.

| EnemyType | First Index | Last Index | 의미 |
|---|---:|---:|---|
| `Boss`, `Elite` | `SpawnIndex` | `1` | `BossElitePos01`, `BossElitePos02` 중 현재 슬롯부터 검사 |
| `Normal` | `StartRowIndex` | `MaxRowIndex` | 요청 레인의 시작 행부터 타일 끝까지 검사 |

Boss/Elite의 `SpawnIndex`는 `0` 또는 `1`이어야 한다. `SpawnIndex = 0`인 후보가 겹치면 Index `1`의 `BossElitePos02`를 다시 검사한다. `SpawnIndex > 1`이면 반복할 고정 위치가 없으므로 `transformSuccess = false`를 반환한다.

#### 전체 구조에 필요한 UE 5.7.4 노드

```text
== (Enum)
% (Integer)
/ (Integer)
== (Integer)
Select                         // Integer, Float, Transform
For Loop With Break
For Each Loop With Break
Get World Location
Get Component Transform
Get Component Rotation
Get Unit Direction (Vector)
Distance 2D
Break Vector
Make Vector
Make Transform
Break Transform
+ (Float)
- (Float)
* (Float)
/ (Float)
* (Vector * Float)
+ (Vector + Vector)
Floor
< (Float)
NOT Boolean
Set CandidateTransform
Set CandidateLocation
Set bHasOverlap
Set bFoundTransform
Function Result
```

`Select` 노드는 연결한 핀의 Type에 따라 `Integer`, `Float`, `Transform`으로 자동 결정한다. Type이 결정되지 않으면 `Select` 노드의 출력 핀을 먼저 대상 핀에 연결한다.

### 7.4 공통 간격 검사 블록

Boss, Elite, Normal 모두 아래 블록으로 후보 위치를 검사한다. 현재 `BP_BaseTile` 배열 이름은 `TileEnemys`이므로 이 식별자를 그대로 사용한다.

#### 필요한 UE 5.7.4 노드

```text
For Each Loop With Break
Get Actor Location
Distance 2D
< (Float)
Branch
Set bHasOverlap
```

#### 노드 연결

```text
TileRef
→ Get TileEnemys
→ For Each Loop With Break.Array

For Each Loop With Break.Loop Body
→ Get Actor Location
    Target = Array Element

CandidateLocation
→ Distance 2D.A

Get Actor Location.Return Value
→ Distance 2D.B

Distance 2D.Return Value
→ < (Float).A

TileRef.MonsterMinCenterDistance
→ < (Float).B

< (Float).Return Value
→ Branch.Condition
```

`Distance 2D < MonsterMinCenterDistance`이면 겹침이다.

```text
Branch.True
→ Set bHasOverlap = true
→ For Each Loop With Break.Break

Branch.False
→ 실행 핀을 연결하지 않음
→ 현재 요소 검사가 끝나면서 다음 Array Element 검사
```

각 후보 검사 전에 반드시 아래 순서로 초기화한다.

```text
Set bHasOverlap = false
→ For Each Loop With Break
```

`For Each Loop With Break.Completed`에서 `NOT bHasOverlap`을 `Branch`로 검사한다. `false`이면 외부 `For Loop With Break`의 다음 후보를 검사한다. `true`이면 `bFoundTransform = true`로 설정하고 외부 반복의 `Break` 핀을 호출한다.

### 7.5 Boss / Elite 후보 Transform

Boss와 Elite는 `BossElitePos01`, `BossElitePos02`만 사용한다.

#### 필요한 UE 5.7.4 노드

```text
== (Enum)                 // EnemyType == Normal 검사
== (Integer)
Get Component Transform   // BossElitePos01, BossElitePos02
Select                    // Type: Transform
```

#### 실행 및 데이터 핀 연결

먼저 Normal 여부를 Pure 비교 노드 하나로 만든다. 이 결과는 후보 반복 범위와 최종 `Select(Transform)`에서 함께 사용한다.

```text
EnemyType
→ == (Enum).A

Normal
→ == (Enum).B
```

```text
For Loop With Break.Index
→ == (Integer).A

== (Integer).B = 0

== (Integer).Return Value
→ Select(Transform).Index

TileRef
→ Get Boss Elite Pos02
→ Get Component Transform
→ Select(Transform).False

TileRef
→ Get Boss Elite Pos01
→ Get Component Transform
→ Select(Transform).True
```

위 연결은 외부 후보 Index `0`에 `BossElitePos01`, Index `1`에 `BossElitePos02`를 선택한다. `SpawnIndex`는 외부 반복의 시작 Index에만 사용한다.

이 `Select(Transform).Return Value`는 7.7의 최종 `Select(Transform).False`에 연결한다. Boss/Elite 후보와 Normal 후보 중 실제로 사용할 Transform을 선택한 다음에만 `CandidateTransform`과 `CandidateLocation`을 설정한다.

### 7.6 Normal 레인과 첫 행 계산

Normal용 계산값은 외부 후보 반복 전에 만든다. `EnemyType != Normal`일 때도 Pure 데이터 노드는 평가될 수 있지만, 최종 `Select`에서 사용되지 않는다.

#### 필요한 UE 5.7.4 노드

```text
% (Integer)
/ (Integer)
== (Integer)
Select                       // Type: Float
Get World Location           // LeftSign, RightSign
Break Vector                 // LeftSignLocation, RightSignLocation 각각
Get World Location           // TileFront, TileBack
Distance 2D
- (Float)
/ (Float)
Floor
Set LaneIndex
Set StartRowIndex
Set SignWorldY
Set NormalStartDistance
Set MaxRowIndex
```

#### 레인 위치

```text
SpawnIndex
→ % (Integer).A

% (Integer).B = 2

% (Integer).Return Value
→ Set LaneIndex.Value
→ == (Integer).A

== (Integer).B = 0
→ Select(Float).Index

TileRef
→ Get Left Sign
→ Get World Location
→ Break Vector
→ Select(Float).True

TileRef
→ Get Right Sign
→ Get World Location
→ Break Vector
→ Select(Float).False

Select(Float).Return Value
→ Set SignWorldY.Value
```

결과적으로 `SpawnIndex` 짝수는 `LeftSign`과 동일한 월드 Y, 홀수는 `RightSign`과 동일한 월드 Y를 사용한다. 표지판의 X와 Z 위치는 Normal에 사용하지 않는다.

#### 행 인덱스와 마지막 검사 행

```text
SpawnIndex
→ / (Integer).A

/ (Integer).B = 2

/ (Integer).Return Value
→ Set StartRowIndex.Value
```

표지판과 Boss/Elite 예약 거리를 계산한다.

```text
TileRef.SignReservedLength
→ + (Float).A

TileRef.MonsterMinCenterDistance
→ * (Float).A

* (Float).B = 2.0

* (Float).Return Value
→ + (Float).B

+ (Float).Return Value
→ Set NormalStartDistance.Value
```

타일 안에서 검사 가능한 마지막 행은 `TileFront`와 `TileBack`의 월드 2D 거리로 구한다.

```text
TileRef
→ Get Tile Front
→ Get World Location
→ Distance 2D.A

TileRef
→ Get Tile Back
→ Get World Location
→ Distance 2D.B

Distance 2D.Return Value
→ - (Float).A

NormalStartDistance
→ - (Float).B

- (Float).Return Value
→ / (Float).A

TileRef.MonsterMinCenterDistance
→ / (Float).B

/ (Float).Return Value
→ Floor
→ Set MaxRowIndex.Value
```

### 7.7 Normal 후보 행 반복

#### 필요한 UE 5.7.4 노드

```text
For Loop With Break
Get Unit Direction (Vector)
* (Vector * Float)
+ (Vector + Vector)
Break Vector                 // 진행 방향 후보 위치
Make Vector
Make Transform
Get Component Rotation
Set CandidateLocation
Set SpawnTransform
Set transformSuccess
NOT Boolean
Branch
```

#### 반복 시작

```text
Set MaxRowIndex.Exec
→ For Loop With Break.Exec

EnemyType == Normal
→ Select(Integer).Index

StartRowIndex
→ Select(Integer).True

SpawnIndex
→ Select(Integer).False

Select(Integer).Return Value
→ For Loop With Break.First Index

EnemyType == Normal
→ 두 번째 Select(Integer).Index

MaxRowIndex
→ 두 번째 Select(Integer).True

두 번째 Select(Integer).False = 1

두 번째 Select(Integer).Return Value
→ For Loop With Break.Last Index
```

#### 반복 본문: 후보 월드 위치

`For Loop With Break.Index`를 `TryRow`로 사용한다.

```text
NormalStartDistance
→ + (Float).A

For Loop With Break.Index
→ * (Integer * Float).A

TileRef.MonsterMinCenterDistance
→ * (Integer * Float).B

* (Integer * Float).Return Value
→ + (Float).B
```

`+ (Float).Return Value`가 현재 행의 `ForwardDistance`다.

진행 방향은 `TileFront`에서 `TileBack`으로 계산한다.

```text
TileRef
→ Get Tile Front
→ Get World Location
→ Get Unit Direction (Vector).From

TileRef
→ Get Tile Back
→ Get World Location
→ Get Unit Direction (Vector).To

Get Unit Direction (Vector).Return Value
→ * (Vector * Float).A

ForwardDistance
→ * (Vector * Float).B
```

먼저 진행 방향 기준 후보 위치를 만든다.

```text
TileRef
→ Get Tile Front
→ Get World Location
→ + (Vector + Vector).A

ForwardDirection * ForwardDistance
→ + (Vector + Vector).B

+ (Vector + Vector).Return Value
→ Break Vector(BaseLocation)

Break Vector(BaseLocation).X
→ Make Vector.X

SignWorldY
→ Make Vector.Y

Break Vector(BaseLocation).Z
→ Make Vector.Z

Make Vector.Return Value
→ Set CandidateLocation.Value
```

`Make Vector.Y`에 `SignWorldY`를 직접 연결하므로, Normal은 표지판이 Local 위치나 타일 회전으로 이동했더라도 선택한 표지판과 정확히 같은 **월드 Y 좌표**를 사용한다.

#### 빈 후보를 찾았을 때

`Set CandidateLocation` 뒤에 7.3의 공통 간격 검사 블록을 둔다.

```text
Set CandidateLocation.Exec
→ Set bHasOverlap = false
→ For Each Loop With Break

For Each Loop With Break.Completed
→ NOT Boolean(bHasOverlap)
→ Branch
```

Normal 후보 Transform은 공통 `Select(Transform)`의 `True` 입력으로 연결한다.

```text
CandidateLocation
→ Make Transform.Location

TileRef
→ Get Tiles
→ Get Component Rotation
→ Make Transform.Rotation

Make Transform.Scale = (1, 1, 1)

Make Transform.Return Value
→ Select(Transform).True

BossEliteCandidateTransform
→ Select(Transform).False

EnemyType == Normal
→ Select(Transform).Index

Select(Transform).Return Value
→ Set CandidateTransform.Value
→ Break Transform
→ Set CandidateLocation
→ Set bHasOverlap = false
→ 공통 간격 검사 블록
```

공통 간격 검사 뒤의 `Branch.True`에서 반복을 끝낸다.

```text
Set bFoundTransform = true
→ For Loop With Break.Break
```

`Branch.False`는 연결하지 않는다. 현재 후보 실행이 끝나면서 다음 외부 반복 Index를 검사한다.

### 7.8 함수 종료 배선

Boss/Elite와 Normal 경로는 모두 외부 `For Loop With Break.Completed`에서 하나의 종료 분기로 합친다.

```text
For Loop With Break.Completed
→ Get bFoundTransform
→ Branch.Condition

Branch.True
→ Function Result 성공 노드
    SpawnTransform   = Get CandidateTransform
    transformSuccess = true

Branch.False
→ Function Result 실패 노드
    SpawnTransform   = Identity Transform
    transformSuccess = false
```

`Function Result` 노드의 `SpawnTransform`과 `transformSuccess` 핀에 값을 직접 연결한다. `Set SpawnTransform`, `Set transformSuccess` 노드는 만들지 않는다.

### 7.9 Transform 공통 설정

```text
Location = CandidateLocation
Rotation = Tiles.GetComponentRotation
Scale    = (1, 1, 1)
```

Enemy가 플레이어 반대 방향을 보아야 하는 경우에는 `Make Transform.Rotation`에만 별도 회전을 적용한다. 위치와 간격 계산은 변경하지 않는다.

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
7. Normal 짝수 슬롯이 LeftSign, 홀수 슬롯이 RightSign과 같은 월드 Y 좌표를 사용하는지 확인
8. Normal이 표지판 및 Boss/Elite 예약 영역 뒤부터 배치되는지 확인
9. 타일 재사용 시 이전 TileEnemies가 반환되고 새 타일 Enemy만 등록되는지 확인
10. 풀 고갈 또는 빈 슬롯에서 새 Actor Spawn 없이 요청이 생략되는지 확인
```
