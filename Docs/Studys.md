# Runner 기술 학습 기록

마지막 갱신: 2026-08-25 KST

## 1. 문서 목적

이 문서는 Runner 프로젝트를 구현하면서 확인한 Unreal Engine 동작 원리, 기술 검토 결과, 대안 비교와 결론을 누적해서 기록한다.

- 모든 내용은 별도 표기가 없으면 Unreal Engine 5.7.4를 기준으로 작성한다.
- 확인된 엔진 동작과 아직 검증하지 않은 설계 제안을 구분한다.
- 코드와 Blueprint를 실제로 적용한 경우 에셋 경로와 PIE 검증 결과를 함께 기록한다.
- 새로운 학습 내용은 기존 기록을 삭제하지 않고 날짜와 주제 단위로 추가한다.
- 프로젝트의 현재 구현 결정은 `CurrentWork.md`가 기준이며, 이 문서는 기술적 근거와 학습 내용을 보존한다.

## 2. 목차

1. 2026-08-24: 자동 러너의 좌우 이동과 전진 이동 구조 검토
2. 2026-08-25: 고정 카메라·충돌 분리와 맵 타일 풀 초기 생성

---

# 2026-08-24 자동 러너의 좌우 이동과 전진 이동 구조 검토

## 1. 보고서 개요

### 1.1 목적

본 보고서는 플레이어의 자동 전진 속도와 좌우 이동 속도를 독립적으로 제어하기 위해 검토한 내용을 정리한다.

주요 검토 항목은 다음과 같다.

1. Enhanced Input을 이용한 좌우 이동
2. `AddMovementInput`의 입력 처리 방식
3. DeltaSeconds 적용 여부
4. 자동 전진과 좌우 이동의 결합 문제
5. `ForwardSpeed`와 `HorizontalSpeed`의 독립 제어 가능 여부
6. 액터 좌표 직접 이동 방식의 문제점
7. 커스텀 Movement Component 적용 가능성
8. 플레이어 고정 및 월드 스크롤 방식

### 1.2 개발 환경

| 항목 | 내용 |
|---|---|
| 엔진 | Unreal Engine 5.7.4 |
| 프로젝트 구현 방식 | Blueprint-only |
| 입력 시스템 | Enhanced Input |
| 플레이어 | `BP_RunnerPlayer` |
| 좌우 입력 액션 | `IA_HorizontalMove` |
| 입력 매핑 컨텍스트 | `IMC_Gameplay` |
| 목표 플랫폼 | Windows 64-bit |

## 2. Enhanced Input 기반 좌우 이동

### 2.1 Input Action 설정

`IA_HorizontalMove`의 Value Type은 `Axis1D`로 설정한다.

| 입력 | 출력값 | Modifier |
|---|---:|---|
| A | -1 | Negate |
| Left Arrow | -1 | Negate |
| D | 1 | 없음 |
| Right Arrow | 1 | 없음 |
| Gamepad Left Stick X | -1부터 1 | Dead Zone |

UE 5.7.4에서 Axis1D Input Action의 `Action Value`는 이미 축 값이다. 별도의 Axis Value 핀은 필요하지 않다. 타입을 명시적으로 분해해야 하는 경우 `Break Input Action Value`의 X 값을 사용할 수 있다.

```text
IA_HorizontalMove Triggered
    Action Value -> Add Movement Input Scale Value

Get Actor Right Vector
    -> Add Movement Input World Direction
```

### 2.2 Default Key Mappings와 Input Mode Filter Options

| 설정 | 역할 |
|---|---|
| Default Key Mappings | Input Action과 실제 키의 연결 |
| Input Mode Filter Options | 현재 입력 모드에서 IMC 전체를 사용할 수 있는지 결정 |

`Default Key Mappings`는 A, D, 방향키와 게임패드 축을 설정한다. `Input Mode Filter Options`는 Gameplay Tag 기반 입력 모드에서 해당 IMC의 사용 가능 여부를 결정하며 개별 키의 축 값이나 방향을 설정하지 않는다.

## 3. AddMovementInput 처리 과정

### 3.1 입력 전달

UE 5.7.4의 `APawn::AddMovementInput`은 액터의 위치를 즉시 변경하지 않는다. 입력 방향과 입력 크기를 곱한 값을 Pawn Movement Component에 전달한다.

기준 소스는 다음과 같다.

- `Engine/Source/Runtime/Engine/Private/Pawn.cpp`
- `Engine/Source/Runtime/Engine/Private/Components/PawnMovementComponent.cpp`
- `Engine/Source/Runtime/Engine/Private/Components/CharacterMovementComponent.cpp`

```cpp
void APawn::AddMovementInput(
    FVector WorldDirection,
    float ScaleValue,
    bool bForce)
{
    UPawnMovementComponent* MovementComponent = GetMovementComponent();

    if (MovementComponent)
    {
        MovementComponent->AddInputVector(
            WorldDirection * ScaleValue,
            bForce);
    }
}
```

### 3.2 입력 누적

같은 프레임에 여러 번 호출된 이동 입력은 `ControlInputVector` 하나에 누적된다.

```cpp
ControlInputVector += WorldAccel;
```

다음과 같이 전진 입력과 좌우 입력을 호출하면 두 입력이 하나의 벡터로 합쳐진다.

```text
AddMovementInput(ForwardVector, 1)
AddMovementInput(RightVector, HorizontalInput)
```

```text
ControlInputVector =
    ForwardVector
    + RightVector * HorizontalInput
```

### 3.3 입력 벡터 제한

CharacterMovement는 합산된 입력 벡터의 길이를 최대 1로 제한한다.

```cpp
FVector UCharacterMovementComponent::ScaleInputAcceleration(
    const FVector& InputAcceleration) const
{
    return GetMaxAcceleration()
        * InputAcceleration.GetClampedToMaxSize(1.0f);
}
```

