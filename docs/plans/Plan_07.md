# Plan_07.md

# Plan_07. DES Simulation Engine Definition

## 1. 목적

이 문서는 웹 기반 물류 라인 이산 사건 시뮬레이션에서 사용할 **DES Simulation Engine**을 정의하기 위한 실행 계획서이다.

Plan_06.md에서 Scenario JSON과 Arrival Schedule을 정의했다면, Plan_07.md는 Layout JSON, Routing Graph, Scenario JSON을 실제로 읽어서 시간순 사건을 처리하는 시뮬레이션 엔진을 설계한다.

이 문서의 핵심 목적은 다음과 같다.

```text id="mjsxeu"
1. 이산 사건 시뮬레이션 엔진의 전체 구조를 정의한다.
2. Simulation Clock과 Event Queue의 역할을 정의한다.
3. Item Runtime State와 Component Runtime State를 정의한다.
4. Source, Conveyor, Diverter, Merge, Buffer, Robot, Pallet Station, Sink의 처리 로직을 정의한다.
5. Blocking, Waiting, Deadlock 처리 기준을 정의한다.
6. Statistics Collector가 어떤 지표를 계산할지 정의한다.
7. Event Log와 State Timeline 생성을 위한 엔진 출력 기준을 만든다.
8. Codex가 DES Engine 모듈을 구현할 수 있는 수준의 명세를 제공한다.
```

이 엔진은 프로젝트의 두뇌다.
Three.js는 움직임을 보여주는 무대이고, DES Engine은 박스와 로봇의 운명을 시간표로 계산하는 작은 관제실이다.

---

## 2. DES Engine의 기본 개념

이산 사건 시뮬레이션은 시간을 일정 간격으로 계속 증가시키는 방식이 아니라, 의미 있는 사건이 발생하는 시점으로 시간을 점프시키는 방식이다.

예시:

```text id="hfj6l9"
0초: BOX_A_000001 생성
0초: Source에서 Conveyor로 진입 시도
10초: Conveyor 출구 도착
10초: Robot 픽업 가능 여부 확인
10초: Robot 작업 시작
18초: Robot 작업 완료
18초: Pallet Station 적재 완료
```

중간의 1초, 2초, 3초를 매번 계산하지 않는다.
중요한 사건이 생기는 시간으로 바로 이동한다.

---

## 3. DES Engine 전체 구조

엔진은 다음 모듈로 구성한다.

```text id="ny6nbv"
SimulationRunner
- 전체 시뮬레이션 실행을 지휘한다.

SimulationClock
- 현재 시뮬레이션 시간을 관리한다.

EventQueue
- 앞으로 발생할 이벤트를 시간순으로 관리한다.

EventDispatcher
- 이벤트 타입에 따라 적절한 handler를 호출한다.

ItemManager
- 런타임 아이템 상태를 관리한다.

ComponentStateManager
- 설비별 상태, 대기열, 점유 상태를 관리한다.

RoutingService
- 다음 컴포넌트, 분기, 합류 판단을 담당한다.

ResourceManager
- Robot, Worker, Machine 같은 resource 점유를 관리한다.

StatisticsCollector
- 처리량, 대기시간, 가동률, WIP, blocked time을 계산한다.

EventLogger
- Event Log와 Item Timeline을 기록한다.

StateTimelineRecorder
- Component State Timeline을 기록한다.

DeadlockDetector
- 더 이상 진행할 수 없는 상태를 감지한다.
```

전체 실행 흐름:

```text id="6pky7k"
1. Layout JSON 검증
2. Scenario JSON 검증
3. RoutingGraph 생성
4. Arrival Schedule을 Arrival Event로 변환
5. Runtime State 초기화
6. Event Queue에 ITEM_CREATE_REQUEST 이벤트 등록
7. Event Queue에서 가장 빠른 이벤트를 꺼냄
8. SimulationClock을 이벤트 시간으로 이동
9. Event Handler 실행
10. Runtime State 변경
11. 필요한 다음 이벤트를 Event Queue에 등록
12. Event Log와 State Timeline 기록
13. 종료 조건까지 반복
14. Summary Statistics 생성
```

---

## 4. 입력과 출력

