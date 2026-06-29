# Plan_08.md

# Plan_08. Event Log Timeline Definition

## 1. 목적

이 문서는 웹 기반 물류 라인 이산 사건 시뮬레이션에서 사용할 **Event Log Timeline** 구조를 정의하기 위한 실행 계획서이다.

Plan_07.md에서 DES Simulation Engine이 이벤트를 처리하고 Simulation Result를 생성하는 방식을 정의했다면, Plan_08.md는 그 결과를 **다시 볼 수 있는 시간의 기록**으로 정리한다.

이 문서의 핵심 목적은 다음과 같다.

```text
1. DES Engine이 생성하는 Event Log의 상세 구조를 정의한다.
2. 아이템별 이동 이력인 Item Timeline을 정의한다.
3. 설비별 상태 변화 이력인 Component State Timeline을 정의한다.
4. Three.js Animation Player가 사용할 재생용 Timeline 구조를 정의한다.
5. 시간 슬라이더에서 특정 시점의 상태를 복원하는 방식을 정의한다.
6. KPI 계산에 필요한 로그와 통계 데이터의 관계를 정의한다.
7. Simulation Result JSON 저장 구조를 정의한다.
8. Codex가 Event Log, Timeline, State Snapshot 모듈을 구현할 수 있도록 명세를 제공한다.
```

Event Log는 시뮬레이션이 지나간 발자국이다.
Timeline은 그 발자국을 다시 걸을 수 있게 만든 길이다.
Three.js Animation Player는 이 길을 따라가며 박스와 로봇을 화면에서 되살린다.

---

## 2. Event Log와 Timeline의 차이

### 2.1 Event Log

Event Log는 시뮬레이션 중 발생한 모든 주요 사건을 시간순으로 저장한 원시 기록이다.

```text
0초: BOX_A_000001 생성
0초: BOX_A_000001 CV_001 진입
10초: BOX_A_000001 CV_001 출구 도착
10초: BOX_A_000001 RB_001 작업 시작
18초: BOX_A_000001 팔레트 적재 완료
18초: BOX_A_000001 완료
```

Event Log는 사건 중심이다.

```text
언제, 어떤 사건이, 어떤 아이템과 설비에서 발생했는가?
```

### 2.2 Timeline

Timeline은 Event Log를 재생과 분석에 적합하게 재구성한 데이터다.

```text
Item Timeline
- 아이템 하나가 어느 시간에 어디에 있었는지 기록

Component State Timeline
- 설비 하나가 어느 시간 구간에 어떤 상태였는지 기록
```

Timeline은 상태 중심이다.

```text
어떤 시간 구간 동안 아이템 또는 설비가 어떤 상태였는가?
```

---

## 3. Simulation Result 전체 구조

```json
{
  "schemaVersion": "1.0.0",
  "resultInfo": {},
  "summary": {},
  "eventLog": [],
  "itemTimelines": [],
  "componentStateTimelines": [],
  "animationTracks": [],
  "componentStats": [],
  "itemStats": [],
  "snapshots": [],
  "warnings": [],
  "errors": [],
  "metadata": {}
}
```

각 영역의 역할은 다음과 같다.

| 영역                      | 역할                          |
| ----------------------- | --------------------------- |
| schemaVersion           | Simulation Result Schema 버전 |
| resultInfo              | 결과 파일 기본 정보                 |
| summary                 | 전체 KPI 요약                   |
| eventLog                | 시간순 원시 이벤트 기록               |
| itemTimelines           | 아이템별 이동 이력                  |
| componentStateTimelines | 설비별 상태 변화 이력                |
| animationTracks         | Three.js 재생용 트랙             |
| componentStats          | 설비별 통계                      |
| itemStats               | 아이템별 통계                     |
| snapshots               | 시간 슬라이더용 상태 스냅샷             |
| warnings                | 경고 목록                       |
| errors                  | 오류 목록                       |
| metadata                | 태그, 비고, 실행 조건 요약            |

---

## 4. Event Log 구조

```ts
export interface SimulationEventLog {
  eventId: string;
  time: number;
  sequence: number;
  eventType: SimulationEventType;
  itemId?: string;
  componentId?: string;
  portId?: string;
  fromComponentId?: string;
  toComponentId?: string;
  fromPortId?: string;
  toPortId?: string;
  stateBefore?: string;
  stateAfter?: string;
  message?: string;
  payload?: Record<string, unknown>;
}
```

