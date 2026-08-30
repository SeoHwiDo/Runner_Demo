# 강화 표지판 활성화·비활성화 Blueprint 작성

기준: Unreal Engine 5.7.4, Blueprint-only  
대상: `BP_BaseTile`, `BP_LevelManager`  
목적: 타일 초기화·재사용 때 `LeftSign`, `RightSign`을 숨기고 Collision을 끈다. Enemy 배치 요청이 있는 타일에서만 두 표지판을 표시하고 Collision을 켠다.

## 1. 전제

`BP_BaseTile` 컴포넌트 트리는 아래와 같다.

```text
LeftSign  : Static Mesh Component
├─ LeftSignText
├─ LeftMount
└─ LeftStatus

RightSign : Static Mesh Component
├─ RightSignText
├─ RightMount
└─ RightStatus
```

`LeftSign`, `RightSign`은 부모 Component다. 표시 전환은 부모에만 적용하고 `Propagate to Children`을 `true`로 둔다. 자식 Component에 같은 노드를 반복해서 연결하지 않는다.

## 2. `BP_BaseTile.SetSignsActive` 함수 생성

### 2.1 함수와 입력

`BP_BaseTile`의 **함수** 목록에서 `+`를 눌러 아래 함수를 만든다.

```text
Function Name : SetSignsActive
Input
  IsActive : Boolean
```

`IsActive = false`는 표지판 비활성화, `IsActive = true`는 표지판 활성화다.

### 2.2 필요한 노드

함수 그래프 빈 공간에 아래 노드를 만든다.

```text
Get LeftSign
Get RightSign

Set Visibility                 // 2개
Set Collision Enabled          // 2개
Select (Collision Enabled)     // 1개
```

`Set Visibility`는 `LeftSign`, `RightSign`을 컴포넌트 패널에서 그래프로 각각 드래그한 뒤, 해당 참조 핀에서 드래그해 검색하면 Target 타입이 자동으로 맞는다.

### 2.3 Collision 값 선택 노드

`Select` 노드를 추가한 뒤 Type을 `Collision Enabled`로 설정한다.

```text
Select (Collision Enabled)
  Index : IsActive
  False : No Collision
  True  : Query Only
```

`Query Only`는 표지판 선택 Trigger를 추가할 때 사용할 최소 Collision 설정이다. 현재 표지판에 Collision을 사용하지 않는 경우에도 활성화/비활성화 상태는 일관되게 유지된다.

### 2.4 데이터 핀 연결

아래 네 Component 제어 노드의 Target을 각각 연결한다.

```text
LeftSign  → Set Visibility(왼쪽).Target
LeftSign  → Set Collision Enabled(왼쪽).Target
RightSign → Set Visibility(오른쪽).Target
RightSign → Set Collision Enabled(오른쪽).Target
```

`IsActive` 출력 핀은 두 `Set Visibility`의 `New Visibility`에 모두 연결한다.

```text
IsActive → Set Visibility(왼쪽).New Visibility
IsActive → Set Visibility(오른쪽).New Visibility
```

두 `Set Visibility`에서 `Propagate to Children`은 `true`로 설정한다.

`Select (Collision Enabled)`의 Return Value는 두 `Set Collision Enabled`의 `New Type`에 모두 연결한다.

```text
Select.Return Value → Set Collision Enabled(왼쪽).New Type
Select.Return Value → Set Collision Enabled(오른쪽).New Type
```

### 2.5 실행 핀 연결

실행 순서는 아래와 같이 고정한다.

```text
SetSignsActive
→ Set Visibility(LeftSign)
→ Set Collision Enabled(LeftSign)
→ Set Visibility(RightSign)
→ Set Collision Enabled(RightSign)
```

완성 형태:

```text
Entry.Exec
  → Set Visibility(LeftSign).Exec
  → Set Collision Enabled(LeftSign).Exec
  → Set Visibility(RightSign).Exec
  → Set Collision Enabled(RightSign).Exec

IsActive
  → Set Visibility(LeftSign).New Visibility
  → Set Visibility(RightSign).New Visibility
  → Select (Collision Enabled).Index

Select (Collision Enabled).Return Value
  → Set Collision Enabled(LeftSign).New Type
  → Set Collision Enabled(RightSign).New Type
```

## 3. 타일 초기화에서 비활성화 호출

대상 함수: `BP_BaseTile.setInitComp`

기존 `setInitComp`의 Bounds 계산, Collision 크기, Component 위치 계산 노드는 삭제하거나 다시 배선하지 않는다.

기존 실행 흐름의 **마지막 실행 출력 핀** 뒤에 `SetSignsActive` 호출 노드를 추가한다.

```text
기존 setInitComp 마지막 Exec
→ SetSignsActive
    IsActive = false
```

동일한 타일을 재사용해도 `setInitComp`가 실행될 때마다 두 표지판은 아래 상태로 돌아간다.

```text
LeftSign / RightSign
  Visible           = false
  Collision Enabled = No Collision
```

## 4. `PlaceUpgradeSigns`에서 활성화 호출

대상 함수: `BP_LevelManager.PlaceUpgradeSigns`

기존 실행 체인은 유지한다.

```text
Shuffle
→ Set LeftSignStatus
→ Set LeftSignValue
→ Set RightSignStatus
→ Set RightSignValue
```

`Set RightSignValue`의 실행 출력 핀 뒤에 `SetSignsActive` 호출을 추가한다.

```text
Set RightSignValue.Exec
→ SetSignsActive.Exec

Tile 함수 입력
→ SetSignsActive.Target

SetSignsActive.IsActive = true
```

완성 후 Enemy 배치 요청이 있는 타일은 아래 순서로 동작한다.

```text
상태·수치 저장
→ 두 표지판 표시
→ 두 표지판 Collision 활성화
→ Boss → Elite → Normal Enemy 배치
```

## 5. 호출 조건

`PlaceUpgradeSigns(Tile)`은 유효한 Enemy 배치 요청이 있을 때만 호출한다.

```text
Enemy 배치 요청 없음
→ SetSignsActive(false) 상태 유지

Enemy 배치 요청 있음
→ PlaceUpgradeSigns(Tile)
→ SetSignsActive(true)
→ Enemy 배치 시작
```

Enemy 풀 대여가 실패해도 표지판은 다시 끄지 않는다. 표지판 활성화 조건은 실제 Enemy 대여 성공이 아니라 유효한 Enemy 배치 요청의 존재다.

## 6. 사용자 확인 항목

```text
1. 타일 최초 생성 시 LeftSign, RightSign이 보이지 않는지 확인
2. 타일 재사용 직후 두 표지판 Collision이 No Collision인지 확인
3. Enemy 배치 요청이 있는 타일에서 두 표지판이 모두 보이는지 확인
4. 표지판 자식인 Text, Mount, Status도 함께 보이는지 확인
5. Enemy 배치 요청이 없는 타일에서 이전 타일 표지판이 남지 않는지 확인
6. 표지판 활성화가 RightSignValue 저장 이후 실행되는지 확인
```

## 7. 현재 적용 상태

```text
BP_BaseTile 에디터 세션에 SetSignsActive 함수 생성됨
IsActive(Boolean) 입력 생성됨
LeftSign 참조 노드 추가됨
RightSign 참조, Visibility·Collision 노드 및 호출 배선은 작성 대기
에셋 저장·Compile·PIE 미실행
```
