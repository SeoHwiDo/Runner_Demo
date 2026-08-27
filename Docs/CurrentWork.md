# Runner 현재 작업

마지막 갱신: **2026-08-27 KST**

목표 완료: **2026-09-03 18:00 KST**

현재 상태: **NEXT-01 맵 풀링 완료, NEXT-03B Enemy Actor 생명주기·카탈로그 기반 준비 완료 및 Manager 초기 풀 생성 구현 단계**

현재 작업: **NEXT-03B 2단계 — `BP_EnemySpawnManager.InitializeEnemyPools`**

## 1. 제품 한 문장

플레이어가 좌우로 이동하며 지속 생성되는 맵에서 몬스터와 구조물을 상대하고, 투사체·플레이어 저장 스테이터스를 확장하는 3D 자동 러너다.

## 2. 이번 구현 범위

사용자의 2026-08-25 지시에 따라 기존 Trello 상세 백로그는 보관한다. 2026-08-26 사용자의 카드 분리 지시에 따라 과대해진 `NEXT-03`을 `NEXT-03A/B/C`로 교체해 아래 11개 카드만 현재 범위로 관리한다.

### 구현 완료

| ID | 카드 | 구현 내용 | 상태 |
|---|---|---|---|
| DONE-01 | 플레이어 캐릭터 생성 | 메시 연결, 애니메이션 연결 | 완료 |
| DONE-02 | 플레이어 이동 구현 | 좌우 이동, 발판 콜리전, 사이드 콜리전 | 완료 |
| DONE-03 | 맵 타일 | 기본형 맵 타일 액터, 머티리얼 연결 | 완료 |
| NEXT-01 | 맵 풀링 및 지속 이동 | 타일 초기 풀, 지속 이동, 이탈 감지, 앞쪽 재배치 | 완료 |

- `DONE-01`~`DONE-03` 완료 표기는 사용자 확인 기준이다.
- `NEXT-01`은 Trello 체크리스트 11개 완료와 2026-08-26 PIE 실행 기록을 기준으로 완료 상태를 동기화했다.

### 앞으로 구현

| ID | 카드 | 구현 내용 | 상태 |
|---|---|---|---|
| NEXT-02 | 프로젝타일 생성 및 발사 | 프로젝타일 생성과 발사 연결 | 대기 |
| NEXT-03A | 몬스터 공통 기반 및 카탈로그 | Enemy 공통 상태·등급·Base Actor, Skeleton 자식과 실제 데이터 카탈로그 인스턴스 | 기반 완료·종별 확장 대기 |
| NEXT-03B | 몬스터 FIFO 풀 관리 | Actor 활성화·비활성화 기반을 사용해 Manager의 카탈로그 초기화와 종별 FIFO 대여·반환 연결 | 진행 중 |
| NEXT-03C | Stage별 몬스터 편성 | 3 Stage와 `Early 5/Mid 4/Boss 1` 편성·선택 | 대기 |
| NEXT-04 | 몬스터 및 구조물 타일에 정형 무작위로 배치 | 타일 회수 전 소유 Enemy 반환, 재배치 후 새 편성 요청 | 대기(설계 확정) |
| NEXT-05 | 프로젝타일 매니저 구현 | 종류, 속도, 생성 속도, 유효 거리를 스테이터스로 적용 | 대기 |
| NEXT-06 | 저장되는 플레이어 스테이터스 구현 | 어빌리티, 재화, 아이템, 펫 | 대기 |

## 3. Trello 동기화

- 보드: https://trello.com/b/JWrQNIlv/runner-arrow-a-row-%EB%AA%A8%EC%9E%91-2026-09-03
- 유지 리스트: `이번 스프린트 → 진행 중 → 검증 → 완료`
- 현재 열린 카드 수: 12개(`ISSUE-01` 보상 내용 정의 포함)
- 완료: `DONE-01`~`DONE-03`, `NEXT-01`
- 진행 중: `NEXT-03B`
- 이번 스프린트: `NEXT-02`, `NEXT-03A`, `NEXT-03C`, `NEXT-04`, `NEXT-05`, `NEXT-06`
- 전체 구현 백로그: `ISSUE-01 몬스터 보상 상자 보상 내용 정의 및 지급`
- 기존 열린 카드 80개는 사용자 지시에 따라 오류 없이 보관 처리했다.
- 카드 구체화: 구현 카드는 독립적으로 검증할 수 있게 분리했다. 체크리스트는 16개·항목 142개이며, `ISSUE-01`은 보상 기획이 정해질 때까지 별도 백로그로 둔다.
- `NEXT-01`: 구현 체크리스트 11개가 모두 완료됐고 카드는 `완료`에 있다. 에디터 로그에서 `TileManager` 컴파일 후 약 25.6초 PIE 실행을 확인했다.
- 기존 `NEXT-03`은 설명과 27개 체크 항목을 보존한 채 보관하고 `NEXT-03A/B/C`로 교체했다.
- `NEXT-03A`: `ST_ActorBaseStatus`, `E_EnemyType`, `BP_EnemyBase`, `ST_EnemyDefinition`, `BP_Enemy_Skel`, `BPDA_EnemyCatalogBase`가 생성됐다. 손상됐던 구조체 참조 오류는 사용자 확인 기준으로 해소됐고, 2026-08-27 `BPDA_EnemyCatalogBase`를 데이터 형식으로 사용하는 실제 `DA_EnemyCatalog` Data Asset 인스턴스를 새로 생성했다. Orc·Dragon 등 종별 자식과 카탈로그 확장은 풀 수직 단위 이후 진행한다. https://trello.com/c/PJaUvETW
- `NEXT-03B`: `EnemyPools`, `PoolIndexByEnemyID`, `PoolIndicesByType`, `ST_EnemyPool`, `ST_EnemyPoolIndexList`가 준비됐다. `BP_EnemyBase`에는 `ActivateFromPool`, `DeactivateFromPool`, 사망 후 Enemy/Reward 외형·Collision 전환 및 피해 적용 함수가 구현 중이며, 현재 요청은 `DA_EnemyCatalog`를 읽어 초기 Actor와 FIFO 대기열을 만드는 `InitializeEnemyPools` 구현이다. https://trello.com/c/jmuJU5pa
- `NEXT-03C`: 3 Stage·`5/4/1` Phase·등급별 후보와 진행 가중치·경계 검증 8개 항목이 모두 미완료다. https://trello.com/c/OYk3NREr
- `NEXT-04`: MapManager가 타일을 이동하기 전에 `ReturnEnemiesOwnedByTile`을 요청하고, 이동이 끝난 뒤 `PopulateTile`로 새 편성을 요청하는 순서를 추가했다. 총 19개 항목이 미완료다.
- 로컬 복구본: `Docs/TrelloImport.csv`