JSON 예시:

```json
{
  "eventId": "EVT_000001",
  "time": 0,
  "sequence": 1,
  "eventType": "ITEM_CREATED",
  "itemId": "BOX_A_000001",
  "componentId": "SRC_001",
  "message": "BOX_A_000001 created at SRC_001",
  "payload": {
    "itemType": "BOX_A",
    "source": "SRC_001",
    "destination": "PALLET_001",
    "routeId": "ROUTE_BOX_A"
  }
}
```

---

## 5. 주요 Event Type별 payload

### 5.1 ITEM_CREATED

```json
{
  "itemType": "BOX_A",
  "source": "SRC_001",
  "destination": "PALLET_001",
  "routeId": "ROUTE_BOX_A"
}
```

### 5.2 ITEM_ENTER_COMPONENT

```json
{
  "componentType": "conveyor",
  "enterPortId": "CV_001_IN"
}
```

### 5.3 ITEM_EXIT_COMPONENT

```json
{
  "componentType": "conveyor",
  "exitPortId": "CV_001_OUT",
  "travelTime": 10
}
```

### 5.4 ITEM_WAIT_START

```json
{
  "reason": "ROBOT_BUSY",
  "queueComponentId": "RB_001",
  "queueLength": 3
}
```

### 5.5 ITEM_WAIT_END

```json
{
  "waitingTime": 12,
  "reason": "ROBOT_AVAILABLE"
}
```

### 5.6 BLOCKING_START

```json
{
  "blockedComponentId": "CV_001",
  "downstreamComponentId": "RB_001",
  "reason": "DOWNSTREAM_FULL"
}
```

### 5.7 ROBOT_PICK_START

```json
{
  "robotId": "RB_001",
  "cycleTime": 8,
  "pickPortId": "RB_001_PICK",
  "placeTarget": "PALLET_001"
}
```

### 5.8 PALLETIZE_COMPLETE

```json
{
  "palletStationId": "PALLET_001",
  "boxIndex": 1,
  "layerIndex": 0,
  "positionInLayer": 0
}
```

---

## 6. Item Timeline 구조

```ts
export interface ItemTimeline {
  itemId: string;
  itemType: string;
  source: string;
  destination?: string;
  routeId?: string;
  createdAt: number;
  completedAt?: number;
  leadTime?: number;
  totalWaitingTime: number;
  totalProcessingTime: number;
  segments: ItemTimelineSegment[];
}
```

Segment 구조:

```ts
export interface ItemTimelineSegment {
  segmentId: string;
  startTime: number;
  endTime: number;
  state:
    | "created"
    | "moving"
    | "waiting"
    | "processing"
    | "blocked"
    | "placed"
    | "completed";
  fromComponentId?: string;
  toComponentId?: string;
  componentId?: string;
  fromPortId?: string;
  toPortId?: string;
  connectionId?: string;
  animationType?: "none" | "linearMove" | "wait" | "robotPickPlace" | "palletPlace";
  payload?: Record<string, unknown>;
}
```

Item Timeline 예시:

```json
{
  "itemId": "BOX_A_000001",
  "itemType": "BOX_A",
  "source": "SRC_001",
  "destination": "PALLET_001",
  "routeId": "ROUTE_BOX_A",
  "createdAt": 0,
  "completedAt": 18,
  "leadTime": 18,
  "totalWaitingTime": 0,
  "totalProcessingTime": 8,
  "segments": [
    {
      "segmentId": "SEG_000001",
      "startTime": 0,
      "endTime": 0,
      "state": "created",
      "componentId": "SRC_001",
      "animationType": "none"
    },
    {
      "segmentId": "SEG_000002",
      "startTime": 0,
      "endTime": 10,
      "state": "moving",
      "fromComponentId": "SRC_001",
      "toComponentId": "CV_001",
      "connectionId": "CONN_001",
      "animationType": "linearMove"
    },
    {
      "segmentId": "SEG_000003",
      "startTime": 10,
      "endTime": 18,
      "state": "processing",
      "componentId": "RB_001",
      "animationType": "robotPickPlace",
      "payload": {
        "pickComponentId": "RB_001",
        "placeTarget": "PALLET_001"
      }
    },
    {
      "segmentId": "SEG_000004",
      "startTime": 18,
      "endTime": 18,
      "state": "completed",
      "componentId": "SINK_001",
      "animationType": "none"
    }
  ]
}
```

