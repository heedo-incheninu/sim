# Plan_05.md

# Plan_05. Network Routing Definition

## 1. 목적

이 문서는 웹 기반 물류 라인 이산 사건 시뮬레이션에서 사용할 **Network Routing 구조**를 정의하기 위한 실행 계획서이다.

Plan_04.md에서 사용자가 Three.js Layout Editor를 통해 설비를 배치하고 포트를 연결했다면, Plan_05.md는 그 연결 정보를 바탕으로 아이템이 실제로 어떤 경로를 따라 이동할 수 있는지 판단하는 라우팅 체계를 정의한다.

이 문서의 핵심 목적은 다음과 같다.

```text
1. Layout JSON의 ports와 connections를 그래프 구조로 변환한다.
2. 아이템의 source, destination, itemType을 기준으로 route를 선택한다.
3. Diverter에서 목적지별 출구를 선택하는 방식을 정의한다.
4. Merge에서 여러 입력 중 어떤 아이템을 먼저 통과시킬지 정의한다.
5. 끊어진 경로, 잘못된 연결, 순환 경로를 검증하는 기준을 만든다.
6. MVP에서는 수동 route를 우선 사용하고, 고도화 단계에서 자동 경로 탐색으로 확장한다.
```

---

## 2. Routing의 기본 개념

Routing은 설비 네트워크에서 아이템이 이동할 다음 위치를 결정하는 기능이다.

기본 흐름은 다음과 같다.

```text
1. Layout JSON에서 components, ports, connections를 읽는다.
2. connections를 기준으로 directed graph를 만든다.
3. Scenario에서 생성된 item의 source, destination, itemType을 확인한다.
4. route가 명시되어 있으면 해당 route를 사용한다.
5. route가 없으면 연결 그래프에서 가능한 경로를 찾는다.
6. Diverter에서는 rule에 따라 output port를 선택한다.
7. Merge에서는 rule에 따라 통과 아이템을 선택한다.
8. 다음 컴포넌트가 받을 수 없으면 waiting 또는 blocked 상태로 전환한다.
```

MVP에서는 다음 원칙을 따른다.

```text
1. routes 배열에 명시된 route를 우선 사용한다.
2. 자동 경로 탐색은 보조 기능으로만 사용한다.
3. Diverter는 byDestination 규칙을 우선 구현한다.
4. Merge는 FIFO 규칙을 우선 구현한다.
5. 복잡한 최적 경로, 대체 경로, 동적 경로 변경은 고도화 단계로 미룬다.
```

---

## 3. Graph 모델

Routing은 connections 배열을 그래프 구조로 변환해서 처리한다.

```text
Node = Port
Edge = Connection
Component = Node들을 포함하는 설비
```

예시:

```text
SRC_001_OUT → CV_001_IN
CV_001_OUT → DIV_001_IN
DIV_001_OUT_A → CV_002_IN
DIV_001_OUT_B → CV_003_IN
```

TypeScript 구조:

```ts
export interface RoutingNode {
  portId: string;
  componentId: string;
  direction: "in" | "out" | "pick" | "place";
  outgoingConnectionIds: string[];
  incomingConnectionIds: string[];
}

export interface RoutingEdge {
  connectionId: string;
  fromPortId: string;
  toPortId: string;
  fromComponentId: string;
  toComponentId: string;
  enabled: boolean;
  type: "transport" | "control" | "virtual";
  travelTimeOverride?: number | null;
}

export interface RoutingGraph {
  nodesByPortId: Record<string, RoutingNode>;
  edgesByConnectionId: Record<string, RoutingEdge>;
  outgoingEdgesByPortId: Record<string, RoutingEdge[]>;
  incomingEdgesByPortId: Record<string, RoutingEdge[]>;
  componentToPortIds: Record<string, string[]>;
}
```

Graph 생성 흐름:

```text
1. ports 배열을 순회하며 RoutingNode를 생성한다.
2. components와 ports 관계를 componentToPortIds에 저장한다.
3. connections 배열을 순회한다.
4. enabled가 false인 connection은 제외한다.
5. from, to port가 존재하는지 확인한다.
6. RoutingEdge를 생성한다.
7. outgoingEdgesByPortId와 incomingEdgesByPortId에 등록한다.
```

---

## 4. Route 모델

Route는 특정 itemType 또는 destination에 대해 아이템이 따라갈 경로를 명시적으로 정의한다.

```json
{
  "id": "ROUTE_BOX_A",
  "name": "BOX_A to Pallet 1",
  "itemType": "BOX_A",
  "source": "SRC_001",
  "destination": "PALLET_001",
  "path": ["SRC_001", "CV_001", "DIV_001", "CV_002", "RB_001", "PALLET_001"],
  "connectionPath": ["CONN_001", "CONN_002", "CONN_003", "CONN_004"],
  "enabled": true
}
```

Route 선택 우선순위:

```text
1. item에 routeId가 직접 지정되어 있으면 해당 route 사용
2. itemType과 destination이 모두 일치하는 route 사용
3. itemType이 일치하는 route 사용
4. source와 destination이 일치하는 route 사용
5. destination이 일치하는 route 사용
6. route가 없으면 자동 경로 탐색 시도
7. 그래도 없으면 ROUTE_NOT_FOUND 오류 발생
```

아이템은 현재 Route에서 어디까지 이동했는지 기억해야 한다.

```ts
export interface ItemRouteState {
  routeId: string;
  path: string[];
  connectionPath?: string[];
  currentPathIndex: number;
  source: string;
  destination: string;
}
```

---

## 5. Next Component 결정

DES Engine은 아이템이 현재 컴포넌트에서 나갈 때 다음 컴포넌트를 알아야 한다.

```ts
getNextComponent(item, currentComponentId): NextComponentResult
```

결과 구조:

```ts
export interface NextComponentResult {
  ok: boolean;
  nextComponentId?: string;
  nextPortId?: string;
  connectionId?: string;
  reason?: string;
}
```

명시 Route 기준 로직:

```text
1. item.routeState를 확인한다.
2. currentComponentId가 routeState.path[currentPathIndex]와 일치하는지 확인한다.
3. 다음 index를 계산한다.
4. 다음 componentId를 가져온다.
5. connectionPath가 있으면 해당 connection을 사용한다.
6. connectionPath가 없으면 current component의 out port에서 다음 component의 in port로 연결된 connection을 찾는다.
7. 찾으면 nextComponentId, nextPortId, connectionId를 반환한다.
8. 찾지 못하면 ROUTE_CONNECTION_NOT_FOUND 오류를 반환한다.
```

---

## 6. Diverter Routing

Diverter는 아이템을 여러 output port 중 하나로 보낸다.

지원 규칙:

```text
byDestination
- 목적지 또는 route 기준으로 output port 선택

shortestQueue
- 하류 대기열이 가장 짧은 output port 선택

roundRobin
- output port를 순서대로 교대 선택

priority
- 아이템 우선순위 또는 목적지 우선순위 기준 선택
```

MVP에서는 `byDestination`을 우선 구현한다.

byDestination 로직:

```text
1. 아이템의 destination을 확인한다.
2. 아이템의 routeState를 확인한다.
3. routeState에서 Diverter 다음 컴포넌트를 찾는다.
4. Diverter의 out port 중 다음 컴포넌트로 연결된 port를 찾는다.
5. 해당 port를 선택한다.
6. 선택한 port의 하류가 받을 수 있으면 이동한다.
7. 받을 수 없으면 Diverter 또는 상류 설비에서 대기한다.
```

---

## 7. Merge Routing

Merge는 여러 입력 라인에서 들어오는 아이템 중 하나를 선택해 output port로 배출한다.

지원 규칙:

```text
fifo
- 먼저 도착한 아이템 우선

roundRobin
- 입력 라인별 교대 배출

mainLinePriority
- 메인 라인 우선

priority
- 아이템 우선순위 기준
```

MVP에서는 `fifo`를 우선 구현한다.

FIFO 로직:

```text
1. 각 input port 대기열을 확인한다.
2. 모든 대기 아이템 중 도착 시간이 가장 빠른 아이템을 찾는다.
3. output port의 하류 설비가 받을 수 있는지 확인한다.
4. 받을 수 있으면 해당 아이템을 배출한다.
5. 받을 수 없으면 Merge는 blocked 상태가 된다.
```

---

## 8. Buffer Routing

Buffer는 아이템이 하류 설비를 기다리는 대기 공간이다.

Routing 관점에서 Buffer는 다음 기능을 가진다.

```text
1. 진입 가능 여부 판단
2. 대기열 순서 결정
3. 하류 설비가 가능해졌을 때 배출 대상 선택
```

FIFO 로직:

```text
1. Buffer에 아이템이 도착한다.
2. 현재 queue length가 capacity보다 작으면 진입 허용
3. capacity 이상이면 상류 설비 blocked
4. 하류 설비가 available이면 queue의 첫 아이템을 배출
```

---

## 9. Robot Routing

Robot은 일반적인 연결 설비와 조금 다르다.
컨베이어처럼 아이템을 단순히 통과시키는 것이 아니라, resource로 점유되고 pick and place 작업을 수행한다.

Routing 관점에서 Robot은 다음을 결정해야 한다.

```text
1. 어떤 pick port에서 아이템을 받을 것인가
2. 어떤 place target으로 아이템을 보낼 것인가
3. 작업 완료 후 아이템의 다음 component는 무엇인가
```

기본 로직:

```text
1. 아이템이 Robot pick port에 도착한다.
2. Robot이 idle인지 확인한다.
3. idle이면 Robot 작업 이벤트를 시작한다.
4. cycleTime 이후 placeTarget에 아이템을 전달한다.
5. placeTarget이 Pallet Station이면 PALLETIZE_COMPLETE 이벤트를 발생시킨다.
6. routeState를 다음 단계로 이동한다.
```

---

## 10. Path Validation

Layout Editor에서 포트 연결은 사용자가 직접 만들기 때문에, 저장 전과 시뮬레이션 실행 전에 경로 검증이 필요하다.

검증 항목:

```text
1. route.source가 존재하는가
2. route.destination이 존재하는가
3. route.path의 모든 component가 존재하는가
4. route.path의 인접 component 간 연결이 존재하는가
5. route.connectionPath의 모든 connection이 존재하는가
6. connectionPath가 path 순서와 일치하는가
7. source component는 source 타입인가
8. destination component는 palletStation 또는 sink 등 적절한 타입인가
9. Diverter에서 다음 경로로 나가는 output port가 존재하는가
10. Merge에서 output port가 하나 이상 존재하는가
```

오류 예시:

```json
{
  "code": "ROUTE_BROKEN_PATH",
  "message": "Route ROUTE_BOX_A has no connection from CV_001 to RB_001.",
  "path": "routes[0].path",
  "severity": "error"
}
```

---

## 11. Deadlock과 Loop 처리

Loop는 route path 안에서 같은 component가 반복되는 경우다.

```text
SRC_001 → CV_001 → CV_002 → CV_001 → SINK_001
```

Loop는 항상 오류는 아니다.
순환 컨베이어나 재작업 라인에서는 의도적으로 발생할 수 있다.

MVP에서는 loop가 있으면 warning으로 처리한다.

Deadlock은 서로가 서로를 기다리면서 흐름이 멈추는 상태다.

MVP에서는 Deadlock을 완전히 해결하지 않고, 감지 후 report한다.

```text
deadlockPolicy = stopAndReport
```

단순 감지 기준:

```text
1. Event Queue에 더 이상 처리 가능한 이벤트가 없다.
2. 아직 completed 되지 않은 item이 있다.
3. 모든 active item이 waiting 또는 blocked 상태다.
4. 일정 시간 이상 상태 변화가 없다.
```

---

## 12. 자동 경로 탐색

고도화 단계에서는 사용자가 route를 직접 정의하지 않아도 source와 destination만으로 경로를 찾을 수 있어야 한다.

MVP에서는 자동 경로 탐색을 보조 기능으로만 사용한다.

기본 탐색은 BFS를 사용한다.

```text
1. source component의 out port들을 시작점으로 둔다.
2. outgoing connection을 따라 이동한다.
3. to port가 속한 component를 방문한다.
4. destination component에 도착하면 경로를 반환한다.
5. 이미 방문한 port는 다시 방문하지 않는다.
```

결과 구조:

```ts
export interface PathSearchResult {
  found: boolean;
  componentPath: string[];
  connectionPath: string[];
  reason?: string;
}
```

---

## 13. Runtime Routing API

DES Engine이 사용할 Routing API는 다음과 같이 정의한다.

```ts
function createRoutingGraph(layout: LayoutJson): RoutingGraph
```

```ts
function selectRouteForItem(
  item: SimulationItem,
  layout: LayoutJson
): LayoutRoute | null
```

```ts
function getNextComponent(
  item: SimulationItem,
  currentComponentId: string,
  context: RoutingContext
): NextComponentResult
```

```ts
function decideDiverterOutput(
  item: SimulationItem,
  diverterId: string,
  context: RoutingContext
): DiverterDecision
```

```ts
function decideMergeRelease(
  mergeId: string,
  waitingItems: SimulationItem[],
  context: RoutingContext
): MergeDecision
```

```ts
function validateRoutes(
  layout: LayoutJson,
  graph: RoutingGraph
): ValidationResult
```

---

## 14. TypeScript 타입 초안

```ts
export interface RoutingNode {
  portId: string;
  componentId: string;
  direction: "in" | "out" | "pick" | "place";
  outgoingConnectionIds: string[];
  incomingConnectionIds: string[];
}

export interface RoutingEdge {
  connectionId: string;
  fromPortId: string;
  toPortId: string;
  fromComponentId: string;
  toComponentId: string;
  enabled: boolean;
  type: "transport" | "control" | "virtual";
  travelTimeOverride?: number | null;
}

export interface RoutingGraph {
  nodesByPortId: Record<string, RoutingNode>;
  edgesByConnectionId: Record<string, RoutingEdge>;
  outgoingEdgesByPortId: Record<string, RoutingEdge[]>;
  incomingEdgesByPortId: Record<string, RoutingEdge[]>;
  componentToPortIds: Record<string, string[]>;
}

export interface ItemRouteState {
  routeId: string;
  path: string[];
  connectionPath?: string[];
  currentPathIndex: number;
  source: string;
  destination: string;
}

export interface NextComponentResult {
  ok: boolean;
  nextComponentId?: string;
  nextPortId?: string;
  connectionId?: string;
  reason?: string;
}

export interface DiverterDecision {
  ok: boolean;
  outputPortId?: string;
  connectionId?: string;
  reason?: string;
}

export interface MergeDecision {
  ok: boolean;
  itemId?: string;
  inputPortId?: string;
  outputPortId?: string;
  reason?: string;
}

export interface PathSearchResult {
  found: boolean;
  componentPath: string[];
  connectionPath: string[];
  reason?: string;
}
```

---

## 15. MVP 구현 범위

### 포함

```text
- Layout JSON에서 RoutingGraph 생성
- enabled connection만 graph에 포함
- 명시 Route 선택
- itemType 기준 Route 선택
- source/destination 기준 Route 선택
- getNextComponent 구현
- Diverter byDestination 구현
- Merge FIFO 구현
- Buffer FIFO 배출 기준 정의
- route path 검증
- connection path 검증
- 끊어진 경로 오류 표시
```

