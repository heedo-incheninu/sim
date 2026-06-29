# Plan_06.md

# Plan_06. Input Scenario Data Definition

## 1. 목적

이 문서는 웹 기반 물류 라인 이산 사건 시뮬레이션에서 사용할 **Input Scenario Data**를 정의하기 위한 실행 계획서이다.

Plan_05.md에서 설비 네트워크의 길 안내 체계인 Network Routing을 정의했다면, Plan_06.md는 그 길 위에 **언제, 어떤 아이템을, 얼마나, 어디로 흘려보낼 것인가**를 정의한다.

이 문서의 핵심 목적은 다음과 같다.

```text
1. Scenario JSON의 전체 구조를 정의한다.
2. Item Master, Arrival Schedule, Process Time, Rule Set의 저장 방식을 정의한다.
3. 시간대별 물동량 입력 방식을 정의한다.
4. 제품별 source, destination, routeId 매핑 방식을 정의한다.
5. CSV 업로드 형식을 정의한다.
6. DES Simulation Engine이 읽을 수 있는 시나리오 데이터 기준을 만든다.
7. Codex가 scenario 타입, 샘플 데이터, 검증 함수를 구현할 수 있도록 명세를 제공한다.
```

Layout JSON이 설비 세계의 지도라면, Scenario JSON은 그 지도 위를 달릴 박스들의 출발 시간표다.
지도만 있으면 설비는 조용한 모형이고, 시나리오가 들어와야 비로소 물류가 움직인다.

---

## 2. Scenario JSON의 역할

Scenario JSON은 시뮬레이션 실행 조건을 저장한다.

포함하는 정보는 다음과 같다.

```text
1. 시뮬레이션 시간 범위
2. 아이템 종류
3. 아이템 크기와 중량
4. 아이템별 출발지와 목적지
5. 시간대별 투입량
6. 투입 패턴
7. routeId 지정
8. 공정 처리 시간
9. 로봇 cycle time override
10. 분기, 합류, 버퍼 운영 규칙 override
11. 랜덤 시드와 실행 옵션
```

Scenario JSON은 설비 배치 자체를 저장하지 않는다.

```text
설비 위치와 연결: layout.json
물동량과 운영 조건: scenario.json
시뮬레이션 결과: simulation_result.json
```

---

## 3. Scenario JSON 전체 구조

MVP 기준 Scenario JSON의 최상위 구조는 다음과 같다.

```json
{
  "schemaVersion": "1.0.0",
  "scenario": {},
  "simulation": {},
  "items": [],
  "arrivalSchedules": [],
  "processOverrides": [],
  "resourceOverrides": [],
  "rules": {},
  "csvImport": {},
  "metadata": {}
}
```

각 영역의 역할은 다음과 같다.

| 영역                | 역할                         |
| ----------------- | -------------------------- |
| schemaVersion     | Scenario JSON Schema 버전    |
| scenario          | 시나리오 이름, 설명, 작성자           |
| simulation        | 시뮬레이션 시작, 종료, 시간 단위, 랜덤 시드 |
| items             | 아이템 마스터                    |
| arrivalSchedules  | 시간대별 투입 스케줄                |
| processOverrides  | 공정별 처리시간 override          |
| resourceOverrides | 로봇 등 resource 조건 override  |
| rules             | 시나리오 단위 운영 규칙              |
| csvImport         | CSV 입력 관련 설정               |
| metadata          | 태그, 수정 이력, 비고              |

---

## 4. simulation 구조

```json
{
  "simulation": {
    "startTime": 0,
    "endTime": 3600,
    "timeUnit": "second",
    "warmupTime": 0,
    "randomSeed": 1,
    "maxEvents": 100000,
    "stopOnDeadlock": true,
    "enableAnimationLog": true
  }
}
```

MVP 기본값:

```text
startTime: 0
endTime: 3600
timeUnit: second
warmupTime: 0
randomSeed: 1
maxEvents: 100000
stopOnDeadlock: true
enableAnimationLog: true
```

---

## 5. Item Master

Item Master는 시뮬레이션에서 생성될 아이템 종류를 정의한다.

