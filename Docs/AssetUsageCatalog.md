# Runner 제공 에셋 통합 활용 카탈로그

- 조사일: 2026-08-24 KST
- 대상: 4기 XLSX, 5기 XLSX, 8기 PDF/Markdown, `D:\경일애셋\언리얼6,7애셋` 전체
- 평가 기준: `활용도`는 일반 프로젝트에서의 재사용성, `Runner`는 현재 자동 러너 MVP에 대한 직접 적합도다.
- 등급: 상 = 바로 후보, 중 = 특정 콘셉트/추가 작업 시 유효, 하 = 현재 MVP와 장르·톤·기술 방향이 맞지 않음.

## 1. 결론

현재 Runner에는 **밝고 읽기 쉬운 스타일라이즈드 판타지** 조합이 가장 안전하다. 실제 보유 폴더에서 `Hyper-Casual Character Pack`을 기본 캐릭터 톤, `Platformer Deathrun` 계열을 도로·장애물, `Loot & Pickup VFX Pack`을 골드·강화 피드백, `Pspsps Cat`을 동료, `Fallen Imp`·`Demon Minion`을 일반 적, `Chasm Golem`·`Infernal Demon`을 보스로 쓰는 구성이 가장 짧은 시간에 일관된 수직 슬라이스를 만들기 좋다.

`Ancient Temple Ruins`, `Necropolis`, `Dieselpunk`, `Megastore`는 품질과 범위는 크지만 현재 러너에는 무겁고 사실적인 편이다. 배경 전체를 가져오기보다 도로 옆 실루엣·원경·소품 일부만 선별하는 편이 낫다. `BPProj`는 UE 4.27 프로젝트이므로 UE 5.7에서 복제본 업그레이드, 머티리얼·블루프린트·스켈레톤 재검증이 필요하다.

가장 먼저 시험할 조합:

1. 플레이어: `Hyper-Casual Character Pack` 또는 `HS05 Paladin`
2. 도로/장애물: `Platformer_1` + `Platformer_8_Underworld`
3. 일반 적: `ma144 Fallen Imp`, `Demon Minion`, Hyper-Casual 변형
4. 보스: `am007 Chasm Golem`, `Infernal Demon`
5. 동료: `Pspsps Cat`
6. 강화/골드: `Loot & Pickup VFX Pack`, `Fantasy Props`
7. 전투 피드백: `Laser FX with Blueprint` 일부 + 5기 `Starter FX`, `Slash and Hit VFX`, `Action-RPG Essentials VFX`
8. HUD: 4기 `HQUI Progress Bars`, `Nine Slice Magic`; 단, 실제 라이브러리 보유와 UE 5.7 호환 확인 후 사용

## 2. 조사 범위와 상태

| 출처 | 확인된 항목 | 상태 |
|---|---:|---|
| 4기 XLSX | 16개 | 구매 목록 문서만 확인; 로컬 실물 미확인 |
| 5기 XLSX | 47개 | 구매 목록 문서만 확인; 로컬 실물 미확인 |
| 6·7기 폴더 | 실제 Marketplace 팩 약 30개 | `Content` 실물 확인 |
| 8기 PDF | 20개 | 구매 신청 목록; 로컬 실물 미확인 |
| 8기 Markdown | 3개 Unreal 프로젝트 묶음 | `Unreal8_Assets` 5.4, `FlaresEffect` 5.7, `CursedMagicNiagara` 5.4로 기재 |

6·7기 폴더는 두 프로젝트로 구성된다.

| 프로젝트 | 엔진 | Content 파일 | Content 용량 | 판단 |
|---|---:|---:|---:|---|
| `Asset2026` | UE 5.4 | 9,942 | 약 25.31GB | UE 5.7 복제본에서 순방향 업그레이드 검증 필요 |
| `BPProj` | UE 4.27 | 1,061 | 약 1.45GB | 변환 위험이 더 큼; 원본 보존 후 선별 마이그레이션 |

`DerivedDataCache`, `Intermediate`, `Saved`, `Collections`, `Developers`, `__ExternalActors__`, 데모 `Maps`는 독립 에셋으로 세지 않았다. `StarterContent`도 Epic 기본 콘텐츠이므로 구매 에셋 목록에서 제외했다. `Platformer_1`과 `Platformer_8_Underworld`는 한 Marketplace 팩의 테마 모듈일 가능성이 높아 하나의 계열로 본다.

## 3. 6·7기 실제 보유 에셋

### 캐릭터·적·보스·동료

