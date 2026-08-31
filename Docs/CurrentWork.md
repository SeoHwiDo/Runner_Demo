# Runner 현재 작업

마지막 갱신: **2026-08-31 KST**

제출일: **2026-09-01 화요일(시간 미정)**

현재 상태: **제출용 필수 기능만 구현하는 최소 범위로 축소**

현재 작업: **SUBMIT-02 — `BP_ProjectileManager.InitProjectilePool` 구현 중. 풀 초기화 완료 후 `RequestProjectile` 상태 확인과 투사체 활성화·반환·피해 연결 및 검증 진행**

Git 보조 정비: **LFS 접근 거부 복구·외부 맵 BuiltData의 미푸시 이력 제외·원격 푸시 검증 완료**

## 1. 제출 목표

오프닝 화면에서 게임, 상점, 저장·불러오기에 접근하고, 인게임에서 오브젝트 풀링과 충돌 전투를 수행하며, 상점에서 재화를 사용해 펫과 영구 스탯을 구매·강화한 결과가 재실행 후에도 유지되는 Unreal Engine 5.7.4 프로젝트 폴더를 제출한다.

제출 후 추가 개선은 진행하지 않는다. 기능 수와 콘텐츠 수는 필수 구조를 확인할 수 있는 최소치로 제한한다.

## 2. 필수 범위

### 2.1 오브젝트 풀링

- 기존 `BP_TileManager`의 타일 재사용 구조를 유지한다.
- `BP_BaseTile`의 전체 배치 크기를 첫 타일 생성 시 한 번 전달받아 타일 범위 Box Collision을 설정한다.
- 타일 범위 Collision은 첫 타일부터 `PoolSize - 1`개의 타일을 포함하고 마지막 1개 타일은 재배치 여유분으로 남긴다.
- 첫 타일만 `RecycleCheck.GetScaledBoxExtent`로 Manager Box를 초기화하고, 이후 타일은 직전 타일의 `NextSpawnPoint` 월드 위치에 배치한다.
- 오류 상태인 `GetTileWorldLength`와 제출 범위에서 제외된 초기 풀 Stage 계산을 제거한다.
- 기존 `BP_EnemySpawnManager`의 고정 크기 Enemy FIFO를 완성한다.
- `InitEnemyPool`, `AcquireEnemy`, `ReturnEnemy`만 핵심 풀 API로 사용한다.
- 타일 재사용 전에 기존 Enemy를 반환하고, 이동 후 새 Enemy를 대여·활성화한다.
- 풀 고갈 시 추가 Spawn이나 활성 Actor 강제 회수 없이 해당 요청을 생략한다.
- 2026-08-31 사용자 요청에 따라 `SeclectSpawnTransform`의 Type별 배치점 선택, 동일 타입 점유 위치 제외, Elite/Boss Bounds 보정 설계를 안내한다. 기존 기능의 실제 적용 상태는 미검증이며 다종 콘텐츠·Stage 확장까지 재개하지 않는다.
- 미구현 `EnemyPlaced`, `PlaceNormalOnTile`, `PlaceBossEliteOnTile`, `AcquireEnemyByID`, `CanAcquireEnemy`, `ReturnEnemiesOwnedByTile` 확장은 구현하지 않는다.

### 2.2 충돌과 전투

- 플레이어 좌우 이동과 맵 경계 Collision을 유지한다.
- 투사체 생성·자동 발사와 Enemy 충돌을 연결한다.
- 투사체 충돌 시 피해를 적용하고 Enemy 체력이 0이면 처치 처리한다.
- Enemy와 플레이어 충돌 시 플레이어 피해 또는 게임 종료를 처리한다.
- 처치 보상은 선택형 보상 상자 없이 재화를 즉시 지급하는 최소 구조로 단순화한다.
- 2026-08-31 사용자 정정 기준으로 `BP_ProjectileManager`는 Player와 Enemy가 함께 쓰는 월드 단위 풀로 구성하며, 현재 재개 지점은 `InitProjectilePool`의 풀 생성·초기 비활성화·Available 배열 등록 흐름이다. 이 함수 완료 후 `RequestProjectile` 상태를 확인하고, `BP_Projectile` 활성화·반환, 최대 거리 확인, 진영별 피해 Interface, Player·Enemy 발사 연결을 구현한다. 저장·컴파일·PIE는 완료 처리하지 않는다. 상세 설계는 `Docs/BlueprintSnippets/GenericProjectilePoolingSystem.md`를 따른다.

