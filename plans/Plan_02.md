# Plan_02.md

# Plan_02. Component Model Definition

## 1. 목적

이 문서는 웹 기반 물류 라인 이산 사건 시뮬레이션에서 사용할 **표준 설비 컴포넌트 모델**을 정의하기 위한 실행 계획서이다.

Plan_01.md가 프로젝트의 범위와 MVP 목표를 정했다면, Plan_02.md는 시뮬레이션 세계에 등장하는 설비 부품들의 공통 문법을 정한다.

이 문서의 핵심 목적은 다음과 같다.

```text
1. 모든 설비 컴포넌트가 가져야 할 공통 구조를 정의한다.
2. Source, Conveyor, Diverter, Merge, Buffer, Robot, Pallet Station, Sink의 역할과 속성을 정의한다.
3. 각 컴포넌트가 어떤 상태를 가지는지 정의한다.
4. 각 컴포넌트가 어떤 이벤트를 처리해야 하는지 정의한다.
5. Layout JSON과 DES Simulation Engine이 같은 컴포넌트 모델을 사용하도록 기준을 만든다.
```

---

## 2. 이 문서의 위치

전체 문서 흐름에서 Plan_02는 다음 위치에 있다.

```text
MasterPlan.md
  ↓
Plan_01.md: Project Scope & MVP Definition
  ↓
Plan_02.md: Component Model Definition
  ↓
Plan_03.md: Layout JSON Schema
```

Plan_02의 산출물은 Plan_03에서 Layout JSON Schema로 확장된다.
즉, Plan_02에서는 “설비가 무엇인가”를 정의하고, Plan_03에서는 “그 설비를 JSON으로 어떻게 저장할 것인가”를 정의한다.

---

## 3. 컴포넌트 모델의 기본 원칙

### 3.1 설비는 3D 모델이 아니라 논리 객체다

Three.js에서 보이는 컨베이어, 로봇, 버퍼는 화면 객체이다.
그러나 시뮬레이션 엔진이 사용하는 컴포넌트는 3D Mesh가 아니라 논리 객체이다.

```text
Component Model을 기준으로 시뮬레이션한다.
Three.js는 Component Model을 화면에 렌더링한다.
```

### 3.2 모든 컴포넌트는 포트를 가진다

```text
Source: out port만 있음
Sink: in port만 있음
Conveyor: in port, out port
Diverter: in port 1개, out port 여러 개
Merge: in port 여러 개, out port 1개
Robot: pick port, place target
```

### 3.3 컴포넌트는 상태를 가진다

MVP에서는 다음 상태를 우선 사용한다.

```text
idle
busy
blocked
waiting
```

### 3.4 컴포넌트는 이벤트를 처리한다

```text
ITEM_ENTER_COMPONENT
ITEM_EXIT_COMPONENT
ITEM_WAIT_START
ITEM_WAIT_END
PROCESS_START
PROCESS_END
ROBOT_PICK_START
ROBOT_PICK_END
```

---

## 4. 공통 Component 구조

```json
{
  "id": "CV_001",
  "type": "conveyor",
  "name": "Infeed Conveyor",
  "description": "Main box infeed conveyor",
  "position": { "x": 0, "y": 0, "z": 0 },
  "rotation": { "x": 0, "y": 0, "z": 0 },
  "dimensions": {
    "length": 6,
    "width": 0.8,
    "height": 0.6
  },
  "ports": {
    "in": ["CV_001_IN"],
    "out": ["CV_001_OUT"]
  },
  "properties": {},
  "visual": {},
  "metadata": {}
}
```

| 필드          | 설명              |
| ----------- | --------------- |
| id          | 컴포넌트 고유 ID      |
| type        | 컴포넌트 종류         |
| name        | 화면에 표시할 이름      |
| description | 설명              |
| position    | Layout 좌표       |
| rotation    | 회전 정보           |
| dimensions  | 길이, 폭, 높이       |
| ports       | 입구와 출구 포트       |
| properties  | 시뮬레이션용 속성       |
| visual      | Three.js 표현용 속성 |
| metadata    | 기타 확장 정보        |

