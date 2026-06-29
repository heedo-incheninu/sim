# Plan_04.md

# Plan_04. Three.js Layout Editor Definition

## 1. 목적

이 문서는 웹 기반 물류 라인 이산 사건 시뮬레이션 프로젝트에서 사용할 **Three.js Layout Editor**를 정의하기 위한 실행 계획서이다.

Plan_03.md에서 Layout JSON의 구조를 정의했다면, Plan_04.md는 사용자가 웹 화면에서 설비 컴포넌트를 직접 배치하고, 이동하고, 회전하고, 길이를 조정하고, 포트끼리 연결하여 Layout JSON을 생성하는 편집기를 설계한다.

이 문서의 핵심 목적은 다음과 같다.

```text
1. Three.js Layout Editor의 화면 구조를 정의한다.
2. 컴포넌트 추가, 선택, 이동, 회전, 길이 조정 방식을 정의한다.
3. 포트 표시와 포트 연결 방식을 정의한다.
4. 선택한 컴포넌트의 속성 패널 구조를 정의한다.
5. 도면 이미지를 배경으로 깔고 스케일을 보정하는 기능을 정의한다.
6. 편집 결과가 Layout JSON으로 저장되고 다시 불러와질 수 있도록 기준을 만든다.
```

---

## 2. Layout Editor의 기본 역할

Three.js Layout Editor는 다음 역할을 한다.

```text
1. 설비 컴포넌트를 화면에 배치한다.
2. 컴포넌트의 위치, 방향, 길이, 크기를 수정한다.
3. 컴포넌트의 포트를 표시한다.
4. 포트와 포트를 연결한다.
5. 선택한 컴포넌트의 속성을 수정한다.
6. 도면 이미지를 배경으로 표시한다.
7. 도면 스케일을 보정한다.
8. 편집 결과를 Layout JSON으로 저장한다.
9. Layout JSON을 불러와 동일한 배치를 복원한다.
```

중요한 원칙은 다음과 같다.

```text
Three.js는 편집 화면이다.
Layout JSON은 실제 데이터다.
화면에서 일어난 모든 편집은 Layout JSON에 반영되어야 한다.
```

---

## 3. 기본 화면 구성

```text
┌─────────────────────────────────────────────────────────────┐
│ Top Toolbar                                                  │
├───────────────┬───────────────────────────────┬─────────────┤
│ Left Panel    │ Three.js Canvas                │ Right Panel │
│ Component     │ Layout View                    │ Properties  │
│ Library       │                               │ Editor      │
├───────────────┴───────────────────────────────┴─────────────┤
│ Bottom Panel: Validation, JSON, Simulation Controls           │
└─────────────────────────────────────────────────────────────┘
```

### 3.1 Top Toolbar

```text
- 새 레이아웃
- 저장
- 불러오기
- JSON 내보내기
- JSON 가져오기
- 실행 취소
- 다시 실행
- 선택 모드
- 이동 모드
- 회전 모드
- 연결 모드
- 도면 배경 업로드
- 스케일 보정
- 검증 실행
```

### 3.2 Left Panel

```text
- Source
- Conveyor
- Diverter
- Merge
- Buffer
- Process Station
- Robot
- Pallet Station
- Sink
```

### 3.3 Three.js Canvas

```text
- 그리드
- 좌표축
- 도면 배경
- 설비 컴포넌트
- 포트
- 연결선
- 컨베이어 방향 화살표
- 로봇 작업 반경
- 선택 표시
- 스냅 가이드
```

### 3.4 Right Panel

```text
- ID
- Type
- Name
- Position
- Rotation
- Dimensions
- Ports
- Properties
- Visual Settings
- Metadata
```

---

## 4. 편집 모드 정의

```text
select
- 컴포넌트, 포트, 연결 선택

move
- 선택한 컴포넌트 이동

rotate
- 선택한 컴포넌트 회전

resize
- 컨베이어 길이 또는 설비 크기 조정

connect
- 포트와 포트 연결

delete
- 선택 객체 삭제

scaleCalibration
- 도면 배경 스케일 보정
```

