# Stage·Tile·Enemy Manager Blueprint 작성표

기준: Unreal Engine 5.7.4, Blueprint-only

이 문서는 에디터를 자동 조작하지 않고 직접 Blueprint에 옮길 수 있도록 함수 시그니처와 노드 연결 순서를 분리한 작성표다. `Component`, 함수 Input/Output, Event Dispatcher는 Blueprint 클립보드 노드만 붙여넣어 생성할 수 없으므로 각 절의 선행 항목을 먼저 만든다.

## 1. 공통 데이터

### `E_LevelPhase`

```text
Early
Mid
Boss
```

### `BP_LevelManager` 변수

```text
NextTileSequence      Integer       기본값 1
MaxStageCount         Integer       기본값 3
TilesPerStage         Integer       기본값 10
EarlyTileCount        Integer       기본값 5
MidTileCount          Integer       기본값 4
bGenerationCompleted  Boolean       기본값 false
bEndTileEntered       Boolean       기본값 false

EnemySpawnManagerRef  BP_EnemySpawnManager Object Reference
TileManagerRef        BP_TileManager Object Reference

EarlyNormalPattern    Integer Array 기본값 [1, 1, 2, 2, 2]
MidNormalPattern      Integer Array 기본값 [1, 1, 2, 2]
MidEliteCount         Integer       기본값 1
BossCount             Integer       기본값 1
```

## 2. `BP_LevelManager.CountStage`

### Output

```text
CanGenerateTile       Boolean
AssignedTileSequence  Integer
StageIndex            Integer
TileIndexInStage      Integer
Phase                 E_LevelPhase
IsFinalTile           Boolean
```

### 노드 연결

```text
CountStage Entry
→ Branch(bGenerationCompleted)

True
→ Return
   CanGenerateTile      = false
   IsFinalTile          = false

False
→ Set AssignedTileSequence = NextTileSequence
→ MaximumTileCount = MaxStageCount * TilesPerStage
→ Branch(AssignedTileSequence > MaximumTileCount)

True
→ Set bGenerationCompleted = true
→ Return
   CanGenerateTile = false

False
→ ZeroBasedSequence = AssignedTileSequence - 1
→ Set StageIndex = (ZeroBasedSequence / TilesPerStage) + 1
→ Set TileIndexInStage = (ZeroBasedSequence % TilesPerStage) + 1
→ Set IsFinalTile = (AssignedTileSequence == MaximumTileCount)
→ Set NextTileSequence = AssignedTileSequence + 1
→ Branch(IsFinalTile)

True
→ Set bGenerationCompleted = true
→ Phase 계산

False
→ Phase 계산
```

Phase 계산 공통 진입:

```text
Branch(TileIndexInStage <= EarlyTileCount)

True
→ Set Phase = Early
→ Return CanGenerateTile = true

False
→ MidEndIndex = EarlyTileCount + MidTileCount
→ Branch(TileIndexInStage <= MidEndIndex)

    True
    → Set Phase = Mid
    → Return CanGenerateTile = true

    False
    → Set Phase = Boss
    → Return CanGenerateTile = true
```

`StageIndex`는 이전 Stage 값에 더하지 않는다. 항상 `AssignedTileSequence`에서 다시 계산한다.

## 3. `BP_LevelManager.SetupPlacedTile`

### Input

```text
TileRef  BP_BaseTile Object Reference
```

### 노드 연결

```text
SetupPlacedTile Entry
→ Is Valid(TileRef)
→ TileRef.ResetForReuse
→ CountStage
→ Branch(CanGenerateTile)

False
→ TileRef.SetEndTileEnabled(false)
→ Return

True
→ TileRef.ApplyStageData(
     AssignedTileSequence,
     StageIndex,
     TileIndexInStage,
     Phase
   )
→ TileRef.SetEndTileEnabled(IsFinalTile)
→ SpawnEnemiesForTile(TileRef, StageIndex, TileIndexInStage, Phase)
```