---

## 5. ID 규칙

```text
Source: SRC_001
Conveyor: CV_001
Diverter: DIV_001
Merge: MRG_001
Buffer: BUF_001
Process Station: PROC_001
Robot: RB_001
Pallet Station: PALLET_001
Sink: SINK_001
```

포트 ID는 컴포넌트 ID를 기준으로 생성한다.

```text
CV_001_IN
CV_001_OUT
DIV_001_IN
DIV_001_OUT_A
DIV_001_OUT_B
MRG_001_IN_A
MRG_001_IN_B
MRG_001_OUT
```

---

## 6. 좌표계와 단위

```text
길이: meter
시간: second
속도: meter/second
수량: item count
```

Three.js와 시뮬레이션의 좌표계를 동일하게 맞춘다.

```text
x: 좌우 방향
y: 높이 방향
z: 전후 방향
```

Top View 편집기에서는 x, z 평면을 사용한다.

---

## 7. Port 모델

Port는 설비 간 물류 흐름을 연결하는 논리적 지점이다.

```json
{
  "id": "CV_001_OUT",
  "componentId": "CV_001",
  "direction": "out",
  "localPosition": { "x": 3, "y": 0, "z": 0 },
  "capacity": 1,
  "isBlockingPoint": true
}
```

| 필드              | 설명                        |
| --------------- | ------------------------- |
| id              | 포트 고유 ID                  |
| componentId     | 소속 컴포넌트 ID                |
| direction       | in 또는 out                 |
| localPosition   | 컴포넌트 기준 상대 위치             |
| capacity        | 해당 포트에 동시에 머무를 수 있는 아이템 수 |
| isBlockingPoint | 막힘 판단 지점 여부               |

---

## 8. Component State 모델

```json
{
  "componentId": "RB_001",
  "state": "busy",
  "currentItemIds": ["BOX_A_001"],
  "queueItemIds": [],
  "lastStateChangedAt": 120,
  "blockedReason": null
}
```

MVP에서 사용할 공통 상태는 다음과 같다.

```text
idle
busy
waiting
blocked
disabled
failed
```

MVP에서는 `idle`, `busy`, `waiting`, `blocked`를 우선 구현한다.

---

## 9. Item 모델과의 관계

```text
Source는 Item을 생성한다.
Conveyor는 Item을 이동시킨다.
Diverter는 Item의 목적지를 기준으로 다음 경로를 선택한다.
Merge는 여러 입력 중 어떤 Item을 먼저 보낼지 결정한다.
Buffer는 Item을 대기시킨다.
Process Station은 Item을 일정 시간 처리한다.
Robot은 Item을 픽업하고 적재한다.
Pallet Station은 Item을 적재 수량으로 기록한다.
Sink는 Item을 완료 처리한다.
```

---

## 10. 표준 컴포넌트 정의

### 10.1 Source

Source는 아이템이 생성되는 시작점이다.

```json
{
  "id": "SRC_001",
  "type": "source",
  "properties": {
    "itemTypes": ["BOX_A"],
    "releaseMode": "schedule",
    "defaultInterval": 6
  },
  "ports": {
    "in": [],
    "out": ["SRC_001_OUT"]
  }
}
```

기본 로직:

```text
1. Arrival Schedule을 읽는다.
2. 생성 시점이 되면 아이템을 생성한다.
3. Source의 out port가 사용 가능한지 확인한다.
4. 가능하면 다음 컴포넌트로 아이템을 보낸다.
5. 불가능하면 Source에서 대기하거나 생성 자체를 지연한다.
```

### 10.2 Conveyor

Conveyor는 아이템을 일정 속도로 이동시키는 설비이다.

```json
{
  "id": "CV_001",
  "type": "conveyor",
  "dimensions": {
    "length": 6,
    "width": 0.8,
    "height": 0.6
  },
  "properties": {
    "speed": 0.6,
    "capacity": 10,
    "accumulationMode": "blocking",
    "minGap": 0.1
  },
  "ports": {
    "in": ["CV_001_IN"],
    "out": ["CV_001_OUT"]
  }
}
```