입력:

```ts id="g1ipsb"
export interface SimulationInput {
  layout: LayoutJson;
  scenario: ScenarioJson;
}
```

출력:

```ts id="m4ymea"
export interface SimulationResult {
  summary: SimulationSummary;
  eventLog: SimulationEventLog[];
  itemTimelines: ItemTimeline[];
  componentStateTimelines: ComponentStateTimeline[];
  componentStats: ComponentStatistics[];
  itemStats: ItemStatistics[];
  warnings: SimulationWarning[];
  errors: SimulationError[];
}
```

주요 출력 항목:

```text id="f3dewo"
summary
- 전체 처리량, 완료 수량, 평균 리드타임 등 요약

eventLog
- 모든 주요 이벤트의 시간순 기록

itemTimelines
- 아이템별 이동 이력

componentStateTimelines
- 설비별 상태 변화 이력

componentStats
- 설비별 busy, idle, blocked, waiting 시간

itemStats
- 아이템별 lead time, waiting time, 완료 여부
```

---

## 5. Simulation Clock

SimulationClock은 현재 시뮬레이션 시간을 관리한다.

```ts id="pg2z4b"
export interface SimulationClock {
  currentTime: number;
  startTime: number;
  endTime: number;
}
```

동작 원칙:

```text id="mya4sc"
1. currentTime은 항상 처리 중인 이벤트의 time으로 이동한다.
2. currentTime은 뒤로 가지 않는다.
3. currentTime이 endTime을 초과하면 일반 이벤트 처리를 중지한다.
4. MVP에서는 currentTime > endTime인 이벤트는 처리하지 않는다.
5. endTime 시점의 WIP는 summary에 기록한다.
```

---

## 6. Event Queue

Event Queue는 앞으로 발생할 이벤트를 시간순으로 저장한다.

```ts id="76j9sy"
export interface SimulationEvent {
  id: string;
  time: number;
  sequence: number;
  type: SimulationEventType;
  itemId?: string;
  componentId?: string;
  portId?: string;
  payload?: Record<string, unknown>;
}
```

MVP에서 사용할 이벤트 타입:

```ts id="uxyk43"
export type SimulationEventType =
  | "ITEM_CREATE_REQUEST"
  | "ITEM_CREATED"
  | "ITEM_RELEASED"
  | "ITEM_ENTER_COMPONENT"
  | "ITEM_EXIT_COMPONENT"
  | "ITEM_WAIT_START"
  | "ITEM_WAIT_END"
  | "BLOCKING_START"
  | "BLOCKING_END"
  | "DIVERTER_ROUTE_DECIDED"
  | "MERGE_RELEASE_DECIDED"
  | "PROCESS_START"
  | "PROCESS_END"
  | "ROBOT_PICK_START"
  | "ROBOT_PICK_END"
  | "ROBOT_PLACE_START"
  | "ROBOT_PLACE_END"
  | "PALLETIZE_COMPLETE"
  | "PALLET_COMPLETE"
  | "ITEM_COMPLETED"
  | "DEADLOCK_DETECTED"
  | "SIMULATION_END";
```

정렬 규칙:

```text id="hib6s6"
1. time 오름차순
2. sequence 오름차순
3. event priority
```

Event Queue API:

```ts id="ft280f"
export interface EventQueue {
  push(event: SimulationEvent): void;
  pop(): SimulationEvent | undefined;
  peek(): SimulationEvent | undefined;
  isEmpty(): boolean;
  size(): number;
  clear(): void;
}
```

---

## 7. Runtime Item State

```ts id="fddpco"
export interface SimulationItem {
  id: string;
  itemType: string;
  dimensions: {
    length: number;
    width: number;
    height: number;
  };
  weight?: number;
  source: string;
  destination?: string;
  routeId?: string;
  routeState?: ItemRouteState;
  priority?: number;
  createdAt: number;
  completedAt?: number;
  currentComponentId?: string;
  currentPortId?: string;
  currentState:
    | "created"
    | "released"
    | "moving"
    | "waiting"
    | "processing"
    | "blocked"
    | "completed";
  waitingStartedAt?: number;
  totalWaitingTime: number;
  totalProcessingTime: number;
  metadata?: Record<string, unknown>;
}
```