초기 타일 생성과 재배치 경로는 직접 `CountStage`를 호출하지 않고 반드시 이 함수 하나를 호출한다.

## 4. `BP_LevelManager.SpawnEnemiesForTile`

### Input

```text
TileRef          BP_BaseTile Object Reference
StageIndex       Integer
TileIndexInStage Integer
Phase            E_LevelPhase
```

### Early

```text
Switch on E_LevelPhase(Phase)
Early
→ PatternIndex = TileIndexInStage - 1
→ EarlyNormalPattern Is Valid Index(PatternIndex)
→ NormalCount = EarlyNormalPattern[PatternIndex]
→ SpawnEnemyTypeForTile(TileRef, Normal, NormalCount, StageIndex)
```

### Mid

```text
Mid
→ PhaseTileIndex = TileIndexInStage - EarlyTileCount
→ PatternIndex = PhaseTileIndex - 1
→ MidNormalPattern Is Valid Index(PatternIndex)
→ NormalCount = MidNormalPattern[PatternIndex]
→ SpawnEnemyTypeForTile(TileRef, Normal, NormalCount, StageIndex)
→ SpawnEnemyTypeForTile(TileRef, Elite, MidEliteCount, StageIndex)
```

### Boss

```text
Boss
→ SpawnEnemyTypeForTile(TileRef, Boss, BossCount, StageIndex)
```

## 5. `BP_LevelManager.SpawnEnemyTypeForTile`

### Input

```text
TileRef      BP_BaseTile Object Reference
EnemyType    E_EnemyType
SpawnCount   Integer
StageIndex   Integer
```

### 노드 연결

```text
Branch(SpawnCount > 0)
True
→ For Loop
   First Index = 0
   Last Index  = SpawnCount - 1

Loop Body
→ SelectEnemyIDByType(EnemyType, StageIndex)
→ Branch(SelectionSuccess)

    False
    → 다음 Loop

    True
    → Branch(EnemyType == Normal)

        True
        → TileRef.GetNormalSpawnTransform(Index, SpawnCount)

        False
        → TileRef.GetEndCenterSpawnTransform

    → EnemySpawnManagerRef.AcquireEnemyByID(
         SelectedEnemyID,
         SpawnTransform,
         TileRef
      )
    → Branch(AcquireSuccess)
    → True: TileRef.TileEnemies Add Unique(AcquiredEnemy)
```

`SelectEnemyIDByType`는 `DA_EnemyCatalog.EnemyDefinitions` 중 다음 조건을 통과한 후보를 가중치 추첨한다.

```text
Definition.MonsterType == EnemyType
Definition.MinStage <= StageIndex
Definition.SpawnWeight > 0
EnemySpawnManagerRef.CanAcquireEnemy(Definition.MonsterID)
```

## 6. `BP_TileManager` 초기 타일 연결

초기 타일 Spawn과 Transform 설정이 모두 끝난 지점에 연결한다.

```text
Spawn Actor BP_BaseTile
→ Set Actor Location(초기 배치 위치)
→ ActiveTiles Add(SpawnedTile)
→ Set LastTile(SpawnedTile)
→ Branch(SpawnedTile.bCountsTowardStage)

True
→ LevelManagerRef.SetupPlacedTile(SpawnedTile)

False
→ SpawnedTile.ResetForReuse
```

플레이어 시작용 안전 타일은 `bCountsTowardStage=false`로 두어 Stage의 10개 전투 타일에서 제외한다.

## 7. `BP_TileManager` 타일 재배치 연결

현재 범위 이탈 타일을 바로 이동시키는 실행선 앞에 Enemy 반환을 넣는다.