기본 로직:

```text
1. 아이템이 in port로 들어온다.
2. capacity가 남아 있으면 진입을 허용한다.
3. travelTime = length / speed로 출구 도착 시간을 계산한다.
4. ITEM_EXIT_COMPONENT 이벤트를 예약한다.
5. 출구 도착 시 다음 컴포넌트가 받을 수 있는지 확인한다.
6. 받을 수 있으면 아이템을 다음 컴포넌트로 보낸다.
7. 받을 수 없으면 blocked 상태가 된다.
```

### 10.3 Diverter

Diverter는 아이템의 목적지 또는 규칙에 따라 여러 출구 중 하나를 선택하는 설비이다.

```json
{
  "id": "DIV_001",
  "type": "diverter",
  "properties": {
    "rule": "byDestination",
    "switchingTime": 1,
    "allowAlternativeRoute": false
  },
  "ports": {
    "in": ["DIV_001_IN"],
    "out": ["DIV_001_OUT_A", "DIV_001_OUT_B"]
  }
}
```

기본 로직:

```text
1. 아이템이 Diverter에 도착한다.
2. 아이템의 destination을 확인한다.
3. rule에 따라 output port를 선택한다.
4. 선택한 output port의 하류 컴포넌트가 받을 수 있는지 확인한다.
5. 가능하면 switchingTime 이후 배출한다.
6. 불가능하면 대기한다.
```

### 10.4 Merge

Merge는 여러 입력 라인이 하나의 출력 라인으로 합쳐지는 설비이다.

```json
{
  "id": "MRG_001",
  "type": "merge",
  "properties": {
    "rule": "fifo",
    "transferTime": 1,
    "mainInputPort": "MRG_001_IN_A"
  },
  "ports": {
    "in": ["MRG_001_IN_A", "MRG_001_IN_B"],
    "out": ["MRG_001_OUT"]
  }
}
```

기본 로직:

```text
1. 여러 입력 포트에서 아이템이 도착한다.
2. 각 입력 포트별 대기열에 아이템을 넣는다.
3. output port의 하류 설비가 받을 수 있는지 확인한다.
4. 받을 수 있으면 rule에 따라 하나의 아이템을 선택한다.
5. 선택된 아이템만 통과시키고 나머지는 대기한다.
```

### 10.5 Buffer

Buffer는 아이템이 임시로 대기하는 공간이다.

```json
{
  "id": "BUF_001",
  "type": "buffer",
  "properties": {
    "capacity": 20,
    "queueDiscipline": "fifo",
    "releaseMode": "whenDownstreamAvailable"
  },
  "ports": {
    "in": ["BUF_001_IN"],
    "out": ["BUF_001_OUT"]
  }
}
```

기본 로직:

```text
1. 아이템이 Buffer에 도착한다.
2. capacity가 남아 있으면 대기열에 넣는다.
3. capacity가 가득 차면 상류 설비에 blocking을 발생시킨다.
4. 하류 설비가 받을 수 있으면 queueDiscipline에 따라 아이템을 배출한다.
```

### 10.6 Process Station

Process Station은 검사, 라벨링, 포장, 밴딩처럼 일정 처리 시간이 필요한 설비이다.

```json
{
  "id": "PROC_001",
  "type": "processStation",
  "properties": {
    "processTime": 5,
    "capacity": 1,
    "processMode": "single"
  },
  "ports": {
    "in": ["PROC_001_IN"],
    "out": ["PROC_001_OUT"]
  }
}
```

### 10.7 Robot

Robot은 아이템을 픽업하고 다른 위치에 놓는 설비이다.

```json
{
  "id": "RB_001",
  "type": "robot",
  "properties": {
    "robotType": "palletizing",
    "cycleTime": 8,
    "workingRadius": 2.8,
    "payload": 20,
    "pickPort": "RB_001_PICK",
    "placeTarget": "PALLET_001",
    "gripperType": "vacuum"
  },
  "ports": {
    "in": ["RB_001_PICK"],
    "out": ["RB_001_OUT"]
  }
}
```

