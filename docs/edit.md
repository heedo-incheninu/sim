# edit.md

# 물류 라인 DES Web Simulator 계획서 수정·개선 사항

## 0. 문서 목적

이 문서는 `MasterPlan.md`와 `Plan_01.md`부터 `Plan_10.md`까지 작성된 개발 계획서에서 **수정하거나 보완해야 할 핵심 사항**을 정리한 문서이다.

현재 전체 구조는 다음 원칙을 잘 따르고 있다.

```text
Layout JSON = 설비 세계의 지도
Scenario JSON = 물동량의 시간표
DES Engine = 사건을 계산하는 두뇌
Event Timeline = 지나간 사건의 필름
Three.js = 편집과 재생 화면
Report = 결과를 판단으로 바꾸는 해석기
```

다만 실제 개발에 들어가기 전에 몇 가지 기준을 더 명확히 하지 않으면, 구현 중 의미가 꼬이거나 테스트 기대값이 맞지 않을 수 있다.

이 문서는 그 위험을 줄이기 위한 수정 지시서다.

---

## 1. 완료 수량 예시와 테스트 기대값 수정

### 1.1 현재 문제

현재 문서에는 다음 조건이 여러 번 등장한다.

```text
투입: 600 box/hour
투입 간격: 6초
컨베이어 이동 시간: 10초
로봇 cycleTime: 8초
시뮬레이션 시간: 3600초
```

이 조건에서 로봇은 8초에 1개를 처리한다.

```text
3600초 / 8초 = 450 box/hour
```

또한 첫 박스는 컨베이어에서 10초 이동한 후 로봇에 도착한다.
따라서 3600초 안에 완료되는 박스 수는 대략 448개 수준이 된다.

그런데 기존 문서에는 다음과 같은 예시가 있다.

```text
완료 수량: 520개
로봇 처리 능력이 투입량보다 충분한가?
```

이 값과 해석은 현재 조건과 맞지 않는다.

### 1.2 수정 방향

테스트 시나리오를 두 가지로 분리한다.

#### A. 병목 없는 기본 테스트

```text
시뮬레이션 시간: 3600초
투입 간격: 10초
투입 수량: 360개/hour
컨베이어 이동 시간: 10초
로봇 cycleTime: 8초
예상 결과: 로봇 대기 거의 없음
```

의도:

```text
- Source → Conveyor → Robot → Pallet Station 흐름이 정상 작동하는지 확인
- EventQueue, travelTime, cycleTime, PALLETIZE_COMPLETE 검증
- 병목이 없는 기준 상태 확인
```

#### B. 로봇 병목 테스트

```text
시뮬레이션 시간: 3600초
투입 간격: 6초
투입 수량: 600개/hour
컨베이어 이동 시간: 10초
로봇 cycleTime: 8초
예상 결과: 로봇 병목 발생
예상 완료 수량: 약 448개
예상 WIP: 약 152개
```

의도:

```text
- Robot queue 증가 확인
- waiting time 증가 확인
- robot utilization 상승 확인
- bottleneck candidate에 RB_001 포함 확인
```

### 1.3 문서 수정 대상

```text
Plan_01.md
Plan_06.md
Plan_07.md
Plan_08.md
Plan_10.md
개발 프롬프트 목록
```

특히 다음 표현은 수정한다.

```text
기존:
로봇 처리 능력이 투입량보다 충분한가?

수정:
투입 간격 6초, 로봇 cycleTime 8초 조건에서는 로봇 처리 능력이 부족하므로 대기와 WIP가 증가해야 한다.
```

---

## 2. Pallet Station과 Sink의 역할 분리

### 2.1 현재 문제

현재 흐름은 다음처럼 표현되어 있다.

```text
Robot → Pallet Station → Sink
```

동시에 `PALLETIZE_COMPLETE`가 발생하면 박스를 completed로 처리하는 내용도 있다.

이 때문에 다음 질문이 생긴다.

```text
박스는 Pallet Station에서 완료인가?
아니면 Sink까지 가야 완료인가?
Sink는 박스 완료 지점인가?
아니면 완성 팔레트 출고 지점인가?
```

