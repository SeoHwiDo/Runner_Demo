# Runner 기술 학습 기록

마지막 갱신: 2026-08-24 KST

## 1. 문서 목적

이 문서는 Runner 프로젝트를 구현하면서 확인한 Unreal Engine 동작 원리, 기술 검토 결과, 대안 비교와 결론을 누적해서 기록한다.

- 모든 내용은 별도 표기가 없으면 Unreal Engine 5.7.4를 기준으로 작성한다.
- 확인된 엔진 동작과 아직 검증하지 않은 설계 제안을 구분한다.
- 코드와 Blueprint를 실제로 적용한 경우 에셋 경로와 PIE 검증 결과를 함께 기록한다.
- 새로운 학습 내용은 기존 기록을 삭제하지 않고 날짜와 주제 단위로 추가한다.
- 프로젝트의 현재 구현 결정은 `CurrentWork.md`가 기준이며, 이 문서는 기술적 근거와 학습 내용을 보존한다.

## 2. 목차

1. 2026-08-24: 자동 러너의 좌우 이동과 전진 이동 구조 검토

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