## 4. 현재 작업

- 카드: `NEXT-03B` 2단계
- 목표: `BP_EnemySpawnManager`가 실제 `DA_EnemyCatalog` 인스턴스의 `EnemyDefinitions`를 읽고, `MonsterID`별 초기 Actor를 한 번만 생성해 `EnemyPools` FIFO와 두 조회 Map을 일관된 인덱스로 구성하도록 `InitializeEnemyPools`를 구현한다.
- 선행 구현:
  - 기본형 맵 타일 액터
  - `TileManager`의 초기 타일 풀 생성과 순차 배치
  - 플레이어와 이동 맵 타일의 물리 Collision 분리
- 확정 설계:
  - 실제 Blueprint 에셋 이름은 현재 프로젝트 구현에 맞춰 `BP_EnemySpawnManager`를 사용하며, 몬스터 Actor 생성·수명·대여·반환·Stage 진행을 소유한다.
  - 공통 기반은 체력·공격력·생존 여부·이름을 담는 `ST_ActorBaseStatus`, 등급을 구분하는 `E_EnemyType`, Collision과 발판 Static Mesh를 가진 `BP_EnemyBase`로 구성한다.
  - 몬스터 등급은 `E_EnemyType`의 `Normal`, `Elite`, `Boss`로 구분한다. 별도 Grade Enum은 만들지 않는다.
  - Skeleton, Orc, Dragon처럼 외형·애니메이션·행동이 다른 몬스터는 `BP_EnemyBase`의 자식 Blueprint로 만들고 공통 체력·사망·풀 반환 로직은 부모에만 둔다.
  - `ST_EnemyDefinition`은 `MonsterID`, `MonsterType`, `MonsterClass`, `BaseStatus`, `InitPoolSize`, `SpawnWeight`, `MinStage`, `EnemyAttackType`을 가진다.
  - `BPDA_EnemyCatalogBase`의 `ST_EnemyDefinition` 배열과 `DA_EnemyCatalog` 인스턴스를 몬스터 에셋과 밸런스의 단일 원천으로 사용한다.
  - 표시용 이름과 조회 키를 분리한다. `ST_ActorBaseStatus`의 이름은 표시용으로 사용하고, 풀과 Map 조회에는 고유 `MonsterID` Name을 사용한다.
  - `BP_EnemyBase`는 원본 상태와 매 대여 때 복사·초기화하는 런타임 상태를 분리하며 카탈로그 원본 상태를 변경하지 않는다.
  - 풀은 `MonsterType → MonsterID → FIFO 대기열` 계층으로 분리한다. 서로 다른 ID의 몬스터는 같은 등급이어도 별도 대기열과 풀 크기를 가진다.
  - 설계 데이터는 하나의 카탈로그 Array로 편집한다. `E_EnemyType`의 `Normal`, `Elite`, `Boss`는 별도 Rank가 아니라 몬스터 등급이자 Type이다.
  - 런타임 풀은 `EnemyPools : ST_EnemyPool Array`, `PoolIndexByEnemyID : Map<Name, Integer>`, `PoolIndicesByType : Map<E_EnemyType, ST_EnemyPoolIndexList>`를 조합한다.
  - `ST_EnemyPool`은 `EnemyID`, `EnemyType`, `EnemyClass`, `AvailableEnemies`, `PoolSize`를 가진다. Skeleton, Orc, Wyvern, Dragon, Phoenix처럼 서로 다른 ID는 각각 독립된 FIFO 배열을 사용한다.
  - `ST_EnemyPoolIndexList`는 `PoolIndices : Integer Array`를 가진다. `PoolIndicesByType`은 `Normal → 소형 일반 몬스터 풀`, `Elite → 대형·강화 몬스터 풀`, `Boss → 보스 몬스터 풀`의 후보 인덱스를 조회하는 파생 데이터다.
  - `EnemyPools`가 실제 Actor와 FIFO 배열을 소유하는 원본이다. 두 Map은 `EnemyPools.Add`가 반환한 실제 인덱스를 초기화 시 기록하는 조회 전용 파생 데이터이며, Map을 기준으로 `EnemyPools`를 만들지 않는다.
  - Blueprint Map에서 구조체를 꺼내면 복사본이므로 `PoolIndicesByType`의 인덱스 배열을 수정한 뒤 같은 Type 키로 구조체를 다시 `Add`한다. `EnemyPools` 요소를 수정한 뒤에는 같은 인덱스에 `Set Array Elem`으로 저장한다.
  - `ActiveMonsters`는 Actor Reference Set으로 중복 활성화·반환을 확인한다. 순서가 없는 Set은 FIFO 대기열이나 무작위 후보 목록으로 사용하지 않는다.
  - BeginPlay에서 등급·이름별 설정 `PoolSize`만큼 한 번만 생성하고 각 비활성 FIFO 대기열에 넣는다.
  - 대여 시 `AvailableMonsters[0]`을 꺼내고, 반환 시 배열 끝에 추가한다.
  - `ActivateFromPool`과 `DeactivateFromPool`은 Enemy Actor 내부 상태와 컴포넌트만 다룬다. FIFO 배열에서 꺼내고 다시 넣는 작업은 `BP_EnemySpawnManager.AcquireEnemyByID`와 `ReturnEnemy`만 수행한다.
  - 몬스터 사망은 풀 반환 조건이 아니다. 사망한 Actor는 같은 인스턴스에서 `RewardBox` 상태로 전환하고, 플레이어가 보상 상자 Collision을 Overlap했을 때 보상 요청 후 `ReturnEnemy`를 호출한다.
  - 살아 있는 Enemy 또는 RewardBox가 MapManager 영역 밖으로 나간 경우에는 개별 Actor가 스스로 풀에 넣지 않는다. MapManager가 타일 재배치 직전에 `EnemySpawnManager.ReturnEnemiesOwnedByTile(Tile)`을 요청한다.
  - 타일 회수 순서는 `소유 Enemy 반환 요청 → TileEnemies 초기화 → 타일을 맨 앞으로 이동 → PopulateTile로 새 Enemy 배치 요청`으로 고정한다.
  - Enemy는 `OwningTile`을, Tile은 `TileEnemies` Actor Reference Array를 보유한다. 타일 회수 때 `Get All Actors Of Class`로 전체 Enemy를 검색하지 않는다.
  - 풀 생명주기는 `InPool`, `ActiveEnemy`, `RewardBox` 세 상태로 구분한다. 현재 `bInPool`·`bIsAlive`를 유지하더라도 상태 전이의 단일 기준은 `E_EnemyPoolState`로 통합한다.
  - `ActivateFromPool` 순서는 `Collision 비활성 유지 → Timer 정리 → PoolOwner/OwningTile/정의 저장 → 상태·HP 초기화 → 보상 상자 숨김 → Transform 적용 → Enemy 표시·Tick 활성화 → Enemy Collision 활성화`다.
  - `DeactivateFromPool` 순서는 `중복 반환 Guard → Enemy/Reward Collision 비활성화 → Timer·AI·이동·VFX 정지 → Enemy/보상 상자 숨김 → 런타임 참조·상태 초기화 → Tick 비활성화`다. Manager는 이 호출이 끝난 뒤에만 FIFO 끝에 Actor를 추가한다.
  - `ConvertToRewardBox`는 `ActiveEnemy → RewardBox` 전이만 수행한다. 전투 Collision·Timer·AI를 끄고 Enemy 외형은 숨기며 RewardBox 외형과 Overlap Collision을 켜되, `bInPool`은 `false`를 유지한다.
  - 보상 종류·수량·데이터·UI·저장 반영은 `ISSUE-01`로 미루고, 현재 단계에서는 보상 지급 요청 Event/Interface 호출 지점만 확보한다.
  - `bInPool`로 중복 반환을 막고 `OwningTile`로 오래된 타일 참조가 재사용된 몬스터를 반환하지 못하게 한다.
  - 풀이 비었을 때 화면의 활성 몬스터를 강제로 회수하지 않고 해당 Spawn 요청을 건너뛴다.
  - 전체 진행은 3개 Stage이며 각 Stage는 약 10개 Tile로 구성한다. Stage 내부 Phase는 `Early 5 Tile / Mid 4 Tile / Boss 1 Tile`을 기본값으로 둔다.
  - Early 구간은 일반 몬스터 1마리로 시작해 2마리까지 증가한다.
  - Mid 구간은 일반 몬스터 패턴을 유지·확장하면서 정예 몬스터를 추가한다. 일반 몬스터 수는 전체 진행도에 따라 타일당 1~3마리 범위에서 결정한다.
  - Stage의 마지막 Tile은 Boss Phase로 사용하며 보스 몬스터를 배치한다.
  - 일반 몬스터의 좌우 위치는 Tile을 2분할한 `LeftSectionCenter` 또는 `RightSectionCenter` 중 하나를 사용한다. 전후 위치는 진입 직후 조우를 막는 `EntrySafetyMargin`과 끝 여백 사이에서 무작위로 정한다.
  - 정예·보스 몬스터는 Tile의 가장 끝 중앙 `EndCenterSpawnPoint`에 배치한다.
  - Stage별 Tile 수, Phase 구간, 일반 몬스터 수, 안전 여백, 최소 전후 간격, 등급·이름별 풀 크기는 Data Asset 또는 Data Table을 단일 원천으로 사용한다.