### 2.3 펫 인벤토리와 영구 스탯

- 인벤토리는 펫 목록으로만 사용한다.
- 펫 데이터는 보유 여부, 장착 여부, 레벨과 스탯 보너스를 가진다.
- 한 번에 장착할 수 있는 펫 수는 최소 구조로 제한한다.
- 플레이어 영구 스탯은 강화 레벨과 적용 값을 분리한다.
- 재화, 펫 보유·장착·레벨, 영구 스탯 강화 레벨을 `SaveGame`에 저장한다.
- 게임 재실행 후 Load하면 저장된 값과 적용 결과가 복원된다.

### 2.4 상점

- 재화를 사용해 미보유 펫을 구매한다.
- 보유 펫의 레벨을 강화한다.
- 플레이어 영구 스탯을 강화한다.
- 구매·강화 비용과 현재 재화를 표시한다.
- 재화 부족과 최대 레벨을 검사한다.
- 성공한 거래 결과를 SaveGame에 반영한다.

### 2.5 화면 구성

화면은 다음 3개만 사용한다.

1. 오프닝 화면
   - 게임 시작
   - 상점 이동
   - 세이브 파일 저장
   - 세이브 파일 불러오기
2. 인게임 화면
   - 플레이어 이동, 자동 발사, Enemy 풀링, 충돌 전투, 재화 획득
   - 오프닝 화면 복귀
3. 상점 화면
   - 펫 구매·강화
   - 영구 스탯 강화
   - 오프닝 화면 복귀

### 2.6 강화 선택·플레이어 스탯·인게임 HUD

- `ST_PlayerStats.StatusValue`는 `E_Stats → Float` Map으로 사용한다. `BP_RunnerPlayer`에는 `BaseStats`, `RunBoundStats`, `PermanentBonusStats`, `CurrentStats`가 있다.
- 확인된 `UpdateStats` 흐름은 `StatsList`의 각 `E_Stats`에 대해 Base·런 중·영구 보너스 Map 값을 합산하고, 결과 Map을 `CurrentStats.StatusValue`에 대입한다. 저장·컴파일·PIE 검증은 아직 수행하지 않았다.
- `RunUpgrade`는 아직 비어 있다. 선택한 `E_Stats`의 값을 `RunBoundStats.StatusValue`에 누적하고 `UpdateStats`를 호출해야 한다.
- 표지판 두 개가 같은 프레임에 겹쳐도 한 개만 적용하도록 `BP_BaseTile.bUpgradeSelected` Guard를 타일 초기화 시 해제하고, 적용 시작 전에 먼저 설정한다. 표지판 상태 Enum은 Player Stat Map과 같은 `E_Stats`로 통일해야 한다.
- `WBP_PlayerHealthBar`의 `UpdateHP`는 현재 HP를 그대로 0~1로 Clamp하면 0 또는 1로 표시된다. `CurrentStatus.HP / BaseStatus.HP` 비율을 Clamp하여 Progress Bar에 전달하도록 수정이 필요하다.
- 사용자 요청 HUD 범위는 플레이어 머리 위 체력바·현재 체력, 좌측의 현재/남은 Stage, 좌측 상단 점수·재화, 우측 상단 무기 강화·펫 상태, Esc 일시정지다. Stage 표시는 3개 Stage와 각 10개 타일 통과 수를 기준으로 한다.

## 3. 제출 카드

| ID | 작업 | 상태 |
|---|---|---|
| SUBMIT-01 | 오브젝트 풀링 기반 인게임 연결 | 완료 |
| SUBMIT-01A | 표지판 단일 선택과 `RunUpgrade` 적용 | 진행 중 |
| SUBMIT-01B | 3×10 타일 기반 Stage 진행 상태와 이벤트 | 대기 |
| SUBMIT-02 | 충돌·전투·재화 획득 | 진행 중 |
| SUBMIT-02A | 플레이어 체력 상태와 월드 체력바 | 대기 |
| SUBMIT-03 | 펫 인벤토리·영구 스탯·SaveGame | 대기 |
| SUBMIT-03A | 영구 스탯 강화 레벨·SaveGame 적용 | 대기 |
| SUBMIT-04 | 펫·스탯 상점 | 대기 |
| SUBMIT-05 | 오프닝·인게임·상점 화면 흐름 | 대기 |
| SUBMIT-05A | 인게임 HUD 정보 패널 | 대기 |
| SUBMIT-05B | Esc 일시정지 메뉴와 입력 복구 | 대기 |
| SUBMIT-06 | 통합·프로젝트 폴더 제출 정리 | 대기 |