---

## 7. Item Timeline 생성 방식

Item Timeline은 Event Log에서 itemId 기준으로 이벤트를 모아 생성한다.

```text
1. eventLog를 itemId 기준으로 그룹화한다.
2. 각 item의 이벤트를 time, sequence 기준으로 정렬한다.
3. 연속 이벤트를 segment로 변환한다.
4. moving, waiting, processing, completed 상태를 구간으로 만든다.
5. leadTime, totalWaitingTime, totalProcessingTime을 계산한다.
```

Segment 생성 규칙:

```text
ITEM_CREATED
- created segment 생성

ITEM_ENTER_COMPONENT + ITEM_EXIT_COMPONENT
- 해당 컴포넌트 체류 또는 이동 segment 생성

ITEM_WAIT_START + ITEM_WAIT_END
- waiting segment 생성

ROBOT_PICK_START + ROBOT_PICK_END
- processing 또는 robotPickPlace segment 생성

PALLETIZE_COMPLETE
- palletPlace segment 생성

ITEM_COMPLETED
- completed segment 생성
```

---

## 8. Component State Timeline 구조

```ts
export interface ComponentStateTimeline {
  componentId: string;
  componentType: string;
  states: ComponentStateSegment[];
}

export interface ComponentStateSegment {
  segmentId: string;
  startTime: number;
  endTime: number;
  state: "idle" | "busy" | "waiting" | "blocked" | "disabled" | "failed";
  itemIds?: string[];
  queueLength?: number;
  reason?: string;
  payload?: Record<string, unknown>;
}
```

예시:

```json
{
  "componentId": "RB_001",
  "componentType": "robot",
  "states": [
    {
      "segmentId": "CSEG_000001",
      "startTime": 0,
      "endTime": 10,
      "state": "idle",
      "itemIds": [],
      "queueLength": 0
    },
    {
      "segmentId": "CSEG_000002",
      "startTime": 10,
      "endTime": 18,
      "state": "busy",
      "itemIds": ["BOX_A_000001"],
      "queueLength": 0
    }
  ]
}
```

---

## 9. Component State Timeline 생성 방식

Component State Timeline은 DES Engine의 `setComponentState` 함수에서 직접 기록하는 것이 가장 정확하다.

```text
1. Component 상태 변경은 반드시 setComponentState를 통해 수행한다.
2. setComponentState는 이전 상태 구간을 종료한다.
3. 새로운 상태 구간을 시작한다.
4. Simulation 종료 시 마지막 상태 구간을 endTime으로 닫는다.
```

상태 변화가 없는 설비도 Timeline을 가져야 한다.

```json
{
  "componentId": "BUF_001",
  "componentType": "buffer",
  "states": [
    {
      "segmentId": "CSEG_000100",
      "startTime": 0,
      "endTime": 3600,
      "state": "idle"
    }
  ]
}
```

---

## 10. Animation Track 구조

Animation Track은 Three.js Animation Player가 바로 사용할 수 있도록 Item Timeline을 한 번 더 가공한 데이터다.

```ts
export interface AnimationTrack {
  trackId: string;
  targetType: "item" | "component" | "robot" | "pallet";
  targetId: string;
  keyframes: AnimationKeyframe[];
}

export interface AnimationKeyframe {
  time: number;
  position?: {
    x: number;
    y: number;
    z: number;
  };
  rotation?: {
    x: number;
    y: number;
    z: number;
  };
  visible?: boolean;
  state?: string;
  payload?: Record<string, unknown>;
}
```

Box 이동 Track 예시:

```json
{
  "trackId": "TRACK_BOX_A_000001",
  "targetType": "item",
  "targetId": "BOX_A_000001",
  "keyframes": [
    {
      "time": 0,
      "position": { "x": 0, "y": 0.8, "z": 0 },
      "visible": true,
      "state": "moving"
    },
    {
      "time": 10,
      "position": { "x": 6, "y": 0.8, "z": 0 },
      "visible": true,
      "state": "waiting"
    },
    {
      "time": 18,
      "position": { "x": 8, "y": 0.8, "z": 3 },
      "visible": true,
      "state": "placed"
    }
  ]
}
```

