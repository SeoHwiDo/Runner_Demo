# BP_TileManager BeginPlay 수정안

기준: Unreal Engine 5.7.4 Blueprint-only

## 목표

`BP_BaseTile`이 Construction Script에서 설정한 `RecycleCheck`와 `NextSpawnPoint`를 `BP_TileManager`의 단일 배치 기준으로 사용한다.

- `RecycleCheck.GetScaledBoxExtent`로 Manager Box 크기를 한 번만 초기화한다.
- Manager Box 안에는 `PoolSize - 1`개의 타일이 들어간다.
- 마지막 1개 타일은 Manager Box 진행 방향 바깥에서 재배치 여유분으로 사용한다.
- 첫 타일 이후에는 직전 타일의 `NextSpawnPoint` 월드 위치에 다음 타일을 배치한다.
- 폐기된 `GetTileWorldLength`와 Stage 계산을 제거한다.

## 선행 조건

- `BP_BaseTile`의 Construction Script는 위치 설정에 `Set Relative Location`을 사용한다.
- `RecycleCheck`의 X·Y Extent는 StaticMesh의 배치 Half Extent와 같다.
- `NextSpawnPoint`는 동일한 타일 Actor를 바로 이어 붙일 수 있는 다음 Actor 원점에 있다.
- `BP_TileManager` Actor와 `Box`의 Scale은 `(1, 1, 1)`이다.
- `PoolSize`는 2 이상이다.

## 제거 대상

- 오류 상태인 `GetTileWorldLength` 호출 노드
- `GetTileWorldLength × 0.5` 계산
- 초기 풀 생성 루프 안의 `CalculateStage` 호출
- 모든 타일에서 실행되는 기존 `InitBoxCollision` 실행 연결

## BeginPlay 실행 순서

기존 `GetActorOfClass → Set LVManager` 이후 풀 생성 루프를 유지한다.

```text
ForLoop
  First Index = 0
  Last Index = PoolSize - 1

Loop Body
→ Spawn Actor BP_BaseTile
  Location = SpawnPoint
  Scale = 현재 프로젝트 설정값
→ Is Valid
→ Branch(Index == 0)
```

첫 타일 경로:

```text
SpawnedTile.RecycleCheck
→ Get Scaled Box Extent
→ TileExtent
→ InitBoxCollision(TileExtent)

Box.Get Component Location
Box.Get Scaled Box Extent

FirstTileX = BoxLocation.X - BoxExtent.X + TileExtent.X

Set Actor Location(SpawnedTile)
  X = FirstTileX
  Y = 기존 SpawnPoint.Y
  Z = 기존 SpawnPoint.Z
→ TilePool.Add(SpawnedTile)
```

두 번째 타일부터의 경로:

```text
Branch False
→ TilePool.Add(SpawnedTile)
```

공통 후처리:

```text
SpawnedTile.NextSpawnPoint
→ Get World Location
→ Set SpawnPoint
```

루프 완료:

```text
ForLoop.Completed
→ TilePool Last Index
→ Get
→ Set LastTile
```

## InitBoxCollision

입력 `TileExtent`는 `BP_BaseTile.RecycleCheck.GetScaledBoxExtent` 결과를 그대로 사용한다.

```text
TileCount = PoolSize - 1

ManagerBoxExtent.X = TileExtent.X * TileCount
ManagerBoxExtent.Y = TileExtent.Y
ManagerBoxExtent.Z = TileExtent.Z

Box.Set Box Extent(ManagerBoxExtent)
```

Manager Box의 기존 위치는 변경하지 않는다.

## 배치 결과

타일 Half Length를 `E`, `PoolSize`를 `N`, Manager Box 중심을 `C`라고 하면 다음과 같다.

```text
ManagerBoxExtent.X = E * (N - 1)
FirstTileCenterX = C - E * (N - 1) + E
TileCenterX(Index) = FirstTileCenterX + (E * 2 * Index)
```

- `Index 0`부터 `N - 2`까지 Manager Box 내부에 들어간다.
- `Index N - 1` 타일의 시작 면은 Manager Box의 끝 면에 맞닿는다.
- 마지막 타일은 Box 바깥의 재배치 여유분이 된다.

## 주의 사항

- `RecycleCheck.GetScaledBoxExtent`를 사용하므로 `BP_BaseTile`의 Actor Scale이 반영된다.
- Manager Box Scale이 1이 아니면 `Set Box Extent` 결과가 다시 확대되므로 Scale을 1로 유지한다.
- `BP_BaseTile` Construction Script에서 월드 Bounds를 Extent에 다시 적용하면 Actor Scale이 중복될 수 있다. BaseTile은 로컬 Bounds로 `RecycleCheck`의 Unscaled Extent를 설정한다.
- 첫 타일 이후에는 타일 길이를 다시 계산하지 않는다.
- Tick에서 `InitBoxCollision` 또는 전체 풀 배치를 다시 실행하지 않는다.

## 사용자 검증 항목

- Blueprint 컴파일 오류가 없는지 확인한다.
- 첫 `PoolSize - 1`개 타일이 Manager Box 내부에 들어가는지 확인한다.
- 마지막 1개 타일이 Box 끝에 맞닿은 바깥쪽 여유분인지 확인한다.
- Construction Script 재실행 후 `NextSpawnPoint`와 타일 위치가 누적 이동하지 않는지 확인한다.
- 타일 재배치 시 앞 타일의 `NextSpawnPoint`에 정확히 이어지는지 확인한다.