전진 입력과 오른쪽 입력이 모두 1이면 정규화 이후 각 방향 성분은 약 0.707이 된다. `MaxWalkSpeed`가 600인 경우 각 방향의 속도 성분은 약 424이고 전체 속도는 600이다. 이 구조에서는 좌우 입력이 추가될 때 전진 속도 성분이 감소한다.

## 4. DeltaSeconds 적용 여부

`AddMovementInput` 자체는 DeltaSeconds를 사용하지 않는다. CharacterMovement가 입력을 소비한 후 가속도와 실제 이동 거리를 계산할 때 DeltaTime을 적용한다.

```cpp
Velocity += Acceleration * DeltaTime;
```

```cpp
Delta = Velocity * DeltaSeconds;
```

처리 순서는 다음과 같다.

```text
AddMovementInput 호출
입력 벡터 누적
입력 벡터 소비
가속도 계산
Velocity에 DeltaTime 적용
이동 거리에 DeltaTime 적용
충돌 및 바닥 이동 처리
```

따라서 CharacterMovement와 `AddMovementInput`을 사용할 때 Scale Value에 DeltaSeconds를 직접 곱하지 않는다.

## 5. 자동 전진과 축별 속도 분리

자동 전진에도 `AddMovementInput`을 사용할 수 있다.

```text
Event Tick
    Add Movement Input

World Direction = Get Actor Forward Vector
Scale Value = 1
```

그러나 좌우 입력을 함께 사용하면 두 입력이 하나의 벡터로 합쳐지고 하나의 `MaxWalkSpeed`에 의해 제한된다. 기본 CharacterMovement는 하나의 입력 벡터, 하나의 `MaxAcceleration`, 하나의 `MaxWalkSpeed`를 사용하므로 전진 속도와 좌우 이동 속도를 모든 순간에 완전히 독립적으로 제어하지 못한다.

## 6. ForwardSpeed와 HorizontalSpeed를 이용한 벡터 역산

### 6.1 목표 속도 벡터

```text
HorizontalVelocity =
    HorizontalSpeed * HorizontalInput
```

```text
DesiredVelocity =
    ForwardVector * ForwardSpeed
    + RightVector * HorizontalVelocity
```

전진 방향과 오른쪽 방향이 직각인 경우 전체 속도는 다음과 같다.

```text
CombinedSpeed =
    sqrt(
        ForwardSpeed * ForwardSpeed
        + HorizontalVelocity * HorizontalVelocity
    )
```

정규화된 이동 방향은 다음과 같다.

```text
MovementDirection =
    DesiredVelocity / CombinedSpeed
```

축별 계수는 다음과 같다.

```text
ForwardCoefficient =
    ForwardSpeed / CombinedSpeed

HorizontalCoefficient =
    HorizontalVelocity / CombinedSpeed
```

### 6.2 계산 예시

```text
ForwardSpeed = 600
HorizontalSpeed = 400
HorizontalInput = 1
```

```text
CombinedSpeed =
    sqrt(600 * 600 + 400 * 400)
    = 약 721.11
```

```text
ForwardCoefficient = 약 0.832
HorizontalCoefficient = 약 0.555
```

```text
MovementDirection =
    ForwardVector * 0.832
    + RightVector * 0.555
```

### 6.3 역산 방식의 한계

정규화된 벡터는 방향의 비율만 표현한다. 기본 CharacterMovement에서 실제 축 속도를 600과 400으로 만들려면 `MaxWalkSpeed`를 `CombinedSpeed`인 약 721.11로 설정해야 한다.

```text
FinalVelocity =
    MovementDirection * MaxWalkSpeed
```

`MaxWalkSpeed`를 사용하지 않고 방향 벡터만 `AddMovementInput`에 전달하면 원하는 축별 속도를 보장할 수 없다.

## 7. 축별 속도 독립을 위한 선택지

| 방식 | 축별 속도 독립 | CharacterMovement 기능 | 구현 비용 |
|---|---:|---:|---:|
| 기본 AddMovementInput | 불완전 | 유지 | 낮음 |
| 액터 좌표 직접 이동 | 가능 | 대부분 직접 처리 | 중간 |
| 커스텀 CharacterMovement | 가능 | 유지 가능 | 높음 |
| 플레이어 고정 및 월드 스크롤 | 가능 | 좌우 이동에 한해 유지 | 중간 |

## 8. 액터 좌표 직접 이동

### 8.1 기본 계산

```text
DesiredVelocity =
    ForwardVector * ForwardSpeed
    + RightVector * HorizontalInput * HorizontalSpeed
```

```text
DeltaLocation =
    DesiredVelocity * DeltaSeconds
```

계산 결과를 `AddActorWorldOffset`에 전달하면 전진 속도와 횡이동 속도를 수학적으로 분리할 수 있다.

```text
Add Actor World Offset

Delta Location = DeltaLocation
Sweep = true
```

### 8.2 CharacterMovement와의 충돌

CharacterMovement가 활성화된 상태에서 액터 좌표까지 직접 변경하면 두 시스템이 동시에 액터를 이동시킨다. 위치 떨림, 충돌 후 위치 보정, 실제 위치와 Velocity 불일치, Tick 순서에 따른 이동 결과 변화가 발생할 수 있다.

직접 이동을 사용하려면 CharacterMovement를 비활성화하거나 Movement Mode를 None으로 변경해야 한다. 이 경우 CharacterMovement가 제공하던 기능을 직접 구현해야 한다.

### 8.3 직접 처리해야 하는 기능

1. 바닥 탐색
2. 중력과 낙하
3. 착지
4. 경사면 이동
5. 계단 오르기
6. 절벽 이탈
7. 이동하는 발판
8. 벽 충돌 후 슬라이딩
9. 침투 상태 해소
10. 넉백과 외부 힘
11. Root Motion
12. 사망, 강화 선택, 일시정지 상태의 이동 차단
13. 네트워크 이동 예측

### 8.4 Sweep의 한계