상태 설명:

```text id="hdm26r"
created
- 아이템이 생성되었지만 아직 설비에 진입하지 않은 상태

released
- Source에서 다음 설비로 이동을 시도한 상태

moving
- 컨베이어 또는 연결 경로를 이동 중인 상태

waiting
- 하류 설비나 resource를 기다리는 상태

processing
- 로봇, 검사기 등에서 처리 중인 상태

blocked
- 경로가 막혀 이동하지 못하는 상태

completed
- Sink 또는 완료 지점에 도착한 상태
```

---

## 8. Runtime Component State

```ts id="n8n8kk"
export interface ComponentRuntimeState {
  componentId: string;
  componentType: ComponentType;
  state: "idle" | "busy" | "waiting" | "blocked" | "disabled" | "failed";
  currentItemIds: string[];
  queueItemIds: string[];
  inputQueues?: Record<string, string[]>;
  outputBlocked?: boolean;
  blockedReason?: string;
  lastStateChangedAt: number;
  busyStartedAt?: number;
  totalBusyTime: number;
  totalIdleTime: number;
  totalWaitingTime: number;
  totalBlockedTime: number;
  processedCount: number;
}
```

Merge처럼 입력 포트가 여러 개인 컴포넌트는 inputQueues를 사용한다.

```ts id="8pm9e1"
inputQueues: {
  "MRG_001_IN_A": ["BOX_A_000001"],
  "MRG_001_IN_B": ["BOX_B_000001"]
}
```

상태 변경은 반드시 공통 함수를 통해 처리한다.

```ts id="ulw9jc"
setComponentState(componentId, nextState, currentTime, reason?)
```

이 함수는 이전 상태의 지속 시간을 계산하고, State Timeline에 상태 변경을 기록한다.

---

## 9. 초기화 단계

DES Engine 실행 전 초기화 순서:

```text id="7tc3nr"
1. validateLayout(layout)
2. validateScenario(scenario, layout)
3. createRoutingGraph(layout)
4. validateRoutes(layout, graph)
5. createArrivalEvents(scenario)
6. component runtime state 초기화
7. item manager 초기화
8. statistics collector 초기화
9. event logger 초기화
10. arrival events를 EventQueue에 ITEM_CREATE_REQUEST로 등록
```

Component State 초기값:

```text id="z6rp5u"
Source: idle
Conveyor: idle
Diverter: idle
Merge: idle
Buffer: idle
Process Station: idle
Robot: idle
Pallet Station: idle
Sink: idle
```

---

## 10. Source 처리 로직

관련 이벤트:

```text id="3sxnl4"
ITEM_CREATE_REQUEST
ITEM_CREATED
ITEM_RELEASED
```

ITEM_CREATE_REQUEST 처리:

```text id="bv1xts"
1. event.payload에서 itemType, source, destination, routeId를 읽는다.
2. Item Master에서 itemType 정보를 찾는다.
3. Runtime Item ID를 생성한다.
4. RoutingService.selectRouteForItem을 호출한다.
5. SimulationItem을 생성한다.
6. ItemManager에 등록한다.
7. ITEM_CREATED 이벤트를 기록한다.
8. Source의 out port에서 다음 컴포넌트 진입 가능 여부를 확인한다.
9. 가능하면 ITEM_RELEASED와 ITEM_ENTER_COMPONENT 이벤트를 예약한다.
10. 불가능하면 Source queue에 넣고 waiting 상태로 둔다.
```

MVP에서는 Source queue가 무한이라고 가정한다.

---

## 11. Conveyor 처리 로직

관련 이벤트:

```text id="oos19g"
ITEM_ENTER_COMPONENT
ITEM_EXIT_COMPONENT
BLOCKING_START
BLOCKING_END
```

진입 처리:

```text id="p1jz2m"
1. Conveyor capacity를 확인한다.
2. 현재 currentItemIds 수가 capacity보다 작으면 진입 허용
3. currentItemIds에 itemId 추가
4. item.currentComponentId를 Conveyor로 갱신
5. item.currentState = moving
6. travelTime = length / speed 계산
7. ITEM_EXIT_COMPONENT 이벤트를 currentTime + travelTime에 예약
8. Conveyor state를 busy로 변경
```