---

## 11. Animation Track 생성 원칙

### 11.1 선형 이동

컨베이어 또는 connection 이동은 선형 보간으로 처리한다.

```text
startTime: 0
endTime: 10
startPosition: CV_001_IN world position
endPosition: CV_001_OUT world position
```

Three.js에서는 현재 재생 시간 t에 대해 다음처럼 보간한다.

```text
progress = (t - startTime) / (endTime - startTime)
position = lerp(startPosition, endPosition, progress)
```

### 11.2 대기 상태

Waiting segment에서는 position이 고정된다.

```text
startTime: 10
endTime: 20
position: pick port position
state: waiting
```

### 11.3 로봇 Pick & Place

MVP에서는 로봇의 실제 관절 궤적이 아니라 박스의 이동 궤적만 단순화해서 표시한다.

```text
pick position
  ↓
lift position
  ↓
place position
```

---

## 12. Snapshot 구조

Snapshot은 시간 슬라이더에서 특정 시점의 전체 상태를 빠르게 복원하기 위한 데이터다.

```ts
export interface SimulationSnapshot {
  time: number;
  items: SnapshotItemState[];
  components: SnapshotComponentState[];
  summary: SnapshotSummary;
}

export interface SnapshotItemState {
  itemId: string;
  itemType: string;
  visible: boolean;
  position?: {
    x: number;
    y: number;
    z: number;
  };
  state: string;
  componentId?: string;
}

export interface SnapshotComponentState {
  componentId: string;
  state: string;
  queueLength: number;
  currentItemIds: string[];
}
```

MVP에서는 Snapshot 구조만 정의하고, 실제 구현은 선택 사항으로 둔다.

---

## 13. KPI 계산용 데이터 구조

```ts
export interface ComponentStatistics {
  componentId: string;
  componentType: string;
  processedCount: number;
  busyTime: number;
  idleTime: number;
  waitingTime: number;
  blockedTime: number;
  utilization: number;
  averageQueueLength?: number;
  maxQueueLength?: number;
}

export interface ItemStatistics {
  itemId: string;
  itemType: string;
  createdAt: number;
  completedAt?: number;
  leadTime?: number;
  totalWaitingTime: number;
  totalProcessingTime: number;
  completed: boolean;
}

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
  bottleneckComponentIds: string[];
}
```

KPI 계산 기준:

```text
throughputPerHour = completedItems / simulationDurationHour

wip = createdItems - completedItems

leadTime = completedAt - createdAt

utilization = busyTime / availableTime
```

---

## 14. Timeline 생성 함수

```ts
function buildItemTimelines(
  eventLog: SimulationEventLog[]
): ItemTimeline[]
```

```ts
function buildComponentStateTimelines(
  stateRecords: ComponentStateSegment[]
): ComponentStateTimeline[]
```

```ts
function buildAnimationTracks(
  layout: LayoutJson,
  itemTimelines: ItemTimeline[],
  componentStateTimelines: ComponentStateTimeline[]
): AnimationTrack[]
```

```ts
function buildComponentStats(
  componentStateTimelines: ComponentStateTimeline[],
  eventLog: SimulationEventLog[],
  simulationStartTime: number,
  simulationEndTime: number
): ComponentStatistics[]
```

```ts
function buildItemStats(
  itemTimelines: ItemTimeline[]
): ItemStatistics[]
```

```ts
function buildSummary(
  itemStats: ItemStatistics[],
  componentStats: ComponentStatistics[],
  simulationStartTime: number,
  simulationEndTime: number,
  errors: SimulationError[]
): SimulationSummary
```

---

## 15. Timeline 검증

검증 항목:

```text
1. 모든 Item Timeline은 ITEM_CREATED 이벤트를 가져야 한다.
2. completed item은 ITEM_COMPLETED 이벤트를 가져야 한다.
3. WAIT_START가 있으면 WAIT_END가 있어야 한다.
4. ROBOT_PICK_START가 있으면 ROBOT_PICK_END가 있어야 한다.
5. segment의 startTime은 endTime보다 작거나 같아야 한다.
6. Item segment는 시간순으로 겹치지 않아야 한다.
7. Component state segment는 시간순으로 이어져야 한다.
8. Animation keyframe은 time 기준으로 정렬되어야 한다.
```

