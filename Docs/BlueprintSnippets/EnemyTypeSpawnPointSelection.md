# EnemyType별 배치점 선택과 타일 경계 보정

기준: Unreal Engine 5.7.4, Blueprint-only

상태: 설계 안내 완료. 실제 Blueprint 수정·컴파일·PIE는 수행하지 않았다.

## 1. 확정 요구사항과 확인 범위

- `BP_LevelManager.SeclectSpawnTransform`에서 전달받은 `enemyType`에 맞는 배치점을 선택한다.
- 현재 타일의 `TileEnemys` 중 같은 `EnemyType`이 사용 중인 배치점을 제외한다.
- Normal은 01/02 두 행의 L/R 포인트를 사용한다. Boss는 Boss 전용 포인트 하나, Elite는 별개의 Elite 전용 포인트 하나만 사용한다. Boss와 Elite는 배치점을 공유하거나 서로의 포인트를 대체 사용하지 않는다.
- Normal은 행(01/02)을 무작위 순서로 검사한다. 선택한 행의 L/R 중 한쪽이라도 점유되어 있으면 행 전체를 건너뛰며, 양쪽 모두 빈 행에서만 L/R 중 하나를 균등 무작위 선택한다. 모든 포인트를 하나의 배열로 합쳐 바로 추첨하지 않는다.
- Elite/Boss는 실제 배치할 Enemy의 Bounds를 이용해 타일의 수평 경계 안으로 보정한다.
- 다른 타입 간 충돌 검사, 다종 콘텐츠, Stage 편성 확장은 이번 안내에 포함하지 않는다.
- 첨부 그래프에는 Entry, `TileEnemys`, `ForEachLoopWithBreak`, `EnemyType` Getter만 있으며 실행선은 미연결이다.
- 저장된 `BP_BaseTile.uasset` 문자열에는 `normal01_L/R`, `normal02_L/R`, `BossElitePos01/02`가 확인된다. 이는 그래프/컴포넌트 계층 검증이 아니며, 현재 에디터의 미저장 상태와 다를 수 있다. `BossElitePos01/02` 중 어느 것을 Boss 또는 Elite로 사용할지는 임의 지정하지 않는다.

## 2. 필요한 데이터

다음 항목은 기존에 존재한다고 가정하지 않는 신규 제안이다.

- `BP_EnemyBase.AssignedSpawnPoint`: Scene Component Object Reference. 실제로 사용한 배치점.
- 함수 입력 `EnemyRef`: 풀에서 대여한 BP_EnemyBase Object Reference. Type만으로는 실제 Bounds를 알 수 없다.
- 함수 출력 `SelectedSpawnPoint`: Scene Component Object Reference.
- 기존 출력 `SpawnTransform`, `transformSuccess` 유지.
- 로컬 `CandidatePoints`: Scene Component Object Reference 배열.

`spawnidx`, `spawnCount`는 이 배치점 선택 방식에는 필요하지 않다. 호출부가 사용 중일 수 있으므로 변경 시 함께 정리한다. Normal의 행과 L/R 선택은 아래의 2단계 무작위 방식으로 처리하며, Boss/Elite 선택 규칙은 유지한다.

Bounds 보정이나 Enemy 이동 후에는 Actor 좌표가 원래 배치점과 달라질 수 있다. 따라서 점유 여부는 좌표의 정확한 일치가 아니라 `AssignedSpawnPoint` 참조로 비교한다.

## 3. Blueprint 실행 순서

```text
Entry
→ Tile / EnemyRef 유효성 및 EnemyRef.EnemyType == enemyType 확인
→ Switch on E_EnemyType
    Normal: Make Array(normal01_L, normal01_R, normal02_L, normal02_R)
    Elite : Make Array(타일의 유일한 Elite 전용 배치점 참조)
    Boss  : Make Array(타일의 유일한 Boss 전용 배치점 참조)
→ Set CandidatePoints
→ For Each Loop(Tile.TileEnemys)
    → Is Valid(Array Element)
    → Array Element != EnemyRef
    → Array Element.EnemyType == 입력 enemyType
    → True: CandidatePoints.Remove Item(Array Element.AssignedSpawnPoint)
→ Completed
→ Normal이면: 아래 3.1절의 행 → L/R 추첨으로 후보를 한 개로 좁힘
→ CandidatePoints 순회
    → 유효한 배치점인지 확인
    → Get World Location
    → Normal: 기준점 위치 사용
    → Elite/Boss: 4절 Bounds 보정, 들어갈 수 없으면 배치 실패
    → SpawnTransform / SelectedSpawnPoint / transformSuccess=true 반환
→ 모든 후보 실패: transformSuccess=false, SelectedSpawnPoint=None 반환
```

