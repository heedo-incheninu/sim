# Plan_09.md

# Plan_09. Three.js Animation Player Definition

## 1. 목적

이 문서는 웹 기반 물류 라인 이산 사건 시뮬레이션에서 사용할 **Three.js Animation Player**를 정의하기 위한 실행 계획서이다.

Plan_08.md에서 Event Log, Item Timeline, Component State Timeline, Animation Track을 정의했다면, Plan_09.md는 그 데이터를 Three.js 화면에서 실제로 재생하는 방법을 정의한다.

이 문서의 핵심 목적은 다음과 같다.

```text
1. Simulation Result를 Three.js 화면에서 재생하는 구조를 정의한다.
2. Timeline과 Animation Track을 기반으로 박스 위치를 계산하는 방식을 정의한다.
3. 시간 슬라이더, 배속, 일시정지, 되감기 기능을 정의한다.
4. 컨베이어 이동, 대기, 로봇 Pick & Place, 팔레트 적재 표현 방식을 정의한다.
5. 설비 상태에 따라 색상을 변경하고 병목 구간을 강조하는 방식을 정의한다.
6. 특정 시뮬레이션 시간의 상태를 복원하는 방식을 정의한다.
7. Codex가 Three.js Animation Player를 구현할 수 있도록 명세를 제공한다.
```

Plan_09는 시뮬레이션의 영사기다.
Plan_07이 사건을 계산했고, Plan_08이 그 사건을 필름으로 만들었다면, Plan_09는 그 필름을 화면에 비추는 단계다.

---

## 2. 이 문서의 위치

전체 문서 흐름에서 Plan_09는 다음 위치에 있다.

```text
MasterPlan.md
  ↓
Plan_01.md: Project Scope & MVP Definition
  ↓
Plan_02.md: Component Model Definition
  ↓
Plan_03.md: Layout JSON Schema Definition
  ↓
Plan_04.md: Three.js Layout Editor Definition
  ↓
Plan_05.md: Network Routing Definition
  ↓
Plan_06.md: Input Scenario Data Definition
  ↓
Plan_07.md: DES Simulation Engine Definition
  ↓
Plan_08.md: Event Log Timeline Definition
  ↓
Plan_09.md: Three.js Animation Player Definition
  ↓
Plan_10.md: Report and Roadmap
```

Plan_09는 다음 데이터를 입력으로 사용한다.

```text
Layout JSON
- 설비 위치, 포트 위치, 연결 구조

Simulation Result
- summary, eventLog, itemTimelines, componentStateTimelines, animationTracks

Animation Settings
- 배속, 표시 옵션, 색상 옵션
```

Plan_09의 출력은 화면이다.

```text
- 박스 이동
- 로봇 작업
- 팔레트 적재
- 설비 상태 색상
- 병목 강조
- 시간 슬라이더 상태
```

---

## 3. Animation Player의 기본 역할

Animation Player는 시뮬레이션 결과를 사용자가 다시 볼 수 있게 해준다.

주요 역할은 다음과 같다.

```text
1. Simulation Result를 불러온다.
2. Layout JSON을 기준으로 설비를 화면에 배치한다.
3. Animation Track 또는 Item Timeline을 기준으로 박스를 생성하고 이동시킨다.
4. Component State Timeline을 기준으로 설비 상태 색상을 변경한다.
5. 현재 재생 시간에 맞는 박스 위치와 설비 상태를 계산한다.
6. 시간 슬라이더와 배속 조절을 제공한다.
7. 병목, 대기, blocked 상태를 시각적으로 강조한다.
8. 사용자가 특정 시간으로 이동하면 해당 시간의 상태를 복원한다.
```

중요한 원칙은 다음과 같다.

```text
Animation Player는 시뮬레이션을 다시 계산하지 않는다.
이미 계산된 Timeline을 재생한다.
```

즉, Animation Player는 DES Engine이 아니다.
여기서는 이벤트를 새로 판단하지 않고, Plan_08에서 생성한 기록을 화면에 보여준다.

---

## 4. 전체 구조

Animation Player의 구조는 다음과 같다.

```text
[Layout JSON]
      ↓
[Static Layout Renderer]
      ↓
설비, 포트, 연결선 표시

[Simulation Result]
      ↓
[Timeline Controller]
      ↓
현재 시간 계산

[Animation Track Evaluator]
      ↓
현재 시간의 item position, visibility, state 계산

[Component State Evaluator]
      ↓
현재 시간의 component state 계산

[Three.js Scene]
      ↓
박스, 로봇, 팔레트, 색상, 병목 표시
```