```json
{
  "id": "BOX_A",
  "name": "Box A",
  "category": "box",
  "dimensions": {
    "length": 0.4,
    "width": 0.3,
    "height": 0.25
  },
  "weight": 5,
  "source": "SRC_001",
  "destination": "PALLET_001",
  "routeId": "ROUTE_BOX_A",
  "priority": 1,
  "visual": {
    "color": "#D9A441"
  },
  "metadata": {
    "sku": "SKU_A"
  }
}
```

검증 규칙:

```text
- id는 items 배열 안에서 고유해야 한다.
- category는 허용된 값이어야 한다.
- dimensions.length, width, height는 0보다 커야 한다.
- source가 지정되면 Layout JSON의 components에 존재해야 한다.
- destination이 지정되면 Layout JSON의 components에 존재해야 한다.
- routeId가 지정되면 Layout JSON의 routes에 존재해야 한다.
```

---

## 6. Arrival Schedule

Arrival Schedule은 특정 시간대에 어떤 아이템을 얼마나 투입할지 정의한다.

```json
{
  "id": "ARR_001",
  "name": "BOX_A uniform input",
  "enabled": true,
  "source": "SRC_001",
  "itemType": "BOX_A",
  "destination": "PALLET_001",
  "routeId": "ROUTE_BOX_A",
  "startTime": 0,
  "endTime": 3600,
  "quantity": 600,
  "arrivalPattern": "uniform",
  "priority": 1
}
```

검증 규칙:

```text
- id는 arrivalSchedules 안에서 고유해야 한다.
- enabled가 false이면 DES Engine은 해당 스케줄을 무시한다.
- source는 Layout JSON의 source component여야 한다.
- itemType은 items 배열에 존재해야 한다.
- startTime은 endTime보다 작아야 한다.
- startTime과 endTime은 simulation 범위 안에 있어야 한다.
- quantity는 0보다 커야 한다.
- arrivalPattern은 허용된 값이어야 한다.
```

---

## 7. Arrival Pattern

지원 패턴:

```text
uniform
- 일정 간격으로 투입

batch
- 특정 시간에 묶음 투입

poisson
- 평균 도착률 기반 확률 투입

csv
- 외부 CSV 시간표 기반 투입

manual
- 사용자가 직접 지정한 개별 투입 이벤트
```

MVP에서는 `uniform`, `batch`, `csv`를 우선 정의하고, 구현은 `uniform`부터 시작한다.

### 7.1 Uniform Pattern

```json
{
  "arrivalPattern": "uniform",
  "startTime": 0,
  "endTime": 3600,
  "quantity": 600
}
```

계산:

```text
duration = endTime - startTime
interval = duration / quantity
```

예시:

```text
3600초 / 600개 = 6초 간격
```

생성 이벤트:

```text
0초: BOX_A_001
6초: BOX_A_002
12초: BOX_A_003
...
```

### 7.2 Batch Pattern

```json
{
  "arrivalPattern": "batch",
  "batches": [
    { "time": 0, "quantity": 50 },
    { "time": 600, "quantity": 50 },
    { "time": 1200, "quantity": 100 }
  ]
}
```

### 7.3 CSV Pattern

```csv
time,source,itemType,destination,routeId,quantity,priority
0,SRC_001,BOX_A,PALLET_001,ROUTE_BOX_A,1,1
6,SRC_001,BOX_A,PALLET_001,ROUTE_BOX_A,1,1
12,SRC_001,BOX_A,PALLET_001,ROUTE_BOX_A,1,1
```

---

## 8. Arrival Event 변환

DES Engine은 arrivalSchedule을 직접 처리하기보다, 시뮬레이션 시작 전에 Arrival Event 목록으로 변환하는 것이 좋다.

```ts
export interface ArrivalEventInput {
  time: number;
  source: string;
  itemType: string;
  destination?: string;
  routeId?: string;
  quantity: number;
  priority?: number;
  scheduleId: string;
}
```

변환 흐름:

```text
1. arrivalSchedules 배열을 읽는다.
2. enabled가 false인 schedule은 제외한다.
3. arrivalPattern별로 이벤트를 생성한다.
4. 생성된 이벤트를 time 기준으로 정렬한다.
5. 같은 time이면 sequence를 부여한다.
6. DES Engine의 Event Queue에 ITEM_CREATE_REQUEST 이벤트로 등록한다.
```