| 에셋 | 구성 근거 | 활용도 | 분위기 | Runner | 추천 활용·주의 |
|---|---|---|---|---|---|
| am007 Chasm Golem | 공격 3, 주문, 피격, 사망, 이동, BP | 상 | 스타일라이즈드 석재 판타지 | 상 | 중간 보스. 큰 실루엣과 명확한 공격 예고에 적합 |
| Astronaut Game Character | 색상 조절 머티리얼, 스켈레탈 메시 | 중 | 준사실적 SF·우주 | 하 | SF 테마일 때 플레이어/적. 현 판타지 조합과 톤 충돌 |
| Dark Knight | 남녀 세트, 다수 이동·전투 애니메이션 | 상 | 어둡고 사실적인 중세 | 중 | 엘리트/플레이어. 캐주얼 배경과 섞으면 비율·재질 불일치 |
| Demon Minion | 무기/비무기 공격·투척·사망, BP·파티클 | 상 | 코믹한 지옥 판타지 | 상 | 일반 적 1순위. 투척형 변형도 만들기 쉬움 |
| HS05 Paladin | 14개 기본 전투 애니메이션, 방패·무기 | 상 | 치비 영웅 판타지 | 상 | 플레이어 또는 방어형 엘리트. Runner의 밝은 톤과 잘 맞음 |
| HS07 Monk | 기본 전투·회피·피격·사망, 무기 | 상 | 치비 동양 판타지 | 중 | 근접 엘리트/보스. 자동 사격 주인공보다 적 NPC에 적합 |
| Hyper-Casual Character Pack | 스틱맨 애니메이션, 직업/코스튬 다수 | 상 | 밝은 로우폴리·하이퍼캐주얼 | 상 | 플레이어·일반 적 변형의 가장 빠른 기반 |
| Infernal Demon | 지상·비행·주문·날개 공격, BP·파티클 | 상 | 다크 판타지 악마 | 상 | 최종 보스 1순위. 비행 단계 전환에 활용 가능 |
| LH03 Mage | 기본 전투·회피·피격·사망, 지팡이 | 상 | 치비 마법 판타지 | 상 | 원거리 엘리트, 마법 보스, 동료 후보 |
| Little Astronaut | 3색 재질, 걷기·달리기·점프·부유 | 중 | 귀여운 SF | 중 | 별도 SF 스킨/동료에는 좋으나 판타지 본편과 분리 권장 |
| ma144 Fallen Imp | 공격 3, 주문, 비행, 피격·사망 | 상 | 귀여운 악마·판타지 | 상 | 비행 일반 적 1순위. 발사선과 충돌 판독이 쉬움 |
| Megaspikan | 다방향 공격, 포효, 유충 소환 세트 | 상 | 괴수·바이오 SF | 중 | 소환형 보스에 매우 좋지만 치비 캐릭터와 스타일 조정 필요 |
| MPMECH | 메카 스켈레탈 메시, 리그, 총기, BP | 중 | 밀리터리 SF | 하 | SF 보스/플레이어 전용. 현재 화살 판타지와 부적합 |
| Pspsps Cat | 전투·대시·채집·사족 이동 등 다수 애니메이션 | 상 | 귀엽고 유쾌한 치비 | 상 | 동료 빌드 1순위. 공격/강화 단계 표현에 유리 |
| Sphere Drone | 이동·회전·사격 애니메이션, 다색 재질 | 상 | 깔끔한 SF·로봇 | 중 | 원거리 동료 구현에는 좋음. 판타지에서는 마법 구체로 재질 수정 |
| The Thin Woman | 공격·비틀거림·전방위 이동·사망 | 중 | 사실적 호러 | 하 | 호러 보스 전용. Runner의 권장 캐주얼 톤과 충돌 |
| Spider Queen | 웹 투척·소환/시전·진입·공격·사망 | 상 | 어두운 동굴·호러 판타지 | 중 | 보스 기믹은 우수. UE4.27 변환과 재질 검증 필요 |

### 환경·소품·애니메이션·VFX