기존 완료 항목인 플레이어 생성, 좌우 이동, 맵 타일, 타일 풀링은 유지한다.

## 4. 구현 순서

1. `SUBMIT-01`: Enemy FIFO 대여·반환과 타일 재사용 연결
2. `SUBMIT-01A`: 표지판 Guard, `RunUpgrade`, `UpdateStats` 재계산 연결
3. `SUBMIT-02`: 투사체 발사, 충돌, 피해, 처치, 재화 지급
4. `SUBMIT-02A`: 현재/최대 HP와 월드 체력바 갱신 연결
5. `SUBMIT-03`·`SUBMIT-03A`: 펫 인벤토리, 영구 스탯, SaveGame 저장·불러오기
6. `SUBMIT-04`: 펫 구매·강화와 영구 스탯 강화
7. `SUBMIT-01B`·`SUBMIT-05A`·`SUBMIT-05B`: Stage 이벤트, HUD, 일시정지 연결
8. `SUBMIT-06`: 전체 흐름 확인과 프로젝트 폴더 제출 정리

병렬 구현이 가능한 UI 작업도 데이터와 SaveGame 구조가 확정된 뒤 연결한다.

## 5. 제외 범위

- 3 Stage, Early/Mid/Boss Phase와 가중치 편성의 게임 콘텐츠 확장. 단, 사용자가 요청한 3×10 타일 기반 HUD 표시 상태는 포함한다.
- 다종 일반 Enemy, Elite, Boss 확장
- 복잡한 정형 무작위 배치와 Seed 재현
- 보상 상자 선택, 재추첨, 빌드 조합
- 화살·비행검·동료의 다중 빌드
- 업적, 다국어, 상세 통계, 고급 접근성
- 추가 맵, 결과 화면, 별도 보스 화면
- Windows 패키징
- 제출 후 리팩터링과 콘텐츠 추가

## 6. 최소 완료 조건

- 오프닝에서 게임과 상점으로 이동하고 다시 돌아올 수 있다.
- 오프닝에서 Save와 Load를 실행할 수 있다.
- 인게임에서 타일과 Enemy가 Spawn/Destroy 반복 없이 재사용된다.
- 투사체와 Enemy 충돌로 피해·처치·재화 획득이 발생한다.
- 상점에서 재화를 사용해 펫 구매·강화와 스탯 강화를 실행할 수 있다.
- 저장 후 프로그램을 종료하고 다시 실행해 재화, 펫, 스탯을 복원할 수 있다.
- Blueprint 컴파일 오류와 진행 불가 오류가 없다.
- UE 5.7.4에서 제출 프로젝트를 열어 오프닝 → 인게임 → 오프닝 → 상점 → 저장 → 재실행 → 불러오기 흐름을 실행할 수 있다.

## 7. 위험과 대응

| 위험 | 대응 |
|---|---|
| 마감 전 기능 과다 | 필수 카드 외 구현 금지, 콘텐츠는 기능 증명 최소치만 사용 |
| Enemy 반환 중 배열 변경 | 반환 대상 복사본을 순회한 뒤 원본 배열을 한 번만 Clear |
| SaveGame 누락 | 재화·펫·스탯 필드를 한 구조에서 저장하고 기본값을 제공 |
| 상점 중복 구매·재화 음수 | 보유 여부, 비용, 최대 레벨을 거래 전에 검사 |
| 화면 전환 후 데이터 초기화 | `GameInstance`에서 현재 데이터를 유지하고 SaveGame으로 영속화 |
| Collision 중복 처리 | 처치·보상 지급 Guard를 두고 Collision을 즉시 비활성화 |

## 8. 결정 로그