모듈 구성은 다음과 같다.

```text
AnimationPlayer
- 전체 재생 컨트롤러

PlaybackController
- play, pause, stop, seek, speed 제어

TimelineController
- 현재 시뮬레이션 시간 관리

TrackEvaluator
- keyframe 기준 현재 위치 계산

ItemRenderer
- 박스 렌더링

ComponentStateRenderer
- 설비 상태 색상 렌더링

RobotAnimationRenderer
- 로봇 Pick & Place 표현

PalletRenderer
- 팔레트 적재 표현

BottleneckOverlay
- 병목과 대기 강조 표시

TimeSlider
- 시간 이동 UI

PlaybackStatsPanel
- 현재 시간의 처리량, WIP, 설비 상태 표시
```

---

## 5. 화면 구성

Animation Player 화면은 Layout Editor와 비슷하지만, 목적이 다르다.
편집이 아니라 재생과 분석이 중심이다.

```text
┌─────────────────────────────────────────────────────────────┐
│ Top Playback Toolbar                                         │
├───────────────────────────────┬─────────────────────────────┤
│ Three.js Animation View        │ Right Analysis Panel         │
│                               │ Current Time / KPI / Legend  │
├───────────────────────────────┴─────────────────────────────┤
│ Bottom Timeline Slider                                       │
└─────────────────────────────────────────────────────────────┘
```

### 5.1 Top Playback Toolbar

포함 기능:

```text
- Play
- Pause
- Stop
- Restart
- Step Forward
- Step Backward
- Speed Select
- Fit View
- Toggle Item Display
- Toggle Bottleneck Overlay
- Toggle Component Labels
- Toggle Port Display
```

### 5.2 Three.js Animation View

표시 요소:

```text
- 설비 레이아웃
- 컨베이어
- 로봇
- 팔레트 스테이션
- Sink
- 이동 중인 박스
- 대기 중인 박스
- 로봇 작업 중인 박스
- 적재된 박스
- 설비 상태 색상
- 병목 강조
```

### 5.3 Right Analysis Panel

표시 정보:

```text
- 현재 시뮬레이션 시간
- 현재 재생 배속
- 현재 WIP
- 현재 완료 수량
- 현재 active item 수
- 현재 busy 설비 목록
- 현재 blocked 설비 목록
- 선택한 설비의 상태
- 선택한 아이템의 Timeline
```

### 5.4 Bottom Timeline Slider

기능:

```text
- 현재 시간 표시
- 슬라이더 드래그로 seek
- 주요 이벤트 tick 표시
- deadlock 발생 시점 표시
- pallet complete 시점 표시
```

---

## 6. Playback Controller

### 6.1 역할

PlaybackController는 재생 상태를 관리한다.

```ts
export type PlaybackState = "idle" | "playing" | "paused" | "stopped" | "ended";
```

### 6.2 상태 구조

```ts
export interface PlaybackControllerState {
  playbackState: PlaybackState;
  currentTime: number;
  startTime: number;
  endTime: number;
  speed: number;
  lastRealTime?: number;
  loop: boolean;
}
```

### 6.3 기본 API

```ts
export interface PlaybackControllerApi {
  play(): void;
  pause(): void;
  stop(): void;
  restart(): void;
  seek(time: number): void;
  setSpeed(speed: number): void;
  stepForward(deltaTime: number): void;
  stepBackward(deltaTime: number): void;
}
```

### 6.4 재생 시간 계산

실제 시간과 시뮬레이션 시간은 분리한다.

```text
simulationDelta = realDelta * playbackSpeed
currentTime = currentTime + simulationDelta
```

예시:

```text
realDelta = 1초
playbackSpeed = 10
simulationDelta = 10초
```

즉, 10배속에서는 실제 1초 동안 시뮬레이션 시간이 10초 진행된다.

### 6.5 지원 배속

MVP에서 지원할 배속은 다음과 같다.

```text
0.5x
1x
2x
5x
10x
30x
60x
```

### 6.6 종료 조건

```text
currentTime >= endTime이면 playbackState를 ended로 변경한다.
loop가 true이면 startTime으로 돌아가 재생한다.
```

MVP에서는 loop는 선택 기능으로 둔다.

---

## 7. Timeline Controller

### 7.1 역할

Timeline Controller는 현재 시뮬레이션 시간에 해당하는 상태를 계산한다.

입력:

```text
currentTime
animationTracks
itemTimelines
componentStateTimelines
layout
```