| 에셋 | 구성 근거 | 활용도 | 분위기 | Runner | 추천 활용·주의 |
|---|---|---|---|---|---|
| Ancient Temple Ruins | 모듈 건축, PCG, 수목·암석, 시네마틱 | 상 | 장엄한 고대 정글 유적 | 중 | 후반 바이옴 원경/측면 장식. 2.74GB라 선별 이관 필수 |
| Dieselpunk | 건물·정유시설·도시 소품 2,093파일 | 상 | 거친 산업·누아르·대체역사 | 하 | 별도 산업 테마에 적합. 10.8GB로 MVP 직접 사용 비추천 |
| Fantasy Props | 항아리·램프·마법병·상자류 22계열 | 상 | 다채로운 동화 판타지 | 상 | 강화 선택대, 파괴 오브젝트, 골드 컨테이너에 적합 |
| Lab | 원심분리기·현미경·유리기구·작업대 | 중 | 현대 실험실·클린 SF | 하 | 연구소 테마 전용. 현 판타지 MVP에는 우선순위 낮음 |
| Laser FX with Blueprint | 레이저·오브·Niagara·시네마틱 BP | 상 | 네온 SF·강한 전투 피드백 | 중 | 비행검 궤적/보스 빔으로 재색상. 과한 광량은 가독성 주의 |
| Loot & Pickup VFX Pack | 상호작용 BP, 스포너, 트레일, 다색 획득 VFX | 상 | 아케이드·보상 중심 | 상 | 골드, 강화 게이트, 보스 보상에 즉시 가치가 큼 |
| Low Poly Sci-Fi Corridor | 모듈 복도, 문·케이블 BP, 파티클 | 상 | 로우폴리 우주선·산업 SF | 하 | SF 러너 스킨에는 우수. 현 판타지 본편에서는 분리 |
| Megastore | 매장·상품·가구·캐릭터, 953파일 | 상 | 밝은 현대 상업 공간 | 하 | 캐주얼 마트 러너에는 좋지만 현재 범위와 톤 불일치 |
| Necropolis - Graveyard Kit | 묘지 환경, 오디오, 안개/VFX, 지형 | 상 | 고딕·음산·사실적 | 중 | 다크 판타지 바이옴 원경에 유효. 2.13GB, 성능·톤 위험 큼 |
| Platformer Deathrun 계열 | 회전·스플라인 이동 BP, 장애물, 언더월드 모듈 | 상 | 밝은 로우폴리 아케이드 | 상 | 도로·장애물·골드·상자 기반 1순위. UE4.27 변환 검증 필요 |
| Two Handed Sword Animations | 콤보·카운터·처형·돌진 등 957파일 | 상 | 소울라이크·중후한 근접전 | 중 | 비행검 소환 포즈/보스에 일부 활용. 자동 사격 러너에는 과잉 |
| Vendors and Customers Animations | 바텐더·점원·손님 상호작용 337파일 | 상 | 현대 생활·상점 시뮬레이션 | 하 | 영구 상점 NPC 연출 일부 외에는 불필요 |
| Horse Animset Pro | 보행·질주·수영·공격·피격 등 729파일 | 상 | 사실적 승마·중세 | 하 | 기마 러너 확장용. UE4.27 스켈레톤 변환 비용이 큼 |

## 4. 4기 구매 목록 16개

아래는 문서에 기재된 항목이며 `언리얼6,7애셋` 폴더에서 실물을 확인한 것은 `The Thinwoman`, `Horse Animset` 두 계열뿐이다.