### NEXT-03B 2단계 상세 계획 — `InitializeEnemyPools`

- 선행 조건:
  - `BP_EnemyBase.ActivateFromPool`, `DeactivateFromPool`과 풀 상태 전이의 기본 그래프가 존재한다.
  - `ST_EnemyDefinition → BPDA_EnemyCatalogBase → DA_EnemyCatalog` 의존성과 실제 Data Asset 인스턴스 생성이 완료됐다.
  - `BP_EnemyBase`의 클래스 기본 상태는 Spawn 직후 Overlap을 발생시키지 않도록 숨김, Tick Off, Enemy/Reward Collision Off, `InPool`로 유지한다.
- Manager 변수:
  - `EnemyCatalog : BPDA_EnemyCatalogBase Object Reference`, 기본값은 실제 Data Asset 인스턴스 `DA_EnemyCatalog`로 지정한다.
  - `EnemyPools : ST_EnemyPool Array`는 실제 종별 FIFO와 Actor 참조를 소유한다.
  - `PoolIndexByEnemyID : Map<Name, Integer>`는 `MonsterID → EnemyPools Index`를 조회한다.
  - `PoolIndicesByType : Map<E_EnemyType, ST_EnemyPoolIndexList>`는 등급별 후보 Pool Index를 조회한다.
  - `ActiveEnemies : BP_EnemyBase Object Reference Set`은 중복 대여·반환을 검사한다.
  - `bPoolsInitialized : Boolean`으로 BeginPlay 중복 초기화를 차단한다.
- 초기화 순서:
  1. `bPoolsInitialized`가 `true`면 즉시 반환해 중복 Actor 생성을 막는다.
  2. `EnemyCatalog` 유효성을 검사한 뒤 `EnemyPools`, `PoolIndexByEnemyID`, `PoolIndicesByType`, `ActiveEnemies`를 비우고 `DA_EnemyCatalog.EnemyDefinitions`를 직접 읽는다.
  3. 각 정의에서 `MonsterID != None`, `MonsterClass` 설정, `InitPoolSize > 0`, ID 비중복을 검사한다. 잘못된 정의는 경고 후 건너뛴다.
  4. `MonsterClass` Soft Class Reference를 `Load Class Asset Blocking`으로 한 번 로드한다. 초기화 루프 안에서 비동기 로드를 사용하지 않는다.
  5. 로컬 `NewAvailableEnemies : BP_EnemyBase Array`를 비우고 `InitPoolSize`만큼 `Always Spawn, Ignore Collisions`로 Actor를 생성한다.
  6. 생성된 Actor에 정의와 `PoolOwner`를 전달해 `InitializeEnemy`를 호출하고, 이어서 `DeactivateFromPool`을 호출한 뒤 `NewAvailableEnemies` 끝에 추가한다.
  7. 실제 생성 성공 수를 `PoolSize`로 사용해 `ST_EnemyPool`을 만들고 `EnemyPools.Add`의 반환값을 `NewPoolIndex`로 저장한다.
  8. `PoolIndexByEnemyID.Add(MonsterID, NewPoolIndex)`로 ID 조회를 등록한다.
  9. `PoolIndicesByType.Find(MonsterType)` 결과의 `PoolIndices`에 `NewPoolIndex`를 추가한다. 기존 키가 없으면 새 `ST_EnemyPoolIndexList`를 만들고, 있으면 수정한 구조체를 같은 키로 다시 `Add`한다.
  10. 전체 정의 처리가 끝나면 인덱스 정합성을 검사하고 성공 시에만 `bPoolsInitialized = true`로 설정한다.