- 2026-08-29: 제출일을 다가오는 화요일인 2026-09-01로 해석했다. 정확한 제출 시간은 미정이다.
- 2026-08-29: 제출 후 개선하지 않는 조건에 따라 기존 MVP와 Stage 확장 계획을 폐기하고 필수 기능만 남겼다.
- 2026-08-29: 인벤토리는 펫 전용으로 사용하고 펫과 영구 스탯은 상점에서 재화를 사용해 구매·강화한다.
- 2026-08-29: 화면은 오프닝, 인게임, 상점 3개로 제한한다.
- 2026-08-29: 오프닝 화면에서 게임 시작, 상점 이동, Save, Load를 제공한다.
- 2026-08-29: `EnemyPlaced`와 Stage별 배치 Helper는 구현하지 않고 기존 Type 기반 Enemy FIFO를 최소 배치 흐름에 직접 연결한다.
- 2026-08-29: `BP_TileManager`의 타일 범위 Collision은 첫 번째로 생성한 `BP_BaseTile.RecycleCheck`의 Scaled BoxExtent와 `PoolSize - 1`로 Extent를 계산한다. 변경된 Manager Box Extent를 다음 실행의 입력으로 재사용하지 않으며 Collision 위치는 유지한다.
- 2026-08-30: `BP_TileManager` 초기 풀은 첫 타일의 `RecycleCheck.GetScaledBoxExtent`로 Manager Box를 한 번만 설정하고, 첫 위치 계산 이후 `NextSpawnPoint` 월드 위치를 다음 Spawn Transform으로 사용한다.
- 2026-08-31: 최신 사용자 지시를 우선하여 Normal은 `Normalxx_L/R`, Boss와 Elite는 각 지정 배치점을 사용하고 동일 타입의 사용 중인 배치점만 제외하는 설계를 안내한다. Bounds 보정 후 위치 변경에 대비해 Enemy에 사용한 배치점 참조를 저장하는 방식을 제안한다. 최초 고정 순회 제안은 후속 사용자 지시에 따라 아래의 2단계 무작위 선택으로 대체한다.
- 2026-08-31: 사용자 정정에 따라 Normal은 행(01/02)을 무작위 순서로 검사하고, L/R 중 한쪽이라도 점유된 행은 전체를 건너뛴다. 양쪽 모두 빈 행에서만 L/R을 균등 무작위 선택한다. 모든 행이 제외되면 빈 포인트가 남아 있어도 배치 실패다. 기존의 한쪽 빈자리 허용 해석은 폐기한다.
- 2026-08-31: Normal은 두 행 중 양쪽 모두 빈 행을 무작위 선택한 뒤 L/R을 무작위 선택한다. Boss는 Boss 전용의 유일한 포인트, Elite는 별도 Elite 전용의 유일한 포인트만 사용한다. 동일 타입이 이미 점유하면 요청을 실패 처리하며 대체 포인트를 찾지 않는다. 앞서 확정한 Boss/Elite Bounds 보정은 유지한다. 배치 용량은 Normal 2개, Boss 1개, Elite 1개이며 생성 수량을 강제하지 않는다.
- 2026-08-31: 런 중 강화와 홈 화면 영구 강화는 분리한다. 최종 스탯은 `BaseStats + PermanentBonusStats + RunBoundStats`이며 런 종료 시 `RunBoundStats`만 초기화한다.
- 2026-08-31: HUD의 Stage 표시는 사용자 요청에 따라 3개 Stage와 각 Stage 10개 타일 통과 수를 사용한다. 이는 HUD 진행 상태이며 Stage별 Enemy·보스 콘텐츠 확장을 의미하지 않는다.

## 9. 변경 로그

- 2026-08-29: 제출 범위를 오브젝트 풀링, 충돌, 펫 인벤토리, 영구 스탯, 상점, SaveGame과 3개 화면으로 축소했다.
- 2026-08-29: 기존 Stage·보스·다중 빌드·보상 선택·추가 콘텐츠 계획과 완료 후 개선 계획을 제거했다.
- 2026-08-29: Trello를 `SUBMIT-01`~`SUBMIT-06` 6개 카드와 완료 조건 35개로 재구성했다. 기존 확장 카드 8개와 빈 `프로젝트 안내`, `전체 구현 백로그` 목록은 복구 가능한 보관 상태로 전환했다.
- 2026-08-29: 사용자 지시에 따라 Windows 패키징을 제외하고 UE 5.7.4 프로젝트 폴더 자체를 제출 대상으로 변경했다.
- 2026-08-29: `Docs/BlueprintSnippets/TileManagerCollisionSizing.md`에 `PoolSize - 1`개 타일을 포함하는 Collision Extent와 Bounds 기반 타일 배치 계산 및 Blueprint 연결 순서를 기록했다. 실제 에셋 적용과 PIE 검증은 대기 상태다.
- 2026-08-30: `Docs/BlueprintSnippets/TileManagerBeginPlayRevision.md`에 `GetTileWorldLength`·Stage 계산 제거, 첫 타일 전용 Collision 초기화와 `NextSpawnPoint` 기반 연속 배치 수정안을 기록했다. 실제 `BP_TileManager` 에셋 적용은 Computer Use 승인 대기 상태이며 컴파일과 PIE는 사용자 검증으로 남겼다.
- 2026-08-31: `Docs/BlueprintSnippets/EnemyTypeSpawnPointSelection.md`에 첨부 함수의 실행선 미연결 상태, 후보 배열에서 동일 타입 점유 배치점 제거, Elite/Boss Bounds 보정과 풀 반환 처리를 기록했다. UE 5.7.4 설치와 로컬 Bounds API를 확인했다. 실제 Blueprint 수정·컴파일·PIE는 미수행이다. 다음 행동은 에디터의 Boss/Elite 배치점 참조와 타일 경계·회전, Enemy 최종 크기 적용 시점을 확인한 뒤 안내 흐름을 연결하고 정상·경계 경로를 검증하는 것이다.
- 2026-08-31: Bounds 보정 수식의 중앙·양쪽 경계·피벗 차이·동일 크기·초과 크기 사례 5건을 수치 검산해 통과했다. 전체 `git diff --check`는 기존 변경 uasset의 Git LFS 임시 파일 쓰기 권한 오류로 실패하여 문서 경로로 검사를 제한했다. 사용자 에셋과 Git 설정은 변경하지 않았다.