`AddActorWorldOffset`의 Sweep은 CharacterMovement의 전체 충돌 처리와 동일하지 않다. 전진과 횡이동을 합친 상태에서 좌우 경계에 충돌하면 횡이동뿐 아니라 전진 이동까지 정지할 수 있다.

이를 방지하려면 횡위치를 숫자로 제한하거나 전진 이동과 횡이동을 별도로 처리해야 한다. 또는 트랙 좌표계에서 전진 거리와 횡오프셋을 분리해야 한다.

충돌로 인해 요청한 이동 거리와 실제 이동 거리가 달라진다면 실제 속도를 위치 차이로 계산해야 한다.

```text
ActualVelocity =
    (CurrentLocation - PreviousLocation)
    / DeltaSeconds
```

이 값은 애니메이션, 발자국 효과, 이동 사운드, 거리 기반 점수, 카메라 연출과 충돌 피해 계산에 사용될 수 있다.

## 9. 커스텀 CharacterMovement

커스텀 CharacterMovement는 전진 속도와 횡이동 속도를 독립시키면서 Capsule 충돌, 바닥 판정, 경사면, 계단, 실제 Velocity와 AnimBP의 `Get Velocity`를 유지하기 위한 방식이다.

```cpp
Velocity =
    ForwardVector * ForwardSpeed
    + RightVector
        * HorizontalInput
        * HorizontalSpeed;
```

횡이동 가속도를 적용하려면 현재 횡속도를 목표 횡속도로 별도 보간한다.

```cpp
const float TargetHorizontalSpeed =
    HorizontalInput * HorizontalSpeed;

const float NewHorizontalSpeed =
    FMath::FInterpConstantTo(
        CurrentHorizontalSpeed,
        TargetHorizontalSpeed,
        DeltaTime,
        HorizontalAcceleration);

Velocity =
    ForwardVector * ForwardSpeed
    + RightVector * NewHorizontalSpeed;
```

이 방식은 C++ 모듈, `UCharacterMovementComponent` 파생 클래스, C++ Character 기반 클래스와 `BP_RunnerPlayer` 부모 클래스 변경이 필요하다. 현재 프로젝트의 Blueprint-only 원칙과 일정에 비해 구현 및 검증 비용이 높다.

## 10. 플레이어 고정 및 월드 스크롤

### 10.1 개념

플레이어가 월드의 전진 방향으로 이동하지 않고 트랙과 게임 콘텐츠를 플레이어 방향으로 이동시키는 방식이다.

```text
플레이어
    좌우 이동만 담당

월드 콘텐츠
    자동 전진 연출 담당
```

```text
HorizontalSpeed
    플레이어 좌우 이동 속도

ForwardSpeed
    월드 스크롤 속도
```

플레이어에게는 좌우 입력만 적용되므로 대각선 입력 벡터가 생성되지 않는다.

```text
AddMovementInput(
    RightVector,
    HorizontalInput
)
```

이때 `CharacterMovement.MaxWalkSpeed`는 `HorizontalSpeed`와 같은 횡이동 전용 속도로 사용할 수 있다. 자동 전진 속도는 CharacterMovement와 분리해 `ForwardSpeed`로 관리한다.

### 10.2 플레이어 축 제한

전진 방향이 월드 X축인 경우 Plane Constraint로 플레이어의 X 위치를 고정할 수 있다.

```text
Constrain to Plane = true
Plane Constraint Normal = (1, 0, 0)
Snap to Plane at Start = true
```

### 10.3 월드 스크롤 계산

```text
ScrollDelta =
    ForwardVector
    * -ForwardSpeed
    * DeltaSeconds
```

진행 거리는 플레이어의 월드 위치가 아니라 별도 값으로 누적한다.

```text
RunDistance =
    RunDistance
    + ForwardSpeed * DeltaSeconds
```

웨이브, 점수와 보스 등장 조건도 `RunDistance`를 기준으로 처리한다.

## 11. 충돌 바닥과 시각적 트랙 분리

플레이어가 서 있는 충돌 바닥까지 이동시키면 CharacterMovement가 바닥을 이동하는 발판으로 인식할 수 있다. 플레이어가 뒤로 끌려가거나 위치 보정으로 떨릴 수 있으므로 고정 충돌 바닥과 이동하는 시각적 트랙을 분리한다.

```text
BP_RunSurface
    고정된 투명 바닥
    Collision 활성화

BP_TrackChunk
    이동하는 도로 메시
    Collision 비활성화
```

적, 장애물과 강화 오브젝트는 개별 Actor로 이동하고 필요한 충돌만 별도로 처리한다.

## 12. 트랙 청크와 관련 시스템

### 12.1 트랙 재활용

도로를 일정한 길이의 여러 청크로 나누고 뒤쪽 재활용 지점을 통과한 청크를 가장 앞쪽으로 이동시킨다.

```text
if ChunkLocationX <= RecycleLocationX
    ChunkLocationX =
        FrontChunkLocationX + ChunkLength
```

### 12.2 플레이어 애니메이션

플레이어의 실제 월드 전진 속도는 0이므로 전진 달리기 애니메이션에는 `ForwardSpeed`를 별도로 전달한다.

```text
VisualForwardSpeed = ForwardSpeed
```

실제 횡이동 속도는 CharacterMovement의 Velocity에서 계산할 수 있다.

```text
ActualLateralSpeed =
    DotProduct(
        CharacterVelocity,
        RightVector
    )
```

### 12.3 투사체 상대 속도

플레이어 투사체가 전진 방향으로 이동하고 적이 월드 스크롤로 플레이어 방향에 접근하면 두 오브젝트의 접근 속도는 다음과 같다.

```text
ClosingSpeed =
    ProjectileSpeed + ForwardSpeed
```

투사체 속도가 월드 좌표 기준인지 트랙 기준 상대 속도인지 프로젝트 전체에서 일관되게 정의해야 한다.

### 12.4 충돌과 상태 관리