출구 처리:

```text id="ia4dfm"
1. RoutingService.getNextComponent를 호출한다.
2. 다음 컴포넌트가 받을 수 있는지 확인한다.
3. 받을 수 있으면 Conveyor에서 item 제거
4. 다음 컴포넌트로 ITEM_ENTER_COMPONENT 이벤트를 현재 시간에 등록
5. Conveyor에 남은 item이 없으면 state를 idle로 변경
6. 받을 수 없으면 item을 Conveyor 출구에서 waiting 또는 blocked 상태로 둔다.
7. Conveyor state를 blocked로 변경한다.
```

이동 시간 계산:

```text id="60jhoi"
travelTime = conveyor.length / conveyor.speed
```

---

## 12. Diverter 처리 로직

관련 이벤트:

```text id="r58919"
ITEM_ENTER_COMPONENT
DIVERTER_ROUTE_DECIDED
ITEM_EXIT_COMPONENT
ITEM_WAIT_START
ITEM_WAIT_END
```

처리 흐름:

```text id="kbfh61"
1. 아이템이 Diverter에 진입한다.
2. Diverter state를 busy로 변경한다.
3. RoutingService.decideDiverterOutput을 호출한다.
4. output port와 connection을 결정한다.
5. switchingTime을 계산한다.
6. currentTime + switchingTime에 ITEM_EXIT_COMPONENT 이벤트를 예약한다.
7. 출구 하류가 막혀 있으면 ITEM_WAIT_START를 기록한다.
```

MVP에서는 byDestination만 구현한다.

---

## 13. Merge 처리 로직

관련 이벤트:

```text id="rerink"
ITEM_ENTER_COMPONENT
MERGE_RELEASE_DECIDED
ITEM_EXIT_COMPONENT
ITEM_WAIT_START
ITEM_WAIT_END
```

진입 처리:

```text id="t1ee4k"
1. 아이템이 특정 input port로 Merge에 도착한다.
2. inputQueues[inputPortId]에 itemId를 추가한다.
3. item.currentState = waiting
4. Merge output port 하류가 받을 수 있는지 확인한다.
5. 받을 수 있으면 decideMergeRelease를 호출한다.
6. 선택된 item을 배출한다.
7. 받을 수 없으면 Merge state를 blocked 또는 waiting으로 둔다.
```

MVP에서는 FIFO만 구현한다.

---

## 14. Buffer 처리 로직

관련 이벤트:

```text id="gupaf6"
ITEM_ENTER_COMPONENT
ITEM_WAIT_START
ITEM_WAIT_END
ITEM_EXIT_COMPONENT
BUFFER_FULL
BUFFER_AVAILABLE
```

진입 처리:

```text id="d3e4iw"
1. Buffer capacity를 확인한다.
2. queueItemIds.length < capacity이면 진입 허용
3. queueItemIds에 itemId 추가
4. item.currentState = waiting
5. ITEM_WAIT_START 기록
6. 하류가 available이면 즉시 배출 시도
7. capacity가 가득 차면 BUFFER_FULL과 upstream blocking 발생
```

배출 처리:

```text id="iqv5ab"
1. 하류 설비가 available인지 확인한다.
2. FIFO 기준으로 queueItemIds[0]을 선택한다.
3. ITEM_WAIT_END 기록
4. Buffer queue에서 제거
5. 다음 컴포넌트로 ITEM_ENTER_COMPONENT 이벤트 등록
6. Buffer에 남은 item이 없으면 idle로 변경
```

---

## 15. Robot 처리 로직

관련 이벤트:

```text id="mqu7i2"
ITEM_ENTER_COMPONENT
ROBOT_PICK_START
ROBOT_PICK_END
ROBOT_PLACE_START
ROBOT_PLACE_END
PALLETIZE_COMPLETE
```

진입 처리:

```text id="yndz7x"
1. Robot state를 확인한다.
2. idle이면 ROBOT_PICK_START 이벤트를 현재 시간에 등록한다.
3. busy이면 Robot queue에 itemId를 추가하고 ITEM_WAIT_START를 기록한다.
```