---

## 9. Resource Overrides

Resource Overrides는 Robot 같은 resource 설비의 시나리오별 조건을 덮어쓰기 위한 구조다.

```json
{
  "resourceOverrides": [
    {
      "componentId": "RB_001",
      "resourceType": "robot",
      "cycleTime": 8,
      "availability": 1.0,
      "enabled": true
    }
  ]
}
```

검증 규칙:

```text
- componentId는 Layout JSON에 존재해야 한다.
- resourceType이 robot이면 대상 component는 robot이어야 한다.
- cycleTime이 있으면 0보다 커야 한다.
- availability는 0보다 크고 1 이하이어야 한다.
```

---

## 10. Scenario Rules

Scenario Rules는 Layout JSON의 기본 rule을 시나리오별로 덮어쓰기 위한 구조다.

```json
{
  "rules": {
    "mergeRuleOverride": "fifo",
    "diverterRuleOverride": "byDestination",
    "bufferRuleOverride": "fifo",
    "blockingRuleOverride": "downstreamBlocking",
    "allowAlternativeRoute": false,
    "deadlockPolicy": "stopAndReport"
  }
}
```

규칙 적용 우선순위:

```text
1. Component properties에 직접 지정된 rule
2. Scenario rules override
3. Layout JSON rules 기본값
4. 시스템 기본값
```

---

## 11. CSV 입력 형식

### 11.1 Arrival Schedule CSV

```csv
scheduleId,source,itemType,destination,routeId,startTime,endTime,quantity,arrivalPattern,priority
ARR_001,SRC_001,BOX_A,PALLET_001,ROUTE_BOX_A,0,3600,600,uniform,1
ARR_002,SRC_001,BOX_B,PALLET_002,ROUTE_BOX_B,0,3600,300,uniform,1
```

### 11.2 Arrival Event CSV

```csv
time,source,itemType,destination,routeId,quantity,priority
0,SRC_001,BOX_A,PALLET_001,ROUTE_BOX_A,1,1
6,SRC_001,BOX_A,PALLET_001,ROUTE_BOX_A,1,1
12,SRC_001,BOX_A,PALLET_001,ROUTE_BOX_A,1,1
```

### 11.3 Item Master CSV

```csv
itemType,name,category,length,width,height,weight,defaultSource,defaultDestination,defaultRouteId,priority
BOX_A,Box A,box,0.4,0.3,0.25,5,SRC_001,PALLET_001,ROUTE_BOX_A,1
BOX_B,Box B,box,0.5,0.35,0.3,7,SRC_001,PALLET_002,ROUTE_BOX_B,1
```

---

## 12. MVP scenario.json 전체 예시

```json
{
  "schemaVersion": "1.0.0",
  "scenario": {
    "id": "SCN_MVP_001",
    "name": "MVP Uniform Palletizing Scenario",
    "description": "600 boxes per hour uniform input for MVP palletizing line",
    "author": "SFA",
    "createdAt": "2026-06-29T00:00:00.000Z",
    "updatedAt": "2026-06-29T00:00:00.000Z"
  },
  "simulation": {
    "startTime": 0,
    "endTime": 3600,
    "timeUnit": "second",
    "warmupTime": 0,
    "randomSeed": 1,
    "maxEvents": 100000,
    "stopOnDeadlock": true,
    "enableAnimationLog": true
  },
  "items": [
    {
      "id": "BOX_A",
      "name": "Box A",
      "category": "box",
      "dimensions": {
        "length": 0.4,
        "width": 0.3,
        "height": 0.25
      },
      "weight": 5,
      "source": "SRC_001",
      "destination": "PALLET_001",
      "routeId": "ROUTE_BOX_A",
      "priority": 1,
      "visual": {
        "color": "#D9A441"
      },
      "metadata": {
        "sku": "SKU_A"
      }
    }
  ],
  "arrivalSchedules": [
    {
      "id": "ARR_001",
      "name": "BOX_A uniform input",
      "enabled": true,
      "source": "SRC_001",
      "itemType": "BOX_A",
      "destination": "PALLET_001",
      "routeId": "ROUTE_BOX_A",
      "startTime": 0,
      "endTime": 3600,
      "quantity": 600,
      "arrivalPattern": "uniform",
      "priority": 1
    }
  ],
  "processOverrides": [],
  "resourceOverrides": [
    {
      "componentId": "RB_001",
      "resourceType": "robot",
      "cycleTime": 8,
      "availability": 1,
      "enabled": true
    }
  ],
  "rules": {
    "mergeRuleOverride": "fifo",
    "diverterRuleOverride": "byDestination",
    "bufferRuleOverride": "fifo",
    "blockingRuleOverride": "downstreamBlocking",
    "allowAlternativeRoute": false,
    "deadlockPolicy": "stopAndReport"
  },
  "csvImport": {
    "enabled": false,
    "sourceType": null,
    "lastImportedFileName": null
  },
  "metadata": {
    "tags": ["mvp", "uniform", "palletizing"],
    "revision": 1,
    "notes": "Initial MVP scenario"
  }
}
```