빠르게 이동하는 적과 장애물은 낮은 프레임레이트에서 충돌을 건너뛸 수 있다. 중요한 충돌에는 Sweep, 이전 위치와 현재 위치 사이 Trace, 충분한 크기의 Overlap Collision과 낮은 프레임레이트 테스트가 필요하다.

플레이어 입력과 월드 스크롤은 별도 시스템이므로 게임 시작 전, 강화 선택, 일시정지, 플레이어 사망과 결과 화면 상태에서 월드 스크롤을 명시적으로 중지해야 한다.

## 13. 검토 결과

### 13.1 기본 AddMovementInput

장점은 Blueprint에서 구현할 수 있고 CharacterMovement의 충돌, 바닥 판정과 DeltaTime 처리를 유지한다는 점이다. 단점은 전진 입력과 횡이동 입력이 하나의 벡터로 합쳐지고 하나의 `MaxWalkSpeed`를 사용하므로 축별 속도를 완전히 독립시킬 수 없다는 점이다.

### 13.2 액터 좌표 직접 이동

장점은 Blueprint에서 전진 속도와 횡이동 속도를 완전히 분리할 수 있다는 점이다. 단점은 CharacterMovement가 제공하던 충돌, 바닥, 경사, 중력, 슬라이딩과 외부 힘 처리를 직접 구현해야 한다는 점이다.

### 13.3 커스텀 CharacterMovement

장점은 축별 속도를 분리하면서 CharacterMovement 기능과 실제 Velocity를 유지할 수 있다는 점이다. 단점은 C++ 도입과 회귀 검증 비용이 발생한다는 점이다.

### 13.4 월드 스크롤

장점은 플레이어의 좌우 이동과 자동 전진 시스템을 분리하고 커스텀 C++ 없이 전진 속도와 횡이동 속도를 독립시킬 수 있다는 점이다. 단점은 트랙, 적, 강화, 장애물과 투사체를 모두 월드 스크롤 좌표 규칙에 맞춰 설계해야 한다는 점이다.

## 14. 검토 결론

현재 프로젝트에 가장 적합한 후보는 플레이어 고정 및 월드 스크롤 방식이다.

```text
BP_RunnerPlayer
    전진 위치 고정
    CharacterMovement로 좌우 이동
    HorizontalSpeed 사용

BP_WorldScrollManager
    ForwardSpeed 관리
    RunDistance 관리
    스크롤 대상 등록 및 이동

BP_RunSurface
    고정된 투명 충돌 바닥

BP_TrackChunk
    이동하는 시각적 트랙
    뒤쪽 도달 시 앞쪽으로 재배치
```

속도 변수의 책임 후보는 다음과 같다.

| 변수 | 책임 |
|---|---|
| `HorizontalSpeed` | 플레이어 좌우 이동 속도 |
| `ForwardSpeed` | 월드 콘텐츠 스크롤 속도 |
| `RunDistance` | 실제 진행 거리 |
| `ProjectileSpeed` | 투사체의 기준 속도 |
| `VisualForwardSpeed` | 플레이어 달리기 애니메이션 속도 |

최종 이동 규칙 후보는 다음과 같다.

```text
플레이어 좌우 이동
    AddMovementInput 사용
    HorizontalSpeed 적용

자동 전진
    월드 스크롤 사용
    ForwardSpeed 적용

충돌 바닥
    고정 상태 유지

시각적 트랙과 콘텐츠
    플레이어 방향으로 이동
```

이 결론은 기술 검토 결과이며 아직 Blueprint 구현과 PIE 검증을 완료하지 않았다. 실제 프로젝트 결정으로 확정할 때는 `CurrentWork.md`의 결정 로그와 M1-01 완료 조건을 갱신하고 Trello를 동기화해야 한다.

---

# 2026-08-25 고정 카메라·충돌 분리와 맵 타일 풀 초기 생성

## 1. 최종 구현 상태

오늘 작업에서는 플레이어가 전진하지 않고 월드의 맵 타일이 이동하는 러너 구조를 구체화했다. 여러 범용화 방식을 검토했지만 MVP에서는 고정된 플레이어 이동 영역과 재사용되는 고정 크기 맵 타일을 분리하는 방식으로 정리했다.

| 영역 | 최종 상태 | 주요 에셋 |
|---|---|---|
| 플레이어 | 고정 발판 위에서 좌우로만 이동 | `/Game/Characters/BP_RunnerPlayer` |
| 카메라 | 플레이어에 부착하지 않고 맵을 바라보는 별도 카메라로 분리 | `/Game/Cameras/BP_InGameCamera` |
| 이동 제한 | 플레이어 전용 고정 바닥·좌우 감지 Collision 사용 | 플레이어 이동 영역 관련 Blueprint |
| 맵 타일 | 고정 크기의 기본 타일 액터와 머티리얼 사용 | `/Game/Levels/BP_BaseTile` |
| 타일 관리 | 시작 시 `PoolSize`만큼 타일을 생성하고 순차 배치한 뒤 참조 보관 | `/Game/Managers/TileManager` |
| 레벨 연결 | GameMode, 카메라, 플레이어와 TileManager 배치 반영 | `/Game/Ingame` |

`NEXT-01` 전체가 완료된 것은 아니다. 초기 생성과 순차 배치까지만 구현했으며, 타일의 `-X` 이동, 맵 범위 이탈 감지, 재배치와 장시간 PIE 검증은 다음 단계다.

## 2. 카메라와 플레이어 이동 책임 분리

### 2.1 고민

플레이어를 따라가는 카메라를 그대로 사용하면 플레이어가 고정돼 있어야 하는 월드 스크롤 연출과 카메라 책임이 섞인다. PlayerController를 추가했을 때 이동 로직까지 Controller로 옮겨야 하는지도 확인할 필요가 있었다.

### 2.2 학습

PlayerController를 추가하는 것만으로 Pawn의 이동 방식이 바뀌지는 않는다. Controller는 입력 매핑과 ViewTarget을 관리하고, 실제 좌우 이동은 기존처럼 `BP_RunnerPlayer`의 `AddMovementInput`에 둘 수 있다.