```text
범위 이탈 TileRef 확인
→ EnemySpawnManagerRef.ReturnEnemiesOwnedByTile(TileRef)
→ Branch(ReturnSuccess)

False
→ 오류 출력 후 재배치 중단

True
→ TileRef.ResetForReuse
→ LastTile.NextSpawnPoint Get World Location
→ TileRef.SetActorLocation
→ Set LastTile = TileRef
→ Branch(TileRef.bCountsTowardStage)

    True
    → LevelManagerRef.SetupPlacedTile(TileRef)

    False
    → 종료
```

순서는 반드시 다음을 유지한다.

```text
기존 Enemy 반환
→ Tile 상태 초기화
→ Tile 위치 이동
→ 새 Sequence·Stage·Phase 할당
→ 새 Enemy 배치
```

## 8. `BP_BaseTile` 선행 항목

### Component

```text
EndTileCollision  Box Collision
```

기본 설정:

```text
Collision Enabled       No Collision
Generate Overlap Events false
Player                   Overlap
나머지                   Ignore
```

### 변수

```text
AssignedTileSequence  Integer
StageIndex            Integer
TileIndexInStage      Integer
Phase                 E_LevelPhase
bIsEndTile            Boolean = false
bEndTriggered         Boolean = false
bCountsTowardStage    Boolean = true
TileEnemies           BP_EnemyBase Object Reference Array
```

### Event Dispatcher

```text
OnEndTileEntered
Input: TileRef BP_BaseTile Object Reference
```

## 9. `BP_BaseTile.ApplyStageData`

### Input

```text
NewTileSequence  Integer
NewStageIndex    Integer
NewTileIndex     Integer
NewPhase         E_LevelPhase
```

### 노드 연결

```text
Set AssignedTileSequence = NewTileSequence
→ Set StageIndex = NewStageIndex
→ Set TileIndexInStage = NewTileIndex
→ Set Phase = NewPhase
```

## 10. `BP_BaseTile.SetEndTileEnabled`

### Input

```text
Enabled Boolean
```

### 노드 연결

```text
Set bIsEndTile = Enabled
→ Set bEndTriggered = false
→ Branch(Enabled)

True
→ EndTileCollision Set Collision Enabled(Query Only)
→ EndTileCollision Set Generate Overlap Events(true)

False
→ EndTileCollision Set Generate Overlap Events(false)
→ EndTileCollision Set Collision Enabled(No Collision)
```

## 11. `BP_BaseTile.ResetForReuse`

```text
SetEndTileEnabled(false)
→ TileEnemies Clear
→ Set AssignedTileSequence = 0
→ Set StageIndex = 0
→ Set TileIndexInStage = 0
→ Set Phase = Early
```

이 함수는 Enemy 반환 완료 후 호출한다. `TileEnemies.Clear`만 먼저 호출하면 `ReturnEnemiesOwnedByTile`이 반환 대상을 찾지 못한다.

## 12. `BP_BaseTile.EndTileCollision BeginOverlap`

```text
On EndTileCollision BeginOverlap
→ Cast To BP_RunnerPlayer(OtherActor)
→ Branch(bIsEndTile AND NOT bEndTriggered)

True
→ Set bEndTriggered = true
→ Set Generate Overlap Events(false)
→ Call OnEndTileEntered(Self)
```

`BP_LevelManager`는 타일 생성 시 Dispatcher를 한 번만 Bind하거나 `BP_TileManager`를 통해 전달받는다.

## 13. `BP_LevelManager.NotifyEndTileEntered`

```text
NotifyEndTileEntered Entry
→ Branch(bEndTileEntered)

True
→ Return

False
→ Set bEndTileEntered = true
→ 신규 Spawn 요청 차단 유지
→ 마지막 구간 진입 Event 호출
```

플레이어의 마지막 타일 진입은 Run Clear가 아니다. 마지막 Boss 사망 Event가 실제 Run Clear를 호출한다.

## 14. `BP_EnemySpawnManager.CanAcquireEnemy`

### Input / Output

```text
EnemyID Name
CanAcquire Boolean
```

### 노드 연결