---

## 13. TypeScript 타입 초안

```ts
export interface ScenarioJson {
  schemaVersion: string;
  scenario: ScenarioInfo;
  simulation: SimulationConfig;
  items: ItemMaster[];
  arrivalSchedules: ArrivalSchedule[];
  processOverrides?: ProcessOverride[];
  resourceOverrides?: ResourceOverride[];
  rules?: ScenarioRules;
  csvImport?: CsvImportInfo;
  metadata?: ScenarioMetadata;
}

export interface ScenarioInfo {
  id: string;
  name: string;
  description?: string;
  author?: string;
  createdAt?: string;
  updatedAt?: string;
}

export interface SimulationConfig {
  startTime: number;
  endTime: number;
  timeUnit: "second";
  warmupTime?: number;
  randomSeed?: number;
  maxEvents?: number;
  stopOnDeadlock?: boolean;
  enableAnimationLog?: boolean;
}

export type ItemCategory = "box" | "tray" | "pallet" | "bundle";

export interface ItemMaster {
  id: string;
  name?: string;
  category: ItemCategory;
  dimensions: {
    length: number;
    width: number;
    height: number;
  };
  weight?: number;
  source?: string;
  destination?: string;
  routeId?: string;
  priority?: number;
  visual?: Record<string, unknown>;
  metadata?: Record<string, unknown>;
}

export type ArrivalPattern =
  | "uniform"
  | "batch"
  | "poisson"
  | "csv"
  | "manual";

export interface ArrivalSchedule {
  id: string;
  name?: string;
  enabled?: boolean;
  source: string;
  itemType: string;
  destination?: string;
  routeId?: string;
  startTime: number;
  endTime: number;
  quantity?: number;
  arrivalPattern: ArrivalPattern;
  priority?: number;
  batches?: BatchArrival[];
  ratePerHour?: number;
  csvSourceId?: string;
}

export interface BatchArrival {
  time: number;
  quantity: number;
}

export interface ArrivalEventInput {
  time: number;
  source: string;
  itemType: string;
  destination?: string;
  routeId?: string;
  quantity: number;
  priority?: number;
  scheduleId: string;
  sequence: number;
}

export interface ResourceOverride {
  componentId: string;
  resourceType: "robot" | "worker" | "machine";
  cycleTime?: number;
  availability?: number;
  enabled?: boolean;
}
```

---

## 14. Scenario 검증 전략

```text
1. 최상위 구조 검증
2. scenario.id, scenario.name 검증
3. simulation 시간 범위 검증
4. item ID 중복 검증
5. item dimensions 검증
6. item source, destination, routeId 참조 검증
7. arrivalSchedule ID 중복 검증
8. arrivalSchedule 시간 범위 검증
9. arrivalSchedule source, itemType, destination, routeId 참조 검증
10. arrivalPattern별 필수 필드 검증
11. processOverrides 대상 component 검증
12. resourceOverrides 대상 component 검증
13. rules 값 검증
```

오류 예시:

```json
{
  "code": "ITEM_TYPE_NOT_FOUND",
  "message": "Arrival schedule ARR_001 references missing itemType BOX_X.",
  "path": "arrivalSchedules[0].itemType",
  "severity": "error"
}
```

---

## 15. DES Engine과의 연결

DES Engine은 Scenario JSON을 다음 순서로 처리한다.