### 2.2 수정 방향

MVP에서는 박스 단위 완료와 팔레트 단위 완료를 분리한다.

#### 박스 단위 흐름

```text
Source → Conveyor → Robot → Pallet Station
```

박스는 로봇이 팔레트에 적재하는 순간 완료로 본다.

```text
PALLETIZE_COMPLETE 발생 시 BOX completed
```

#### 팔레트 단위 흐름

```text
Pallet Station → Sink
```

팔레트가 maxBoxes에 도달하면 팔레트 완료 이벤트를 발생시킨다.

```text
PALLET_COMPLETE 발생 시 완성 팔레트 출고 가능
```

Sink는 박스가 아니라 **완성 팔레트의 출고 지점**으로 정의한다.

### 2.3 MVP 권장 기준

MVP에서는 다음처럼 단순화한다.

```text
BOX 완료 기준:
PALLETIZE_COMPLETE

Sink:
선택 사항
완성 팔레트 출고를 표현할 때만 사용
```

즉, 첫 번째 MVP 테스트 라인은 다음으로 충분하다.

```text
SRC_001 → CV_001 → RB_001 → PALLET_001
```

Sink를 유지할 경우 의미를 다음처럼 명확히 한다.

```text
SINK_001 = Completed Pallet Sink
BOX_A는 Sink로 이동하지 않는다.
완성된 Pallet만 Sink로 이동한다.
```

### 2.4 문서 수정 대상

```text
MasterPlan.md
Plan_01.md
Plan_03.md
Plan_06.md
Plan_07.md
Plan_08.md
Plan_09.md
Plan_10.md
```

특히 `Source → Conveyor → Robot → Pallet Station → Sink`라고 되어 있는 MVP 흐름은 다음 중 하나로 수정한다.

```text
권장:
Source → Conveyor → Robot → Pallet Station

또는:
Source → Conveyor → Robot → Pallet Station
Pallet Station → Sink는 팔레트 완료 후 출고 흐름
```

---

## 3. Robot 이벤트명 수정

### 3.1 현재 문제

현재 이벤트 이름은 다음처럼 정의되어 있다.

```text
ROBOT_PICK_START
ROBOT_PICK_END
ROBOT_PLACE_START
ROBOT_PLACE_END
PALLETIZE_COMPLETE
```

그런데 MVP에서는 `cycleTime`을 pick, move, place 전체 작업 시간으로 사용하고 있다.

즉, `ROBOT_PICK_END`라는 이름은 실제 의미와 맞지 않는다.

```text
현재 cycleTime = pick + move + place 전체 작업 시간
하지만 이벤트명은 pick만 끝난 것처럼 보임
```

### 3.2 수정 방향

MVP에서는 로봇 작업을 하나의 task로 단순화한다.

수정 이벤트:

```text
ROBOT_TASK_START
ROBOT_TASK_END
PALLETIZE_COMPLETE
```

의미:

```text
ROBOT_TASK_START
- 로봇이 아이템 하나에 대한 pick, move, place 전체 작업을 시작

ROBOT_TASK_END
- 로봇의 cycleTime이 끝남

PALLETIZE_COMPLETE
- 박스가 Pallet Station에 적재 완료됨
```

### 3.3 고도화 단계 이벤트

나중에 실제 로봇 모션을 세분화할 경우 다음 이벤트를 사용할 수 있다.

```text
ROBOT_PICK_START
ROBOT_PICK_END
ROBOT_MOVE_START
ROBOT_MOVE_END
ROBOT_PLACE_START
ROBOT_PLACE_END
PALLETIZE_COMPLETE
```

하지만 MVP에서는 사용하지 않는다.

### 3.4 문서 수정 대상

```text
Plan_02.md
Plan_03.md
Plan_05.md
Plan_07.md
Plan_08.md
Plan_09.md
개발 프롬프트 09~20
```

수정 예시:

```text
기존:
ROBOT_PICK_START
ROBOT_PICK_END

수정:
ROBOT_TASK_START
ROBOT_TASK_END
```

---