카메라 전환의 최소 흐름은 다음과 같다.

```text
BP_InGameCamera BeginPlay
    -> Get Player Controller
    -> Set View Target with Blend
        New View Target = Self
        Blend Time = 0
```

### 2.3 해결

카메라를 플레이어 컴포넌트에서 분리해 독립 Actor로 구성하고, PlayerController가 해당 카메라를 ViewTarget으로 사용하도록 했다. 그 결과 카메라의 월드 위치는 고정하면서 플레이어 좌우 이동 로직을 그대로 유지할 수 있었다.

## 3. 플레이어 이동 Collision과 시각적 타일 분리

### 3.1 고민

초기에는 이동하는 타일의 바닥 Collision이 플레이어를 지지하고, 좌우 Collision이 이동을 제한하는 구조를 검토했다. 그러나 타일 전체가 계속 `-X`로 이동하면 CharacterMovement가 바닥을 이동 발판으로 해석하거나 플레이어 위치를 보정할 수 있다.

### 3.2 학습

시각적 월드 스크롤과 플레이어의 물리적 지지는 같은 Actor가 담당할 필요가 없다.

```text
고정 이동 Collision
    플레이어 지지와 좌우 이동 제한 담당

이동 맵 타일
    시각적 도로와 타일별 콘텐츠 담당
    플레이어 지지 Collision과 분리
```

이렇게 분리하면 맵 타일을 `Sweep = false`로 이동하거나 순간 재배치해도 플레이어의 CharacterMovement 바닥 판정에는 영향을 주지 않는다.

### 3.3 해결

플레이어는 별도의 고정 이동용 Collision 위에서 움직이고, 맵 타일은 플레이어와 물리적으로 상호작용하지 않는 시각적 Actor로 유지했다. 타일에 필요한 플레이어 감지는 별도의 Overlap Collision으로만 처리한다.

## 4. Collision 채널에서 Block과 Overlap의 역할

### 4.1 고민

플레이어 전용 Object Channel을 추가하고, 바닥과 좌우 Collision이 Player만 Block하도록 만들면 입력 차단 불린을 없앨 수 있는지 검토했다.

### 4.2 학습

하나의 Collision Response는 동일한 Object Channel에 대해 동시에 Block과 Overlap이 될 수 없다.

| Collision 역할 | Collision Enabled | Player 응답 |
|---|---|---|
| 물리적 바닥 또는 벽 | Query and Physics | Block |
| 감지 센서 | Query Only | Overlap |

Player Object Type은 보통 캐릭터의 Capsule에 적용하고 Skeletal Mesh는 충돌 책임에서 제외하는 편이 명확하다. 감지 Collision에는 `Generate Overlap Events`가 필요하다.

### 4.3 해결

플레이어를 물리적으로 막는 고정 이동 영역과 상태를 감지하는 타일 Trigger를 분리했다. 타일 Trigger의 BeginOverlap과 EndOverlap은 `PlayerEnter` 상태만 관리하며 플레이어 지지나 이동 차단 Collision과 섞지 않는다.

## 5. `Accessed None`과 Actor 참조 수명

### 5.1 발생한 문제

다음 런타임 오류가 반복됐다.

```text
Accessed None trying to read property PlayerRef in BP_TileManager_C
Node: Set bBlockRight
Blueprint: BP_BaseTileCollision
```

### 5.2 원인

오류가 발생한 노드보다 중요한 정보는 `PlayerRef`가 `None`이었다는 사실이다. Overlap 이벤트가 정상 실행돼도, 별도로 캐시한 Player 참조가 생성 순서상 아직 지정되지 않았거나 잘못된 소유 Blueprint의 변수를 읽으면 접근 오류가 발생한다.

### 5.3 해결 원칙

Overlap 이벤트에서 이미 전달되는 `Other Actor`를 실제 이벤트의 근거로 사용한다.

```text
On Component Begin Overlap
    Other Actor
    -> Cast To BP_RunnerPlayer
    -> Cast 결과로 필요한 상태 변경
```

캐시 참조가 반드시 필요하다면 값을 읽기 전에 `Is Valid`를 사용하고, 누가 언제 설정하고 해제하는지 수명 규칙을 명확히 해야 한다. 이번 구조처럼 이벤트가 대상 Actor를 직접 전달하는 경우에는 불필요한 `PlayerRef`를 제거하는 편이 안전하다.

## 6. 단일 Boolean과 Overlap 순서 문제

인접한 두 타일의 Trigger가 경계에서 동시에 플레이어와 겹칠 수 있다. 이때 다음 순서가 발생할 수 있다.

```text
다음 타일 BeginOverlap -> PlayerEnter = true
이전 타일 EndOverlap   -> PlayerEnter = false
```

`PlayerEnter`가 각 타일 내부 상태라면 단일 Boolean으로 충분하다. 여러 타일의 감지 결과를 플레이어 또는 Manager의 전역 상태 하나로 합친다면 Boolean 대신 Overlap 개수를 사용해야 한다.

```text
BeginOverlap
    TileOverlapCount += 1

EndOverlap
    TileOverlapCount = Max(TileOverlapCount - 1, 0)

PlayerEnter
    TileOverlapCount > 0
```

핵심 학습은 이벤트의 최종 발생 순서에 의존하는 Boolean은 경계에서 잘못된 상태를 만들 수 있다는 점이다.

## 7. 공용 타일 크기와 동적 스케일링 검토

### 7.1 검토한 방식

`BP_TileCommon`에 공용 크기 변수를 두고, Static Mesh와 Box Collision의 원본 Bounds를 측정해 목표 크기에 맞게 Scale과 위치를 계산하는 방식을 검토했다.

```text
MeshScale = TargetSize / OriginalMeshSize
CollisionScale = TargetSize / OriginalCollisionSize
```

Collision의 크기뿐 아니라 중심 위치도 함께 계산해야 메시와 바닥, 좌우 벽이 정확히 맞는다.

