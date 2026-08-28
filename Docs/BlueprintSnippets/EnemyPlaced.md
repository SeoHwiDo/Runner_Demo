# `BP_LevelManager.EnemyPlaced` Blueprint 작성표

기준: Unreal Engine 5.7.4, Blueprint-only

목표: `BP_TileManager`가 재사용 타일을 앞쪽으로 이동하고 Stage 계산을 마친 뒤 `BP_LevelManager.EnemyPlaced(TileRef)`를 호출한다. `BP_LevelManager`는 현재 Stage 구간에 맞는 Enemy를 FIFO 풀에서 대여하고, `BP_BaseTile`의 배치 컴포넌트 Transform으로 초기화·활성화한다.

## 1. 확인된 기존 API와 컴포넌트

```text
BP_BaseTile
  BossElitePos01 : Scene Component
  BossElitePos02 : Scene Component
  NormalEnemyBox : Box Collision

BP_LevelManager
  CalculateStage()
  StageIdx          : Integer
  TileIdxInStage    : Integer
  EnemyManager      : BP_EnemySpawnManager Object Reference

BP_EnemySpawnManager
  AcquireEnemy(mosterType : E_EnemyType)
    return  : BP_EnemyBase Object Reference
    success : Boolean
  ReturnEnemy(Enemy)

BP_EnemyBase
  ActivateFromPool(
    SpawnTransform : Transform,
    NewOwningTile  : BP_BaseTile Object Reference
  )
    Success : Boolean
```

`AcquireEnemy`는 풀 안의 `AvailableEnemies[0]`을 꺼내고 수정한 `ST_EnemyPool`을 `EnemyPools`에 다시 저장하는 기존 FIFO 구현을 그대로 사용한다.

## 2. 선행 변수

### `BP_BaseTile`

다음 변수를 추가한다.

```text
TileEnemies : BP_EnemyBase Object Reference Array
```

목적:

- 타일 재사용 전에 해당 타일이 소유한 Enemy만 반환한다.
- `Get All Actors Of Class`로 전체 Enemy를 검색하지 않는다.
- 활성화에 성공한 Enemy만 `Add Unique`로 등록한다.

### `BP_LevelManager`

다음 변수 기본값을 사용한다.

```text
EarlyNormalPattern : Integer Array = [1, 1, 2, 2, 2]
MidNormalPattern   : Integer Array = [1, 1, 2, 2]
MidEliteCount      : Integer       = 1
BossCount          : Integer       = 1
```

## 3. `BP_LevelManager.BeginPlay` — `EnemyManager` 1회 캐시

현재 `EnemyManager`가 지정되지 않았으므로 `BeginPlay`에서 한 번만 찾는다. Tick이나 `EnemyPlaced` 호출마다 검색하지 않는다.

```text
Event BeginPlay
→ Get Actor Of Class
    Actor Class = BP_EnemySpawnManager
→ Cast To BP_EnemySpawnManager

Cast Success
→ Set EnemyManager

Cast Failed
→ Print String
    In String     = "[LevelManager] BP_EnemySpawnManager를 찾지 못했습니다."
    Print To Log  = true
    Print To Screen = Development에서만 true
```

레벨에 `BP_EnemySpawnManager` 인스턴스는 정확히 하나만 배치한다. 두 개 이상 배치할 가능성이 생기면 `Get Actor Of Class` 대신 GameMode 또는 명시적 참조 주입으로 교체한다.

## 4. 호출 위치 — `BP_TileManager.Box EndOverlap`

`BP_TileManager`에 이미 있는 `LVManager`를 사용한다. `BP_BaseTile`이 `BP_LevelManager`를 직접 참조하게 만들지 않는다.

타일 재사용 순서는 다음으로 고정한다.

```text
Box OnComponentEndOverlap
→ Cast To BP_BaseTile(OtherActor)
→ 기존 TileEnemies 반환
→ TileEnemies Clear
→ 재사용 타일 Set Actor Location
→ Set LastTile
→ LVManager.CalculateStage
→ Branch(IsEnd)

False
→ LVManager.EnemyPlaced(As BP_BaseTile)

True
→ 마지막 타일 처리
→ 필요한 경우 EnemyPlaced를 먼저 호출한 뒤 End 상태를 적용
```

`EnemyPlaced`는 반드시 타일의 새 위치와 `CalculateStage` 결과가 확정된 다음 호출한다. 이동 전에 호출하면 Enemy가 타일의 이전 World Transform을 사용한다.

## 5. `BP_LevelManager.EnemyPlaced`

### 함수 시그니처

```text
Input
  TileRef : BP_BaseTile Object Reference

Output
  Success : Boolean
```

### 지역 변수

```text
bAnyPlaced : Boolean = false
PatternIndex : Integer
NormalCount : Integer
```