- 후보에서 같은 타입의 **모든** 점유 위치를 제거해야 하므로 기존 Enemy 순회에는 일반 `For Each Loop`를 사용한다.
- `TileEnemys`에는 현재 타일이 소유하는 대여·배치 완료 Enemy만 유지한다. 반환된 Actor는 Is Valid여도 활성 Enemy가 아니므로 반드시 배열에서 제거한다.
- 동일 타입 Enemy의 AssignedSpawnPoint가 비어 있으면 점유를 판정할 수 없다. 기존 배치 경로도 이 변수를 기록하도록 고치고, 이 불일치가 발생하면 이번 요청을 실패 처리한다.
- 후보가 없을 때 배열 Get이나 원점 배치를 하지 않는다.
- Normal은 타일당 최대 2개(행당 1개), Boss와 Elite는 각각 최대 1개다. 이는 배치 용량이며 해당 수량을 항상 생성하라는 의미는 아니다.
- Boss/Elite는 전용 포인트가 하나이므로 후보 추첨과 대체 포인트 탐색이 없다. 동일 타입의 기존 Enemy가 전용 위치를 점유하면 실패한다.
- 동일 타입의 점유만 검사한다. 서로 다른 타입의 실제 Bounds 간 겹침은 이 함수의 검사 범위에 포함하지 않는다.

### 3.1 Normal의 행 → L/R 무작위 선택

추가 로컬 변수:

- `RowOrder`: Integer 배열.
- `RowPoints`: Scene Component Object Reference 배열.

동일 타입의 점유 배치점을 모두 제거한 `CandidatePoints`를 기준으로 다음 순서로 처리한다.

```text
RowOrder = Make Array(1, 2)
→ Shuffle(RowOrder)
→ For Each Loop With Break(RowOrder)
    → Switch on Int(Array Element)
        1: RowPoints = Make Array(normal01_L, normal01_R)
        2: RowPoints = Make Array(normal02_L, normal02_R)
    → 양쪽 포인트의 Is Valid 확인
    → CandidatePoints.Contains(RowPoints[0])
        AND CandidatePoints.Contains(RowPoints[1])
    → False: 이 행 전체를 건너뛰고 다음 Loop Body에서 검사
    → True:
        Shuffle(RowPoints)
        SelectedSpawnPoint = RowPoints[0]
        선택된 포인트의 Transform과 성공 반환
→ 모든 행을 검사해도 성공하지 못하면 배치 실패 반환
```

예: 01_L은 점유되고 01_R은 비어 있다면 01행 전체를 건너뛴다. 02_L/R이 모두 비어 있으면 둘 중 하나를 1/2 확률로 선택한다. 양쪽 모두 빈 행이 없으면 빈 포인트가 남아 있어도 배치 실패다. 점유를 유지하는 동안 Normal은 행당 최대 1개가 배치된다. 다음 Enemy를 추첨하기 전에 이전 Enemy의 배치 성공과 점유 등록을 완료한다. 행 순서를 한 번만 섞어 유한 순회하므로 점유 행을 반복 추첨하는 무한 반복이 없다.

### 3.2 Boss와 Elite의 단일 전용 포인트

```text
Boss 요청
→ 현재 타일에 배치된 Boss 존재: 실패
→ 없음: Boss 전용 포인트 선택 → Bounds 보정 → 결과 반환

Elite 요청
→ 현재 타일에 배치된 Elite 존재: 실패
→ 없음: Elite 전용 포인트 선택 → Bounds 보정 → 결과 반환
```

`TileEnemys`에는 현재 타일의 배치 완료 Enemy만 있어야 한다. 유일한 포인트를 사용하는 두 타입은 동일 타입이 존재하는지만 검사해도 점유 판정이 된다. 이 분기를 직접 구성할 때는 포인트 배열이나 랜덤 노드가 필요 없다. 전용 포인트가 유효하지 않거나 Bounds가 타일에 들어가지 않으면 실패하고 대여한 Enemy를 반환한다. 다른 타입이나 Normal의 포인트로 우회하지 않는다.

전용 포인트는 배치 기준점이다. 앞서 확정한 Bounds 보정은 유지하므로 실제 최종 위치는 타일 안으로 보정될 수 있다.

## 4. Elite/Boss Bounds 보정

아래 간단한 계산의 적용 조건은 **수평 직사각형 타일의 경계가 월드 X/Y축에 정렬되어 있고, Bounds 측정 이후 Enemy의 최종 회전·크기를 변경하지 않는 것**이다. 실제 타일 회전은 이번 작업에서 확인하지 않았으므로 에디터 적용 시 확인해야 한다. 회전 타일은 월드 AABB만으로 실제 타일 내부를 보장할 수 없으며 타일 로컬 공간에서 계산해야 한다.