### 7.2 폐기한 이유

현재 MVP의 타일은 고정된 형태이며 런타임에 서로 다른 원본 메시를 동일 규격으로 자동 보정할 필요가 없다. 범용 크기 상속, Construction Script 스케일링과 Collision 위치 보정까지 도입하면 현재 요구보다 복잡도가 커진다.

### 7.3 학습과 결론

범용화는 실제 변형 요구가 생겼을 때 도입해야 한다. 오늘은 동적 공용 크기 구조를 폐기하고 고정 크기 `BP_BaseTile`을 기준으로 삼았다. 이 결정으로 초기 생성 간격과 향후 재배치 조건을 단순한 `TileLength` 하나로 표현할 수 있게 됐다.

## 8. 초기 타일 풀 생성과 순차 배치

### 8.1 구현 흐름

`TileManager`의 BeginPlay에서 `PoolSize`만큼 타일을 생성한다.

```text
Event BeginPlay
    -> ForLoop (0 ~ PoolSize - 1)
        -> SpawnActor BP_BaseTile
            SpawnTransform.Location = SpawnPoint
        -> Add 생성 Actor to TilePool
        -> SpawnPoint 갱신
```

다음 타일의 배치점은 현재 배치점에 진행 방향과 타일 길이를 더해 계산한다.

```text
NextSpawnPoint =
    CurrentSpawnPoint
    + TileForwardVector * TileLength
```

모든 타일이 동일한 축 정렬 상태라면 월드 `+X * TileLength`를 더해도 되지만, 타일 회전을 허용할 경우 `GetActorForwardVector` 또는 타일의 `NextTilePoint`를 사용하는 방식이 안전하다.

### 8.2 학습

풀링의 첫 단계는 Spawn과 Destroy를 반복하지 않도록 필요한 Actor를 한 번 생성하고 참조를 보관하는 것이다. 현재 `TilePool` 배열은 이후 이동·회수·초기화 대상의 소유권을 Manager에 모으는 역할을 한다.

초기 생성만 구현했다고 풀링 전체가 끝난 것은 아니다. 반환 시 상태 초기화, 재배치, 재시작 시 배열 초기화와 Actor 수 유지 검증까지 완료해야 실제 풀링으로 볼 수 있다.

## 9. 맵 범위 Collision을 이용한 재배치 설계

이 절은 다음 구현을 위한 확정 설계이며 아직 Blueprint와 PIE 검증을 완료하지 않았다.

### 9.1 전체 맵 범위 방식

맵의 유효 영역과 정확히 같은 Query Only Box Collision을 두고, 타일의 전용 Bounds Collision이 완전히 빠져나갈 때 `EndOverlap`으로 회수한다.

```text
MapBounds OnComponentEndOverlap
    -> Other Actor Cast To BP_BaseTile
    -> 빠져나간 타일을 앞쪽 SpawnPoint로 이동
```

`EndOverlap`은 시각적 Mesh가 아니라 실제로 겹치던 Collision 형상이 완전히 분리됐을 때 발생한다. 따라서 타일의 재배치 감지용 Bounds가 시각적 타일 길이와 일치해야 한다.

### 9.2 고정 SpawnPoint가 성립하는 조건

다음 조건이 모두 맞으면 고정된 앞쪽 SpawnPoint로 재배치해도 타일이 순환한다.

1. 맵 범위 Collision의 진행축 길이가 타일 한 개 길이와 정확히 같다.
2. 두 타일이 같은 길이이고 처음부터 빈틈없이 붙어 있다.
3. 모든 타일이 같은 속도로 같은 축을 따라 이동한다.
4. 첫 타일이 완전히 나갈 때 다음 타일이 정확히 맵 범위를 채운다.
5. 재배치 지점은 맵 범위의 앞쪽 끝에 고정돼 있다.

### 9.3 프레임 단위 초과 이동

수학적으로 경계가 일치하더라도 실제 게임은 프레임 단위로 위치를 갱신한다. 한 프레임에 이동하는 거리가 `ScrollSpeed * DeltaSeconds`이므로 `EndOverlap`이 발생했을 때 타일이 경계를 몇 유닛 지나쳤을 수 있다. 고정 SpawnPoint에 정확히 Snap하면 이 초과 거리만큼 작은 틈이나 겹침이 생길 수 있다.

가장 안전한 재배치 기준은 현재 맨 앞 타일의 실제 위치다.

```text
RecycleLocation =
    CurrentFrontTileLocation
    + FrontTileForwardVector * TileLength
```

또는 각 타일 앞에 `NextTilePoint` Scene Component를 두고 그 World Transform을 사용한다. 고정 SpawnPoint 방식은 PIE에서 틈이 없다는 것이 확인될 때 유지하고, 틈이 보이면 현재 앞 타일 기준 방식으로 전환한다.

## 10. Tick과 타일 이동 책임

타일 이동은 매 프레임 위치 갱신이 필요한 연속 동작이므로 Tick 사용이 허용되는 영역이다. 다만 Manager가 Tick마다 `TilePool` 전체를 순회하면 타일 수가 늘어날수록 비용이 증가한다.

현재 후보는 각 타일이 동일한 `ScrollSpeed`로 자신을 이동하고 Manager는 생성·회수·배치만 책임지는 방식이다. 다른 후보는 작은 고정 풀을 Manager가 한 번에 이동시켜 Tick 순서를 통제하는 방식이다. 최종 선택 시 다음을 함께 확인해야 한다.

- 타일별 Tick 순서에 따라 경계 틈이 달라지는가
- 모든 타일에 동일한 `ScrollSpeed`와 Pause 상태가 적용되는가
- Tick 안에서 `Get All Actors Of Class`나 반복 Cast를 사용하지 않는가
- 풀 크기가 커졌을 때 전체 배열 순회 비용이 문제가 되는가

## 11. Cascade AnimTrail 자동 변환의 한계

