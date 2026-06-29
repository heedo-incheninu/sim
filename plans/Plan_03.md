# Plan_03.md

# Plan_03. Layout JSON Schema Definition

## 1. 목적

이 문서는 웹 기반 물류 라인 이산 사건 시뮬레이션에서 사용할 **Layout JSON Schema**를 정의하기 위한 실행 계획서이다.

Plan_02.md에서 Source, Conveyor, Diverter, Merge, Buffer, Robot, Pallet Station, Sink 등 표준 컴포넌트의 역할과 속성을 정의했다면, Plan_03.md는 그 컴포넌트를 하나의 레이아웃 파일 안에 어떻게 저장하고, 연결하고, 검증할지 정한다.

이 문서의 핵심 목적은 다음과 같다.

```text
1. Layout JSON의 전체 구조를 정의한다.
2. 컴포넌트, 포트, 연결, 경로, 규칙의 저장 방식을 정의한다.
3. Three.js Layout Editor와 DES Simulation Engine이 동일한 JSON을 사용하도록 기준을 만든다.
4. 저장, 불러오기, 검증, 버전 관리를 위한 Schema 원칙을 정의한다.
5. Codex가 TypeScript 타입과 검증 함수를 구현할 수 있는 수준의 명세를 제공한다.
```

Layout JSON은 이 프로젝트의 설계도이자 공통 언어다.
Three.js는 이 설계도를 보고 화면을 그리고, DES Engine은 이 설계도를 보고 물류 흐름을 계산한다.

---

## 2. Layout JSON의 역할

Layout JSON은 다음 정보를 저장한다.

```text
1. 프로젝트 기본 정보
2. 좌표계와 단위 정보
3. 설비 컴포넌트 목록
4. 포트 목록
5. 포트 간 연결 관계
6. 목적지별 경로 정의
7. 분기, 합류, Blocking 기본 규칙
8. Three.js 표시용 시각 정보
9. 검증 및 버전 관리 정보
```

Layout JSON은 다음 정보를 저장하지 않는다.

```text
1. 시간대별 물동량
2. 제품별 투입 수량
3. 시뮬레이션 시작 및 종료 시간
4. 이벤트 로그
5. 시뮬레이션 결과 KPI
```

이 정보들은 각각 다른 파일에서 관리한다.

```text
layout.json
- 설비 배치와 연결 구조

scenario.json
- 물동량, 제품, 운영 조건

simulation_result.json
- 시뮬레이션 결과와 이벤트 로그
```

---

## 3. Layout JSON 전체 구조

MVP 기준 Layout JSON의 최상위 구조는 다음과 같다.

```json
{
  "schemaVersion": "1.0.0",
  "project": {},
  "coordinateSystem": {},
  "components": [],
  "ports": [],
  "connections": [],
  "routes": [],
  "rules": {},
  "visualSettings": {},
  "metadata": {}
}
```

각 영역의 역할은 다음과 같다.

| 영역               | 역할                     |
| ---------------- | ---------------------- |
| schemaVersion    | Layout JSON Schema 버전  |
| project          | 프로젝트 이름, ID, 작성자, 설명   |
| coordinateSystem | 단위, 좌표계, 스케일           |
| components       | 설비 컴포넌트 목록             |
| ports            | 전체 포트 목록               |
| connections      | 포트 간 연결 관계             |
| routes           | 목적지별 이동 경로             |
| rules            | 기본 분기, 합류, Blocking 규칙 |
| visualSettings   | Three.js 화면 표시 설정      |
| metadata         | 생성일, 수정일, 태그 등 부가 정보   |

---

## 4. project 구조

```json
{
  "project": {
    "id": "project_001",
    "name": "Palletizing Line MVP",
    "description": "Simple palletizing line layout for MVP simulation",
    "author": "SFA",
    "createdAt": "2026-06-29T00:00:00.000Z",
    "updatedAt": "2026-06-29T00:00:00.000Z"
  }
}
```

| 필드          | 필수 | 설명         |
| ----------- | -- | ---------- |
| id          | 필수 | 프로젝트 고유 ID |
| name        | 필수 | 프로젝트 이름    |
| description | 선택 | 프로젝트 설명    |
| author      | 선택 | 작성자 또는 조직  |
| createdAt   | 선택 | 생성 시간      |
| updatedAt   | 선택 | 수정 시간      |

---

## 5. coordinateSystem 구조