오류 예시:

```json
{
  "code": "TIMELINE_INVALID_SEGMENT",
  "message": "Item BOX_A_000001 has a segment whose endTime is earlier than startTime.",
  "path": "itemTimelines[0].segments[2]",
  "severity": "error"
}
```

---

## 16. 저장 파일 구조

최종 결과 저장 파일은 다음 구조를 따른다.

```json
{
  "schemaVersion": "1.0.0",
  "resultInfo": {},
  "summary": {},
  "eventLog": [],
  "itemTimelines": [],
  "componentStateTimelines": [],
  "animationTracks": [],
  "componentStats": [],
  "itemStats": [],
  "snapshots": [],
  "warnings": [],
  "errors": [],
  "metadata": {}
}
```

저장 옵션:

```text
summaryOnly
- summary, componentStats, itemStats만 저장

analysis
- eventLog, timelines 포함

animation
- animationTracks 포함

full
- 모든 데이터 저장
```

MVP에서는 `analysis` 수준을 기본값으로 한다.

---

## 17. 작업 목록

```text
타입 정의 작업
- src/types/result.ts 생성
- SimulationResult 타입 정의
- SimulationResultInfo 타입 정의
- SimulationEventLog 타입 정의
- ItemTimeline 타입 정의
- ItemTimelineSegment 타입 정의
- ComponentStateTimeline 타입 정의
- ComponentStateSegment 타입 정의
- AnimationTrack 타입 정의
- AnimationKeyframe 타입 정의
- ComponentStatistics 타입 정의
- ItemStatistics 타입 정의

Event Log 작업
- src/simulation/logging/EventLogger.ts 생성
- eventId 자동 생성
- time, sequence 기록
- eventType 기록
- itemId, componentId, portId 기록
- payload 기록
- message 생성

Timeline 생성 작업
- src/simulation/timeline/buildItemTimelines.ts 생성
- eventLog를 itemId 기준으로 그룹화
- moving segment 생성
- waiting segment 생성
- processing segment 생성
- completed segment 생성
- leadTime 계산
- totalWaitingTime 계산
- totalProcessingTime 계산

Component State Timeline 작업
- src/simulation/timeline/StateTimelineRecorder.ts 생성
- 상태 변경 segment 생성
- 마지막 state segment 닫기
- componentId 기준 timeline 생성
- idle 상태 설비 timeline 생성

Statistics 작업
- src/simulation/statistics/buildComponentStats.ts 생성
- busyTime 계산
- idleTime 계산
- blockedTime 계산
- utilization 계산
- processedCount 계산
- bottleneck ranking 계산
```

---

## 18. 완료 기준

```text
- Event Log의 상세 구조가 정의되어 있다.
- Event Type별 payload 기준이 정의되어 있다.
- Item Timeline 구조가 정의되어 있다.
- Item Timeline 생성 방식이 정의되어 있다.
- Component State Timeline 구조가 정의되어 있다.
- Component State Timeline 생성 방식이 정의되어 있다.
- Animation Track 구조가 정의되어 있다.
- Snapshot 구조가 정의되어 있다.
- KPI 계산용 데이터 구조가 정의되어 있다.
- simulation_result.json 저장 구조가 정의되어 있다.
- TypeScript 타입 초안이 작성되어 있다.
- Codex가 Timeline과 Result 모듈을 구현할 수 있는 수준이다.
```

---

## 19. Codex 작업 지시 예시

```text
MasterPlan.md부터 Plan_08.md까지 읽고,
src/types/result.ts 파일을 만들어줘.

Plan_08.md의 TypeScript 타입 초안을 기준으로 다음 타입을 구현해.
- SimulationResult
- SimulationResultInfo
- SimulationEventLog
- ItemTimeline
- ItemTimelineSegment
- ComponentStateTimeline
- ComponentStateSegment
- AnimationTrack
- AnimationKeyframe
- ComponentStatistics
- ItemStatistics

아직 Three.js Animation Player는 구현하지 마.
```

---

## 20. 다음 단계

Plan_08이 완료되면 다음 문서로 이동한다.

```text
Plan_09.md
```

Plan_09에서는 Plan_08에서 정의한 Timeline과 Animation Track을 Three.js 화면에서 재생하는 **Three.js Animation Player**를 정의한다.
::: 