오늘 발생한 파티클 변환 오류는 AnimNotify 시간 설정 문제가 아니라 Cascade 자동 변환기가 AnimTrail Emitter를 지원하지 않는 문제였다.

```text
CascadeParticleSizeConverter does not support emitters with renderer type
ParticleModuleTypeDataAnimTrail_0
```

즉 `ParticleModuleSize`를 변환하는 과정에서 Emitter의 Renderer Type이 AnimTrail이어서 변환이 중단됐다. 해결 선택지는 다음과 같다.

1. 원본 Cascade Trail을 그대로 사용한다.
2. Niagara Ribbon 기반 Sword Trail을 새로 구성한다.
3. AnimNotify State는 Trail의 시작·종료 신호만 담당하고 실제 이펙트 구현을 교체한다.

자동 변환 결과가 생성됐다는 사실만으로 재생 가능하다고 판단하면 안 된다. 변환 로그에서 미지원 모듈과 Renderer Type을 확인하고, 변환이 중단된 Emitter는 수동 재구성해야 한다.

## 12. 검증 결과와 남은 과제

### 12.1 확인한 결과

UE 5.7.4 `CompileAllBlueprints` 결과는 다음과 같다.

```text
Blueprint 오류: 0
Blueprint 경고: 0
로드 실패: 0
```

명령 전체 종료 코드는 샌드박스에서 Zen과 기본 DDC 경로를 쓸 수 없어 1이었다. `-DDC-ForceMemoryCache`를 사용해 Blueprint 컴파일 자체는 완료했으므로 Blueprint 실패와 도구 실행 환경의 캐시 오류를 구분해야 한다.

### 12.2 아직 확인하지 않은 항목

- PIE에서 초기 타일 간격 확인
- 타일 전체의 `-X` 지속 이동
- 맵 범위 `EndOverlap` 회수
- 재배치 순간의 틈과 중첩
- 장시간 실행 시 타일 Actor 수 유지
- 재시작과 레벨 재진입 시 Pool과 SpawnPoint 초기화

## 13. 오늘의 핵심 결론

1. 플레이어 지지 Collision과 이동하는 시각적 타일을 분리하면 CharacterMovement와 월드 스크롤의 충돌을 줄일 수 있다.
2. Overlap 대상이 이벤트 핀으로 전달될 때는 수명이 불명확한 전역 `PlayerRef`보다 `Other Actor`를 사용하는 편이 안전하다.
3. 물리적 제한은 Block, 상태 감지는 Overlap으로 역할을 분리해야 한다.
4. 고정 규격만 필요한 단계에서는 공용 동적 스케일링보다 고정 `TileLength`가 구현과 검증에 유리하다.
5. 풀링은 초기 생성뿐 아니라 회수, 상태 초기화, 재배치와 Actor 수 검증까지 포함한다.
6. Collision 경계가 수학적으로 맞아도 프레임 초과 이동 때문에 틈이 생길 수 있으므로 실제 앞 타일 Transform을 기준으로 재배치하는 대안을 준비해야 한다.
7. Blueprint 컴파일 성공 여부와 DDC/Zen 같은 실행 환경 오류는 별도로 판정해야 한다.

## 14. Enemy 오브젝트 풀의 생명주기와 책임 분리

### 14.1 풀 소유권과 Actor 내부 상태는 다른 책임이다

처음에는 `ActivateFromPool`과 `DeactivateFromPool`이 Actor를 FIFO 배열에서 꺼내고 다시 넣는 작업까지 담당하는 구조를 생각할 수 있다. 하지만 Actor가 자신의 컨테이너까지 수정하면 중복 반환, 잘못된 MonsterID 대기열 삽입, 활성 Actor 강제 회수 같은 오류를 한곳에서 통제하기 어렵다.

따라서 책임을 다음과 같이 분리했다.

```text
BP_EnemySpawnManager
    - MonsterID별 FIFO 대기열 소유
    - AcquireEnemyByID / ReturnEnemy
    - 중복 반환과 풀 고갈 검사

BP_EnemyBase
    - ActivateFromPool
    - DeactivateFromPool
    - 자신의 HP, Collision, Timer, Tick, 표시 상태 초기화
```

Manager는 “어떤 Actor를 빌리고 돌려받는가”를 결정하고, Enemy는 “빌려진 뒤 어떤 상태가 되어야 하는가”만 처리한다. 이 경계를 지키면 반환 경로가 늘어나도 FIFO 컨테이너 변경은 `ReturnEnemy` 한곳에서만 일어난다.

### 14.2 사망과 풀 반환은 같은 사건이 아니다

이번 게임에서는 Enemy가 사망한 뒤 보상 상자로 남아 플레이어의 Overlap을 기다린다. 따라서 사망 시 즉시 `DeactivateFromPool`을 호출하면 보상 상자도 함께 사라진다.

상태 흐름은 다음과 같이 분리해야 한다.

```text
InPool
    -> AcquireEnemyByID
ActiveEnemy
    -> Death
RewardBox
    -> Player Overlap 또는 Tile 회수
InPool
```

`bIsAlive`와 `bInPool` 두 Boolean만으로 표현하면 `bIsAlive=false`, `bInPool=false`가 사망 연출 중인지 RewardBox인지 구분되지 않는다. `E_EnemyPoolState`의 `InPool`, `ActiveEnemy`, `RewardBox`를 단일 상태 기준으로 사용하고 Boolean은 필요한 경우 파생 값이나 호환 값으로만 유지하는 편이 안전하다.

보상 종류·수량·UI·저장은 아직 결정하지 않았으므로 풀링 구현에는 보상 지급 요청 Event 또는 Interface 호출 지점만 두고 실제 내용은 별도 이슈로 분리했다.

### 14.3 활성화는 Collision을 마지막에, 비활성화는 처음에 처리한다