기본 로직:

```text
1. 픽업 포인트에 아이템이 도착한다.
2. 로봇이 idle 상태인지 확인한다.
3. idle이면 아이템을 점유하고 ROBOT_PICK_START 이벤트를 발생시킨다.
4. cycleTime 동안 busy 상태가 된다.
5. cycleTime 이후 아이템을 placeTarget으로 이동시킨다.
6. Pallet Station에 적재 완료 이벤트를 전달한다.
7. 로봇은 idle 상태로 돌아간다.
```

### 10.8 Pallet Station

Pallet Station은 로봇이 박스를 적재하는 위치이다.

```json
{
  "id": "PALLET_001",
  "type": "palletStation",
  "properties": {
    "palletSize": {
      "length": 1.1,
      "width": 1.1,
      "height": 0.15
    },
    "maxBoxes": 100,
    "boxesPerLayer": 20,
    "maxLayers": 5,
    "currentBoxCount": 0,
    "pattern": "grid"
  },
  "ports": {
    "in": ["PALLET_001_IN"],
    "out": ["PALLET_001_OUT"]
  }
}
```

### 10.9 Sink

Sink는 아이템 흐름의 완료 지점이다.

```json
{
  "id": "SINK_001",
  "type": "sink",
  "properties": {
    "category": "complete",
    "countMode": "item"
  },
  "ports": {
    "in": ["SINK_001_IN"],
    "out": []
  }
}
```

---

## 11. 컴포넌트별 이벤트 책임

| 컴포넌트            | 주요 책임                | 주요 이벤트                                                    |
| --------------- | -------------------- | --------------------------------------------------------- |
| Source          | 아이템 생성               | ITEM_CREATED, ITEM_RELEASED                               |
| Conveyor        | 이동 시간 계산, blocking   | ITEM_ENTER_COMPONENT, ITEM_EXIT_COMPONENT, BLOCKING_START |
| Diverter        | 출구 선택                | DIVERTER_ROUTE_DECIDED                                    |
| Merge           | 합류 순서 결정             | MERGE_RELEASE_DECIDED                                     |
| Buffer          | 대기열 관리               | ITEM_WAIT_START, ITEM_WAIT_END, BUFFER_FULL               |
| Process Station | 처리 시간 관리             | PROCESS_START, PROCESS_END                                |
| Robot           | resource 점유, 픽업 및 적재 | ROBOT_PICK_START, ROBOT_PICK_END, PALLETIZE_COMPLETE      |
| Pallet Station  | 적재 수량 관리             | PALLETIZE_COMPLETE, PALLET_COMPLETE                       |
| Sink            | 완료 처리                | ITEM_COMPLETED                                            |

---

## 12. TypeScript 타입 초안

```ts
export type ComponentType =
  | "source"
  | "conveyor"
  | "diverter"
  | "merge"
  | "buffer"
  | "processStation"
  | "robot"
  | "palletStation"
  | "sink";

export type ComponentState =
  | "idle"
  | "busy"
  | "waiting"
  | "blocked"
  | "disabled"
  | "failed";

export interface Vector3Like {
  x: number;
  y: number;
  z: number;
}

export interface Dimensions {
  length?: number;
  width?: number;
  height?: number;
  radius?: number;
}

export interface Port {
  id: string;
  componentId: string;
  direction: "in" | "out" | "pick" | "place";
  localPosition?: Vector3Like;
  capacity?: number;
  isBlockingPoint?: boolean;
}

export interface BaseComponent {
  id: string;
  type: ComponentType;
  name?: string;
  description?: string;
  position: Vector3Like;
  rotation: Vector3Like;
  dimensions?: Dimensions;
  ports: {
    in: string[];
    out: string[];
  };
  properties: Record<string, unknown>;
  visual?: Record<string, unknown>;
  metadata?: Record<string, unknown>;
}
```

---

## 13. 검증 규칙