- 2026-08-31: Normal 선택 안내를 무작위 행 순회 → 한쪽이라도 점유 시 행 전체 제외 → 양쪽 모두 빈 행에서 L/R 추첨으로 정정했다. 문서 흐름에서 한쪽 점유와 모든 행 제외 시 실패 분기를 검토했다. 실제 에셋 변경·컴파일·PIE는 미수행이며 다음 행동은 이 순서를 그래프에 연결해 검증하는 것이다.

- 2026-08-31: 최신 요청의 타입별 배치 규칙과 Boss/Elite 단일 포인트 분기를 설계 문서에 반영했다. 문서 흐름을 검토했으며 실제 에셋 변경·컴파일·PIE는 미수행이다. 다음 행동은 실제 전용 포인트 참조를 연결하고 빈 행·행 점유·전용 포인트 점유·Bounds 초과 경로를 검증하는 것이다.

- 2026-08-31 (Git 1차 조치 정정): 외부 `.umap` 6개와 관련 `BuiltData.uasset` 3개를 최신 트리에서 제외했지만, BuiltData는 미푸시 조상 커밋 `82e1c25`에 남아 있어 LFS 업로드 대상에서 빠지지 않았다. 당시 최신 트리 확인만으로 제외가 충분하다고 보고한 판단을 정정한다.
- 2026-08-31 (Git 원인): 실제 `DESKTOP-CTA5QIP\user` 계정으로도 로컬 `.git/lfs/objects` 파일을 읽지 못했다. LFS 객체 4,414개에 비어 있는 DACL과 비활성화된 상속이 확인됐다. 실패 해시 `dc2a216...`는 필요한 `Content/Ingame.umap`이며, `76985ee...`와 `1a687205...`는 외부 맵의 day/night BuiltData였다. 앞선 권한 복구는 파일 단위에서 완료되지 않았으며 이번 오류의 직접 원인은 로컬 파일 접근 권한이다.
- 2026-08-31 (Git 복구): 기존 ACL을 `.git/codex-repair/lfs-20260831-142512/empty-file-acls.json`에 백업하고 빈 ACL 파일에만 소유자 user의 FullControl을 복구했다. 빈 ACL 잔여 0개, 오류 객체 3개의 SHA-256 일치를 확인했다. 미지원 LFS 잠금 API 경고는 해당 원격의 로컬 `locksverify=false` 설정으로 비활성화했다.
- 2026-08-31 (Git 이력): 미푸시 커밋 2개를 `35988d2`와 `5aa730b`로 재작성해 BuiltData 3개를 미푸시 이력에서 제거했다. 수정 전후 최신 tree, 실제 index 해시, 작업 상태가 같음을 검증했다. 원래 이력은 로컬 `refs/codex/backups/lfs-repair-20260831-142853`에 보존했다. 작업 중인 Blueprint와 미추적 파일은 변경하지 않았다.
- 2026-08-31 (Git 검증): `git lfs fsck --objects main` 통과, 업로드 후보의 제외 BuiltData 0건, 필요한 Ingame 맵 포함을 확인했다. 일반 `git push origin refs/heads/main:refs/heads/main`이 LFS 1410/1410 전송과 원격 `b7c2d4b -> 5aa730b` 갱신으로 성공했다. 강제 푸시는 사용하지 않았다. Blueprint 내용 변경이 없어 UE 컴파일·PIE는 이 정비의 검증 범위에 포함하지 않았다. 다음 행동은 기존 SUBMIT-01 기능 작업 재개이며, Git 정비만으로 SUBMIT-06 전체를 완료 처리하지 않는다.
- 2026-08-31: 사용자가 공유한 `BP_RunnerPlayer.UpdateStats` 그래프에서 `BaseStats`, `RunBoundStats`, `PermanentBonusStats`의 `StatusValue` Map을 `StatsList` 기준으로 합산해 `CurrentStats`에 대입하는 연결을 확인했다. `RunUpgrade` 호출, 표지판 충돌 Guard, Blueprint 컴파일과 PIE는 아직 확인하지 않았다.
- 2026-08-31: 사용자가 공유한 `WBP_PlayerHealthBar.UpdateHP` 그래프에서 HP 값을 직접 0~1로 Clamp하는 연결을 확인했다. 현재/최대 HP 비율을 전달하도록 수정이 필요하며, 체력바 표시를 완료 처리하지 않았다.
- 2026-08-31: Trello 원격 보드의 실제 상태를 재조회했다. 완료 목록의 `SUBMIT-01`은 유지하고, `SUBMIT-02` 및 새 `SUBMIT-01A`를 진행 중으로 기록했다. HUD·체력·영구 스탯·Stage 진행·일시정지의 누락 범위는 하위 카드로 추가했다.
- 2026-08-31: 범용 투사체 풀링 설계를 `Docs/BlueprintSnippets/GenericProjectilePoolingSystem.md`에 기록했다. 당시 기록한 `InitProjectilePool` 구현 완료 표기는 같은 날 사용자 정정으로 대체됐다. 현재 `InitProjectilePool`은 구현 중이며, `RequestProjectile`의 실제 구현·검증 상태도 재확인이 필요하다. 풀 반환·투사체 충돌·피해 전달·발사자 연결·UE 5.7.4 컴파일과 PIE는 미검증이다.
- 2026-08-31: 투사체 자동 반환 기준을 시간에서 발사 시작 위치 기준 최대 이동 거리로 변경했다. `ST_ProjectileRequest.MaxTravelDistance`와 활성 투사체 전용 Tick 거리 비교를 사용하며, 실제 Blueprint 반영·컴파일·PIE는 미검증이다.
- 2026-08-31: 사용자 정정에 따라 현재 구현 재개 지점을 `BP_ProjectileManager.InitProjectilePool`로 변경했다. 풀 생성·초기 비활성화·Available 배열 등록을 완료한 뒤 `RequestProjectile` 상태 확인, `BP_Projectile.ActivateProjectile`, `DeactivateProjectile`, `RequestReturn`, 최대 거리 확인, 충돌·피해 전달, Player·Enemy 발사 연결 순서로 진행한다. 실제 Blueprint 저장·컴파일·PIE는 미검증이다.