ROBOT_PICK_START 처리:

```text id="430h4g"
1. Robot state를 busy로 변경한다.
2. currentItemIds에 itemId를 등록한다.
3. item.currentState = processing
4. cycleTime을 결정한다.
5. currentTime + cycleTime에 ROBOT_PICK_END 이벤트를 예약한다.
```

cycleTime 결정 우선순위:

```text id="yjt3vc"
1. Scenario resourceOverrides의 cycleTime
2. Robot component.properties.cycleTime
3. 시스템 기본값
```

ROBOT_PICK_END 처리:

```text id="dzofq6"
1. Robot currentItemIds에서 itemId를 제거한다.
2. placeTarget을 확인한다.
3. placeTarget이 Pallet Station이면 PALLETIZE_COMPLETE 이벤트를 현재 시간에 등록한다.
4. Robot queue에 대기 item이 있으면 다음 ROBOT_PICK_START를 등록한다.
5. 대기 item이 없으면 Robot state를 idle로 변경한다.
```

---

## 16. Pallet Station 처리 로직

Pallet Station은 일반 ComponentRuntimeState 외에 적재 상태를 가진다.

```ts id="1b501n"
export interface PalletStationRuntimeState extends ComponentRuntimeState {
  currentBoxCount: number;
  currentLayer: number;
  completedPalletCount: number;
}
```

PALLETIZE_COMPLETE 처리:

```text id="8p9lqh"
1. Pallet Station의 currentBoxCount를 1 증가시킨다.
2. item.currentState를 completed 또는 placed로 변경한다.
3. boxesPerLayer에 도달하면 PALLET_LAYER_COMPLETE 이벤트를 기록한다.
4. maxBoxes에 도달하면 PALLET_COMPLETE 이벤트를 기록한다.
5. route상 다음 컴포넌트가 Sink이면 ITEM_COMPLETED 이벤트를 등록한다.
```

---

## 17. Sink 처리 로직

관련 이벤트:

```text id="l9bzxl"
ITEM_ENTER_COMPONENT
ITEM_COMPLETED
```

처리 흐름:

```text id="2521wd"
1. 아이템이 Sink에 도착한다.
2. ITEM_COMPLETED 이벤트를 기록한다.
3. item.currentState = completed
4. item.completedAt = currentTime
5. leadTime = completedAt - createdAt 계산
6. Sink processedCount 증가
7. StatisticsCollector에 완료 정보를 전달한다.
```

---

## 18. Blocking과 Waiting 처리

Blocking은 하류 설비가 받을 수 없어 상류 설비가 아이템을 내보내지 못하는 상태다.

Blocking 처리 흐름:

```text id="ekveog"
1. 아이템이 다음 컴포넌트로 이동하려고 한다.
2. canAcceptItem(nextComponent)를 호출한다.
3. false이면 현재 컴포넌트 또는 포트에서 item을 대기시킨다.
4. BLOCKING_START 이벤트를 기록한다.
5. 현재 컴포넌트 state를 blocked로 변경한다.
6. 하류 설비가 available해지면 BLOCKING_END 이벤트를 기록한다.
7. 대기 item 이동을 재시도한다.
```

Waiting 시간 계산:

```text id="c7rapw"
ITEM_WAIT_START 시점에 item.waitingStartedAt 기록
ITEM_WAIT_END 시점에 currentTime - waitingStartedAt을 totalWaitingTime에 더함
```

---

## 19. Deadlock 감지

MVP의 Deadlock 감지 기준:

```text id="j7jdds"
1. Event Queue가 비어 있다.
2. completed 되지 않은 active item이 있다.
3. active item이 모두 waiting 또는 blocked 상태다.
4. 더 이상 retry 가능한 이동이 없다.
```

처리 방식:

```text id="ylqpr2"
1. DEADLOCK_DETECTED 이벤트 기록
2. SimulationResult.errors에 deadlock 오류 추가
3. scenario.simulation.stopOnDeadlock이 true이면 시뮬레이션 중지
4. false이면 warning만 남기고 종료 시점까지 진행 시도
```

---

## 20. Statistics Collector