- 런타임 불변 조건:
  - 초기화 이후 `EnemyPools`에 `Remove Index`, `Insert`, `Sort`를 사용하지 않는다. 요소 순서를 바꾸면 두 조회 Map의 인덱스가 무효화된다.
  - 풀 인덱스를 SaveGame에 저장하지 않는다. 카탈로그 순서가 바뀌면 실행마다 인덱스가 달라질 수 있다.
  - 중복 `MonsterID`, 로드 실패 Class, Spawn 실패 Actor, 범위를 벗어난 Index는 오류로 기록하고 잘못된 참조를 배열에 넣지 않는다.
  - 풀 고갈 시 새 Actor를 런타임에 무제한 생성하거나 활성 Actor를 강제 회수하지 않고 해당 Spawn 요청을 건너뛴다.
- 완료 조건:
  - `Skeleton` 정의가 `EnemyPools`의 실제 반환 인덱스와 `PoolIndexByEnemyID`에 동일하게 등록된다.
  - `Normal`, `Elite`, `Boss`의 `PoolIndicesByType`에 해당 Type과 일치하는 Pool Index만 포함된다.
  - `PoolIndexByEnemyID`의 모든 값이 `EnemyPools.IsValidIndex`를 통과하고, 조회한 Pool의 `EnemyID`가 Map 키와 일치한다.
  - 초기화 재호출 시 Actor가 중복 생성되지 않는다.
  - Blueprint 컴파일 오류와 새 경고가 없고 PIE에서 Skeleton 초기 풀 수, 비활성 상태, ID 조회 결과를 확인한다.
- 현재 1단계 완료 조건:
  - Unreal Engine 5.7.4 Blueprint-only로 구현한다.
  - `ActivateFromPool` 호출 후 `bInPool=false`, 생존 상태, 최대 체력, 지정 Transform, Enemy 표시·Collision·필요 Tick이 복원된다.
  - `DeactivateFromPool` 호출 후 Enemy와 RewardBox Collision, Timer, 이동/AI, VFX, 표시와 Tick이 모두 정지하고 `OwningTile` 등 런타임 참조가 제거된다.
  - 활성화→비활성화→재활성화 시 이전 타겟·Timer·체력·RewardBox 상태가 새 대여에 남지 않는다.
  - 사망 시 즉시 비활성화되지 않고 RewardBox 상태로 남는다.
  - 관련 Blueprint 컴파일 오류와 새 경고가 없다.
  - PIE에서 수동으로 Skeleton을 `Activate → Deactivate → Activate`했을 때 상태 초기화를 확인한다.
- 영향 예상: `Content/Enemy/EnemyBase/BP_EnemyBase`, `Content/Struct/E_EnemyPoolState`(신규), 이후 `Content/Managers/BP_EnemySpawnManager`
- 현재 확인: `BP_EnemyBase`에는 `ActivateFromPool`, `DeactivateFromPool`, `CurrentHP`, `bIsAlive`, `PoolOwner`, `OwningTile`, `PoolState`와 Enemy/Reward 외형·Collision 전환 기반이 존재한다. 피해 적용 함수는 `CurrentHP - IncomingDamage`, `bIsAlive = CurrentHP > 0` 방향으로 검토됐지만 사용자 수정 완료와 PIE 결과는 아직 확인하지 않았다. `BP_EnemySpawnManager`에는 `EnemyPools`, `PoolIndexByEnemyID`, `PoolIndicesByType` 컨테이너와 `ST_EnemyPool`, `ST_EnemyPoolIndexList`가 준비됐으며, 새 `DA_EnemyCatalog`가 실제 Data Asset 인스턴스로 생성됐다. Manager의 초기화·FIFO 대여·반환 그래프는 아직 없다.
- 완료 작업: `NEXT-01`의 타일 지속 이동·이탈 감지·재배치는 Trello 완료 상태와 2026-08-26 PIE 실행 기록을 기준으로 완료 처리했다.
- 다음 행동: UE 5.7.4에서 `BP_EnemySpawnManager.EnemyCatalog`를 `BPDA_EnemyCatalogBase Object Reference`로 만들고 기본값에 새 `DA_EnemyCatalog`를 지정한다. 이어서 `InitializeEnemyPools`에서 카탈로그 유효성·중복 ID·Class 로드·초기 Spawn·비활성 FIFO 구성·ID/Type 조회 Map 기록·중복 초기화 방지를 구현하고 Skeleton 풀로 PIE 검증한다. 성공 후 `AcquireEnemyByID`와 `ReturnEnemy`를 연결한다.

## 5. 구현 순서

1. `NEXT-03B` 1단계 Enemy Actor 활성화·비활성화 생명주기
2. `NEXT-03B` 2단계 카탈로그 기반 `InitializeEnemyPools`와 조회 인덱스 구성
3. `NEXT-03B` 3단계 Manager FIFO 대여·반환과 RewardBox 전환
4. `NEXT-03A` 구조체 참조 복구·종별 자식 Blueprint·카탈로그 마무리
5. `NEXT-03C` Stage별 몬스터 편성·선택
6. `NEXT-04` 타일 SpawnPoint 기반 몬스터 배치·회수
7. `NEXT-02` 프로젝타일 생성 및 발사
8. `NEXT-05` 프로젝타일 매니저
9. `NEXT-06` 저장되는 플레이어 스테이터스

## 6. 위험과 검증

