# 실제 Blueprint 클립보드 붙여넣기 순서

## 선행 설정

1. `BP_BaseTile`에 `TileEnemies : BP_EnemyBase Object Reference Array`를 만든다.
2. `BP_LevelManager`의 EventGraph를 연다.
3. `01_LevelManager_EnemyManagerCache.txt` 전체를 복사해 EventGraph에 붙여넣고 기존 `Event BeginPlay` 실행 핀을 첫 노드에 연결한다.
4. `02_LevelManager_EnemyPlacedEvents.txt` 전체를 복사해 같은 EventGraph의 빈 곳에 붙여넣는다.
5. Compile한다.

## TileManager 호출

`BP_TileManager`에서 타일 재배치와 `CalculateStage`가 끝난 실행선에 다음을 둔다.

```text
LVManager → Call EnemyPlaced
TileRef = 재배치한 As BP_BaseTile
```

Custom Event는 첫 Compile 후 `LVManager` 핀에서 `EnemyPlaced`로 검색할 수 있다.

## 주의

- 이 블록은 `EnemyPlaced`, `PlaceNormalOnTile`, `PlaceBossEliteOnTile` Custom Event 세 개를 함께 생성한다.
- Normal 위치는 `NormalEnemyBox.GetComponentBounds` 안에서 무작위 추첨한다.
- Elite/Boss는 현재 수량 1 규칙에 맞춰 `BossElitePos01`을 사용한다.
- `BossElitePos02`는 수량 2 확장 때 연결한다.
- 붙여넣은 뒤 빨간 노드 또는 고아 핀이 생기면 저장하지 말고 스크린샷을 전달한다.