MVP에서 계산할 지표:

```text id="4u9xaz"
- createdItems
- completedItems
- wip
- throughputPerHour
- averageLeadTime
- averageWaitingTime
- averageProcessingTime
- component utilization
- component busy time
- component idle time
- component blocked time
- robot utilization
- pallet completed count
- bottleneck candidates
```

Summary 구조:

```ts id="zs6v1f"
export interface SimulationSummary {
  simulationStartTime: number;
  simulationEndTime: number;
  createdItems: number;
  completedItems: number;
  wip: number;
  throughputPerHour: number;
  averageLeadTime: number;
  averageWaitingTime: number;
  averageProcessingTime: number;
  deadlockDetected: boolean;
}
```

설비 가동률:

```text id="xs7x57"
utilization = busyTime / availableTime
```

병목 후보 기준:

```text id="4g96bu"
1. blockedTime이 긴 설비
2. waitingTime이 긴 설비
3. utilization이 높은 resource
4. queue length 평균이 높은 설비
```

---

## 21. Simulation Runner

최상위 API:

```ts id="9twul6"
export interface SimulationRunner {
  run(input: SimulationInput): SimulationResult;
}
```

실행 의사 코드:

```ts id="07fyfc"
function run(input: SimulationInput): SimulationResult {
  validateInput(input);

  const graph = createRoutingGraph(input.layout);
  const arrivalEvents = createArrivalEvents(input.scenario);

  const context = initializeSimulationContext(input, graph);

  for (const arrival of arrivalEvents) {
    context.eventQueue.push(createItemCreateRequestEvent(arrival));
  }

  while (!context.eventQueue.isEmpty()) {
    const event = context.eventQueue.pop();

    if (!event) break;
    if (event.time > context.clock.endTime) break;

    context.clock.currentTime = event.time;

    dispatchEvent(event, context);

    if (context.processedEventCount > context.maxEvents) {
      recordMaxEventError(context);
      break;
    }

    if (shouldCheckDeadlock(event)) {
      const deadlock = detectDeadlock(context);
      if (deadlock) {
        handleDeadlock(context);
        break;
      }
    }
  }

  return buildSimulationResult(context);
}
```

---

## 22. Simulation Context

```ts id="ws8tu1"
export interface SimulationContext {
  layout: LayoutJson;
  scenario: ScenarioJson;
  graph: RoutingGraph;
  clock: SimulationClock;
  eventQueue: EventQueue;
  items: Record<string, SimulationItem>;
  components: Record<string, ComponentRuntimeState>;
  eventLogs: SimulationEventLog[];
  itemSequenceByType: Record<string, number>;
  processedEventCount: number;
  maxEvents: number;
  warnings: SimulationWarning[];
  errors: SimulationError[];
}
```

Context는 시뮬레이션 중 모든 모듈이 공유하는 런타임 상태다.

---

## 23. MVP 구현 범위

### 포함

```text id="2o0uqo"
- SimulationRunner
- SimulationClock
- EventQueue
- EventDispatcher
- SimulationContext
- SimulationItem
- ComponentRuntimeState
- Source 처리
- Conveyor 처리
- Robot 처리
- Pallet Station 처리
- Sink 처리
- Diverter byDestination 처리
- Merge FIFO 처리
- Buffer FIFO 처리
- Blocking 기본 처리
- Waiting 시간 계산
- Deadlock 감지 기초
- Event Log 생성
- Summary Statistics 생성
```

### 제외

```text id="gmivno"
- Zone Conveyor 정밀 누적
- 박스 간 물리 충돌
- 실제 로봇 역기구학
- 확률 고장
- 작업자 스케줄
- Shift와 Break
- 동적 rerouting
- 최적화 엔진
```

---

## 24. 작업 목록