| 위험 | 대응 |
|---|---|
| 맵 타일 경계에서 틈·중첩 발생 | 재배치 기준점과 타일 길이를 데이터화하고 경계 PIE 확인 |
| `DA_EnemyCatalog`를 Blueprint Class로 잘못 만들거나 Manager 변수를 Class Reference로 지정 | 실제 Data Asset 인스턴스를 사용하고 Manager에는 `BPDA_EnemyCatalogBase Object Reference`로 연결 |
| 피해 적용 계산 또는 생존 판정이 반대로 연결됨 | `Clamp(CurrentHP - IncomingDamage, 0, MaxHP)`와 `CurrentHP > 0`을 PIE에서 확인 |
| 타일·프로젝타일 무제한 생성 | 풀 크기와 활성 수 제한, 반환 시 상태 초기화 |
| FIFO 배열의 중복 반환·오래된 타일 참조 | `bInPool`과 `OwningTile` 검증 후 반환, 타일 회수 시 참조 배열 초기화 |
| 사망 직후 Actor를 반환해 보상 상자가 사라짐 | 사망은 `RewardBox` 상태 전이만 수행하고 플레이어 획득 또는 타일 회수 때만 Manager 반환 호출 |
| 비활성화 중 Collision이 남아 중복 Overlap 발생 | `DeactivateFromPool`의 첫 처리로 Enemy·Reward Collision을 모두 끄고 Timer/AI보다 먼저 차단 |
| 재대여한 Enemy에 이전 상태가 남음 | Collision을 마지막에 켜고, 그 전에 Timer·HP·Target·OwningTile·RewardBox 표시를 전부 초기화 |
| 등급·이름별 하위 풀이 없거나 고갈됨 | 잘못된 키는 경고 후 건너뛰고, 고갈 시 활성 몬스터 강제 회수 없이 해당 요청만 생략 |
| 표시 이름을 Map 키로 사용해 이름 변경·중복 시 풀 조회 실패 | 표시용 이름과 별도로 변경되지 않는 `MonsterID` Name을 사용 |
| Blueprint Map의 구조체/배열 값을 수정했지만 원본에 반영되지 않음 | ID→PoolIndex Map은 조회 전용으로 두고 실제 변경 가능한 풀은 Array에서 수정 후 `Set Array Elem` |
| `EnemyPools` 순서 변경으로 조회 Map의 인덱스 무효화 | 초기화 후 `Remove Index`·`Insert`·`Sort`를 금지하고 재초기화 시 전체 컨테이너를 함께 재구성 |
| 초기화 중복 호출로 비활성 Actor가 중복 생성됨 | `bPoolsInitialized` Guard를 두고 재초기화가 필요하면 별도 정리 함수로 기존 Actor를 먼저 회수·제거 |
| Soft Class 로드 또는 Actor Spawn 실패 | 정의별 유효성 검사 후 실패 항목을 건너뛰고 실제 생성 성공 수만 `PoolSize`에 기록 |
| Data Table의 기본 생존 상태가 런타임 피해로 오염됨 | `BaseStatus`를 원본으로 유지하고 대여 시 `CurrentStatus`로 복사해 체력과 생존 상태 초기화 |
| 몬스터 풀이 모두 사용 중 | 활성 몬스터 강제 회수 대신 Spawn 요청 건너뛰기, PIE에서 적정 PoolSize 측정 |
| Tile 진입 직후 몬스터 조우 또는 몬스터끼리 겹침 | `EntrySafetyMargin`, 끝 여백, 최소 전후 간격으로 무작위 범위를 제한하고 배치 실패 시 요청 생략 |
| Stage Phase 경계가 한 Tile씩 어긋남 | `TileIndexInStage`의 `1~5/6~9/10` 범위를 단일 함수로 판정하고 `5→6`, `9→10`, `10→다음 Stage` PIE 확인 |
| 정형 무작위 배치가 진행을 막음 | 좌우 Section 중앙과 유효 전후 범위를 데이터화하고 안전 경로 규칙 유지 |
| 프로젝타일 스테이터스와 실제 동작 불일치 | 종류·속도·생성 속도·유효 거리의 단일 데이터 원천 유지 |
| 저장 데이터 누락·구버전 | SaveVersion과 안전한 기본값 적용 |

## 7. 결정 로그