## 4. Robot Port 구조 일관화

### 4.1 현재 문제

현재 Robot 포트는 다음처럼 혼합되어 있다.

```json
"ports": {
  "in": ["RB_001_PICK"],
  "out": ["RB_001_OUT"]
}
```

그런데 실제 port 객체에서는 direction이 다음처럼 되어 있다.

```json
{
  "id": "RB_001_PICK",
  "direction": "pick"
}
```

즉, component의 ports 그룹에서는 `in`에 들어가 있지만, port의 direction은 `pick`이다.

구현 자체는 가능하지만, 초기 개발에서는 혼란을 만들 수 있다.

### 4.2 수정 방향 A: MVP 단순화 방식

MVP에서는 Robot도 일반 컴포넌트처럼 `in/out` 구조로 통일한다.

```json
"ports": {
  "in": ["RB_001_IN"],
  "out": ["RB_001_OUT"]
}
```

Port 객체:

```json
{
  "id": "RB_001_IN",
  "componentId": "RB_001",
  "direction": "in"
}
```

Robot properties:

```json
{
  "pickPort": "RB_001_IN",
  "placeTarget": "PALLET_001"
}
```

이 방식이 MVP에 가장 적합하다.

### 4.3 수정 방향 B: 고도화 방식

고도화 단계에서는 포트 그룹을 확장할 수 있다.

```json
"ports": {
  "in": [],
  "out": ["RB_001_OUT"],
  "pick": ["RB_001_PICK"],
  "place": []
}
```

그러나 이 경우 Layout 타입, Validator, Editor, Routing 모두 변경해야 하므로 MVP에서는 피한다.

### 4.4 최종 권장

MVP에서는 방법 A를 사용한다.

```text
Robot pick port도 in port로 표현한다.
RB_001_PICK 대신 RB_001_IN을 사용한다.
pickPort property는 RB_001_IN을 참조한다.
```

### 4.5 문서 수정 대상

```text
Plan_02.md
Plan_03.md
Plan_04.md
Plan_05.md
Plan_07.md
Plan_09.md
개발 프롬프트 03, 04, 07, 08, 10, 16, 18
```

---

## 5. Blocking 해제와 Retry 이벤트 명확화

### 5.1 현재 문제

문서에는 하류 설비가 가능해지면 blocked 상태가 해제된다고 되어 있다.

그러나 다음 내용이 명확하지 않다.

```text
누가 retry를 발생시키는가?
언제 blocked item을 다시 이동시키는가?
어떤 이벤트 이름을 사용하는가?
```

이 부분이 없으면 아이템이 한 번 막힌 뒤 계속 멈춰 있을 수 있다.

### 5.2 수정 방향

MVP에서는 다음 이벤트를 추가한다.

```text
RETRY_RELEASE
```

### 5.3 RETRY_RELEASE 정의

```text
RETRY_RELEASE
- blocked 또는 waiting 상태의 아이템이 다시 다음 컴포넌트로 이동 가능한지 확인하는 이벤트
```

### 5.4 발생 조건

다음 상황에서 RETRY_RELEASE를 예약한다.

```text
1. Robot 작업 완료 후 queue 또는 upstream blocked item이 있는 경우
2. Buffer에서 item이 빠져 capacity가 생긴 경우
3. Merge output이 비워진 경우
4. Conveyor에서 item이 빠져 capacity가 생긴 경우
```

### 5.5 처리 로직

```text
1. blocked component 또는 waiting queue를 확인한다.
2. 가장 먼저 대기 중인 item을 선택한다.
3. canAcceptItem(nextComponent)를 다시 호출한다.
4. 가능하면 ITEM_WAIT_END 또는 BLOCKING_END를 기록한다.
5. 다음 컴포넌트로 ITEM_ENTER_COMPONENT 이벤트를 등록한다.
6. 아직 불가능하면 대기 상태를 유지한다.
```

### 5.6 문서 수정 대상

```text
Plan_07.md
Plan_08.md
Plan_09.md
개발 프롬프트 11, 12
```

이벤트 타입에 다음을 추가한다.