```text
PoolIndexByEnemyID Find(EnemyID)
→ Branch(Found)

False
→ Return false

True
→ EnemyPools Is Valid Index(PoolIndex)
→ Get EnemyPools[PoolIndex]
→ Break ST_EnemyPool
→ AvailableEnemies Length > 0
→ Return 결과
```

## 15. `BP_EnemySpawnManager.AcquireEnemyByID`

### Input / Output

```text
EnemyID        Name
SpawnTransform Transform
OwningTile     BP_BaseTile Object Reference

Success        Boolean
AcquiredEnemy  BP_EnemyBase Object Reference
```

### 노드 연결

```text
PoolIndexByEnemyID Find(EnemyID)
→ Found와 EnemyPools Is Valid Index 검사
→ Pool = EnemyPools[PoolIndex]
→ Pool.AvailableEnemies Length > 0 검사
→ Enemy = Pool.AvailableEnemies[0]
→ Pool.AvailableEnemies Remove Index(0)
→ Set Members in ST_EnemyPool(AvailableEnemies)
→ EnemyPools Set Array Elem(PoolIndex, 수정한 Pool)
→ Enemy Is Valid 검사
→ Enemy.ActivateFromPool(SpawnTransform, Self, OwningTile)
→ ActiveEnemies Add(Enemy)
→ Return Success=true, AcquiredEnemy=Enemy
```

구조체는 Map/Array에서 복사본으로 나오므로 수정한 `ST_EnemyPool`을 반드시 `Set Array Elem`으로 원래 Index에 저장한다.

## 16. `BP_EnemySpawnManager.ReturnEnemy`

```text
Enemy Is Valid
→ Enemy.PoolState != InPool
→ ActiveEnemies Contains(Enemy)
→ PoolIndexByEnemyID Find(Enemy.MonsterID)
→ EnemyPools Is Valid Index
→ Enemy.DeactivateFromPool
→ Pool = EnemyPools[PoolIndex]
→ Pool.AvailableEnemies Add Unique(Enemy)
→ EnemyPools Set Array Elem(PoolIndex, 수정한 Pool)
→ ActiveEnemies Remove(Enemy)
→ Return Success=true
```

## 17. `BP_EnemySpawnManager.ReturnEnemiesOwnedByTile`

### Input / Output

```text
TileRef BP_BaseTile Object Reference
Success Boolean
```

### 노드 연결

```text
TileRef Is Valid
→ TileRef.TileEnemies를 LocalEnemies 지역 배열에 복사
→ For Each Loop(LocalEnemies)

Loop Body
→ Enemy Is Valid
→ Enemy.OwningTile == TileRef
→ ReturnEnemy(Enemy)

Completed
→ TileRef.TileEnemies Clear
→ Return Success=true
```

순회 중 원본 `TileEnemies`를 변경하지 않도록 지역 배열 복사본을 사용한다.

## 18. 필수 경계 검증

```text
Sequence 1  → Stage 1 / Tile 1  / Early
Sequence 5  → Stage 1 / Tile 5  / Early
Sequence 6  → Stage 1 / Tile 6  / Mid
Sequence 9  → Stage 1 / Tile 9  / Mid
Sequence 10 → Stage 1 / Tile 10 / Boss
Sequence 11 → Stage 2 / Tile 1  / Early
Sequence 20 → Stage 2 / Tile 10 / Boss
Sequence 30 → Stage 3 / Tile 10 / Boss / IsFinalTile=true
Sequence 31 → CanGenerateTile=false
```

PIE에서는 다음 순서를 확인한다.

```text
초기 전투 타일마다 Sequence가 한 번만 증가
→ 반환 전 기존 Enemy가 모두 InPool로 복귀
→ 이동 후 같은 Tile Actor에 새 Sequence 할당
→ 30번째 타일만 EndTileCollision 활성화
→ 31번째 이후 바닥 타일에는 Enemy 미생성
→ EndTileCollision 중복 Overlap 1회 차단
→ 마지막 Boss 사망 전에는 Run Clear 미호출
```