- 2026-08-25: 사용자 지시에 따라 Trello의 기존 상세 백로그를 보관하고, 구현 완료 3개와 앞으로 구현할 6개만 현재 작업 범위로 다시 구성한다.
- 2026-08-25: 구현 완료 항목은 사용자 확인 기준으로 기록하며, 별도의 UE 5.7.4 컴파일·PIE 재검증 결과는 추후 기능 검증 시 추가한다.
- 2026-08-25: 현재 작업은 `NEXT-01 맵 풀링 및 지속 이동`으로 지정한다.
- 2026-08-25: 초기 타일 풀은 `TileManager`가 생성·순차 배치하고, 맵 범위 콜리전을 완전히 벗어난 타일을 앞쪽 배치 지점으로 회수하는 구조로 진행한다.
- 2026-08-26: 몬스터는 `BP_MonsterSpawnManager`가 수명과 Stage 진행을 관제하고 타일은 배치할 몬스터를 대여·반환한다. 풀은 `Normal/Elite/Boss → MonsterName → FIFO` 계층으로 분리하며 먼저 반환된 Actor를 먼저 재사용한다.
- 2026-08-26: 풀 고갈 시 화면의 활성 몬스터를 강제로 회수하지 않고 Spawn을 건너뛴다. 중복 반환과 오래된 타일 참조는 `bInPool`, `OwningTile` 검증으로 방지한다.
- 2026-08-26: 게임은 3개 Stage, Stage당 약 10개 Tile을 기본으로 하고 각 Stage를 `Early 5/Mid 4/Boss 1`로 구성한다. Early는 일반 1→2마리, Mid는 일반 패턴과 정예 추가, 마지막 Tile은 보스 배치로 진행한다.
- 2026-08-26: 일반 몬스터는 좌우 2분할 Section 중앙과 진입 안전 여백 이후의 무작위 전후 위치에 배치하고, 정예·보스는 Tile 가장 끝 중앙에 배치한다. 세부 수치는 Data Asset 또는 Data Table에서 조정한다.
- 2026-08-26: 기존 구현 명칭을 기준으로 몬스터 등급 Enum은 `E_MonsterType`을 사용하고 별도 `E_MonsterGrade`를 만들지 않는다. `ST_ActorBaseStatus`와 `BP_MonsterBase`는 공통 런타임 기반으로 유지한다.
- 2026-08-26: 몬스터 에셋은 `ST_MonsterDefinition` 카탈로그에서 관리하고, 실행 시 등급별 ID Array, `MonsterID→PoolIndex` Map, 실제 FIFO Pool Array, 활성 Actor Set으로 역할을 분리한다.
- 2026-08-26: `NEXT-03` 한 카드가 기반·카탈로그·풀·Stage 편성을 모두 포함해 비대해졌으므로 `NEXT-03A 공통 기반 및 카탈로그`, `NEXT-03B FIFO 풀 관리`, `NEXT-03C Stage별 편성`으로 분리한다.
- 2026-08-26: 실제 생성된 에셋 명칭을 기준으로 `Monster` 접두사 설계를 `Enemy` 계열(`E_EnemyType`, `BP_EnemyBase`, `ST_EnemyDefinition`, `BP_EnemySpawnManager`)에 맞춘다. 데이터 필드의 `MonsterID`, `MonsterType` 명칭은 기존 구현을 유지한다.
- 2026-08-26: 구현 우선순위를 `NEXT-03B`로 바꾸고, 먼저 `BP_EnemyBase.ActivateFromPool`과 `DeactivateFromPool`의 Actor 내부 생명주기를 완성한다. FIFO 컨테이너 변경은 Manager만 담당한다.
- 2026-08-26: Enemy 사망은 즉시 풀 반환하지 않고 같은 Actor를 `RewardBox` 상태로 전환한다. 반환은 보상 상자 Player Overlap 또는 MapManager의 타일 회수 요청 두 경로로 제한한다.
- 2026-08-26: MapManager의 타일 재배치는 `ReturnEnemiesOwnedByTile → 타일 이동 → PopulateTile` 순서로 EnemySpawnManager와 연동한다. 보상 내용은 별도 `ISSUE-01`로 이관한다.
- 2026-08-26: `E_EnemyType`의 `Normal`, `Elite`, `Boss`를 몬스터 등급이자 Type으로 유지한다. 각 Type 아래에는 Skeleton 같은 소형 일반 몬스터, Orc·Wyvern 같은 정예 몬스터, Dragon·Phoenix 같은 보스 몬스터의 독립 풀을 둔다.
- 2026-08-26: `EnemyPools` Array를 런타임 원본으로 사용하고, `PoolIndexByEnemyID`와 `PoolIndicesByType`은 `EnemyPools.Add` 반환 인덱스로 초기화하는 조회 전용 파생 데이터로 사용한다. 인덱스는 하드코딩하지 않는다.
- 2026-08-26: `NEXT-03B`의 다음 수직 단위는 카탈로그 기반 `InitializeEnemyPools` 구현으로 정한다. Actor 생명주기 완료 후 초기 풀 생성·ID 조회·Type별 후보 인덱스와 중복 초기화 방지를 구현한다.
- 2026-08-27: `BPDA_EnemyCatalogBase`는 `PrimaryDataAsset` 기반 데이터 형식 Blueprint로 유지하고, 런타임 카탈로그는 이 형식의 실제 `DA_EnemyCatalog` Data Asset 인스턴스로 사용한다. Manager는 Class Defaults가 아니라 `BPDA_EnemyCatalogBase Object Reference`에서 `EnemyDefinitions`를 직접 읽는다.
- 2026-08-27: 현재 사용자 요청을 `NEXT-03B 2단계 InitializeEnemyPools`로 확정한다. 이번 수직 단위에서는 카탈로그 기반 초기 생성과 FIFO 컨테이너/조회 Map 구성까지만 구현하고 Stage 편성·타일 배치는 뒤 카드에서 처리한다.

## 8. 변경 로그