```json
{
  "coordinateSystem": {
    "unit": "meter",
    "timeUnit": "second",
    "angleUnit": "degree",
    "axis": {
      "horizontal": "x",
      "vertical": "y",
      "depth": "z"
    },
    "topViewPlane": "xz",
    "scale": {
      "drawingUnitPerMeter": 1
    }
  }
}
```

MVP 기본값은 다음과 같다.

```text
unit: meter
timeUnit: second
angleUnit: degree
topViewPlane: xz
```

Three.js 내부 회전 계산은 radian을 많이 사용하지만, Layout JSON에서는 사람이 읽기 쉬운 degree를 우선 사용한다.
Three.js 렌더링 단계에서 degree를 radian으로 변환한다.

---

## 6. components 배열

components는 레이아웃에 포함된 모든 설비 컴포넌트를 저장한다.

```json
{
  "id": "CV_001",
  "type": "conveyor",
  "name": "Infeed Conveyor",
  "description": "Main infeed conveyor",
  "position": { "x": 3, "y": 0, "z": 0 },
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
  "properties": {
    "speed": 0.6,
    "capacity": 10,
    "accumulationMode": "blocking"
  },
  "visual": {
    "color": "#4A90E2",
    "labelVisible": true
  },
  "metadata": {
    "tag": "infeed"
  }
}
```

허용 Component Type:

```text
source
conveyor
diverter
merge
buffer
processStation
robot
palletStation
sink
```

공통 검증 규칙:

```text
- id는 전체 components 안에서 고유해야 한다.
- type은 허용된 Component Type 중 하나여야 한다.
- position.x, position.y, position.z는 숫자여야 한다.
- rotation.x, rotation.y, rotation.z는 숫자여야 한다.
- ports.in과 ports.out에 적힌 포트 ID는 ports 배열에 존재해야 한다.
- properties는 객체여야 한다.
```

---

## 7. component type별 Schema

### 7.1 Source Component

```json
{
  "id": "SRC_001",
  "type": "source",
  "name": "Box Source",
  "position": { "x": 0, "y": 0, "z": 0 },
  "rotation": { "x": 0, "y": 0, "z": 0 },
  "dimensions": {
    "length": 0.8,
    "width": 0.8,
    "height": 0.6
  },
  "ports": {
    "in": [],
    "out": ["SRC_001_OUT"]
  },
  "properties": {
    "itemTypes": ["BOX_A"],
    "releaseMode": "schedule",
    "defaultInterval": 6
  }
}
```

검증 규칙:

```text
- in port는 없어야 한다.
- out port는 최소 1개 있어야 한다.
- releaseMode는 schedule, manual, batch 중 하나여야 한다.
```

### 7.2 Conveyor Component

```json
{
  "id": "CV_001",
  "type": "conveyor",
  "name": "Infeed Conveyor",
  "position": { "x": 3, "y": 0, "z": 0 },
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
  "properties": {
    "speed": 0.6,
    "capacity": 10,
    "accumulationMode": "blocking",
    "minGap": 0.1
  }
}
```

검증 규칙:

```text
- length는 0보다 커야 한다.
- speed는 0보다 커야 한다.
- capacity는 1 이상이어야 한다.
- in port는 최소 1개 있어야 한다.
- out port는 최소 1개 있어야 한다.
```

### 7.3 Diverter Component

```json
{
  "id": "DIV_001",
  "type": "diverter",
  "name": "Destination Diverter",
  "position": { "x": 6, "y": 0, "z": 0 },
  "rotation": { "x": 0, "y": 0, "z": 0 },
  "dimensions": {
    "length": 1,
    "width": 1,
    "height": 0.6
  },
  "ports": {
    "in": ["DIV_001_IN"],
    "out": ["DIV_001_OUT_A", "DIV_001_OUT_B"]
  },
  "properties": {
    "rule": "byDestination",
    "switchingTime": 1,
    "allowAlternativeRoute": false
  }
}
```

검증 규칙:

```text
- in port는 최소 1개 있어야 한다.
- out port는 최소 2개 있어야 한다.
- rule은 byDestination, shortestQueue, roundRobin, priority 중 하나여야 한다.
```

### 7.4 Merge Component

```json
{
  "id": "MRG_001",
  "type": "merge",
  "name": "Main Merge",
  "position": { "x": 8, "y": 0, "z": 0 },
  "rotation": { "x": 0, "y": 0, "z": 0 },
  "dimensions": {
    "length": 1,
    "width": 1,
    "height": 0.6
  },
  "ports": {
    "in": ["MRG_001_IN_A", "MRG_001_IN_B"],
    "out": ["MRG_001_OUT"]
  },
  "properties": {
    "rule": "fifo",
    "transferTime": 1,
    "mainInputPort": "MRG_001_IN_A"
  }
}
```