## 10. Trello 동기화

- 보드: https://trello.com/b/JWrQNIlv/runner-arrow-a-row-%EB%AA%A8%EC%9E%91-2026-09-03
- 열린 목록: `이번 스프린트`, `진행 중`, `검증`, `완료`
- 진행 중: `SUBMIT-02`, `SUBMIT-01A`
- 이번 스프린트: `SUBMIT-01B`, `SUBMIT-02A`, `SUBMIT-03`, `SUBMIT-03A`, `SUBMIT-04`, `SUBMIT-05`, `SUBMIT-05A`, `SUBMIT-05B`, `SUBMIT-06`
- 완료: `SUBMIT-01`, 기존 `NEXT-01`, `DONE-01`~`DONE-03`
- 2026-08-31: 카드 설명에 인수 조건과 확인 상태를 기록했다. 현재 보드의 제출 카드에는 체크리스트가 없으므로 과거 문서의 `35개 중 5개` 수치는 삭제한다.
- 추가 카드: `SUBMIT-01A` 표지판 단일 선택·`RunUpgrade`, `SUBMIT-01B` 3×10 Stage 진행, `SUBMIT-02A` 체력·월드 체력바, `SUBMIT-03A` 영구 스탯·SaveGame, `SUBMIT-05A` HUD, `SUBMIT-05B` Esc 일시정지.
- 보드 제목의 기존 날짜 `2026-09-03`은 유지한다.