---

## 5. 컴포넌트 추가 방식

컴포넌트 추가 흐름은 다음과 같다.

```text
1. 사용자가 Left Panel에서 컴포넌트 타입을 선택한다.
2. 사용자가 Canvas 위 원하는 위치를 클릭한다.
3. 시스템이 기본 속성을 가진 컴포넌트를 생성한다.
4. 컴포넌트 ID와 포트 ID를 자동 생성한다.
5. Layout 상태에 component와 ports를 추가한다.
6. Three.js Canvas에 컴포넌트를 렌더링한다.
7. Right Panel에 속성 편집 화면을 표시한다.
```

기본 생성값:

```text
Source
- dimensions: 0.8m x 0.8m x 0.6m
- out port 1개

Conveyor
- length: 3m
- width: 0.8m
- height: 0.6m
- speed: 0.6m/s
- capacity: 5
- in port 1개, out port 1개

Diverter
- length: 1m
- width: 1m
- in port 1개, out port 2개
- rule: byDestination

Merge
- length: 1m
- width: 1m
- in port 2개, out port 1개
- rule: fifo

Buffer
- length: 2m
- width: 0.8m
- capacity: 10

Robot
- workingRadius: 2.8m
- cycleTime: 8s
- robotType: palletizing

Pallet Station
- length: 1.1m
- width: 1.1m
- maxBoxes: 100

Sink
- dimensions: 0.8m x 0.8m x 0.6m
- in port 1개
```

ID 자동 생성 규칙:

```text
SRC_001
CV_001
DIV_001
MRG_001
BUF_001
PROC_001
RB_001
PALLET_001
SINK_001
```

---

## 6. 컴포넌트 선택과 표시

선택된 컴포넌트는 다음 방식으로 강조한다.

```text
- 외곽선 강조
- 반투명 하이라이트
- 이동 핸들 표시
- 회전 핸들 표시
- 포트 표시
```

Hover 시 표시 정보:

```text
- component id
- component type
- name
- 주요 속성
```

예시:

```text
CV_001
Type: Conveyor
Length: 6m
Speed: 0.6m/s
Capacity: 10
```

---

## 7. 컴포넌트 이동

컴포넌트 이동은 Three.js Mesh를 직접 움직이는 것이 아니라 Layout 상태의 `position`을 수정하는 방식으로 처리한다.

```text
1. 사용자가 컴포넌트를 드래그한다.
2. 마우스 위치를 x, z 평면 좌표로 변환한다.
3. Layout 상태의 component.position을 갱신한다.
4. Three.js 렌더링이 상태 변경을 반영한다.
5. 연결선과 포트 위치도 함께 갱신된다.
```

스냅 기능:

```text
기본 스냅 간격: 0.5m
Shift 키 누르면 스냅 해제
Ctrl 키 누르면 0.1m 미세 이동
```

---

## 8. 컴포넌트 회전

Top View에서는 y축 회전을 기본으로 한다.

```text
rotation.y = 방향 각도
```

지원 방식:

```text
- 우측 패널에서 각도 입력
- 회전 핸들 드래그
- 90도 회전 버튼
- 단축키 R
```

회전하면 다음 정보가 함께 갱신되어야 한다.

```text
- component.rotation
- port world position
- connection visual line
- conveyor direction arrow
- robot working radius 표시 방향
```

---

## 9. 컨베이어 길이 조정

MVP에서는 Conveyor length 조정을 핵심 기능으로 구현한다.

지원 방식:

```text
- 우측 패널에서 length 숫자 입력
- 컨베이어 양 끝 핸들을 드래그
- 단축키 + / - 로 길이 증가 감소
```

MVP에서는 중심 기준 조정을 기본으로 한다.

```text
center 기준
- position은 유지
- 양쪽 끝이 동시에 늘어나거나 줄어든다.
```

길이 변경 시 갱신 대상:

```text
- component.dimensions.length
- CV_001_IN localPosition
- CV_001_OUT localPosition
- connection visual line
- Three.js conveyor mesh scale
- direction arrow 위치
```

---

## 10. 포트 표시와 연결

포트는 설비 간 물류 흐름을 연결하는 논리적 지점이다.

표시 조건:

```text
- 컴포넌트 선택 시 해당 컴포넌트의 포트 표시
- Connect Mode에서는 모든 연결 가능한 포트 표시
- 평상시에는 포트 숨김
```

연결 흐름:

```text
1. 사용자가 Connect Mode로 전환한다.
2. out port를 클릭한다.
3. 연결 가능한 in port들이 강조된다.
4. 사용자가 대상 in port를 클릭한다.
5. connection 객체를 생성한다.
6. Layout JSON의 connections 배열에 추가한다.
7. 화면에 연결선을 표시한다.
8. Layout Validator를 실행하여 오류 여부를 확인한다.
```

연결 가능 조건:

```text
from port는 out 또는 place여야 한다.
to port는 in 또는 pick이어야 한다.
이미 동일한 connection이 있으면 중복 생성하지 않는다.
disabled component와는 연결하지 않는다.
```

---

## 11. 속성 패널

모든 컴포넌트에 표시할 공통 속성:

```text
id
type
name
description
position.x
position.y
position.z
rotation.x
rotation.y
rotation.z
dimensions
visual
metadata
```

컴포넌트별 속성:

```text
Source
- itemTypes
- releaseMode
- defaultInterval

Conveyor
- length
- width
- height
- speed
- capacity
- accumulationMode
- minGap

Diverter
- rule
- switchingTime
- allowAlternativeRoute

Merge
- rule
- transferTime
- mainInputPort

Buffer
- capacity
- queueDiscipline
- releaseMode

Process Station
- processTime
- capacity
- processMode

Robot
- robotType
- cycleTime
- workingRadius
- payload
- pickPort
- placeTarget
- gripperType

Pallet Station
- maxBoxes
- boxesPerLayer
- maxLayers
- pattern

Sink
- category
- countMode
```

---

## 12. 도면 배경 기능

MVP에서는 도면을 자동 인식하지 않고, 배경지도로 사용한다.

기능 흐름:

```text
1. 사용자가 도면 이미지 파일을 업로드한다.
2. 이미지를 Three.js 바닥 평면에 텍스처로 표시한다.
3. 사용자가 투명도를 조정한다.
4. 사용자가 도면을 잠근다.
5. 사용자가 도면 위에 컴포넌트를 배치한다.
```

지원 파일 형식:

```text
png
jpg
jpeg
webp
```

Layout JSON의 visualSettings에 저장한다.

```json
{
  "drawingBackground": {
    "enabled": true,
    "imageUrl": "/drawings/sample_layout.png",
    "opacity": 0.35,
    "locked": true,
    "scale": 1,
    "position": { "x": 0, "y": -0.01, "z": 0 },
    "rotation": { "x": 0, "y": 0, "z": 0 },
    "size": {
      "width": 20,
      "height": 12
    }
  }
}
```

---

## 13. 도면 스케일 보정

스케일 보정 흐름:

```text
1. 사용자가 스케일 보정 모드를 선택한다.
2. 도면 위에서 기준 거리의 시작점과 끝점을 클릭한다.
3. 실제 거리 값을 입력한다.
4. 시스템이 drawingUnitPerMeter 또는 image scale을 계산한다.
5. 이후 컴포넌트 길이와 위치가 실제 meter 단위로 맞춰진다.
```

예시:

```text
도면에서 두 점 사이 화면 거리: 5.2 units
사용자가 입력한 실제 거리: 10m

scale = 5.2 / 10 = 0.52 drawing units per meter
```

---

## 14. Layout 상태 관리

원본 상태는 항상 LayoutJson object다.

추천 상태 구조:

```ts
interface LayoutEditorState {
  layout: LayoutJson;
  selectedComponentId: string | null;
  selectedConnectionId: string | null;
  selectedPortId: string | null;
  mode: EditorMode;
  history: LayoutJson[];
  future: LayoutJson[];

  addComponent: (type: ComponentType, position: Vector3Like) => void;
  updateComponent: (id: string, patch: Partial<LayoutComponent>) => void;
  removeComponent: (id: string) => void;

  addConnection: (fromPortId: string, toPortId: string) => void;
  removeConnection: (connectionId: string) => void;

  selectComponent: (id: string | null) => void;
  selectConnection: (id: string | null) => void;
  setMode: (mode: EditorMode) => void;

  loadLayout: (layout: LayoutJson) => void;
  exportLayout: () => LayoutJson;
}
```

Editor Mode 타입:

```ts
type EditorMode =
  | "select"
  | "move"
  | "rotate"
  | "resize"
  | "connect"
  | "delete"
  | "scaleCalibration";
```

---

## 15. Three.js 렌더링 구조

Layout JSON의 components 배열을 순회하면서 컴포넌트를 렌더링한다.

```text
source → SourceMesh
conveyor → ConveyorMesh
diverter → DiverterMesh
merge → MergeMesh
buffer → BufferMesh
processStation → ProcessStationMesh
robot → RobotMesh
palletStation → PalletStationMesh
sink → SinkMesh
```

추천 폴더 구조:

```text
src/three/
  scene/
    LayoutScene.tsx
    CameraController.tsx
    Grid.tsx
    DrawingBackground.tsx

  objects/
    SourceObject.tsx
    ConveyorObject.tsx
    DiverterObject.tsx
    MergeObject.tsx
    BufferObject.tsx
    RobotObject.tsx
    PalletStationObject.tsx
    SinkObject.tsx
    PortObject.tsx
    ConnectionLine.tsx

  controls/
    SelectionController.tsx
    TransformController.tsx
    ConnectionController.tsx
    ScaleCalibrationController.tsx
```

---

## 16. 카메라와 조작 방식

MVP에서는 Top View 카메라를 기본으로 한다.

```text
Orthographic Camera
Top View
x-z 평면 보기
```

지원 조작:

```text
마우스 휠: zoom in/out
마우스 우클릭 드래그: pan
Fit View 버튼: 전체 레이아웃 보기
Home 키: 원점 보기
```

---

## 17. 단축키

```text
V: Select Mode
M: Move Mode
R: Rotate Mode
C: Connect Mode
Delete: 선택 객체 삭제
Ctrl + Z: Undo
Ctrl + Y: Redo
Ctrl + S: Save
F: Fit View
Esc: 선택 해제 또는 현재 작업 취소
Shift + Drag: 스냅 해제
```

---

## 18. 검증 표시

검증 시점:

```text
- 저장 전
- 시뮬레이션 실행 전
- connection 생성 후
- component 속성 변경 후
- 사용자가 Validate 버튼 클릭 시
```

오류 표시 형식:

```text
[ERROR] PORT_NOT_FOUND
Connection CONN_001 references missing port CV_999_OUT.
Path: connections[0].from
```

오류가 특정 컴포넌트나 연결과 관련 있으면 화면에서 강조한다.

```text
- 오류 컴포넌트 빨간 외곽선
- 오류 연결선 빨간색
- warning은 노란색
```

---

## 19. 저장 및 불러오기

저장 방식:

```text
1. 현재 Layout 상태를 가져온다.
2. validateLayout을 실행한다.
3. 오류가 있으면 저장 전 사용자에게 표시한다.
4. warning만 있으면 저장 가능하다.
5. JSON 파일로 다운로드한다.
```

불러오기 방식:

```text
1. 사용자가 layout.json 파일을 선택한다.
2. JSON을 파싱한다.
3. schemaVersion을 확인한다.
4. validateLayout을 실행한다.
5. 오류가 없으면 Layout 상태에 반영한다.
6. Three.js 화면에 렌더링한다.
```

---

## 20. MVP 구현 범위

### 포함