- 2026-08-25: 현재 에셋 `BP_RunnerPlayer`, `NewAnimBlueprint`, `BP_BaseTile`, `BP_MapOutLine`, `TileManager`, `M_BaseTile`, `MI_BaseTile`을 기준으로 9개 Trello 카드의 구현 내용을 UE 5.7.4 Blueprint 단위로 구체화했다. 완료 카드 3개의 신규 구현 항목 23개는 완료, 앞으로 구현할 카드 6개의 신규 항목 73개는 미완료로 기록했다. 재조회 결과 체크리스트 14개·항목 111개이며 누락·조회 오류는 0건이다.
- 2026-08-25: CurrentWork를 사용자 지정 9개 카드 구조로 재작성했다. 기존 Trello 열린 카드 80개를 모두 보관하고 새 카드 9개를 생성했다. 재조회 결과 완료 3개·진행 중 1개·이번 스프린트 5개만 열려 있으며, 완료 카드와 사용자 지정 하위 체크리스트 상태가 모두 일치한다.
- 2026-08-25: `NEXT-01`의 초기 타일 생성과 순차 배치 구현을 사용자 확인 기준으로 기록했다. UE 5.7.4 `CompileAllBlueprints`에서 Blueprint 오류 0개·경고 0개·로드 실패 0개를 확인했으며, 샌드박스 DDC/Zen 쓰기 제한으로 명령 종료 코드만 1이었다. 타일 이동·범위 이탈 감지·재배치와 PIE 검증은 다음 단계로 남겼다.
- 2026-08-25: Trello `NEXT-01` 카드 설명에 커밋 `81fcddf`, 초기 생성·순차 배치, Blueprint 컴파일 결과와 남은 작업을 반영했다. `초기 활성 타일을 진행 축 방향으로 TileLength 간격 배치` 항목을 완료 처리했으며 카드는 `진행 중`을 유지했다.
- 2026-08-25: `Docs/Studys.md`에 고정 카메라, 플레이어 이동 Collision과 시각적 타일 분리, `PlayerRef` 수명 오류, 고정 타일 결정, 초기 풀 생성, Collision 기반 재배치 설계와 검증 결과를 학습 기록으로 추가했다.
- 2026-08-26: 최신 사용자 지시에 따라 현재 작업을 `NEXT-03/NEXT-04 FIFO 몬스터 풀링 및 타일 배치`로 전환했다. 실제 Blueprint 구현 전 단계로 Manager 소유권, FIFO 대기열, 활성화·비활성화, 풀 고갈, 중복 반환과 타일 소유권 규칙을 확정했다.
- 2026-08-26: Trello `NEXT-03`과 `NEXT-04`의 설명·체크리스트를 FIFO 풀 설계에 맞게 최신화하고 `NEXT-04`를 `진행 중`으로 이동했다. 구현 및 PIE 검증 전이므로 두 카드의 체크 항목은 완료 처리하지 않았다.
- 2026-08-26: 사용자 지정 `BP_MonsterSpawnManager` 중심 구조, 등급·이름별 FIFO 하위 풀, 3 Stage와 `5/4/1` Phase 구성, 일반·정예·보스의 Tile 배치 규칙을 `NEXT-03/NEXT-04` 계획과 Trello 설명·체크리스트에 반영했다. 재조회 결과 `NEXT-03` 16개, `NEXT-04` 18개 항목이 모두 미완료이며 두 카드는 `진행 중`이다.
- 2026-08-26: 사용자 확인 기반의 `ST_ActorBaseStatus`, `E_MonsterType`, `BP_MonsterBase` 현황과 Skeleton·Orc·Dragon 자식 Blueprint, 몬스터 카탈로그, Array·Map·Set 역할 분리 설계를 `NEXT-03` 계획과 Trello에 추가했다. 재조회 결과 체크리스트 27개 중 기반 구성 3개 완료, 나머지 구현·검증 24개 미완료이며 카드는 `진행 중`을 유지한다.
- 2026-08-26: 과대해진 기존 Trello `NEXT-03`을 기록 보존용으로 보관하고, 독립 완료 가능한 `NEXT-03A/B/C` 세 카드로 분리했다. 기반 완료 상태는 `NEXT-03A`에만 승계하고 FIFO와 Stage 편성 항목은 미완료로 유지했다. 재조회 결과 열린 카드 11개, 열린 체크리스트 16개·항목 134개이며 분리 카드의 누락은 없다.
- 2026-08-26: Trello 재조회 중 남은 작업이 기록된 `NEXT-01`이 `완료` 리스트에 있음을 확인해 CurrentWork와 맞도록 `진행 중` 리스트로 복귀시켰다.
- 2026-08-26: 최신 프로젝트와 Trello를 다시 대조해 `NEXT-01`을 완료로 동기화했다. 에디터 로그에서 `TileManager` 컴파일 후 약 25.6초 PIE 실행을 확인했다.
- 2026-08-26: `NEXT-03A`의 실제 에셋 `ST_EnemyDefinition`, `BP_Enemy_Skel`, `BPDA_EnemyCatalogBase`, `DA_EnemyCatalog`와 `NEXT-03B`의 `BP_EnemySpawnManager` 컨테이너 생성을 현황에 반영했다. UE 5.7.4 전체 Blueprint 컴파일은 0개 오류·0개 경고·로드 실패 0개였으나, 카탈로그의 `EnemyDefinitions` 구조체 직렬화에서 `Unknown structure` 오류 1건을 별도 미완료 위험으로 기록했다.
- 2026-08-26: 사용자 지정 생명주기에 맞춰 현재 작업을 `NEXT-03B`로 전환했다. 사망→RewardBox, 보상 획득/타일 회수→풀 반환, 타일 회수 전 반환·재배치 후 재배치 요청의 경계를 문서화하고 보상 내용은 `ISSUE-01`로 분리했다.
- 2026-08-26: 현재 작업물을 의존성 순서로 분리해 플레이어 경로(`6e0207c`), 카메라(`e9c3379`), 타일 Manager(`1fd4fff`), Enemy 공통 데이터(`ca7bed0`), Enemy 기반·Manager(`ff2e145`), Skeleton Knight 예제(`fceb3c7`), Skeleton Enemy·카탈로그(`fc413d4`) 커밋으로 기록했다. 외부 예제는 실제 참조하는 `Skeleton`, `Skeleton_Knight_01`과 Enemy 발판의 돌 Texture 2개만 포함했다.
- 2026-08-26: `BP_EnemySpawnManager`의 최신 컨테이너와 신규 `ST_EnemyPool`, `ST_EnemyPoolIndexList`를 기준으로 `InitializeEnemyPools` 구현 순서, 실패 처리, 인덱스 불변 조건과 PIE 완료 조건을 다음 작업으로 문서화했다.
- 2026-08-27: 사용자 생성 결과를 확인해 새 `DA_EnemyCatalog`를 실제 Data Asset 인스턴스로 기록했다. `NEXT-03A`의 구조체 참조 오류 표기를 해소 상태로 갱신하고, `NEXT-03B` 현재 작업을 Actor 생명주기 정리에서 `BP_EnemySpawnManager.InitializeEnemyPools` 구현으로 전환했다.
- 2026-08-27: 루트 `CurrentWork.md`의 BatchNamer 완료 기록을 이 문서의 완료 이력으로 병합하고, 단일 현황판 경로를 `Docs/CurrentWork.md`로 확정했다.

## 9. 완료된 외부 도구 작업 기록 — BatchNamer

이 절은 기존 루트 `CurrentWork.md`에 있던 완료 작업을 보존한 기록이다. 현재 게임 구현 우선순위와 다음 행동은 문서 상단의 `NEXT-03B`를 기준으로 한다.

### 9.1 완료 범위

- 카드 ID: `TOOL-002`
- 상태: 완료
- 목표: BatchNamer에 순번 추가, 기존 숫자 제거, 기존 숫자 자리수 정규화 기능을 추가하고 UE 5.7.4 Win64 배포본을 갱신한다.
- 영향 파일:
  - `Plugins/BatchNamer/BatchNamer.uplugin`
  - `Plugins/BatchNamer/Source/BatchNamerCore`
  - `Plugins/BatchNamer/Source/BatchNamer`
  - `Plugins/BatchNamer/Build/PackagePlugin.ps1`
  - `Plugins/BatchNamer/README.md`, `CHANGELOG.md`, `LICENSE.md`
  - `Runner_Demo.uproject`
- 완료 조건:
  - 순번 추가가 시작 번호와 자리수 설정을 반영한다.
  - 기존 이름의 연속 숫자 묶음을 제거할 수 있다.
  - 기존 이름의 연속 숫자 묶음을 지정 자리수로 0 채움하며 긴 숫자는 자르지 않는다.
  - 제거와 자리수 정규화가 동시에 적용되지 않도록 UI 상태를 제어한다.
  - 자동화 테스트와 UE 5.7.4 Win64 BuildPlugin 검증을 통과한다.

### 9.2 남은 수동 확인과 위험