```text id="hedgn1"
타입 정의 작업
- src/types/simulation.ts 생성
- SimulationInput 타입 정의
- SimulationResult 타입 정의
- SimulationEvent 타입 정의
- SimulationEventType 타입 정의
- SimulationItem 타입 정의
- ComponentRuntimeState 타입 정의
- SimulationContext 타입 정의
- SimulationSummary 타입 정의
- SimulationEventLog 타입 정의

Event Queue 작업
- src/simulation/engine/EventQueue.ts 생성
- push 구현
- pop 구현
- peek 구현
- time, sequence 기준 정렬 구현
- 단위 테스트 작성

초기화 작업
- initializeSimulationContext.ts 생성
- component runtime state 초기화
- itemSequenceByType 초기화
- arrival events를 EventQueue에 등록
- resource override 적용

Event Handler 작업
- handleItemCreateRequest 구현
- handleItemEnterComponent 구현
- handleItemExitComponent 구현
- handleConveyorEnter 구현
- handleConveyorExit 구현
- handleRobotEnter 구현
- handleRobotPickStart 구현
- handleRobotPickEnd 구현
- handlePalletizeComplete 구현
- handleItemCompleted 구현
- handleDiverterEnter 구현
- handleMergeEnter 구현
- handleBufferEnter 구현

State와 통계 작업
- setComponentState 구현
- item waiting time 계산
- component busy, idle, blocked time 계산
- summary 계산
- componentStats 계산
- bottleneck candidates 계산
```

---

## 25. 테스트 시나리오

### 25.1 Source → Conveyor → Robot → Pallet → Sink

조건:

```text id="qlbrkq"
시뮬레이션 시간: 3600초
투입 수량: 600개
투입 간격: 6초
컨베이어 길이: 6m
컨베이어 속도: 0.6m/s
컨베이어 이동 시간: 10초
로봇 cycleTime: 8초
```

검증:

```text id="smzup2"
- ITEM_CREATE_REQUEST가 600개 생성되는가
- Conveyor travelTime이 10초로 계산되는가
- Robot cycleTime이 8초로 적용되는가
- PALLETIZE_COMPLETE 이벤트가 생성되는가
- ITEM_COMPLETED 이벤트가 생성되는가
- completedItems가 계산되는가
- robot utilization이 계산되는가
```

### 25.2 Robot 병목 테스트

조건:

```text id="oitcg5"
투입 간격: 4초
로봇 cycleTime: 8초
```

검증:

```text id="rlznbe"
- Robot queue가 증가하는가
- item waiting time이 증가하는가
- robot utilization이 100%에 가까워지는가
- 병목 후보로 Robot이 표시되는가
```

---

## 26. 완료 기준

```text id="7rjkg6"
- DES Engine의 전체 구조가 정의되어 있다.
- SimulationClock과 EventQueue가 정의되어 있다.
- Event 타입과 Event 처리 흐름이 정의되어 있다.
- SimulationItem과 ComponentRuntimeState가 정의되어 있다.
- Source 처리 로직이 정의되어 있다.
- Conveyor 처리 로직이 정의되어 있다.
- Diverter 처리 로직이 정의되어 있다.
- Merge 처리 로직이 정의되어 있다.
- Buffer 처리 로직이 정의되어 있다.
- Robot 처리 로직이 정의되어 있다.
- Pallet Station 처리 로직이 정의되어 있다.
- Sink 처리 로직이 정의되어 있다.
- Blocking, Waiting, Deadlock 처리 기준이 정의되어 있다.
- Summary Statistics 계산 기준이 정의되어 있다.
- Event Log 기록 기준이 정의되어 있다.
- Codex가 DES Engine MVP를 구현할 수 있는 수준이다.
```

---

## 27. Codex 작업 지시 예시

```text id="tfb218"
MasterPlan.md부터 Plan_07.md까지 읽고,
src/types/simulation.ts 파일을 만들어줘.

Plan_07.md의 타입 정의를 기준으로 다음 타입을 구현해.
- SimulationInput
- SimulationResult
- SimulationEvent
- SimulationEventType
- SimulationItem
- ComponentRuntimeState
- SimulationContext
- SimulationSummary
- SimulationEventLog
- SimulationError
- SimulationWarning

아직 Three.js Animation Player는 구현하지 마.
```

---

## 28. 다음 단계

Plan_07이 완료되면 다음 문서로 이동한다.

```text id="58usam"
Plan_08.md
```

Plan_08에서는 DES Engine이 생성한 eventLog와 runtime state를 기반으로 **Event Log Timeline** 구조를 정의한다.
::: 