```text
- Top View Three.js Canvas
- Grid 표시
- Source 추가
- Conveyor 추가
- Robot 추가
- Pallet Station 추가
- Sink 추가
- 컴포넌트 선택
- 컴포넌트 이동
- Conveyor 길이 수정
- 컴포넌트 회전
- 포트 표시
- 포트 연결
- 속성 패널
- Layout JSON 저장
- Layout JSON 불러오기
- 검증 결과 표시
```

### 제외

```text
- DXF 자동 인식
- DWG 지원
- 자동 라우팅 생성
- 3D 모델 GLB 편집
- 실제 로봇 관절 편집
- 복잡한 충돌 검사
- 다중 사용자 동시 편집
```

---

## 21. 작업 목록

### 21.1 화면 구조 작업

* [ ] 전체 Editor Layout 구현
* [ ] Top Toolbar 구현
* [ ] Left Component Library 구현
* [ ] Three.js Canvas 영역 구현
* [ ] Right Properties Panel 구현
* [ ] Bottom Validation Panel 구현

### 21.2 Three.js 기본 작업

* [ ] Orthographic Camera 구현
* [ ] Grid 구현
* [ ] 기본 조명 구현
* [ ] 좌표축 표시
* [ ] 화면 pan/zoom 구현
* [ ] Fit View 구현

### 21.3 컴포넌트 렌더링 작업

* [ ] SourceObject 구현
* [ ] ConveyorObject 구현
* [ ] RobotObject 구현
* [ ] PalletStationObject 구현
* [ ] SinkObject 구현
* [ ] DiverterObject 구현
* [ ] MergeObject 구현
* [ ] BufferObject 구현
* [ ] PortObject 구현
* [ ] ConnectionLine 구현

### 21.4 편집 기능 작업

* [ ] 컴포넌트 추가
* [ ] 컴포넌트 선택
* [ ] 컴포넌트 이동
* [ ] 컴포넌트 회전
* [ ] Conveyor 길이 조정
* [ ] 포트 선택
* [ ] 포트 연결
* [ ] 연결 삭제
* [ ] 컴포넌트 삭제

### 21.5 속성 패널 작업

* [ ] 공통 속성 편집
* [ ] Conveyor 속성 편집
* [ ] Robot 속성 편집
* [ ] Pallet Station 속성 편집
* [ ] Sink 속성 편집
* [ ] 속성 입력 검증
* [ ] 속성 변경 시 Layout 상태 갱신

---

## 22. 완료 기준

```text
- 사용자가 웹 화면에서 Source, Conveyor, Robot, Pallet Station, Sink를 추가할 수 있다.
- 사용자가 컴포넌트를 선택하고 이동할 수 있다.
- 사용자가 컴포넌트를 회전할 수 있다.
- 사용자가 Conveyor 길이를 수정할 수 있다.
- 컴포넌트의 포트가 표시된다.
- 사용자가 포트와 포트를 연결할 수 있다.
- 연결 결과가 Layout JSON의 connections 배열에 반영된다.
- 우측 속성 패널에서 주요 속성을 수정할 수 있다.
- 수정 결과가 Layout JSON에 반영된다.
- Layout JSON을 저장할 수 있다.
- Layout JSON을 불러와 동일한 배치를 복원할 수 있다.
- Layout Validator 오류가 화면에 표시된다.
```

---

## 23. Codex 작업 지시 예시

```text
MasterPlan.md, Plan_01.md, Plan_02.md, Plan_03.md, Plan_04.md를 읽고,
Vite + React + TypeScript 프로젝트에 Layout Editor 기본 화면을 만들어줘.

구성:
- Top Toolbar
- Left Component Library
- Center Three.js Canvas
- Right Properties Panel
- Bottom Validation Panel

아직 DES Simulation Engine은 구현하지 마.
```

---

## 24. 다음 단계

Plan_04가 완료되면 다음 문서로 이동한다.

```text
Plan_05.md
```

Plan_05에서는 Layout Editor에서 생성한 ports와 connections를 기준으로 **Network Routing**을 정의한다.
::: 