```text
- id는 전체 프로젝트에서 고유해야 한다.
- type은 허용된 ComponentType 중 하나여야 한다.
- position은 숫자 좌표를 가져야 한다.
- rotation은 숫자 좌표를 가져야 한다.
- ports에 등록된 포트 ID는 실제 Port 목록에 존재해야 한다.
```

컴포넌트별 검증:

```text
Source
- out port가 최소 1개 있어야 한다.

Sink
- in port가 최소 1개 있어야 한다.
- out port는 없어야 한다.

Conveyor
- length는 0보다 커야 한다.
- speed는 0보다 커야 한다.
- capacity는 1 이상이어야 한다.

Diverter
- in port는 1개 이상이어야 한다.
- out port는 2개 이상이어야 한다.

Merge
- in port는 2개 이상이어야 한다.
- out port는 1개 이상이어야 한다.

Buffer
- capacity는 1 이상이어야 한다.

Robot
- cycleTime은 0보다 커야 한다.
- workingRadius는 0보다 커야 한다.
- placeTarget이 정의되어야 한다.

Pallet Station
- maxBoxes는 1 이상이어야 한다.
- boxesPerLayer는 1 이상이어야 한다.
```

---

## 14. MVP 구현 우선순위

```text
1순위
- Source
- Conveyor
- Robot
- Pallet Station
- Sink

2순위
- Diverter

3순위
- Merge
- Buffer

4순위
- Process Station
```

---

## 15. 작업 목록

```text
문서 작업
- 공통 Component 구조 확정
- Port 구조 확정
- Component State 구조 확정
- Item과 Component 관계 정의
- 컴포넌트별 필수 속성 정의
- 컴포넌트별 처리 이벤트 정의
- 검증 규칙 정의

타입 설계 작업
- ComponentType 타입 정의
- ComponentState 타입 정의
- Port 타입 정의
- BaseComponent 타입 정의
- SourceComponent 타입 정의
- ConveyorComponent 타입 정의
- DiverterComponent 타입 정의
- MergeComponent 타입 정의
- BufferComponent 타입 정의
- RobotComponent 타입 정의
- PalletStationComponent 타입 정의
- SinkComponent 타입 정의

샘플 데이터 작업
- Source 샘플 작성
- Conveyor 샘플 작성
- Robot 샘플 작성
- Pallet Station 샘플 작성
- Sink 샘플 작성
- Diverter 샘플 작성
- Merge 샘플 작성
- Buffer 샘플 작성
```

---

## 16. 완료 기준

```text
- 모든 MVP 컴포넌트의 역할이 정의되어 있다.
- 모든 MVP 컴포넌트의 필수 속성이 정의되어 있다.
- 모든 MVP 컴포넌트의 포트 구조가 정의되어 있다.
- 모든 MVP 컴포넌트의 상태 모델이 정의되어 있다.
- 모든 MVP 컴포넌트의 주요 이벤트 책임이 정의되어 있다.
- TypeScript 타입 초안이 작성되어 있다.
- 컴포넌트 검증 규칙이 작성되어 있다.
- Plan_03에서 Layout JSON Schema로 확장할 수 있는 수준이다.
```

---

## 17. Codex 작업 지시 예시

```text
MasterPlan.md, Plan_01.md, Plan_02.md를 읽고,
src/types/component.ts 파일을 생성해.

다음 타입을 정의해.
- ComponentType
- ComponentState
- Vector3Like
- Dimensions
- Port
- BaseComponent
- SourceComponent
- ConveyorComponent
- DiverterComponent
- MergeComponent
- BufferComponent
- ProcessStationComponent
- RobotComponent
- PalletStationComponent
- SinkComponent

아직 시뮬레이션 로직은 구현하지 마.
```

---

## 18. 다음 단계

Plan_02가 완료되면 다음 문서로 이동한다.

```text
Plan_03.md
```

Plan_03에서는 Plan_02에서 정의한 Component Model을 기반으로 **Layout JSON Schema**를 상세히 정의한다.
::: 