1. Runner 에디터를 재시작할 때 BatchNamer 1.2.0 UI가 로드되는지 확인한다.
2. 필요하면 임시 에셋과 액터로 숫자 제거, 자리수 정규화, 순번 추가 및 Undo/Redo를 수동 스모크 테스트한다.

- 공개 라이선스 종류와 최종 `CreatedBy` 표기는 사용자가 지정하지 않아 권리 보유 기본 문구와 배포 전 확인 항목으로 남겼다.
- 로컬 Git LFS 실행 문제로 일반 `git status`가 실패할 수 있다. BatchNamer 추적 대상은 텍스트 소스이며 생성 ZIP은 Git에서 제외되는 `Builds`에 두었다.
- Trello 연결 부재로 `TOOL-001` 카드를 생성하거나 완료 상태로 동기화하지 못했다.
- 사용자 에디터 세션과 에셋을 보호하기 위해 실제 UI 이름 변경 및 Undo/Redo 검증은 실행하지 않았다.
- `TOOL-002` 작업 당시 로드된 1.1 DLL을 덮어쓰지 않고 1.2 DLL을 버전명으로 별도 배치했으며, 다음 시작에서 사용할 모듈 매핑을 갱신했다.

### 9.3 결정 기록

- 2026-08-26: 사용자의 `fallguys_demo`는 인접 프로젝트 `C:\shd\Unreal\fallGuys`를 뜻하는 것으로 해석했다.
- 2026-08-26: 프로젝트 게임 런타임과 분리하기 위해 `BatchNamerCore`와 `BatchNamer`를 모두 Editor 전용 모듈로 구성했다.
- 2026-08-26: 이름 규칙과 검증은 `Core` 모듈로 분리하고 Slate, Content Browser, Level Editor 의존성은 UI 모듈에만 두었다.
- 2026-08-26: 생성 배포본은 Git 추적 대상이 아닌 `Builds/BatchNamer-1.1.0-UE5.7-Win64.zip`에 두었다.
- 2026-08-26: 숫자 제거는 이름에 포함된 모든 연속 숫자 묶음을 제거하는 것으로 정의했다.
- 2026-08-26: 기존 숫자 자리수 정규화는 각 연속 숫자 묶음의 왼쪽을 0으로 채우며, 지정 자리수보다 긴 숫자는 유지한다.
- 2026-08-26: 처리 순서는 `찾기/바꾸기 → 기존 숫자 제거 또는 자리수 정규화 → 접두사/접미사 → 새 순번 추가`로 고정했다.
- 2026-08-26: 숫자 제거와 기존 숫자 자리수 정규화는 상호 배타 옵션으로 제공한다.
- 2026-08-26: BatchNamer 버전을 1.2.0으로 올리고 새 배포본을 `Builds/BatchNamer-1.2.0-UE5.7-Win64.zip`에 두었다.

### 9.4 검증 결과

- `TOOL-002` UE 5.7.4 `RunUAT BuildPlugin -TargetPlatforms=Win64 -Rocket`: 성공, 종료 코드 0
- 배포본 최소 테스트 프로젝트에서 `Mounting External plugin BatchNamer` 확인
- 자동화 테스트 `BatchNamer.Core.RuleTransformation`, `BatchNamer.Core.Validation`: 모두 성공
- 숫자 묶음 전체 제거, 묶음별 0 채움, 긴 숫자 유지, 제거 우선순위와 새 순번 추가 확인
- 플러그인 버전: 1.2.0
- 배포 ZIP: `Builds/BatchNamer-1.2.0-UE5.7-Win64.zip`, 16,364,065 bytes
- ZIP 루트 `BatchNamer/BatchNamer.uplugin` 확인
- SHA-256: `72C32D3C9B5B1F37683D1FEEE7D67DAE2C3D85AA7F518F05292BAF039F7CB363`
- 플러그인과 현황판 후행 공백 검사: 문제 없음
- 실제 UI 이름 변경과 Undo/Redo 수동 테스트: 미실행

#### `TOOL-001` 기준선

- Unreal Engine: 5.7.4, Windows 64-bit
- `RunUAT BuildPlugin -TargetPlatforms=Win64 -Rocket`: 성공, 종료 코드 0
- Runner `UnrealEditor Win64 Development` 타깃 빌드: 성공, 종료 코드 0
- 배포본 최소 테스트 프로젝트 로드와 자동화 테스트 2개: 성공
- `Runner_Demo.uproject`, `BatchNamer.uplugin` JSON 파싱: 성공
- 배포 ZIP: `Builds/BatchNamer-1.1.0-UE5.7-Win64.zip`, 16,347,974 bytes
- SHA-256: `945F97E8903B45341365EAD9B4D739825939F62988F46C0AA8968FE24E43B02D`
- 실제 에셋·액터 UI 이름 변경과 Undo/Redo 수동 테스트: 미실행

### 9.5 변경 기록

- 2026-08-26: `TOOL-001` 작업을 등록했다.
- 2026-08-26: `fallGuys/Plugins/BatchNamer`의 프로젝트 내부 구현을 Runner의 독립 플러그인 소스로 옮겼다.
- 2026-08-26: `BatchNamerCore` 규칙·검증 모듈과 `BatchNamer` 에디터 UI 모듈로 분리하고 자동화 테스트를 추가했다.
- 2026-08-26: 설치, 사용, 호환성, 배포 절차와 변경 기록 및 라이선스 고지를 추가했다.
- 2026-08-26: 재현 가능한 PowerShell 패키징 스크립트와 UE 5.7.4 Win64 배포 ZIP을 생성했다.
- 2026-08-26: `TOOL-001` 구현과 자동 검증을 완료했다. Trello는 연결 부재로 미동기화 상태다.
- 2026-08-26: `TOOL-002` 숫자 추가·제거·자리수 정규화 확장 작업을 등록했다.
- 2026-08-26: Core 규칙에 기존 숫자 제거와 숫자 묶음별 자리수 정규화를 추가했다.
- 2026-08-26: UI에 `Remove existing numbers`, `Pad existing numbers`, `Add sequential number` 옵션과 상호 배타 상태 제어를 추가했다.
- 2026-08-26: 숫자 처리 순서와 예시를 README와 CHANGELOG에 반영하고 버전을 1.2.0으로 갱신했다.
- 2026-08-26: UE 5.7.4 Win64 BuildPlugin과 배포본 자동화 테스트를 통과하고 `TOOL-002`를 완료했다. Trello는 연결 부재로 미동기화 상태다.