```text
RETRY_RELEASE
```

---

## 6. Connection 이동 시간 기준 명확화

### 6.1 현재 문제

Layout JSON의 connection에는 다음 필드가 있다.

```json
"travelTimeOverride": null
```

그런데 MVP 설명에서는 컨베이어 이동 시간만 사용한다.

즉, 다음 질문이 생긴다.

```text
Connection 자체에도 이동 시간이 있는가?
Source에서 Conveyor까지 이동 시간은 0인가?
Conveyor에서 Robot까지 이동 시간은 0인가?
```

### 6.2 수정 방향

MVP에서는 connection 이동 시간을 0으로 간주한다.

```text
Connection = 논리적 연결
실제 시간 소요 이동 = Conveyor와 Robot에서만 발생
```

MVP의 시간 소요 설비:

```text
Conveyor
- travelTime = length / speed

Robot
- taskTime = cycleTime

Process Station
- processTime
```

Connection travelTime은 고도화 단계에서 사용한다.

```text
connection.properties.travelTimeOverride는 MVP에서는 무시한다.
향후 shuttle, lift, virtual transfer 등에 사용한다.
```

### 6.3 문서 수정 대상

```text
Plan_03.md
Plan_05.md
Plan_07.md
Plan_08.md
Plan_09.md
```

---

## 7. 타입 중복 방지 원칙 추가

### 7.1 현재 문제

문서마다 다음 타입이 반복 등장한다.

```text
SimulationSummary
SimulationResult
SimulationEventLog
ComponentStatistics
ItemStatistics
```

Codex가 구현하면 파일별로 중복 정의할 위험이 있다.

예상 문제:

```text
src/types/simulation.ts에 SimulationSummary
src/types/result.ts에 SimulationSummary
src/types/report.ts에 또 다른 Summary 타입
```

이렇게 되면 import 관계가 꼬이고 타입 호환성이 깨진다.

### 7.2 수정 방향

타입 소유권을 명확히 한다.

```text
src/types/common.ts
- Vector3Like
- Dimensions
- ValidationError
- ValidationResult

src/types/component.ts
- ComponentType
- ComponentState
- Port
- BaseComponent
- Component별 타입

src/types/layout.ts
- LayoutJson
- ProjectInfo
- CoordinateSystem
- LayoutConnection
- LayoutRoute
- LayoutRules

src/types/scenario.ts
- ScenarioJson
- ItemMaster
- ArrivalSchedule
- ResourceOverride

src/types/routing.ts
- RoutingGraph
- RoutingNode
- RoutingEdge
- ItemRouteState

src/types/simulation.ts
- SimulationInput
- SimulationContext
- SimulationEvent
- SimulationItem
- ComponentRuntimeState

src/types/result.ts
- SimulationResult
- SimulationSummary
- SimulationEventLog
- ItemTimeline
- ComponentStateTimeline
- ComponentStatistics
- ItemStatistics

src/types/animation.ts
- PlaybackState
- TimelineFrame
- EvaluatedItemState

src/types/report.ts
- ReportModel
- BottleneckReport
- Recommendation
- ComparisonReport
```

### 7.3 문서 수정 대상

```text
MasterPlan.md
Plan_07.md
Plan_08.md
Plan_10.md
개발 프롬프트 01, 09, 13, 21
```

---

## 8. React Three Fiber 사용 여부 명확화

### 8.1 현재 문제

문서에서는 Three.js라고 표현하고, 개발 프롬프트에는 `@react-three/fiber`를 포함했다.

둘 다 맞지만 개발 지시에는 더 명확한 표현이 필요하다.

### 8.2 수정 방향

프로젝트 기술 스택을 다음처럼 명확히 한다.

```text
렌더링 엔진:
- Three.js

React 통합:
- @react-three/fiber

상태 관리:
- Zustand

빌드:
- Vite

언어:
- TypeScript
```

즉, 문서에서 `Three.js 화면`이라고 표현하되, 구현 지시에서는 다음 문장을 넣는다.

```text
React 환경에서는 @react-three/fiber를 사용해 Three.js scene을 구성한다.
순수 Three.js imperative 코드보다 React component 기반으로 작성한다.
```