1. 실제 배치 영역을 나타내는 타일 Mesh 또는 Box에 `Get Component Bounds`를 호출한다. 모든 자식·트리거를 포함할 수 있는 `Tile.Get Actor Bounds`는 사용하지 않는다. `RecycleCheck`도 실제 배치 영역과 일치하는 경우에만 사용한다.
2. Enemy의 최종 Mesh·Scale·Rotation이 준비된 상태에서 Bounds를 읽는다. 함수 내부에서는 Enemy를 이동하거나 활성화하지 않는다.
3. 풀 보관 상태에서는 Collision이 꺼져 있으므로 Actor의 `Get Actor Bounds`를 사용할 때 `Only Colliding Components=false`로 설정한다. 감지용 Sphere나 VFX까지 포함되어 과대 측정된다면 배치용 Mesh/전용 Bounds 컴포넌트의 `Get Component Bounds`를 사용한다.
4. Bounds 중심과 Actor 피벗 간 차이를 보정한다. BoxExtent는 전체 크기가 아니라 반크기다.

```text
TileMin = TileOrigin - TileExtent
TileMax = TileOrigin + TileExtent
Offset  = EnemyBoundsOrigin - EnemyRef.GetActorLocation

MinX = TileMin.X + EnemyExtent.X - Offset.X
MaxX = TileMax.X - EnemyExtent.X - Offset.X
MinY = TileMin.Y + EnemyExtent.Y - Offset.Y
MaxY = TileMax.Y - EnemyExtent.Y - Offset.Y

MinX > MaxX 또는 MinY > MaxY → 배치 실패

Result.X = Clamp(SpawnPointWorldLocation.X, MinX, MaxX)
Result.Y = Clamp(SpawnPointWorldLocation.Y, MinY, MaxY)
Result.Z = SpawnPointWorldLocation.Z
```

Z는 타일 두께 안으로 Clamp하지 않는다. 배치점의 높이는 해당 Enemy 피벗이 바닥에 맞도록 설정되어 있어야 한다. 발 위치 보정은 별도 요구사항이다.

`Make Transform`은 위 Result 위치와 **Bounds 측정에 사용한 Enemy 회전·크기**로 구성한다. 배치점의 Scale을 그대로 복사하면 Enemy 크기가 달라져 Bounds 계산과 어긋날 수 있다. `ActivateFromPool`이 Mesh·Scale·Rotation을 다시 바꾼다면 측정과 실제 적용 순서를 먼저 수정해야 한다.

## 5. 호출부와 풀 반환

```text
AcquireEnemy
→ 최종 Enemy 외형·회전·크기 준비
→ SeclectSpawnTransform(Tile, EnemyType, EnemyRef)
→ 성공: ActivateFromPool(SpawnTransform, Tile)
        배치 완료 시 AssignedSpawnPoint = SelectedSpawnPoint
        Tile.TileEnemys Add Unique(EnemyRef)
        이후 다음 Enemy 배치 요청
→ 실패: ReturnEnemy(EnemyRef), 이번 요청 생략
```

활성화가 실패하면 점유를 기록하지 않고 대여한 Enemy를 반환한다. 다음 배치 요청은 이전 Enemy의 점유 등록까지 끝난 후 실행한다.

반환 시 기존 TileEnemys에서 제거하고 `AssignedSpawnPoint=None`으로 초기화한다. 타일 전체 반환은 배열 복사본을 순회해 원본 변경 중 순회를 피한다. 풀 고갈 시 추가 Spawn이나 활성 Enemy 강제 회수는 하지 않는다.

## 6. 검증 상태

- 설치 엔진 `Engine/Build/Build.version`에서 5.7.4 확인.
- 로컬 5.7.4 `Actor.h`와 `KismetSystemLibrary.h`에서 Bounds API 확인.
- Bounds 수식 수치 검산 5건 통과: 중앙 유지, 왼쪽 보정, 오른쪽 피벗 보정, 타일과 같은 크기, 타일 초과 크기 거부. 이는 에디터 검증을 대체하지 않는다.
- 에디터 검증 대기: 빈 타일, 동일 타입 일부 점유, 다른 타입만 존재, 전체 점유, 반환 후 재배치, 대형 Enemy, 피벗 오프셋, 풀 Collision 비활성 상태, 타일 이동 후 재배치.
- Normal 2단계 추첨의 문서 흐름 검토: 한쪽 점유 시 행 전체 제외, 양쪽 모두 빈 행에서만 L/R 추첨, 모든 행이 제외되면 실패하는 경로를 확인했다. 실제 랜덤 동작과 점유 갱신의 PIE 검증은 대기 상태다.
- Boss/Elite 전용 포인트 각각 1개, 중복 점유 시 실패, 다른 포인트 대체 사용 없음으로 설계를 확정했다. 문서 분기 검토만 수행했으며 에셋 적용과 PIE는 대기 상태다.
- 실제 uasset 변경, 컴파일, PIE를 수행하지 않았으므로 SUBMIT-01 완료 조건을 충족한 것으로 표시하지 않는다.

이 문서는 이번 사용자 요구에 관한 안내이며, 과거 `EnemyPlaced.md`와 Clipboard 07의 거리·행 계산 블록을 그대로 붙여 넣어 적용하지 않는다.