풀에서 꺼낸 Actor는 Transform과 HP를 초기화하기 전에 Collision이 켜지면 이전 위치나 잘못된 상태에서 Overlap을 발생시킬 수 있다. 반대로 반환 중인 Actor는 다른 정리보다 Collision을 먼저 꺼야 중복 피격과 보상 중복 획득을 막을 수 있다.

```text
ActivateFromPool
    Collision Off 유지
    -> Timer와 런타임 참조 초기화
    -> HP와 상태 초기화
    -> RewardBox 표시 해제
    -> Spawn Transform 적용
    -> Enemy 표시와 Tick 활성화
    -> Enemy Collision On

DeactivateFromPool
    Enemy/Reward Collision Off
    -> Timer, AI, 이동, VFX 정지
    -> 표시 해제
    -> OwningTile, Target, 런타임 상태 초기화
    -> Tick Off
```

`PoolOwner`, `MonsterID`처럼 Actor의 풀 정체성을 나타내는 값은 반환 때 지우지 않는다. `OwningTile`, Target, 상태 이상처럼 한 번의 대여에서만 유효한 값만 초기화한다.

### 14.4 Blueprint FIFO는 Array의 앞 제거와 뒤 추가로 표현한다

먼저 반환된 Actor를 먼저 재사용하려면 각 `MonsterID`마다 별도의 대기열이 필요하다.

```text
Acquire: AvailableEnemies[0] 사용 후 Remove Index 0
Return:  AvailableEnemies 끝에 Add
```

Blueprint의 `Set`은 중복 검사에는 적합하지만 순서가 없으므로 FIFO 대기열로 사용할 수 없다. `ActiveEnemies Set`은 활성 여부와 중복 반환 검사에 사용하고, 실제 대여 순서는 `AvailableEnemies Array`가 담당한다.

또한 Blueprint Map에서 구조체 값을 가져오면 복사본을 수정하게 되는 경우가 있으므로 다음 구조를 선택했다.

```text
PoolIndexByMonsterID : Map<Name, Integer>  // 조회 전용
MonsterPools         : ST_EnemyPool Array // 실제 변경 대상
```

Map에서 Pool Index를 찾고 Array의 구조체를 수정한 뒤 `Set Array Elem`으로 다시 기록하면 중첩 배열 변경이 원본에 반영되지 않는 문제를 피할 수 있다.

### 14.5 타일 회수는 소유권으로 처리한다

MapManager가 화면 밖 Tile을 맨 앞으로 옮길 때 Enemy가 살아 있거나 RewardBox 상태일 수 있다. 이때 월드의 모든 Enemy를 검색하면 타일마다 `Get All Actors Of Class`와 전체 배열 탐색이 반복된다.

그래서 Enemy에는 `OwningTile`, Tile에는 `TileEnemies` Array를 두고 양방향으로 소유권을 기록한다.

```text
MapManager: Tile 이탈 감지
    -> EnemySpawnManager.ReturnEnemiesOwnedByTile(Tile)
    -> TileEnemies 초기화
    -> Tile을 맨 앞으로 이동
    -> EnemySpawnManager.PopulateTile(Tile, Stage/Phase/Seed)
```

반환 시 `Enemy.OwningTile == 요청 Tile`인지 다시 검사하면, 재사용된 Enemy를 이전 Tile의 오래된 참조가 잘못 반환하는 문제도 막을 수 있다.

### 14.6 원본 데이터와 런타임 상태를 분리한다

`ST_ActorBaseStatus`와 `ST_EnemyDefinition`은 카탈로그의 원본 설정이다. 전투 중 `CurrentHP`나 생존 상태를 이 원본 구조체에 직접 기록하면 다음 대여 또는 같은 정의를 사용하는 다른 Actor까지 오염될 수 있다.

따라서 카탈로그 값은 읽기 전용 원본으로 두고, `ActivateFromPool`에서 최대 체력과 필요한 값을 런타임 변수로 복사한다. 매 대여 시 같은 초기화 함수를 거치게 하면 새로 Spawn한 Actor와 재사용 Actor의 시작 상태 차이도 줄어든다.

### 14.7 Unreal 바이너리 에셋은 직접 참조와 재귀 의존성을 함께 커밋해야 한다

`BP_Enemy_Skel`만 커밋해도 로컬에서는 이미 설치된 예제 에셋 때문에 정상으로 보일 수 있다. 하지만 새 Checkout에서는 Skeletal Mesh, Skeleton, Physics Asset, Animation, Material Instance, Material Function과 Texture가 빠져 로드 오류가 발생한다.

이번에는 `BP_Enemy_Skel`의 직접 참조를 확인하고 `Skeleton_Knight_01`과 공통 `Skeleton` 재질 폴더를 별도 LFS 커밋으로 분리했다. Enemy 발판 Material이 참조하는 Fantastic Village의 돌 Texture도 전체 팩 대신 직접 의존하는 2개만 포함했다.

기능 에셋과 대용량 예제 원본을 별도 커밋으로 나누면 다음 이점이 있다.

- 기능 변경과 외부 에셋 추가의 목적을 Git 기록에서 구분할 수 있다.
- 에셋 라이선스 또는 용량 문제가 생겼을 때 예제 커밋만 독립적으로 검토할 수 있다.
- LFS 업로드 실패 시 어느 커밋의 대용량 객체가 원인인지 찾기 쉽다.

### 14.8 아직 검증이 필요한 항목

- `ActivateFromPool → DeactivateFromPool → ActivateFromPool` 재사용 PIE
- 반환 직후 Enemy와 Reward Collision이 모두 꺼지는지 확인
- 이전 HP, Target, Timer, RewardBox 표시가 재대여 Actor에 남지 않는지 확인
- RewardBox Player Overlap의 중복 지급·중복 반환 방지
- Tile 회수 시 ActiveEnemy와 RewardBox가 모두 반환되는지 확인
- `BPDA_EnemyCatalogBase.EnemyDefinitions`의 `Unknown structure` 참조 복구
- `BP_Enemy_Skel`의 `EquipSocket_L`과 Skeletal Mesh 설정 경고 제거