### 검증과 Stage 구간 분기

```text
EnemyPlaced Entry
→ Set bAnyPlaced = false
→ Is Valid(TileRef)
→ Branch

False
→ Return Success = false

True
→ Is Valid(EnemyManager)
→ Branch

False
→ Return Success = false

True
→ Branch(TileIdxInStage <= 5)
```

#### Early: Tile 1~5

```text
True
→ Set PatternIndex = TileIdxInStage - 1
→ EarlyNormalPattern Is Valid Index(PatternIndex)
→ Branch

True
→ Set NormalCount = EarlyNormalPattern[PatternIndex]
→ PlaceEnemyTypeOnTile(
     TileRef,
     Normal,
     NormalCount
   )
→ Return Success = 반환된 bAnyPlaced

False
→ Return Success = false
```

#### Mid: Tile 6~9

Early Branch의 False 실행선:

```text
→ Branch(TileIdxInStage <= 9)

True
→ Set PatternIndex = TileIdxInStage - 6
→ MidNormalPattern Is Valid Index(PatternIndex)
→ Branch

True
→ Set NormalCount = MidNormalPattern[PatternIndex]
→ PlaceEnemyTypeOnTile(TileRef, Normal, NormalCount)
→ PlaceEnemyTypeOnTile(TileRef, Elite, MidEliteCount)
→ Return Success = Normal 또는 Elite가 한 마리 이상 활성화됐는지

False
→ Return Success = false
```

#### Boss: Tile 10

Mid Branch의 False 실행선:

```text
→ PlaceEnemyTypeOnTile(TileRef, Boss, BossCount)
→ Return Success = 반환된 bAnyPlaced
```

함수 성공은 요청 수 전부가 아니라 한 마리 이상 배치됐는지를 뜻한다. 풀 고갈 시 활성 Enemy를 강제 회수하거나 새 Actor를 생성하지 않는다.

## 6. `BP_LevelManager.PlaceEnemyTypeOnTile`

### 함수 시그니처

```text
Input
  TileRef    : BP_BaseTile Object Reference
  EnemyType : E_EnemyType
  SpawnCount : Integer

Output
  AnyPlaced : Boolean
```

### 지역 변수

```text
SpawnIndex      : Integer
AcquiredEnemy   : BP_EnemyBase Object Reference
SpawnTransform  : Transform
ActivationOK    : Boolean
```

### 공통 실행 흐름

```text
PlaceEnemyTypeOnTile Entry
→ Set AnyPlaced = false
→ Branch(SpawnCount > 0)

False
→ Return AnyPlaced = false

True
→ For Loop
    First Index = 0
    Last Index  = SpawnCount - 1

Loop Body
→ Set SpawnIndex = Index
→ EnemyManager.AcquireEnemy(EnemyType)
→ Branch(success AND Is Valid(return))

False
→ 다음 Loop

True
→ Set AcquiredEnemy = return
→ SelectSpawnTransform(TileRef, EnemyType, SpawnIndex, SpawnCount)
→ Branch(TransformSuccess)

False
→ EnemyManager.ReturnEnemy(AcquiredEnemy)
→ 다음 Loop

True
→ AcquiredEnemy.ActivateFromPool(SpawnTransform, TileRef)
→ Set ActivationOK = Success
→ Branch(ActivationOK)

False
→ EnemyManager.ReturnEnemy(AcquiredEnemy)
→ 다음 Loop

True
→ TileRef.TileEnemies Add Unique(AcquiredEnemy)
→ Set AnyPlaced = true
→ 다음 Loop

Completed
→ Return AnyPlaced
```

대여 후 Transform 계산이나 활성화가 실패하면 반드시 `ReturnEnemy`를 호출한다. 그렇지 않으면 Actor가 `ActiveEnemies`에는 남고 화면에는 나오지 않는 풀 누수가 발생한다.

## 7. `BP_LevelManager.SelectSpawnTransform`

### 함수 시그니처

```text
Input
  TileRef    : BP_BaseTile Object Reference
  EnemyType  : E_EnemyType
  SpawnIndex : Integer
  SpawnCount : Integer

Output
  SpawnTransform  : Transform
  TransformSuccess : Boolean
```

## 7.1 Normal — `NormalEnemyBox` 내부 정형 무작위

`NormalEnemyBox`의 Local X 전체 범위를 `SpawnCount`개 구간으로 나눈다. 각 Enemy는 자기 구간 안에서만 X를 추첨하고 Y는 Box 전체 폭에서 추첨한다. Z는 Box Component의 Local 원점 평면을 사용한다.