출력:

```text
현재 visible item 목록
현재 item position
현재 item state
현재 component state
현재 robot state
현재 pallet state
```

### 7.2 API

```ts
export interface TimelineController {
  evaluate(time: number): TimelineFrame;
}
```

### 7.3 TimelineFrame 구조

```ts
export interface TimelineFrame {
  time: number;
  items: EvaluatedItemState[];
  components: EvaluatedComponentState[];
  robots: EvaluatedRobotState[];
  pallets: EvaluatedPalletState[];
}
```

### 7.4 EvaluatedItemState

```ts
export interface EvaluatedItemState {
  itemId: string;
  itemType: string;
  visible: boolean;
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
  state: string;
  componentId?: string;
}
```

### 7.5 EvaluatedComponentState

```ts
export interface EvaluatedComponentState {
  componentId: string;
  componentType: string;
  state: string;
  queueLength: number;
  currentItemIds: string[];
  colorState: "idle" | "busy" | "waiting" | "blocked" | "error";
}
```

---

## 8. Track Evaluator

### 8.1 역할

Track Evaluator는 Animation Track의 keyframes를 기준으로 현재 시간의 위치, 회전, 표시 여부를 계산한다.

### 8.2 기본 API

```ts
function evaluateTrack(
  track: AnimationTrack,
  time: number
): EvaluatedTrackState
```

### 8.3 EvaluatedTrackState

```ts
export interface EvaluatedTrackState {
  targetId: string;
  targetType: "item" | "component" | "robot" | "pallet";
  visible?: boolean;
  position?: { x: number; y: number; z: number };
  rotation?: { x: number; y: number; z: number };
  state?: string;
  payload?: Record<string, unknown>;
}
```

### 8.4 Keyframe 탐색

현재 시간 t에 대해 다음 keyframe을 찾는다.

```text
previousKeyframe = time <= t인 마지막 keyframe
nextKeyframe = time > t인 첫 keyframe
```

경우별 처리:

```text
previous와 next가 모두 있음
- 보간 처리

previous만 있음
- previous 상태 유지

next만 있음
- 아직 나타나기 전, visible false

둘 다 없음
- visible false
```

### 8.5 보간 방식

위치 보간:

```text
position = lerp(previous.position, next.position, progress)
progress = (time - previous.time) / (next.time - previous.time)
```

회전 보간:

```text
rotation = lerp(previous.rotation, next.rotation, progress)
```

MVP에서는 회전 보간은 단순 선형 보간으로 처리한다.

### 8.6 상태 보간

상태는 보간하지 않고 이전 keyframe 상태를 유지한다.

```text
state = previous.state
```

---

## 9. Item Animation

### 9.1 역할

Item Animation은 박스, 트레이 같은 아이템을 화면에 표시하고 이동시킨다.

MVP에서는 box 형태만 우선 구현한다.

### 9.2 박스 생성

아이템 타입의 dimensions를 기준으로 box mesh를 생성한다.

```text
BOX_A dimensions:
length: 0.4
width: 0.3
height: 0.25
```

Three.js 표현:

```text
BoxGeometry(length, height, width)
```

주의:

```text
Three.js에서 y축은 높이로 사용한다.
따라서 geometry args 순서는 x, y, z 기준에 맞춰야 한다.
```

### 9.3 표시 상태

아이템은 다음 상태에 따라 다르게 표시한다.

```text
moving
- 일반 박스 색상

waiting
- 약간 강조 또는 테두리 표시

processing
- 로봇 작업 중 표시

blocked
- 붉은 테두리 또는 깜박임

placed
- 팔레트 위 고정

completed
- 숨김 또는 Sink 위치에 짧게 표시 후 제거
```

### 9.4 성능 최적화

아이템 수가 많아지면 모든 박스를 개별 Mesh로 만드는 것은 부담이 될 수 있다.

MVP에서는 개별 Mesh로 시작한다.

고도화 단계에서는 다음을 검토한다.

```text
- InstancedMesh
- 일정 수 이상은 집계 표현
- 화면 밖 아이템 렌더링 생략
- completed item 숨김 처리
```

---

## 10. Conveyor Animation

### 10.1 역할

컨베이어 자체는 Layout JSON 기준으로 고정 표시된다.
움직이는 것은 컨베이어 위의 박스다.

### 10.2 컨베이어 방향 표시

컨베이어에는 방향 화살표를 표시한다.

```text
CV_001_IN → CV_001_OUT
```

방향은 port world position 또는 component.rotation을 기준으로 계산한다.