검증 규칙:

```text
- in port는 최소 2개 있어야 한다.
- out port는 최소 1개 있어야 한다.
- rule은 fifo, roundRobin, mainLinePriority, priority 중 하나여야 한다.
```

### 7.5 Buffer Component

```json
{
  "id": "BUF_001",
  "type": "buffer",
  "name": "Robot Buffer",
  "position": { "x": 9, "y": 0, "z": 0 },
  "rotation": { "x": 0, "y": 0, "z": 0 },
  "dimensions": {
    "length": 2,
    "width": 0.8,
    "height": 0.6
  },
  "ports": {
    "in": ["BUF_001_IN"],
    "out": ["BUF_001_OUT"]
  },
  "properties": {
    "capacity": 20,
    "queueDiscipline": "fifo",
    "releaseMode": "whenDownstreamAvailable"
  }
}
```

### 7.6 Robot Component

```json
{
  "id": "RB_001",
  "type": "robot",
  "name": "Palletizing Robot 1",
  "position": { "x": 10, "y": 0, "z": 2 },
  "rotation": { "x": 0, "y": 0, "z": 0 },
  "dimensions": {
    "radius": 2.8,
    "height": 2.2
  },
  "ports": {
    "in": ["RB_001_PICK"],
    "out": ["RB_001_OUT"]
  },
  "properties": {
    "robotType": "palletizing",
    "cycleTime": 8,
    "workingRadius": 2.8,
    "payload": 20,
    "pickPort": "RB_001_PICK",
    "placeTarget": "PALLET_001",
    "gripperType": "vacuum"
  }
}
```

검증 규칙:

```text
- cycleTime은 0보다 커야 한다.
- workingRadius는 0보다 커야 한다.
- pickPort는 ports.in에 존재해야 한다.
- placeTarget은 components 안에 존재하는 palletStation ID여야 한다.
```

### 7.7 Pallet Station Component

```json
{
  "id": "PALLET_001",
  "type": "palletStation",
  "name": "Pallet Station 1",
  "position": { "x": 11, "y": 0, "z": 3 },
  "rotation": { "x": 0, "y": 0, "z": 0 },
  "dimensions": {
    "length": 1.1,
    "width": 1.1,
    "height": 0.15
  },
  "ports": {
    "in": ["PALLET_001_IN"],
    "out": ["PALLET_001_OUT"]
  },
  "properties": {
    "maxBoxes": 100,
    "boxesPerLayer": 20,
    "maxLayers": 5,
    "pattern": "grid"
  }
}
```

### 7.8 Sink Component

```json
{
  "id": "SINK_001",
  "type": "sink",
  "name": "Completed Sink",
  "position": { "x": 13, "y": 0, "z": 3 },
  "rotation": { "x": 0, "y": 0, "z": 0 },
  "dimensions": {
    "length": 0.8,
    "width": 0.8,
    "height": 0.6
  },
  "ports": {
    "in": ["SINK_001_IN"],
    "out": []
  },
  "properties": {
    "category": "complete",
    "countMode": "item"
  }
}
```

---

## 8. ports 배열

ports 배열은 레이아웃에 존재하는 모든 포트를 별도 목록으로 저장한다.

```json
{
  "id": "CV_001_OUT",
  "componentId": "CV_001",
  "direction": "out",
  "localPosition": { "x": 3, "y": 0, "z": 0 },
  "capacity": 1,
  "isBlockingPoint": true,
  "label": "OUT"
}
```

검증 규칙:

```text
- id는 ports 배열 안에서 고유해야 한다.
- componentId는 components 배열에 존재해야 한다.
- direction은 in, out, pick, place 중 하나여야 한다.
- components.ports에 참조된 모든 포트는 ports 배열에 존재해야 한다.
- ports 배열에 존재하는 모든 포트는 하나의 componentId를 가져야 한다.
```

---

## 9. connections 배열

connections는 포트와 포트 사이의 연결 관계를 정의한다.

```json
{
  "id": "CONN_001",
  "from": "CV_001_OUT",
  "to": "DIV_001_IN",
  "type": "transport",
  "bidirectional": false,
  "properties": {
    "travelTimeOverride": null,
    "enabled": true
  },
  "visual": {
    "points": [
      { "x": 6, "y": 0, "z": 0 },
      { "x": 7, "y": 0, "z": 0 }
    ]
  }
}
```

검증 규칙:

```text
- from 포트는 ports 배열에 존재해야 한다.
- to 포트는 ports 배열에 존재해야 한다.
- from 포트의 direction은 out 또는 place여야 한다.
- to 포트의 direction은 in 또는 pick이어야 한다.
- 동일한 id의 connection이 중복되면 안 된다.
- enabled가 false인 connection은 routing에서 제외한다.
```

---

## 10. routes 배열

routes는 아이템이 출발지에서 목적지까지 이동할 수 있는 경로를 정의한다.

```json
{
  "id": "ROUTE_BOX_A",
  "name": "BOX_A to Pallet 1",
  "itemType": "BOX_A",
  "source": "SRC_001",
  "destination": "PALLET_001",
  "path": [
    "SRC_001",
    "CV_001",
    "DIV_001",
    "CV_002",
    "RB_001",
    "PALLET_001",
    "SINK_001"
  ],
  "connectionPath": [
    "CONN_001",
    "CONN_002",
    "CONN_003"
  ],
  "enabled": true
}
```

MVP에서는 다음 원칙을 따른다.

```text
1. Route는 수동 정의한다.
2. itemType 또는 destination 기준으로 Route를 선택한다.
3. 자동 최단 경로 탐색은 Plan_05에서 고도화한다.
```

---

## 11. rules 구조

```json
{
  "rules": {
    "defaultBlockingRule": "downstreamBlocking",
    "defaultMergeRule": "fifo",
    "defaultDiverterRule": "byDestination",
    "defaultBufferRule": "fifo",
    "allowAlternativeRoute": false,
    "deadlockPolicy": "stopAndReport"
  }
}
```

MVP 기본값:

```text
defaultBlockingRule: downstreamBlocking
defaultMergeRule: fifo
defaultDiverterRule: byDestination
defaultBufferRule: fifo
allowAlternativeRoute: false
deadlockPolicy: stopAndReport
```

---

## 12. visualSettings 구조

visualSettings는 Three.js 화면 표시를 위한 설정이다.
DES Engine은 이 정보를 계산에 사용하지 않는다.

```json
{
  "visualSettings": {
    "grid": {
      "visible": true,
      "size": 50,
      "division": 50
    },
    "drawingBackground": {
      "enabled": true,
      "imageUrl": "/drawings/sample_layout.png",
      "opacity": 0.35,
      "locked": true,
      "scale": 1
    },
    "labels": {
      "componentLabelVisible": true,
      "portLabelVisible": false
    }
  }
}
```

---

## 13. TypeScript 타입 초안

```ts
export interface LayoutJson {
  schemaVersion: string;
  project: ProjectInfo;
  coordinateSystem: CoordinateSystem;
  components: LayoutComponent[];
  ports: LayoutPort[];
  connections: LayoutConnection[];
  routes: LayoutRoute[];
  rules: LayoutRules;
  visualSettings?: VisualSettings;
  metadata?: LayoutMetadata;
}

export interface Vector3Like {
  x: number;
  y: number;
  z: number;
}

export interface ProjectInfo {
  id: string;
  name: string;
  description?: string;
  author?: string;
  createdAt?: string;
  updatedAt?: string;
}

export interface CoordinateSystem {
  unit: "meter";
  timeUnit: "second";
  angleUnit: "degree" | "radian";
  axis: {
    horizontal: "x";
    vertical: "y";
    depth: "z";
  };
  topViewPlane: "xz";
  scale: {
    drawingUnitPerMeter: number;
  };
}

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

export interface LayoutComponent {
  id: string;
  type: ComponentType;
  name?: string;
  description?: string;
  position: Vector3Like;
  rotation: Vector3Like;
  dimensions?: Record<string, number>;
  ports: {
    in: string[];
    out: string[];
  };
  properties: Record<string, unknown>;
  visual?: Record<string, unknown>;
  metadata?: Record<string, unknown>;
}

export type PortDirection = "in" | "out" | "pick" | "place";

export interface LayoutPort {
  id: string;
  componentId: string;
  direction: PortDirection;
  localPosition?: Vector3Like;
  capacity?: number;
  isBlockingPoint?: boolean;
  label?: string;
}

export type ConnectionType = "transport" | "control" | "virtual";

export interface LayoutConnection {
  id: string;
  from: string;
  to: string;
  type: ConnectionType;
  bidirectional?: boolean;
  properties?: {
    travelTimeOverride?: number | null;
    enabled?: boolean;
  };
  visual?: {
    points?: Vector3Like[];
  };
}

export interface LayoutRoute {
  id: string;
  name?: string;
  itemType?: string;
  source: string;
  destination: string;
  path: string[];
  connectionPath?: string[];
  enabled?: boolean;
}
```