```text
EnemyType == Normal
→ TileRef.NormalEnemyBox Is Valid
→ NormalEnemyBox.Get Unscaled Box Extent
→ Break Vector(BoxExtent)

SafeSpawnCount = Max(SpawnCount, 1)
FullLocalX     = BoxExtent.X * 2.0
SegmentWidth   = FullLocalX / SafeSpawnCount
SegmentMinX    = -BoxExtent.X + (SegmentWidth * SpawnIndex)
SegmentMaxX    = SegmentMinX + SegmentWidth

LocalX = Random Float In Range(SegmentMinX, SegmentMaxX)
LocalY = Random Float In Range(-BoxExtent.Y, BoxExtent.Y)
LocalOffset = Make Vector(LocalX, LocalY, 0.0)

BoxWorldTransform = NormalEnemyBox.Get Component Transform
WorldLocation     = Transform Location(BoxWorldTransform, LocalOffset)
WorldRotation     = NormalEnemyBox.Get World Rotation

SpawnTransform = Make Transform
  Location = WorldLocation
  Rotation = WorldRotation
  Scale    = (1, 1, 1)

Return TransformSuccess = true
```

`Get Scaled Box Extent`와 Component Transform Scale을 동시에 사용하면 Scale이 두 번 적용될 수 있다. 위 그래프는 `Get Unscaled Box Extent + Transform Location` 조합으로 통일한다.

## 7.2 Elite/Boss — `BossElitePos01`, `BossElitePos02`

```text
EnemyType != Normal
→ SpawnIndex % 2
→ Select Scene Component
    0 = TileRef.BossElitePos01
    1 = TileRef.BossElitePos02
→ Is Valid(SelectedComponent)

False
→ Return TransformSuccess = false

True
→ SelectedComponent.Get Component Transform
→ Set SpawnTransform
→ Return TransformSuccess = true
```

현재 `MidEliteCount=1`, `BossCount=1`이므로 기본적으로 `BossElitePos01`이 사용된다. 이후 수량을 2로 올리면 두 번째 Actor는 `BossElitePos02`를 사용한다.

## 8. 타일 재사용 전 Enemy 반환

`BP_TileManager`에서 재사용 타일을 이동하기 전에 실행한다.

```text
As BP_BaseTile.TileEnemies
→ Set LocalEnemies = TileEnemies 복사본
→ For Each Loop(LocalEnemies)

Loop Body
→ Is Valid(Array Element)
→ Branch(Array Element.OwningTile == As BP_BaseTile)

True
→ EnemyManager.ReturnEnemy(Array Element)

Completed
→ As BP_BaseTile.TileEnemies Clear
→ 타일 이동 실행
```

순회 중 원본 `TileEnemies`를 바로 수정하지 않는다. 복사본을 순회한 뒤 원본을 Clear한다.

## 9. Stage별 MonsterID 제한에 대한 현재 범위

현재 `AcquireEnemy`는 `E_EnemyType`별 후보 풀을 무작위 선택하며 `ST_EnemyDefinition.MinStage`와 `SpawnWeight`를 직접 검사하지 않는다. 따라서 이 작성표가 보장하는 Stage 규칙은 다음이다.

```text
Early → Normal
Mid   → Normal + Elite
Boss  → Boss
```

같은 Type 안에서도 Stage별 `MonsterID`, `MinStage`, `SpawnWeight`를 적용하려면 후속으로 다음 API를 추가한다.

```text
SelectEnemyIDByType(EnemyType, StageIdx)
AcquireEnemyByID(EnemyID)
```

이 확장은 현재 `EnemyPlaced`의 배치 Transform과 타일 소유권 흐름을 변경하지 않는다.

## 10. 검증 체크리스트

```text
1. BeginPlay에서 EnemyManager가 한 번만 설정된다.
2. Tile 1~5에 Normal [1,1,2,2,2]가 요청된다.
3. Tile 6~9에 Normal [1,1,2,2]와 Elite 1이 요청된다.
4. Tile 10에 Boss 1이 요청된다.
5. Normal Enemy가 NormalEnemyBox 범위 안에 배치된다.
6. Normal 2마리가 서로 다른 Local X 구간을 사용한다.
7. Elite/Boss가 BossElitePos01에서 활성화된다.
8. SpawnCount=2일 때 두 번째 Elite/Boss가 BossElitePos02를 사용한다.
9. ActivateFromPool 성공 후 OwningTile이 전달한 TileRef와 일치한다.
10. 활성화 성공 Actor만 TileEnemies에 한 번 등록된다.
11. 대여 후 Transform/활성화 실패 Actor는 FIFO에 반환된다.
12. 타일 이동 전 기존 TileEnemies가 모두 반환된다.
13. 타일 이동과 CalculateStage 후 EnemyPlaced가 호출된다.
14. 풀 고갈 시 새 Actor 생성이나 활성 Actor 강제 회수가 없다.
15. Blueprint 컴파일 오류와 새 경고가 없다.
```
