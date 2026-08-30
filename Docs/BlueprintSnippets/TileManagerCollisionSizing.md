# BP_TileManager 타일 범위 Collision 자동 크기 계산

기준: Unreal Engine 5.7.4 Blueprint-only

## 목표

`BP_BaseTile`의 배치 크기를 기준으로 `BP_TileManager`의 타일 범위 Box Collision을 자동 설정한다. Collision에는 `PoolSize - 1`개의 타일이 빈틈없이 들어가며 마지막 1개 타일은 재배치 여유분으로 남긴다. 타일 위치는 설정된 Collision의 중심과 Extent를 기준으로 계산한다.

## 전제

- 타일은 `BP_TileManager`의 로컬 `+X` 방향으로 배치한다.
- 입력값 `TileBoxExtent`는 생성된 `BP_BaseTile.RecycleCheck`의 `Get Scaled Box Extent` 결과다.
- `PoolSize`는 2 이상이어야 한다.
- Box Collision의 Relative Scale은 `(1, 1, 1)`을 사용한다.
- 타일 범위 Box Collision의 기존 위치를 유지하고 해당 Bounds 안에서 타일 배치 위치를 계산한다.
- 변경 대상인 `BP_TileManager.Box`의 현재 Extent를 입력값으로 다시 사용하지 않는다.

## 계산식

```text
ContainedTileCount = PoolSize - 1

CollisionExtent.X = TileBoxExtent.X * ContainedTileCount
CollisionExtent.Y = TileBoxExtent.Y
CollisionExtent.Z = TileBoxExtent.Z
```

예: `PoolSize = 10`, `TileBoxExtent.X = 1,000`이면 전체 길이 2,000인 타일 9개를 포함한다.

```text
ContainedTileCount = 9
CollisionFullSize.X = 18,000
CollisionExtent.X = 9,000
```

타일은 Collision의 최소 X 경계에서 타일 Half Size만큼 안쪽을 첫 중심으로 사용한다.

```text
TileFullSize.X = TileBoxExtent.X * 2.0
FirstTileCenterX = CollisionCenter.X - CollisionExtent.X + TileBoxExtent.X
TileCenterX = FirstTileCenterX + TileFullSize.X * TileIndex
```

`TileIndex = 0`부터 `ContainedTileCount - 1`까지는 Collision 안에 배치한다. 남은 마지막 1개 타일은 진행 방향 바깥의 재배치 여유분으로 둔다.

## BP_TileManager 함수

함수명: `UpdateTileRangeCollision`

입력:

| 이름 | 형식 | 의미 |
|---|---|---|
| `TileBoxExtent` | Vector | 생성된 `BP_BaseTile.RecycleCheck`의 월드 Scale 반영 Half Extent |

실행 순서:

```text
PoolSize
→ Integer - Integer(1)
→ ContainedTileCount
→ Branch(ContainedTileCount > 0 AND TileBoxExtent.X > 0)
```

유효한 경우:

```text
TileBoxExtent.X * ContainedTileCount
→ CollisionExtentX

Make Vector
  X = CollisionExtentX
  Y = TileBoxExtent.Y
  Z = TileBoxExtent.Z
→ 타일 범위 Box Collision.Set Box Extent
  Update Overlaps = false
```

`TileBoxExtent.Y` 또는 `TileBoxExtent.Z`가 실제 타일 범위와 다르면 기존 Manager Box Extent의 Y·Z를 유지하고 X만 갱신한다.

유효하지 않은 경우에는 Collision 값을 변경하지 않고 반환한다. 제출 전 임시 출력은 제거한다.

## 호출 위치

초기 타일 풀 생성 중 첫 번째 `BP_BaseTile` 생성에 성공했을 때 한 번만 호출한다.

```text
Spawn Actor BP_BaseTile
→ Is Valid
→ Branch(Loop Index == 0)
  True
  → As BP_BaseTile.RecycleCheck.Get Scaled Box Extent
  → UpdateTileRangeCollision
```

`PoolSize` 또는 `TileBoxExtent`를 런타임에 변경하지 않는 현재 구조에서는 매 타일 생성이나 Tick마다 다시 계산하지 않는다.

`Set Box Extent`는 절대값을 설정하므로 같은 `TileBoxExtent`로 반복 실행해도 결과가 커지지 않는다. 반복 실행 때 크기가 증가한다면 변경 대상인 `BP_TileManager.Box`의 변경된 Extent를 다음 입력으로 다시 사용하고 있는지 확인한다. 입력은 첫 번째로 생성한 `BP_BaseTile.RecycleCheck`에서 한 번만 가져온다.

## 검증 항목

- `PoolSize = N`일 때 첫 타일부터 `N - 1`번째 타일까지 Collision 내부에 들어간다.
- 마지막 1개 타일은 Collision 외부의 재배치 여유분으로 유지된다.
- `Set Box Extent` 입력값이 전체 크기가 아니라 절반 크기인지 확인한다.
- Collision 위치가 기존 값에서 바뀌지 않는다.
- 타일 중심이 Collision 최소 경계와 타일 Half Size를 기준으로 계산된다.
- 타일의 `EndOverlap`이 한 프레임에 중복 처리되지 않는다.
- Blueprint 컴파일 후 PIE에서 타일 간 틈과 동시 다중 반환이 발생하지 않는다.