### 8.3 문서 수정 대상

```text
MasterPlan.md
Plan_04.md
Plan_09.md
개발 프롬프트 00, 16, 20
```

---

## 9. Uniform 투입 방식 고정

### 9.1 현재 상태

현재 Uniform 계산은 다음으로 되어 있다.

```text
duration = endTime - startTime
interval = duration / quantity
첫 아이템은 startTime에 생성
```

이 방식은 좋다.

다만 혼동을 막기 위해 다음 문장을 추가해야 한다.

### 9.2 추가 문장

```text
quantity=600, startTime=0, endTime=3600인 경우
0초부터 3594초까지 6초 간격으로 총 600개를 생성한다.

endTime은 마지막 생성 시각이 아니라 시뮬레이션 종료 경계다.
따라서 interval은 duration / quantity로 계산한다.
duration / (quantity - 1)을 사용하지 않는다.
```

### 9.3 문서 수정 대상

```text
Plan_06.md
개발 프롬프트 06
```

---

## 10. MVP 범위 재정의

### 10.1 현재 문제

문서에서 MVP라는 말이 넓게 사용된다.

MVP 안에 다음 기능들이 섞여 있다.

```text
Diverter
Merge
Buffer
Robot
Pallet
Animation
Report
Export
Comparison
Recommendation
도면 배경
스케일 보정
```

이대로 개발하면 MVP가 너무 커진다.

### 10.2 수정 방향

MVP를 단계별로 나눈다.

```text
MVP-0: 타입, 샘플, 검증
MVP-1: Source → Conveyor → Robot → Pallet Station 실행
MVP-2: Event Log와 Summary
MVP-3: Static Three.js Viewer
MVP-4: Animation Player
MVP-5: Layout Editor 기본 기능
MVP-6: Diverter, Merge, Buffer
MVP-7: Report, Export
```

### 10.3 최우선 완료 기준

문서 전체에 다음 기준을 추가한다.

```text
MVP의 가장 작은 완료 기준은 Source → Conveyor → Robot → Pallet Station 라인이 정상 실행되는 것이다.
```

### 10.4 문서 수정 대상

```text
MasterPlan.md
Plan_01.md
Plan_07.md
Plan_10.md
개발 프롬프트 순서 문서
```

---

## 11. 권장 수정 문구 모음

아래 문구를 MasterPlan.md 또는 Plan_01.md에 추가하는 것을 권장한다.

```text
MVP의 기본 완료 기준은 Source → Conveyor → Robot → Pallet Station 라인이 정상 실행되는 것이다.
Sink는 MVP에서 선택 사항이며, 박스 단위 완료는 Pallet Station의 PALLETIZE_COMPLETE 시점으로 본다.
Robot cycleTime은 MVP에서 pick, move, place 전체 작업 시간을 의미한다.
Connection 자체의 이동 시간은 MVP에서 0으로 간주하고, 시간 소요 이동은 Conveyor와 Robot에서만 발생한다.
Robot pick port는 MVP에서 일반 in port로 표현한다.
Blocked 또는 waiting 상태의 아이템은 RETRY_RELEASE 이벤트를 통해 이동을 재시도한다.
```

---

## 12. 개발 프롬프트 수정 사항

### 12.1 Prompt 03 수정

기존:

```text
CV_001_OUT → RB_001_PICK
```

수정:

```text
CV_001_OUT → RB_001_IN
```

기존:

```text
RB_001_PICK
direction: pick
```

수정:

```text
RB_001_IN
direction: in
```

### 12.2 Prompt 06 수정

Uniform 변환 조건을 명확히 한다.

```text
startTime 0, endTime 3600, quantity 600이면
0, 6, 12, ... 3594초 이벤트가 생성되어야 한다.
interval은 (endTime - startTime) / quantity로 계산한다.
duration / (quantity - 1)을 사용하지 않는다.
```

### 12.3 Prompt 10 수정

기존:

```text
SRC_001 → CV_001 → RB_001 → PALLET_001 → SINK_001
```

수정:

```text
SRC_001 → CV_001 → RB_001 → PALLET_001
```

추가:

```text
BOX_A는 PALLETIZE_COMPLETE 시점에 completed로 처리한다.
SINK_001은 MVP에서 선택 사항이며, 완성 팔레트 출고 지점으로만 사용한다.
```

### 12.4 Prompt 11 수정

추가:

```text
Robot 작업 이벤트는 ROBOT_TASK_START, ROBOT_TASK_END를 사용한다.
ROBOT_TASK_END 이후 PALLETIZE_COMPLETE를 발생시킨다.
Robot이 queue의 다음 item을 처리할 수 있으면 RETRY_RELEASE 또는 즉시 ROBOT_TASK_START를 등록한다.
```

### 12.5 Prompt 13 수정

기존 이벤트:

```text
ROBOT_PICK_START
ROBOT_PICK_END
```

수정 이벤트:

```text
ROBOT_TASK_START
ROBOT_TASK_END
```

### 12.6 Prompt 20 수정

Robot Pick & Place 표현은 화면 표현 용어로만 사용한다.

```text
이벤트명은 ROBOT_TASK_START, ROBOT_TASK_END를 사용하되,
Animation Player에서는 이를 pick → lift → place 동작으로 시각화한다.
```

---

## 13. 우선 수정 순서

문서 전체를 한 번에 고치기보다 아래 순서로 수정하는 것이 좋다.

```text
1. MasterPlan.md
- MVP 완료 기준 추가
- Pallet Station과 Sink 역할 정리
- Robot cycleTime 의미 정리
- Connection travelTime MVP 기준 추가

2. Plan_01.md
- 테스트 시나리오를 병목 없는 테스트와 로봇 병목 테스트로 분리
- 완료 수량 예시 수정

3. Plan_02.md
- Robot port 구조를 RB_001_IN / RB_001_OUT으로 수정
- Robot 이벤트를 ROBOT_TASK_START / END로 수정

4. Plan_03.md
- mvpLayout에서 RB_001_PICK을 RB_001_IN으로 수정
- Sink 역할 주석 수정

5. Plan_06.md
- uniform interval 기준 명확화
- scenario 예시에서 기대 결과 문구 수정

6. Plan_07.md
- Robot task 이벤트명 수정
- RETRY_RELEASE 추가
- Pallet Station 완료 기준 수정

7. Plan_08.md
- Timeline 이벤트명 수정
- PALLETIZE_COMPLETE 기준 completed 처리 수정

8. Plan_09.md
- Animation 용어는 Pick & Place 유지
- 이벤트명은 ROBOT_TASK_START / END로 수정

9. Plan_10.md
- 완료 수량 예시와 bottleneck 예시 수정

10. 개발 프롬프트 문서
- Prompt 03, 06, 10, 11, 13, 20 수정
```

---

## 14. 최종 정리

지금까지 만든 문서의 전체 방향은 좋다.

특히 다음 구조는 유지해야 한다.

```text
Layout JSON 중심 설계
Three.js와 DES Engine 분리
Scenario JSON으로 물동량 분리
Event Log와 Timeline 분리
Animation Player는 재생만 담당
Report는 결과 해석 담당
```

다만 개발 전에 반드시 고쳐야 할 핵심은 다음 5개다.

```text
1. 완료 수량 예시와 테스트 기대값 수정
2. Pallet Station과 Sink 역할 분리
3. Robot 이벤트명을 cycleTime 의미에 맞게 수정
4. Robot port 구조를 MVP에서 in/out으로 단순화
5. Blocking 해제와 RETRY_RELEASE 이벤트 명확화
```

이 다섯 개를 반영하면 MasterPlan과 Plan_01~Plan_10은 훨씬 안정적인 개발 지시서가 된다.

현재 문서는 좋은 뼈대를 가지고 있다.
이 edit.md는 그 뼈대의 관절 방향을 맞추는 교정표다.
관절만 맞추면 컨베이어 세계는 훨씬 덜 삐걱거리고, 박스들은 자기 길을 더 조용히 찾아갈 것이다.