| 에셋 | 유형 | 활용도 | 분위기 | Runner | 추천 활용·주의 |
|---|---|---|---|---|---|
| [Tung Tung Tung Sahur / Italian Brainrot](https://www.fab.com/listings/29661740-91d7-467e-a0f3-48dd42964fc1) | 캐릭터 | 하 | 밈·괴상·코미디 | 하 | 유행 의존성이 강해 본편 톤과 IP/밈 수명 주의 |
| [Survival Animations](https://www.fab.com/listings/8a14fa35-6d5a-4c85-8302-93d0c675fd97) | 애니메이션 | 중 | 생존·현실적 | 하 | 채집/부상 게임에 적합; 자동 러너에는 직접 가치 낮음 |
| [HQUI: Buttons V2](https://www.fab.com/listings/09a8914c-87c4-4b9a-8894-2f21d4f77a13) | UI | 상 | 중립, 군사·RPG까지 커스텀 | 상 | 메뉴·설정 버튼. 다양한 상태/스타일 클래스와 게임패드 메뉴에 유용 |
| [High School Environment](https://www.fab.com/listings/59adaf95-5c58-4ec4-bdbd-750efaefd61d) | 환경 | 중 | 현대 학교·청춘·호러 | 하 | 학교 호러/일상 게임용 |
| [The Thinwoman: Nightgown](https://www.fab.com/listings/e9a30002-7c7a-4652-bb19-72e6218e041c) | 캐릭터 | 중 | 호러·추격 | 하 | 실물 있음. 다크 보스 스킨 외에는 제외 권장 |
| [Ghost Hunting Essentials](https://www.fab.com/listings/48372739-fb3c-46c8-884c-08fe7686c238) | 소품 | 중 | 초자연 조사·호러 | 하 | 탐지기/카메라류 추정; Runner 장르와 무관 |
| [Horror Bride](https://www.fab.com/listings/34f020e8-add7-438d-b8ca-0af0292babf9) | 캐릭터 | 중 | 고딕 호러 | 하 | 호러 보스/NPC용 |
| [Grid-Based Builder](https://www.fab.com/listings/1da64f6d-cdc9-4dee-8fb2-7b67ea0165cf) | 게임시스템 | 상 | RTS·건설 | 하 | Blueprint 기반 그리드 건축. 현재 러너에는 시스템 과잉 |
| [Horse Animset](https://www.fab.com/listings/e0fc6520-8c9d-425d-9d25-3b6dc9ed56f8) | 캐릭터/애니메이션 | 상 | 승마·중세 | 하 | 실물 있음. 기마 확장 전까지 보류 |
| [Turn-Based Tactic Strategy Template](https://www.fab.com/listings/bd11e459-47bd-45b4-a7ce-af1a227bc5ac) | 템플릿 | 상 | 전술·보드게임 | 하 | 턴제 전술 시작점. Runner 구조와 직접 결합하지 않음 |
| [Quest System](https://www.fab.com/listings/26d8944a-f5b3-4373-a7cb-f36c870937b3) | 게임시스템 | 상 | RPG·스토리 | 하 | 100% Blueprint지만 MVP 제외 범위인 퀘스트 시스템 |
| [Battle Royale Island Pack](https://www.fab.com/listings/58b474ca-58d2-4dfc-82ae-a53ee10de372) | 환경 | 중 | 열대·배틀로얄 | 하 | 넓은 섬 환경은 선형 러너에 과대 |
| [HQUI: Progress Bars](https://www.fab.com/listings/64bec1c7-b1c2-465e-b0a0-c2ad6769f997) | UI | 상 | 커스텀 HUD | 상 | 체력·보스 HP·강화 진행 표시 1순위 |
| [Nine Slice Magic](https://www.fab.com/listings/1b044d1f-f0e1-446b-abd3-9d052a6a629e) | UI | 상 | 장르 중립 | 상 | 단일 머티리얼 9-slice 위젯. 다양한 해상도 UI 프레임에 유용 |
| [NPC Eyes Sight System](https://www.fab.com/listings/6b54716a-dd21-414d-b78f-384068de14b7) | 플러그인 | 상 | 잠입·호러 | 하 | C++ 백그라운드 스레드 기반. Blueprint-only 규칙상 승인 없이 사용 불가 |
| [Brr Brr Patapim](https://www.fab.com/listings/32e3cae6-c62b-4687-99ed-677b2ba567a2) | 캐릭터 | 하 | 밈·초현실 코미디 | 하 | 본편 미술 방향을 흐릴 가능성이 큼 |

## 5. 5기 구매 목록 47개

5기는 VFX 21개, 환경 8개, 머티리얼 5개, 오디오 4개, 애니메이션 4개, 기타 5개로 전투·스타일라이즈드 표현에 강하다.

### VFX·머티리얼

| 에셋 | 활용도 | 분위기 | Runner | 추천 활용·주의 |
|---|---|---|---|---|
| [Nature VFX with Vine Growth](https://www.fab.com/listings/a830d4a9-9428-43ed-9fe5-2a70c9bd3e0d) | 상 | 자연 마법·생명력 | 중 | 덩굴 강화/소환 연출 |
| [Anime Stylized VFX](https://www.fab.com/listings/d85dee78-9584-4b4b-a3b7-4b96de33c2b8) | 상 | 애니·과장 | 상 | 명중·필살기·강화 피드백 |
| [Stylized Rendering System](https://www.fab.com/listings/6129b04e-3859-4a2d-90b3-000958a563d2) | 상 | 셀셰이딩 | 중 | 전체 미술을 바꾸므로 프로토타입에서 먼저 호환성 확인 |
| [Blood Magic Skill VFX Pack](https://www.fab.com/listings/aeb1b647-5e64-4009-91c7-5651aeafed4b) | 상 | 어두운 혈마법 | 중 | 적/보스 공격; 밝은 본편에서는 색조 절제 |
| [NIAGARA Blood VFX Pack](https://www.fab.com/listings/d6cce54e-79ba-493d-b919-172e5dc8577e) | 상 | 고어·호러 | 하 | 연령등급과 가독성 위험. 비혈액 색으로 변형 시만 검토 |
| [Cel-Shader & Outline](https://www.fab.com/listings/115968e6-8e5b-474d-9c56-95d800bc3ad0) | 상 | 만화·외곽선 | 상 | 캐릭터/강화 선택 실루엣 강화 |
| [Stylized VFX v.02](https://www.fab.com/listings/0b16ec5e-0971-4ee0-a505-4981a50a88e5) | 상 | 화려한 판타지 | 상 | 화살/마법/보상 연출 |
| [Stylized VFX vol.3](https://www.fab.com/listings/68b9a9d3-374a-4f73-aefe-8fcdd2a20944) | 상 | 화려한 판타지 | 상 | 보스·강화 연출 |
| [Dissolution VFX Pack using Niagara](https://www.fab.com/listings/d00da02c-eb9b-4324-a279-0e0796801ab5) | 상 | 소멸·SF/마법 | 상 | 적 사망 시 시체 누적 방지와 시각 피드백 |
| [Starter FX Pack](https://www.fab.com/listings/4ff15fc7-8e65-4385-b67f-feaaa645680a) | 상 | 장르 중립 | 상 | MVP 기본 명중·폭발·버프 후보 |
| [Footsteps VFX Pack](https://www.fab.com/listings/2a855bf4-1abf-443e-b5ff-3c1e8975ab8a) | 중 | 지면 반응 | 중 | 자동 전진 속도감; 대량 생성 비용 측정 필요 |
| [Air Magic Spells Niagara FX](https://www.fab.com/listings/48499658-f1d4-4836-be70-1064e0d0e5a9) | 상 | 바람·청량 판타지 | 상 | 화살 관통/속도 강화와 잘 맞음 |
| [Stylized Water](https://www.fab.com/listings/f2bdf5fa-bf17-4460-ab45-a16d5959d866) | 상 | 밝은 카툰 | 중 | 강/해안 바이옴 측면 배경 |
| [Ambient FX in Unreal Engine](https://www.fab.com/listings/02333be9-ab7b-4468-b9b5-7c7abd9ded76) | 상 | 먼지·환경 분위기 | 중 | 화면 깊이 강화; 모바일급 과밀 효과는 피함 |
| [VFX Spark Pack](https://www.fab.com/listings/1bf775cb-39e6-4a14-b611-0c917340ba2d) | 상 | 금속·전기·충돌 | 상 | 검/화살 명중, 골드 반짝임 |
| [Surface Decals](https://www.fab.com/listings/e70174bb-4116-4561-879a-89c84f72639d) | 상 | 사실적 표면 | 중 | 도로 반복감 완화; 데칼 수 관리 |
| [ArchViz Glass](https://www.fab.com/listings/9f3cb67d-0e4e-4b1f-9884-1d900313861a) | 중 | 건축 시각화·현대 | 하 | Runner 핵심 표현과 무관 |
| [Cartoon Sky Shader](https://www.fab.com/listings/2bcc9fb8-fb7e-45ad-95b5-caa95679c408) | 상 | 밝은 카툰 | 상 | 권장 스타일의 하늘/시간대 변주 |
| [Comix VFX](https://www.fab.com/listings/1a1df993-fd30-4cd2-8f37-70746c16ad48) | 상 | 코믹북·의성어 | 중 | 하이퍼캐주얼 방향일 때 명중 강조 |
| [StylizedToonBeam01](https://www.fab.com/listings/86a433f2-9772-48f6-a04d-a48bd499b511) | 중 | 애니 레이저 | 중 | 보스 빔/비행검 에너지; 판타지 색상으로 조정 |
| [Action-RPG Essentials VFX Pack V2.0](https://www.fab.com/listings/8bebe1ca-0d93-4495-8bed-c07851d92fe1) | 상 | 액션 RPG | 상 | 전투 피드백 핵심 후보 |
| [Dynamic Impact and Collision Effects](https://www.fab.com/listings/d9a83197-8153-42cb-9dff-fca43c413b5d) | 상 | 물리 충돌·타격 | 상 | 투사체 명중/환경 충돌 |
| [Slash and Hit VFX](https://www.fab.com/listings/9d886fdb-8011-49e7-a10d-0d8897fe9dc2) | 상 | 근접 액션 | 상 | 비행검 빌드 1순위 |
| [Niagara Magical Slashes](https://www.fab.com/listings/19f2e633-c1be-45e0-a249-36fcad82cd31) | 상 | 마법 검격 | 상 | 비행검 강화 단계 표현 |
| [Muzzle Flashes Vol.1 Niagara](https://www.fab.com/listings/56ac6e20-b3d0-4bbb-8d19-53a2dded97d1) | 상 | 총기·현대 | 하 | 화살 게임에는 직접 부적합 |
| [Roar Spell VFX](https://www.fab.com/listings/de3c9ae3-ebfb-4702-8cc2-1c7c2608c0f4) | 중 | 괴수·충격파 | 상 | 보스 포효/광역 경고에 적합 |

### 환경·오디오·애니메이션·기타

| 에셋 | 유형 | 활용도 | 분위기 | Runner | 추천 활용·주의 |
|---|---|---|---|---|---|
| [Stylized Trees Asia Nature Environment](https://www.fab.com/listings/7a400781-3a27-4121-a8dc-0bc62c4e2949) | 환경 | 상 | 동양·카툰 자연 | 상 | 도로 측면 수목·원경 |
| [Sci-Fi Urban Pack](https://www.fab.com/listings/204aa460-f646-4f20-b551-bdc0e3471bef) | 환경 | 상 | 미래 도시 | 하 | SF 스킨 전용 |
| [ProInstance Tools Plugin](https://www.fab.com/listings/68d92f15-d5c0-4f9b-8c09-64f3065030d6) | 플러그인 | 상 | 장르 중립 | 중 | 반복 배치·ISM/HISM에 유용하나 새 플러그인 승인 필요 |
| [Modular Fantasy RPG Castle](https://www.fab.com/listings/a7168c64-ffb4-4130-a564-f24b78ca35c6) | 환경 | 상 | 중세 판타지 | 중 | 보스 구간 원경/성문. 전체 맵 이식은 과대 |
| [Action Game Sounds Pack](https://www.fab.com/listings/4950a0c3-ace9-4cce-86dc-ce551263b6ce) | 오디오 | 상 | 액션 | 상 | 명중·처치·보스 전환 사운드 |
| [Horror Hospital Pack](https://www.fab.com/listings/bfd3b555-395f-48d8-9cdd-94fb351a757f) | 환경 | 중 | 병원 호러 | 하 | 현재 톤과 무관 |
| [Atmospheric House](https://www.fab.com/listings/9b9bfddd-4988-44e0-a4a0-47fda6b7b81c) | 환경 | 중 | 사실적·음산한 주택 | 하 | 탐색 호러용 |
| [Horror Corridor](https://www.fab.com/listings/ef8de17e-3f55-4a8e-9edb-319676b5f88b) | 환경 | 중 | 폐쇄·호러 | 하 | 선형 구조는 맞지만 톤과 가독성 불일치 |
| [Abandoned School](https://www.fab.com/listings/aed2a439-1c05-4359-ab17-c07486516ec0) | 환경 | 중 | 폐교·호러 | 하 | 현재 판타지 러너와 부적합 |
| [Japanese School Library](https://www.fab.com/listings/15c6384b-5eb6-4df6-a72e-d2ba9c6d912c) | 환경 | 중 | 현대 일본 학교 | 하 | 별도 일상/호러 프로젝트용 |
| [Water Gun Emoji](https://www.fab.com/listings/81a85009-4ef8-45e3-9b83-d72491ce0f49) | 소품 | 하 | 장난감·밈 | 하 | 코미디 스킨 외 불필요 |
| [Toy Gun Water Gun](https://www.fab.com/listings/648839b5-2418-414a-9c32-0c783f828beb) | 소품 | 중 | 장난감·여름 | 하 | 판타지 무기와 불일치 |
| [SFX OF FPS, FOOT STEP](https://www.fab.com/listings/b230804d-f357-43d9-aa47-bc307284b63d) | 오디오 | 중 | FPS·현대 | 중 | 발소리만 선별 가능 |
| [Replicated Interaction Kit Vol 2](https://www.fab.com/listings/649907f7-4470-4717-b12a-8b411851c096) | 시스템 | 상 | 사실적 상호작용 | 하 | 12개 BP, 서랍/수납장 상호작용. Runner에는 불필요 |
| [Water Sounds](https://www.fab.com/listings/52285d25-3977-4b2f-84df-dea01736a055) | 오디오 | 상 | 자연·물 | 중 | 수변 바이옴 앰비언스 |
| [KAWAII ANIMATIONS 100](https://www.fab.com/listings/39a9f490-752e-4f33-86d4-2735d8c6a75b) | 애니메이션 | 상 | 귀엽고 코믹 | 상 | 하이퍼캐주얼 캐릭터 승리·선택·상점 연출 |
| [Muscle Cat](https://www.fab.com/listings/23032b25-28c2-49e5-9501-73ec2533caf3) | 캐릭터 | 중 | 유쾌한 밈 | 중 | 동료/비밀 보스 스킨. 톤 과장 주의 |
| [UI Sound Effect](https://www.fab.com/listings/2e562302-8463-4adb-a892-9519ae269cee) | 오디오 | 상 | 장르 중립 | 상 | 메뉴·강화 선택·상점 피드백 |
| [Sword and Shield Animation Pack](https://www.fab.com/listings/ac3b99c6-27d2-450e-98ca-a998f9e83442) | 애니메이션 | 상 | 중세 전사 | 중 | Paladin/적 보스용 |
| [Mage Animation Pack](https://www.fab.com/listings/8dba821a-b491-48b5-8e18-508e4f815b02) | 애니메이션 | 상 | 판타지 마법 | 상 | 동료/보스 시전 |
| [Archer Animations Pack](https://www.fab.com/listings/455e908c-fcfc-4094-be2c-8883bdc532ee) | 애니메이션 | 상 | 판타지 궁수 | 상 | 자동 사격 플레이어 1순위 |

## 6. 8기 구매 신청 20개

8기 Markdown의 줄바꿈 때문에 `Soulslike Fog Wall`과 `Modular Medieval Worlds`, `3D Fire and Explosions`와 `Realistic Gun VFX`, `Spatial Distortion`과 `Niagara VFX Bundle`이 한 줄처럼 보이지만 PDF에서는 각각 별도 행이다.

| 에셋 | 유형 | 활용도 | 분위기 | Runner | 추천 활용·주의 |
|---|---|---|---|---|---|
| [Element Projectile FX Pack](https://www.fab.com/listings/c7f7a506-38b7-461f-87ed-0bf0d3eb3601) | VFX | 상 | 원소 마법 | 상 | 화살 속성 강화의 핵심 후보 |
| [Wizard Spells Pack](https://www.fab.com/listings/db6d0c26-058b-4932-ac37-5e85d49f0519) | VFX | 상 | 마법사·판타지 | 상 | 보스/동료/강화 연출 |
| [Pro HUD Pack: Horror Skin](https://www.fab.com/listings/71d02886-4bcd-47af-a7bf-737458a09789) | UI | 중 | 고딕 호러 | 하 | 현재 밝은 톤과 불일치; 구조만 참고 |
| [RPG Sword Buffs VFX Pack](https://www.fab.com/listings/c831cf67-0964-4e3f-9126-75f5f8f29833) | VFX | 상 | 판타지 무기 강화 | 상 | 비행검 빌드 1순위 |
| [VDB Dust Effects Pack](https://www.fab.com/listings/a6c3ccaf-c9fc-46d5-bd96-e35dc14122b3) | VFX | 상 | 영화적 먼지·폭발 | 중 | 보스 등장/충돌. 볼륨 비용 측정 필요 |
| [Vanish Effect Pack](https://www.fab.com/listings/8d2ca8d7-fe82-40a8-aa9d-5b4840a14186) | VFX | 상 | 순간이동·마법 | 상 | 적 사망/보스 페이즈 전환 |
| [Soulslike Fog Wall VFX](https://www.fab.com/listings/c0ea3cb0-7ec5-403e-b16b-9027bfd38b19) | VFX | 상 | 소울라이크·금지 구역 | 상 | 보스 진입 게이트에 적합 |
| [Flares Effect](https://www.fab.com/listings/658ff94f-0051-40ef-8c7f-858c52b839c2) | VFX | 중 | 렌즈·에너지 | 중 | 보상/클리어 강조; UE 5.7 프로젝트로 기재 |
| [Modular Medieval Worlds](https://www.fab.com/listings/497c793c-7262-4a8a-9bb5-78a423217982) | 환경 | 상 | 중세 마을·수도원 | 중 | 도로 측면/보스 성문. 전체 이식은 범위 과대 |
| [Cursed Magic Niagara](https://www.fab.com/listings/83d31880-441c-424e-8d22-71d778cf2905) | VFX | 상 | 저주·다크 판타지 | 중 | 적/보스 공격; UE 5.4에서 업그레이드 필요 |
| [3D Fire and Explosions](https://www.fab.com/listings/98c8264f-5eeb-4d9e-a668-33eab83e3393) | VFX | 상 | 사실적 화염·폭발 | 중 | 보스/함정. 스타일라이즈드 톤과 혼합 주의 |
| [Realistic Gun VFX](https://www.fab.com/listings/21573c1a-e494-4194-9926-0c03f48a5563) | VFX | 상 | 현실적 총기 | 하 | 화살 판타지와 무관 |
| [Easy Fog](https://www.fab.com/listings/0a1c0c21-ff23-4fb7-9dc5-7beb6e16ce64) | VFX/환경 | 상 | 안개·몽환·호러 | 중 | 원경 분리와 바이옴 전환; 과도하면 적 판독 저하 |
| [Spatial Distortion VFX Pack](https://www.fab.com/listings/c589bc94-be76-4c01-a81b-d74e22bc5c9a) | VFX | 상 | 공간 왜곡·SF/마법 | 상 | 강화 선택, 보스 워프, 피격 충격 |
| [Niagara VFX Bundle](https://www.fab.com/listings/ebcdf5fb-077c-46ac-be75-1393a7c699b6) | VFX | 상 | 다목적·화려함 | 상 | 범용성이 높지만 필요한 효과만 선별 |
| [ARPGVFXPack04](https://www.fab.com/listings/e9c2e600-4df9-42bc-be1b-8f5f10e85ac7) | VFX | 상 | 액션 RPG | 상 | 보스/스킬 피드백 |
| [AOE and Spell Decal VFX with SFX](https://www.fab.com/listings/8908c188-5b4d-4131-94dd-701c1c200c52) | VFX/오디오 | 상 | 마법진·전투 경고 | 상 | 보스 공격 텔레그래프 1순위 |
| [ARPGVFXPack01](https://www.fab.com/listings/53a3e3c3-55da-4574-b8a5-84bb0774e456) | VFX | 상 | 액션 RPG | 상 | 기본 공격/스킬 변주 |
| [ARPGVFXPack03](https://www.fab.com/listings/4ea54c1a-a334-4115-89ea-957aa6d3d4b0) | VFX | 상 | 액션 RPG | 상 | 강화/보스 스킬 변주 |
| [18 Packs & 3300+ Sounds Mega Bundle](https://www.fab.com/listings/f96764ef-99ae-4331-bef8-e0e1a76db430) | 오디오 | 상 | 다목적 | 상 | 전투·UI·환경 소리 원천. 중복/음량 표준화 필요 |

PDF 금액에는 불일치가 있다. 행에 인쇄된 최종 가격 20개를 그대로 더하면 **₩797,280**인데 합계는 **₩794,140**이다. `Element Projectile FX Pack` 최종가를 ₩35,690이 아닌 원가 ₩32,540으로, `Cursed Magic Niagara`를 ₩59,180이 아닌 원가 ₩59,190으로 보면 합계와 정확히 일치한다. 구매 증빙 확인 시 원본 영수증과 대조해야 한다.

## 7. Runner 도입 우선순위

### Must 후보 - 이번 MVP에 직접 가치

- `Hyper-Casual Character Pack`, `Platformer Deathrun`, `Loot & Pickup VFX`, `Pspsps Cat`
- `Fallen Imp`, `Demon Minion`, `Chasm Golem`, `Infernal Demon`
- 5기 `Archer Animations`, `Starter FX`, `Action-RPG Essentials`, `Slash and Hit`, `UI Sound Effect`
- 8기 `Element Projectile`, `RPG Sword Buffs`, `AOE and Spell Decal`, `Mega Sound Bundle`
- 4기 `HQUI Progress Bars`, `Nine Slice Magic`

### Should 후보 - 톤을 맞추거나 성능 확인 후

- `HS05 Paladin`, `LH03 Mage`, `Fantasy Props`, `Cartoon Sky Shader`, `Stylized Trees`
- `Laser FX`, `Spatial Distortion`, `Vanish Effect`, `Soulslike Fog Wall`
- `Ancient Temple Ruins`와 `Necropolis`의 원경/소품 일부

### 보류

- 현대/호러 환경 전체, 대형 Dieselpunk/Megastore, 총기 VFX, 말 애니메이션, 턴제/건설/퀘스트 시스템
- C++ 기반 `NPC Eyes Sight System`: Blueprint-only 정책과 충돌
- `ProInstance Tools Plugin`: 유용하지만 새 플러그인 추가이므로 사용자 승인 전 사용 금지

## 8. 기술·라이선스 체크리스트

1. 원본 프로젝트를 직접 변환하지 말고 복제본에서 UE 5.7로 연다.
2. 필요한 폴더만 `Migrate`하고 데모 GameMode/Input/StarterContent는 가져오지 않는다.
3. 스켈레탈 메시는 Skeleton, Physics Asset, AnimBP, Root Motion, Retarget 결과를 각각 확인한다.
4. Niagara/VDB/머티리얼은 Shader Model, 플러그인 의존성, 투명도 오버드로, 동시 재생 수를 확인한다.
5. Marketplace BP는 프로젝트 규칙에 맞게 인터페이스/컴포넌트 경계로 감싸고 레벨 BP로 복사하지 않는다.
6. 문서의 구매 신청·목록과 로컬 파일 보유는 **사용권 증명과 동일하지 않다**. 실제 구매 계정, 조직/좌석 범위, Fab 라이선스 유형, KitBash3D 라이선스를 릴리스 전에 확인한다.
7. 서로 다른 팩의 사실적·치비·하이퍼캐주얼 캐릭터를 한 화면에 섞지 말고, 한 바이옴 안에서는 비율·텍셀 밀도·러프니스·외곽선 규칙을 통일한다.
8. 대형 환경 팩은 전체 이식하지 않고 필요한 메시·머티리얼·텍스처만 선별한다.

## 9. 확인된 한계

- 4·5·8기 항목은 문서와 일부 공식 Fab 설명을 기준으로 평가했으며, 로컬 `.uasset` 실물·정확한 엔진 버전·모든 의존성은 확인하지 못했다.
- 6·7기 평가는 실제 폴더명, 파일 수, 하위 폴더, 샘플 에셋명, 프로젝트 엔진 버전을 근거로 했다. Unreal Editor에서 각 에셋을 열어 컴파일/PIE한 것은 아니다.
- 8기 PDF는 3페이지 전체 텍스트와 표 셀을 추출해 대조했지만, 로컬 PDF의 앱 브라우저 렌더는 보안 정책상 열 수 없어 이미지 기반 레이아웃 검증은 하지 못했다.
- Marketplace 페이지의 가격·지원 엔진·라이선스는 바뀔 수 있으므로 실제 도입 직전에 다시 확인해야 한다.
