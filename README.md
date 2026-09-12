# Runner_Demo

Unreal Engine 5.7.4, Blueprint-only, Windows 64-bit 싱글 플레이 러너이다.
`Arrow a Row`의 핵심 재미(자동 전진, 자동 사격, 좌우 이동, 강화 선택)만 참고했고 원작의 이름, 코드, 모델, 텍스처, 음원, UI, 문구는 복제하지 않았다.

## 조작법

- A / D: 좌우 이동 (`AxisMapping MoveRight`)
- 자동 전진, 자동 발사
- Esc: 일시정지
- 오프닝: 게임 시작, 상점 이동, Save, Load
- 상점: 펫 구매·강화, 영구 스탯 강화

## 시연 영상

- https://youtu.be/HeW1g0u7sFc

## 소스코드 안내

이 프로젝트는 C++ 없이 Blueprint로만 작성됐다. 소스코드는 아래 `.uasset`, `.umap`이다.

- `Content/Players/`: `BP_RunnerPlayer`, 이동, 체력, `BaseStats`, `RunBoundStats`, `PermanentBonusStats`, `CurrentStats`, `UpdateStats`
- `Content/Managers/`: `BP_TileManager`, `BP_EnemySpawnManager`, `BP_ProjectileManager`
- `Content/Enemy/`, `Content/Projectile/`: 풀링, 충돌, 피해, 처치, 재화 지급
- `Content/GameFrameWork/`: `BP_RunnerGameInstance`, `GameMode`, `PlayerController`
- `Content/UI/`: `WBP_PlayerHealthBar` 등 메뉴, HUD, 상점
- `Content/Struct/`: `ST_PlayerStats`, `ST_EnemyDefinition`, 투사체 요청 구조체
- `Content/Levels/`, `Content/Ingame.umap`, `Content/Main.umap`: 제출용 레벨
- `Docs/BlueprintSnippets/`: 핵심 흐름의 텍스트 설계서. GitHub에서 코드 리뷰 대신 이 문서를 본다.

## 실행 방법

1. Unreal Engine 5.7.4를 설치한다.
2. 이 저장소를 클론한다.
3. `Runner_Demo.uproject`를 우클릭해 `Generate Visual Studio project files` 없이 바로 에디터로 연다.
4. `Main.umap` 또는 `Ingame.umap`을 열어 PIE로 실행한다.

패키지 빌드는 제출 범위에서 제외됐다. 플레이 영상과 빌드는 `GitHub Releases`에 별도로 올린다.

## 외부 에셋 고지 (중요)

이 Public 저장소에는 Fab 및 외부 마켓플레이스 에셋 원본을 포함하지 않는다.
Fab Standard License는 프로젝트에 포함한 형태의 배포와 private 저장소 공유만 허용하고, 에셋 단독 재배포를 금지한다.

제외 폴더 (`.gitignore` 처리, 캐릭터 포함 전체):

- `Content/Z_Ex_Assets/Fantastic_Village_Pack/` - 타일, 머티리얼, 표지판 계열
- `Content/Z_Ex_Assets/Kobo_Dungeon/` - 상자, 소품 계열
- `Content/Z_Ex_Assets/SKnight_modular/` - 스켈레톤 적 계열
- `Content/Z_Ex_Assets/ST_Characters_JC/` - 캐릭터 계열
- `Content/Collections/`, `Content/Developers/` - 로컬 에디터 데이터

이 상태로 맵을 열면 해당 메시·텍스처 참조가 깨져 보인다. 게임 규칙, 풀링, 전투, 저장 로직 검토에는 지장이 없다.

원본 Private 프로젝트 보유자가 로컬에서 복원하는 방법:

1. Fab에서 해당 팩을 본인 계정으로 구매·클레임한다.
2. Private 프로젝트에서 `Migrate`로 `Content/Z_Ex_Assets/`에 넣는다.
3. 에디터에서 `Main.umap`의 깨진 참조를 재연결한다.

## 이 브랜치에 대하여

- `public-portfolio` 브랜치는 `main`의 커밋 메시지와 날짜를 유지한 채 외부 에셋만 전 이력에서 제거한 것이다. 커밋 해시는 원본과 다르다.
- Private 원본(`main`)은 별도로 보관되며 이 브랜치와 머지하지 않는다.

## 라이선스

- 직접 작성한 Blueprint, 설정, 문서: 별도 명시가 없으면 포트폴리오 열람용이다. 재사용 조건은 작성자에게 문의한다.
- Fab·마켓플레이스 에셋: 각 판매자의 라이선스를 따르며 본 저장소에 포함되지 않는다.
- 패키지된 게임 실행 파일과 영상 배포는 Fab EULA상 프로젝트 형태 배포에 해당하며 원본 에셋 공개와 다르다.