```text
1. Layout JSON 검증
2. Scenario JSON 검증
3. RoutingGraph 생성
4. Route 선택 가능 여부 검증
5. Arrival Schedule을 Arrival Event 목록으로 변환
6. Event Queue에 ITEM_CREATE_REQUEST 이벤트 등록
7. Resource Override 적용
8. Process Override 적용
9. 시뮬레이션 시작
```

---

## 16. MVP 구현 범위

### 포함

```text
- Scenario JSON 전체 구조
- SimulationConfig
- Item Master
- Uniform Arrival Schedule
- Batch Arrival Schedule 구조
- CSV Arrival 형식 정의
- Resource Override 중 Robot cycleTime
- Scenario Rules 기본값
- Arrival Schedule을 Arrival Event로 변환
- Scenario Validator
- MVP scenario 샘플
```

### 제외

```text
- Poisson 실제 구현
- Shift와 Break 반영
- 설비 고장 시나리오
- 작업자 스케줄
- 제품별 복잡한 우선순위
- 확률 기반 불량률
- 재작업 루프 자동 생성
```

---

## 17. 작업 목록

```text
타입 정의 작업
- src/types/scenario.ts 생성
- ScenarioJson 타입 정의
- ScenarioInfo 타입 정의
- SimulationConfig 타입 정의
- ItemMaster 타입 정의
- ArrivalSchedule 타입 정의
- BatchArrival 타입 정의
- ArrivalEventInput 타입 정의
- ResourceOverride 타입 정의
- ScenarioRules 타입 정의

샘플 데이터 작업
- src/data/samples/mvpScenario.ts 생성
- BOX_A Item Master 작성
- 1시간 600개 uniform arrival 작성
- RB_001 cycleTime override 작성
- Scenario rules 기본값 작성

변환 함수 작업
- src/simulation/scenario/createArrivalEvents.ts 생성
- uniform schedule 변환 구현
- batch schedule 변환 구조 작성
- csv schedule 변환 인터페이스 작성
- Arrival Event 시간순 정렬
- sequence 부여

검증 함수 작업
- src/data/validators/scenarioValidator.ts 생성
- validateScenario(scenario, layout) 구현
- simulation 시간 범위 검증
- item ID 중복 검증
- item dimensions 검증
- arrivalSchedule 참조 검증
- arrivalPattern별 필수 필드 검증
- resourceOverride 검증
```

---

## 18. 완료 기준

```text
- Scenario JSON 전체 구조가 정의되어 있다.
- SimulationConfig 구조가 정의되어 있다.
- Item Master 구조가 정의되어 있다.
- Arrival Schedule 구조가 정의되어 있다.
- Arrival Pattern별 처리 방식이 정의되어 있다.
- Arrival Event 변환 방식이 정의되어 있다.
- Process Override와 Resource Override 구조가 정의되어 있다.
- Scenario Rules 구조가 정의되어 있다.
- CSV 입력 형식이 정의되어 있다.
- MVP scenario.json 예시가 작성되어 있다.
- TypeScript 타입 초안이 작성되어 있다.
- 검증 전략이 정의되어 있다.
- Codex가 scenario 모듈을 구현할 수 있는 수준이다.
```

---

## 19. Codex 작업 지시 예시

```text
MasterPlan.md, Plan_01.md, Plan_02.md, Plan_03.md, Plan_04.md, Plan_05.md, Plan_06.md를 읽고,
src/types/scenario.ts 파일을 만들어줘.

Plan_06.md의 TypeScript 타입 초안을 기준으로 다음 타입을 구현해.
- ScenarioJson
- ScenarioInfo
- SimulationConfig
- ItemMaster
- ArrivalSchedule
- BatchArrival
- ArrivalEventInput
- ProcessOverride
- ResourceOverride
- ScenarioRules

아직 DES Engine 본체는 구현하지 마.
```

---

## 20. 다음 단계

Plan_06이 완료되면 다음 문서로 이동한다.

```text
Plan_07.md
```

Plan_07에서는 Layout JSON, Routing Graph, Scenario JSON을 실제로 읽어서 이산 사건 시뮬레이션을 수행하는 **DES Simulation Engine**을 정의한다.
::: 