---

## 14. Layout 검증 전략

Layout JSON은 저장 전과 시뮬레이션 실행 전에 반드시 검증한다.

```text
1. 최상위 구조 검증
2. project 검증
3. coordinateSystem 검증
4. component ID 중복 검증
5. port ID 중복 검증
6. component와 port 참조 검증
7. connection의 from/to 포트 검증
8. route의 path 검증
9. component type별 properties 검증
10. 순환 또는 끊어진 연결 검증
```

검증 오류 반환 구조:

```ts
export interface ValidationError {
  code: string;
  message: string;
  path: string;
  severity: "error" | "warning";
}

export interface ValidationResult {
  valid: boolean;
  errors: ValidationError[];
  warnings: ValidationError[];
}
```

---

## 15. 저장과 불러오기 원칙

저장 원칙:

```text
- 저장 시 schemaVersion을 포함한다.
- components, ports, connections, routes 순서로 저장한다.
- 사람이 읽기 쉽도록 pretty JSON 형식으로 저장한다.
- updatedAt을 갱신한다.
```

불러오기 원칙:

```text
- schemaVersion을 먼저 확인한다.
- 현재 버전과 다르면 migration 가능 여부를 확인한다.
- JSON 구조를 검증한다.
- 검증에 실패하면 시뮬레이션 실행을 막는다.
- warning만 있으면 실행은 허용하되 사용자에게 표시한다.
```

---

## 16. 작업 목록

### 16.1 타입 정의 작업

* [ ] `src/types/layout.ts` 생성
* [ ] `LayoutJson` 타입 정의
* [ ] `ProjectInfo` 타입 정의
* [ ] `CoordinateSystem` 타입 정의
* [ ] `LayoutComponent` 타입 정의
* [ ] `LayoutPort` 타입 정의
* [ ] `LayoutConnection` 타입 정의
* [ ] `LayoutRoute` 타입 정의
* [ ] `LayoutRules` 타입 정의
* [ ] `ValidationError` 타입 정의
* [ ] `ValidationResult` 타입 정의

### 16.2 샘플 데이터 작업

* [ ] `src/data/samples/mvpLayout.ts` 생성
* [ ] Source → Conveyor → Robot → Pallet Station → Sink 샘플 작성
* [ ] 분기 테스트용 Source → Conveyor → Diverter → Robot 2대 샘플 작성
* [ ] JSON export 가능한 구조인지 확인

### 16.3 검증 함수 작업

* [ ] `src/data/validators/layoutValidator.ts` 생성
* [ ] `validateLayout(layout)` 구현
* [ ] `validateProject(project)` 구현
* [ ] `validateComponents(components)` 구현
* [ ] `validatePorts(ports, components)` 구현
* [ ] `validateConnections(connections, ports)` 구현
* [ ] `validateRoutes(routes, components, connections)` 구현
* [ ] component type별 properties 검증 구현

---

## 17. 완료 기준

```text
- Layout JSON의 전체 구조가 정의되어 있다.
- project 구조가 정의되어 있다.
- coordinateSystem 구조가 정의되어 있다.
- components 구조가 정의되어 있다.
- ports 구조가 정의되어 있다.
- connections 구조가 정의되어 있다.
- routes 구조가 정의되어 있다.
- rules 구조가 정의되어 있다.
- visualSettings와 metadata 구조가 정의되어 있다.
- MVP layout.json 전체 예시가 작성되어 있다.
- TypeScript 타입 초안이 작성되어 있다.
- 검증 전략과 오류 반환 구조가 정의되어 있다.
- Codex가 타입, 샘플, 검증 함수를 구현할 수 있는 수준이다.
```

---

## 18. Codex 작업 지시 예시

```text
MasterPlan.md, Plan_01.md, Plan_02.md, Plan_03.md를 읽고,
src/types/layout.ts 파일을 만들어줘.

Plan_03.md의 TypeScript 타입 초안을 기준으로 다음 타입을 구현해.
- LayoutJson
- ProjectInfo
- CoordinateSystem
- LayoutComponent
- LayoutPort
- LayoutConnection
- LayoutRoute
- LayoutRules
- ValidationError
- ValidationResult

아직 Three.js 화면과 시뮬레이션 엔진은 구현하지 마.
```

---

## 19. 다음 단계

Plan_03이 완료되면 다음 문서로 이동한다.

```text
Plan_04.md
```

Plan_04에서는 Layout JSON을 실제로 만들고 수정할 수 있는 **Three.js Layout Editor**를 정의한다.
::: 