### 제외

```text
- 동적 최적 경로 탐색
- queue length 기반 실시간 rerouting
- Dijkstra, A* 기반 가중치 탐색
- 복잡한 deadlock 자동 해소
- 다중 목적지 동시 최적화
- AGV, AMR 동적 경로 충돌 회피
```

---

## 16. 작업 목록

### 16.1 타입 정의 작업

* [ ] `src/types/routing.ts` 생성
* [ ] `RoutingNode` 타입 정의
* [ ] `RoutingEdge` 타입 정의
* [ ] `RoutingGraph` 타입 정의
* [ ] `ItemRouteState` 타입 정의
* [ ] `NextComponentResult` 타입 정의
* [ ] `DiverterDecision` 타입 정의
* [ ] `MergeDecision` 타입 정의
* [ ] `PathSearchResult` 타입 정의

### 16.2 Graph 생성 작업

* [ ] `src/simulation/routing/createRoutingGraph.ts` 생성
* [ ] ports 기반 node 생성
* [ ] connections 기반 edge 생성
* [ ] outgoingEdgesByPortId 생성
* [ ] incomingEdgesByPortId 생성
* [ ] componentToPortIds 생성
* [ ] enabled false connection 제외

### 16.3 Route 선택 작업

* [ ] `selectRouteForItem` 구현
* [ ] routeId 직접 지정 우선
* [ ] itemType + destination 매칭
* [ ] itemType 매칭
* [ ] source + destination 매칭
* [ ] destination 매칭
* [ ] route 없음 오류 처리

### 16.4 Next Component 작업

* [ ] `getNextComponent` 구현
* [ ] routeState 기반 다음 component 계산
* [ ] connectionPath 기반 connection 선택
* [ ] connectionPath 없을 때 graph에서 연결 찾기
* [ ] 다음 port 반환
* [ ] 경로 끝 도달 처리

### 16.5 검증 작업

* [ ] `validateRoutes` 구현
* [ ] route source 존재 검증
* [ ] route destination 존재 검증
* [ ] path component 존재 검증
* [ ] connectionPath 존재 검증
* [ ] path와 connectionPath 연결 일치 검증
* [ ] route loop warning 구현
* [ ] broken path error 구현

---

## 17. 완료 기준

```text
- ports와 connections를 RoutingGraph로 변환하는 구조가 정의되어 있다.
- Route 선택 우선순위가 정의되어 있다.
- getNextComponent 로직이 정의되어 있다.
- Diverter byDestination 로직이 정의되어 있다.
- Merge FIFO 로직이 정의되어 있다.
- Buffer의 routing 역할이 정의되어 있다.
- Robot의 routing 역할이 정의되어 있다.
- route path 검증 기준이 정의되어 있다.
- broken path, loop, deadlock 처리 기준이 정의되어 있다.
- TypeScript 타입 초안이 작성되어 있다.
- Codex가 routing 모듈을 구현할 수 있는 수준이다.
```

---

## 18. Codex 작업 지시 예시

```text
MasterPlan.md, Plan_01.md, Plan_02.md, Plan_03.md, Plan_04.md, Plan_05.md를 읽고,
src/types/routing.ts 파일을 만들어줘.

Plan_05.md의 TypeScript 타입 초안을 기준으로 다음 타입을 구현해.
- RoutingNode
- RoutingEdge
- RoutingGraph
- ItemRouteState
- NextComponentResult
- DiverterDecision
- MergeDecision
- PathSearchResult

아직 DES Engine 본체는 구현하지 마.
```

---

## 19. 다음 단계

Plan_05가 완료되면 다음 문서로 이동한다.

```text
Plan_06.md
```

Plan_06에서는 시뮬레이션에 투입할 물동량과 운영 조건을 담는 **Input Scenario Data**를 정의한다.
::: 