### 10.3 컨베이어 상태 색상

Component State Timeline을 기준으로 컨베이어 색상을 변경한다.

```text
idle: 회색
busy: 파란색
waiting: 노란색
blocked: 빨간색
```

MVP에서는 이 색상 체계를 사용하되, 나중에 visualSettings로 이동한다.

### 10.4 이동 보간

박스가 컨베이어를 이동하는 경우:

```text
startPosition = in port world position
endPosition = out port world position
startTime = segment.startTime
endTime = segment.endTime
```

현재 위치:

```text
position = lerp(startPosition, endPosition, progress)
```

---

## 11. Waiting Animation

### 11.1 역할

Waiting은 박스가 특정 위치에서 멈춰 있는 상태다.

예시:

```text
- Robot pick port 앞에서 대기
- Merge input port에서 대기
- Buffer 내부에서 대기
- Conveyor 출구에서 blocked
```

### 11.2 위치 표시

Waiting segment는 payload 또는 componentId를 기준으로 위치를 결정한다.

우선순위:

```text
1. segment.payload.position
2. segment.toPortId world position
3. segment.componentId 중심점
4. 이전 segment의 마지막 position
```

### 11.3 대기열 표시

같은 위치에 여러 아이템이 대기하면 겹쳐 보일 수 있다.

MVP에서는 간단히 offset을 적용한다.

```text
queueIndex = 대기열 내 순서
offset = queueIndex * 0.15m
```

### 11.4 색상 표시

```text
waiting: 노란색 테두리
blocked: 빨간색 테두리
```

---

## 12. Robot Pick & Place Animation

### 12.1 역할

Robot Animation은 로봇이 박스를 집어 팔레트 위치로 옮기는 장면을 보여준다.

MVP에서는 실제 6축 관절 운동을 구현하지 않는다.
대신 다음 방식으로 표현한다.

```text
1. 로봇 상태 색상 변경
2. 박스가 pick position에서 lift position으로 이동
3. lift position에서 place approach position으로 이동
4. place position으로 내려감
5. 팔레트 위에 고정
```

### 12.2 keyframe 구성

ROBOT_PICK_START와 ROBOT_PICK_END 사이를 4개 구간으로 나눈다.

예시:

```text
startTime = 10
endTime = 18
duration = 8

10초: pick position
12초: lift position
16초: place approach
18초: place position
```

비율:

```text
0%: pick
25%: lift
75%: place approach
100%: place
```

### 12.3 위치 계산

```text
pickPosition = robot pick port world position
placePosition = pallet target position
liftPosition = pickPosition + y축 1m
placeApproach = placePosition + y축 1m
```

### 12.4 로봇 모델 표현

MVP 로봇 모델은 단순 모델을 사용한다.

```text
- 원기둥 베이스
- 기둥
- 단순 팔
- 그리퍼
- 작업 반경 원
```

고도화 단계:

```text
- Blender GLB 로봇 모델
- 간단한 관절 회전
- 실제 6축 로봇 애니메이션
```

---

## 13. Pallet Animation

### 13.1 역할

Pallet Animation은 로봇이 적재한 박스를 팔레트 위에 누적 표시한다.

### 13.2 적재 위치 계산

PALLETIZE_COMPLETE 이벤트의 payload를 사용한다.

```text
boxIndex
layerIndex
positionInLayer
```

MVP에서는 grid pattern을 사용한다.

```text
boxesPerLayer = 20
가로 개수 = 5
세로 개수 = 4
```

```text
row = floor(positionInLayer / columns)
col = positionInLayer % columns
layer = layerIndex
```

좌표:

```text
x = palletCenter.x - palletWidth / 2 + boxWidth / 2 + col * boxWidth
z = palletCenter.z - palletLength / 2 + boxLength / 2 + row * boxLength
y = palletTopY + boxHeight / 2 + layer * boxHeight
```

### 13.3 적재된 박스 표시

적재 완료된 박스는 이동 아이템에서 팔레트 고정 아이템으로 전환한다.

```text
state: placed
visible: true
position: pallet slot position
```

---

## 14. Component State Rendering

Component State Rendering은 설비 상태에 따라 색상과 강조 효과를 적용한다.

상태 색상 기본값:

```text
idle: 회색
busy: 파란색
waiting: 노란색
blocked: 빨간색
disabled: 어두운 회색
failed: 검붉은색
```

상태 업데이트 흐름:

```text
1. TimelineController가 currentTime 기준 component state를 계산한다.
2. ComponentStateRenderer가 해당 component mesh를 찾는다.
3. state에 맞는 material을 적용한다.
4. blocked 또는 failed 상태는 강조 효과를 적용한다.
```

병목 후보로 선정된 설비는 별도 overlay를 표시한다.

```text
- 외곽선 두껍게 표시
- 붉은 pulse 효과
- label에 bottleneck rank 표시
```

---

## 15. Time Slider

Time Slider는 사용자가 특정 시뮬레이션 시간으로 이동할 수 있게 한다.

기능:

```text
- 현재 시간 표시
- 전체 시뮬레이션 시간 범위 표시
- 드래그로 seek
- 클릭으로 특정 시간 이동
- 주요 이벤트 tick 표시
```

seek 처리:

```text
1. playbackState를 paused로 변경한다.
2. currentTime을 선택 시간으로 설정한다.
3. TimelineController.evaluate(currentTime)을 호출한다.
4. Three.js Scene을 해당 상태로 즉시 갱신한다.
```

---

## 16. Current Time State 복원

특정 시간으로 이동했을 때 화면이 즉시 그 시간 상태로 복원되어야 한다.

복원 대상:

```text
- 아이템 visible 여부
- 아이템 위치
- 아이템 상태
- 설비 상태 색상
- 로봇 상태
- 팔레트 적재 상태
- 대기열 표시
```

MVP에서는 Animation Track과 Timeline을 직접 평가한다.

```text
1. 모든 item track을 currentTime 기준으로 evaluate
2. visible true인 item만 화면에 표시
3. 모든 component state timeline을 currentTime 기준으로 evaluate
4. component material 갱신
5. pallet placed item 목록 갱신
```

---

## 17. 선택과 상세 정보

### 17.1 아이템 선택

사용자가 박스를 클릭하면 Right Analysis Panel에 item timeline을 표시한다.

```text
- itemId
- itemType
- currentState
- currentComponent
- createdAt
- completedAt
- leadTime
- totalWaitingTime
- totalProcessingTime
- routeId
```

### 17.2 설비 선택

사용자가 설비를 클릭하면 설비 상태와 통계를 표시한다.

```text
- componentId
- componentType
- currentState
- currentQueueLength
- processedCount
- busyTime
- idleTime
- blockedTime
- utilization
- bottleneck rank
```

---

## 18. Display Options

```ts
export interface AnimationDisplayOptions {
  showItems: boolean;
  showCompletedItems: boolean;
  showLabels: boolean;
  showPorts: boolean;
  showConnections: boolean;
  showBottleneckOverlay: boolean;
  showRobotWorkingRadius: boolean;
  showWaitingItems: boolean;
  showBlockedHighlight: boolean;
}
```

MVP 기본값:

```text
showItems: true
showCompletedItems: true
showLabels: true
showPorts: false
showConnections: true
showBottleneckOverlay: true
showRobotWorkingRadius: true
showWaitingItems: true
showBlockedHighlight: true
```

---

## 19. Animation Player 상태 관리

```ts
export interface AnimationPlayerState {
  layout: LayoutJson | null;
  result: SimulationResult | null;
  playback: PlaybackControllerState;
  currentFrame: TimelineFrame | null;
  selectedItemId: string | null;
  selectedComponentId: string | null;
  displayOptions: AnimationDisplayOptions;
}
```

주요 액션:

```ts
export interface AnimationPlayerActions {
  loadLayout(layout: LayoutJson): void;
  loadResult(result: SimulationResult): void;
  play(): void;
  pause(): void;
  stop(): void;
  seek(time: number): void;
  setSpeed(speed: number): void;
  selectItem(itemId: string | null): void;
  selectComponent(componentId: string | null): void;
  updateDisplayOptions(options: Partial<AnimationDisplayOptions>): void;
}
```

---

## 20. Three.js 렌더링 구조

추천 폴더 구조:

```text
src/three/animation/
  AnimationPlayer.tsx
  PlaybackToolbar.tsx
  TimeSlider.tsx
  TimelineController.ts
  TrackEvaluator.ts
  ItemAnimationLayer.tsx
  ComponentStateLayer.tsx
  RobotAnimationLayer.tsx
  PalletAnimationLayer.tsx
  BottleneckOverlay.tsx
  PlaybackStatsPanel.tsx
```

레이어 구조:

```text
StaticLayoutLayer
- 설비, 컨베이어, 연결선

ItemAnimationLayer
- 이동 중인 박스

RobotAnimationLayer
- 로봇 작업 표현

PalletAnimationLayer
- 팔레트 적재 박스

ComponentStateLayer
- 설비 상태 색상

OverlayLayer
- 병목, 라벨, 선택 강조
```

---

## 21. Rendering Loop

기본 흐름:

```text
1. requestAnimationFrame 실행
2. realDelta 계산
3. playbackState가 playing이면 currentTime 갱신
4. TimelineController.evaluate(currentTime)
5. currentFrame을 Three.js scene에 적용
6. render 호출
```

의사 코드:

```ts
function animationLoop(realTime: number) {
  const realDelta = realTime - lastRealTime;

  if (playbackState === "playing") {
    const simulationDelta = realDelta * speed;
    const nextTime = currentTime + simulationDelta;
    seek(nextTime);
  }

  renderScene();
  requestAnimationFrame(animationLoop);
}
```

---

## 22. MVP 구현 범위

### 포함

```text
- Simulation Result 불러오기
- Layout JSON 기준 Static Layout 렌더링
- Playback Controller
- Play, Pause, Stop, Seek
- Speed 1x, 5x, 10x, 30x, 60x
- Time Slider
- Animation Track 평가
- 박스 선형 이동
- 박스 waiting 표시
- Robot Pick & Place 단순 표현
- Pallet 적재 표시
- Component State 색상 표시
- Bottleneck Overlay
- 아이템 선택
- 설비 선택
```

### 제외

```text
- 실제 로봇 역기구학
- GLB 로봇 관절 애니메이션
- 물리 기반 박스 충돌
- InstancedMesh 최적화
- 대용량 Timeline pagination
- Web Worker 기반 평가
```

---

## 23. 작업 목록

```text
타입 정의 작업
- src/types/animation.ts 생성
- PlaybackState 타입 정의
- PlaybackControllerState 타입 정의
- TimelineFrame 타입 정의
- EvaluatedItemState 타입 정의
- EvaluatedComponentState 타입 정의
- EvaluatedRobotState 타입 정의
- EvaluatedPalletState 타입 정의
- AnimationDisplayOptions 타입 정의
- AnimationPlayerState 타입 정의

Playback 작업
- PlaybackController.ts 생성
- play 구현
- pause 구현
- stop 구현
- seek 구현
- setSpeed 구현
- currentTime 업데이트 구현
- endTime 도달 처리

Timeline 평가 작업
- TrackEvaluator.ts 생성
- keyframe 탐색 구현
- position lerp 구현
- rotation lerp 구현
- visible 처리 구현
- state 유지 처리 구현

Three.js Layer 작업
- StaticLayoutLayer.tsx 구현
- ItemAnimationLayer.tsx 구현
- ComponentStateLayer.tsx 구현
- RobotAnimationLayer.tsx 구현
- PalletAnimationLayer.tsx 구현
- BottleneckOverlay.tsx 구현
```

---

## 24. 완료 기준

```text
- Animation Player의 전체 구조가 정의되어 있다.
- Playback Controller 구조가 정의되어 있다.
- Time Slider와 Seek 동작이 정의되어 있다.
- Animation Track 평가 방식이 정의되어 있다.
- 박스 선형 이동 방식이 정의되어 있다.
- Waiting 상태 표시 방식이 정의되어 있다.
- Robot Pick & Place 단순 애니메이션 방식이 정의되어 있다.
- Pallet 적재 위치 계산 방식이 정의되어 있다.
- Component State 색상 표시 방식이 정의되어 있다.
- Bottleneck Overlay 표시 방식이 정의되어 있다.
- Three.js 렌더링 레이어 구조가 정의되어 있다.
- Codex가 Animation Player MVP를 구현할 수 있는 수준이다.
```

---

## 25. Codex 작업 지시 예시

```text
MasterPlan.md부터 Plan_09.md까지 읽고,
src/types/animation.ts 파일을 만들어줘.

Plan_09.md의 타입 정의를 기준으로 다음 타입을 구현해.
- PlaybackState
- PlaybackControllerState
- TimelineFrame
- EvaluatedItemState
- EvaluatedComponentState
- EvaluatedRobotState
- EvaluatedPalletState
- AnimationDisplayOptions
- AnimationPlayerState

아직 Three.js 화면은 만들지 마.
```

---

## 26. 다음 단계

Plan_09가 완료되면 다음 문서로 이동한다.

```text
Plan_10.md
```

Plan_10에서는 시뮬레이션 결과를 분석하고, 리포트와 향후 고도화 방향을 정의한다.
::: 
