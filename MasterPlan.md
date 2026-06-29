# MasterPlan.md

# 웹 기반 물류 라인 이산 사건 시뮬레이션 Master Plan

## 0. 문서 목적

이 문서는 복잡한 물류 자동화 라인을 웹에서 구성하고, 이산 사건 시뮬레이션으로 물동량 흐름, 대기, 병목, 로봇 작업, 처리량을 검토하기 위한 전체 개발 계획서이다.

목표는 단순한 3D 애니메이션이 아니라, 실제 설계 검토에 사용할 수 있는 **웹 기반 물류 셀 시뮬레이터**를 만드는 것이다.

핵심 방향은 다음과 같다.

```text
Three.js는 화면과 편집기를 담당한다.
DES Simulation Engine은 계산과 판단을 담당한다.
Layout JSON은 둘 사이의 공통 언어가 된다.
```

즉, Three.js 오브젝트를 직접 읽어서 시뮬레이션을 추론하는 방식이 아니라, 사용자가 Three.js 화면에서 설비와 경로를 배치하면 그 결과가 Layout JSON으로 저장되고, 이산 사건 시뮬레이션 엔진은 이 JSON을 기준으로 계산한다.

---

## 1. 프로젝트 개요

### 1.1 프로젝트 이름

```text
Logistics DES Web Simulator
```

### 1.2 프로젝트 목표

컨베이어, 분기기, 합류부, 버퍼, 로봇, 파렛타이징 스테이션 등으로 구성된 물류 자동화 라인을 웹에서 배치하고, 시간대별 물동량과 공정 조건을 입력하여 이산 사건 시뮬레이션을 수행한다.

시뮬레이션 결과는 Three.js 기반 3D 또는 2.5D 화면에서 재생하고, 처리량, 대기시간, 설비 가동률, 병목 구간을 리포트로 제공한다.

### 1.3 사용 목적

이 시스템은 다음 목적에 활용한다.

```text
1. 고객 설명용
- 컨베이어와 로봇의 흐름을 직관적으로 설명
- 정적인 CAD 도면을 움직이는 시뮬레이션으로 변환
- 비전문가도 물류 흐름을 이해할 수 있도록 시각화

2. 설계 검토용
- 로봇 대수 검토
- 컨베이어 속도 검토
- 버퍼 용량 검토
- 분기 및 합류 병목 확인
- 팔레타이징 처리량 검토

3. 운영 분석용
- 시간대별 물동량 변화 반영
- 제품 Mix 반영
- 설비 정지 또는 고장 시나리오 반영
- 일 생산량 또는 시간당 처리량 예측
```

### 1.4 1차 목표

1차 개발 목표는 **설계 검토용 MVP**이다.

MVP는 다음 기능을 포함한다.

```text
- Source에서 박스 생성
- 컨베이어 이동
- 분기기에서 목적지별 경로 선택
- 합류부에서 FIFO 또는 교대 방식 처리
- 버퍼에서 대기
- 로봇 픽업 및 파렛타이징
- 처리량 계산
- 대기시간 계산
- 로봇 가동률 계산
- 병목 구간 표시
- Three.js 화면에서 결과 재생
```

---

## 2. 전체 시스템 구조

### 2.1 핵심 구조

```text
[도면 이미지 또는 수동 배치]
        ↓
[Three.js Layout Editor]
        ↓
[Layout JSON]
        ↓
[DES Simulation Engine]
        ↓
[Event Log / State Timeline]
        ↓
[Three.js Animation Player]
        ↓
[Analysis Report]
```

### 2.2 역할 분리

| 영역                        | 역할                                   |
| ------------------------- | ------------------------------------ |
| Three.js Layout Editor    | 설비 배치, 방향 설정, 길이 조정, 포트 연결, 도면 배경 표시 |
| Layout JSON               | 설비, 연결관계, 경로, 속성, 시나리오를 저장하는 공통 데이터  |
| DES Simulation Engine     | 이산 사건 기반으로 이동, 대기, 분기, 합류, 로봇 작업 계산  |
| Event Log                 | 시뮬레이션 결과를 시간순 사건으로 저장                |
| Three.js Animation Player | Event Log를 기반으로 박스 이동과 로봇 작업을 재생     |
| Analysis Report           | 처리량, 대기시간, 가동률, 병목 구간을 출력            |

### 2.3 가장 중요한 설계 원칙

```text
Three.js는 계산하지 않는다.
Three.js는 배치하고 보여준다.

DES Engine은 그리지 않는다.
DES Engine은 사건과 상태를 계산한다.

Layout JSON은 둘의 공통 언어다.
```

피해야 할 구조는 다음과 같다.

```text
Three.js 오브젝트의 위치와 모양을 직접 읽어서
시뮬레이션 엔진이 컨베이어 방향과 흐름을 추론하는 방식
```

이 방식은 도면이 복잡해질수록 오류가 커진다.
컨베이어, 리프트, 분기기, 합류부, 로봇이 섞이면 3D 형상만 보고 논리 흐름을 정확히 추론하기 어렵다.

권장 구조는 다음과 같다.

```text
사용자가 Three.js 화면에서 배치한다.
하지만 저장되는 것은 3D 오브젝트가 아니라 Layout JSON이다.

사용자가 경로를 연결한다.
하지만 저장되는 것은 선 그림이 아니라 from-to 포트 연결관계다.

사용자가 물동량을 입력한다.
하지만 저장되는 것은 단순 수량이 아니라 시간별 Arrival Schedule이다.
```

---

## 3. 기술 스택

### 3.1 프론트엔드

```text
Vite
React
TypeScript
Three.js
Zustand 또는 Redux Toolkit
```

### 3.2 시뮬레이션 엔진

```text
TypeScript 기반 Custom DES Engine
Event Queue
Component State Manager
Routing Engine
Statistics Collector
```

### 3.3 3D 모델

```text
기본 모델: Three.js Geometry 기반 단순 모델
고급 모델: Blender에서 만든 GLB 또는 glTF 모델
```

### 3.4 데이터 저장

1차 MVP에서는 파일 기반 JSON 저장을 사용한다.

```text
layout.json
scenario.json
simulation_result.json
```

고도화 단계에서는 다음 저장 방식을 검토한다.

```text
LocalStorage
IndexedDB
SQLite WASM
Server DB
```

### 3.5 배포

```text
Vercel
Netlify
GitHub Pages
사내 서버
```

---

## 4. 폴더 구조 제안

```text
logistics-des-simulator/
  README.md
  package.json
  vite.config.ts
  tsconfig.json

  docs/
    MasterPlan.md

    plans/
      Plan_01_ProjectScope.md
      Plan_02_ComponentModel.md
      Plan_03_LayoutJsonSchema.md
      Plan_04_ThreeJsLayoutEditor.md
      Plan_05_NetworkRouting.md
      Plan_06_InputScenarioData.md
      Plan_07_DesSimulationEngine.md
      Plan_08_EventLogTimeline.md
      Plan_09_ThreeJsAnimationPlayer.md
      Plan_10_ReportAndRoadmap.md

  public/
    drawings/
    models/
    sample-data/

  src/
    main.tsx
    App.tsx

    editor/
      components/
      panels/
      hooks/
      layoutEditorStore.ts

    three/
      scene/
      objects/
      controls/
      materials/
      animation/

    simulation/
      engine/
      events/
      components/
      routing/
      statistics/

    data/
      schemas/
      samples/
      importers/
      exporters/

    types/
      layout.ts
      scenario.ts
      simulation.ts
      component.ts

    report/
      charts/
      tables/
      exporters/
```

---

## 5. 데이터 흐름

### 5.1 기본 흐름

```text
1. 사용자가 설비 컴포넌트를 배치한다.
2. 사용자가 설비 속성을 입력한다.
3. 사용자가 포트 간 연결을 만든다.
4. 시스템은 Layout JSON을 생성한다.
5. 사용자가 물동량과 공정 조건을 입력한다.
6. 시스템은 Scenario JSON을 생성한다.
7. DES Engine은 Layout JSON과 Scenario JSON을 읽는다.
8. Event Queue를 기반으로 시뮬레이션을 실행한다.
9. Event Log와 State Timeline을 생성한다.
10. Three.js Animation Player가 결과를 재생한다.
11. Report 모듈이 KPI를 계산하고 표시한다.
```

### 5.2 데이터 파일 관계

```text
layout.json
- 설비 배치
- 설비 속성
- 포트
- 연결관계
- 기본 라우팅

scenario.json
- 제품 종류
- 시간대별 투입량
- 목적지
- 공정시간
- 운영 규칙

simulation_result.json
- 이벤트 로그
- 설비 상태 이력
- 박스별 이동 이력
- KPI 결과
```

---

## 6. 핵심 데이터 모델

### 6.1 Layout JSON 예시

```json
{
  "project": {
    "id": "sample_project_001",
    "name": "Palletizing Line Simulation",
    "unit": "meter"
  },
  "components": [
    {
      "id": "SRC_001",
      "type": "source",
      "name": "Box Infeed Source",
      "position": { "x": 0, "y": 0, "z": 0 },
      "rotation": { "x": 0, "y": 0, "z": 0 },
      "ports": {
        "in": [],
        "out": ["SRC_001_OUT"]
      }
    },
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
      "properties": {
        "speed": 0.6,
        "capacity": 10,
        "accumulationMode": "blocking"
      },
      "ports": {
        "in": ["CV_001_IN"],
        "out": ["CV_001_OUT"]
      }
    },
    {
      "id": "RB_001",
      "type": "robot",
      "name": "Palletizing Robot",
      "position": { "x": 9, "y": 0, "z": 2 },
      "rotation": { "x": 0, "y": 0, "z": 0 },
      "properties": {
        "cycleTime": 8,
        "workingRadius": 2.8,
        "pickPort": "PICK_001_IN",
        "placeTarget": "PALLET_001"
      },
      "ports": {
        "in": ["RB_001_IN"],
        "out": ["RB_001_OUT"]
      }
    }
  ],
  "connections": [
    {
      "id": "CONN_001",
      "from": "SRC_001_OUT",
      "to": "CV_001_IN",
      "type": "transport"
    },
    {
      "id": "CONN_002",
      "from": "CV_001_OUT",
      "to": "RB_001_IN",
      "type": "transport"
    }
  ]
}
```

### 6.2 Scenario JSON 예시

```json
{
  "simulation": {
    "startTime": 0,
    "endTime": 3600,
    "timeUnit": "second"
  },
  "items": [
    {
      "id": "BOX_A",
      "name": "Box A",
      "dimensions": {
        "length": 0.4,
        "width": 0.3,
        "height": 0.25
      },
      "destination": "PALLET_001"
    }
  ],
  "arrivalSchedule": [
    {
      "id": "ARR_001",
      "source": "SRC_001",
      "itemType": "BOX_A",
      "startTime": 0,
      "endTime": 3600,
      "quantity": 600,
      "arrivalPattern": "uniform"
    }
  ],
  "rules": {
    "mergeRule": "fifo",
    "diverterRule": "byDestination",
    "blockingRule": "downstreamBlocking"
  }
}
```

### 6.3 Simulation Result 예시

```json
{
  "summary": {
    "simulationTime": 3600,
    "createdItems": 600,
    "completedItems": 520,
    "throughputPerHour": 520,
    "averageLeadTime": 310,
    "averageWaitingTime": 95
  },
  "componentStats": [
    {
      "componentId": "RB_001",
      "busyTime": 3400,
      "idleTime": 200,
      "utilization": 0.944
    }
  ],
  "eventLog": [
    {
      "time": 0,
      "eventType": "ITEM_CREATED",
      "itemId": "BOX_A_001",
      "componentId": "SRC_001"
    },
    {
      "time": 10,
      "eventType": "ITEM_ENTER_COMPONENT",
      "itemId": "BOX_A_001",
      "componentId": "CV_001"
    },
    {
      "time": 20,
      "eventType": "ROBOT_PICK_START",
      "itemId": "BOX_A_001",
      "componentId": "RB_001"
    },
    {
      "time": 28,
      "eventType": "PALLETIZE_COMPLETE",
      "itemId": "BOX_A_001",
      "componentId": "PALLET_001"
    }
  ]
}
```

---

## 7. 표준 컴포넌트 정의

### 7.1 Source

박스, 트레이, 팔레트 등 이동 대상이 생성되는 시작점이다.

주요 속성:

```text
- id
- position
- output port
- 생성 대상 item type
- 생성 스케줄
```

### 7.2 Conveyor

박스가 일정 속도로 이동하는 설비이다.

주요 속성:

```text
- length
- width
- height
- speed
- capacity
- accumulation mode
- in port
- out port
```

기본 규칙:

```text
앞 공간이 있으면 이동한다.
출구 설비가 막혀 있으면 컨베이어 끝에서 대기한다.
대기한 박스 뒤쪽 박스도 순차적으로 멈춘다.
```

### 7.3 Diverter

목적지 또는 규칙에 따라 박스를 여러 출구 중 하나로 보내는 설비이다.

주요 속성:

```text
- input port
- output ports
- rule
- switching time
```

기본 규칙:

```text
목적지가 정해져 있으면 목적지 기준으로 출구를 선택한다.
선택한 출구가 막혀 있으면 대기한다.
대체 경로가 허용된 경우에만 다른 출구를 선택한다.
```

### 7.4 Merge

여러 입력 라인이 하나의 출력 라인으로 합쳐지는 설비이다.

주요 속성:

```text
- input ports
- output port
- merge rule
- transfer time
```

기본 규칙:

```text
FIFO
교대 배출
메인 라인 우선
우선순위 기반 배출
```

### 7.5 Buffer

박스가 임시로 대기하는 공간이다.

주요 속성:

```text
- capacity
- queue discipline
- input port
- output port
```

기본 규칙:

```text
공간이 있으면 진입한다.
가득 차면 상류 설비를 막는다.
기본 배출 규칙은 FIFO이다.
```

### 7.6 Process Station

검사, 라벨링, 포장, 밴딩처럼 처리 시간이 필요한 설비이다.

주요 속성:

```text
- process time
- capacity
- resource requirement
- failure option
```

기본 규칙:

```text
아이템이 도착하면 설비가 비어 있는지 확인한다.
비어 있으면 처리 시작한다.
처리 중에는 설비를 점유한다.
처리가 끝나면 다음 설비로 이동한다.
```

### 7.7 Robot

픽업, 적재, 디팔레타이징, 팔레타이징을 수행하는 설비이다.

주요 속성:

```text
- working radius
- cycle time
- pick port
- place target
- gripper type
- payload
- palletizing pattern
```

기본 규칙:

```text
픽업 포인트에 박스가 도착한다.
로봇이 available이면 작업을 시작한다.
cycle time 동안 로봇은 busy 상태가 된다.
작업 완료 후 박스는 팔레트 또는 다음 위치로 이동한다.
```

### 7.8 Pallet Station

로봇이 박스를 적재하는 위치이다.

주요 속성:

```text
- pallet size
- box pattern
- layer count
- max boxes
- current count
```

기본 규칙:

```text
박스가 적재되면 현재 적재 수량을 증가시킨다.
한 층이 완료되면 layer count를 갱신한다.
팔레트가 완료되면 완료 이벤트를 발생시킨다.
```

### 7.9 Sink

완료, 출고, 리젝트, 폐기 지점이다.

주요 속성:

```text
- input port
- completed count
- category
```

기본 규칙:

```text
아이템이 Sink에 도착하면 해당 아이템의 이동은 완료된다.
완료 수량과 리드타임을 기록한다.
```

---

## 8. 이산 사건 시뮬레이션 엔진 설계

### 8.1 DES 개념

이산 사건 시뮬레이션은 시간을 일정 간격으로 계속 계산하지 않고, 중요한 사건이 발생하는 시점으로 시간을 점프시키는 방식이다.

예시:

```text
0초: 박스 생성
10초: 컨베이어 출구 도착
12초: 분기기 통과
20초: 로봇 픽업 위치 도착
20초: 로봇 작업 시작
28초: 팔레트 적재 완료
```

### 8.2 엔진 구성요소

```text
SimulationClock
- 현재 시뮬레이션 시간

EventQueue
- 앞으로 발생할 사건을 시간순으로 저장

ItemManager
- 박스, 트레이, 팔레트의 상태 관리

ComponentManager
- 설비별 상태 관리

Router
- 목적지와 연결 그래프를 기준으로 다음 설비 결정

ResourceManager
- 로봇, 작업자, 검사기처럼 점유되는 자원 관리

StatisticsCollector
- 처리량, 대기시간, 가동률, WIP 계산
```

### 8.3 주요 이벤트 타입

```text
ITEM_CREATED
ITEM_ENTER_COMPONENT
ITEM_EXIT_COMPONENT
ITEM_WAIT_START
ITEM_WAIT_END
PROCESS_START
PROCESS_END
ROBOT_PICK_START
ROBOT_PICK_END
PALLETIZE_COMPLETE
ITEM_COMPLETED
BLOCKING_START
BLOCKING_END
```

### 8.4 컨베이어 기본 로직

```text
1. 아이템이 컨베이어에 진입한다.
2. 컨베이어 길이와 속도 기준으로 출구 도착 시간을 계산한다.
3. 출구 도착 이벤트를 예약한다.
4. 출구 도착 시 다음 설비가 진입 가능한지 확인한다.
5. 가능하면 다음 설비로 이동한다.
6. 불가능하면 대기 상태가 된다.
7. 다음 설비에 공간이 생기면 대기 해제 이벤트가 발생한다.
```

### 8.5 분기 기본 로직

```text
1. 아이템이 분기기에 도착한다.
2. 아이템 목적지를 확인한다.
3. 목적지에 맞는 output port를 선택한다.
4. 선택한 하류 설비가 진입 가능한지 확인한다.
5. 가능하면 통과한다.
6. 불가능하면 분기기 또는 상류에서 대기한다.
```

### 8.6 합류 기본 로직

```text
1. 여러 입력 라인에서 아이템이 합류부에 도착한다.
2. 합류 규칙에 따라 배출 우선순위를 결정한다.
3. 출력 라인이 비어 있으면 하나의 아이템만 통과시킨다.
4. 나머지 아이템은 대기한다.
```

### 8.7 로봇 기본 로직

```text
1. 픽업 포인트에 아이템이 도착한다.
2. 로봇이 available인지 확인한다.
3. 로봇이 비어 있으면 ROBOT_PICK_START 이벤트를 발생시킨다.
4. 로봇은 cycle time 동안 busy 상태가 된다.
5. cycle time 이후 ROBOT_PICK_END 이벤트가 발생한다.
6. 팔레트 스테이션에 적재 완료 이벤트를 기록한다.
7. 로봇은 다시 available 상태가 된다.
```

---

## 9. Three.js Layout Editor 설계

### 9.1 기본 목표

Three.js Layout Editor는 사용자가 설비를 웹 화면에서 배치하고, 속성을 수정하고, 포트 간 연결을 설정하는 도구이다.

처음에는 완전한 3D CAD 방식보다 **Top View 기반 2.5D 편집기**로 시작한다.

### 9.2 화면 구성

```text
좌측 패널
- 컴포넌트 라이브러리
- Source
- Conveyor
- Diverter
- Merge
- Buffer
- Robot
- Pallet Station
- Sink

중앙 화면
- Three.js Canvas
- 도면 배경
- 설비 배치
- 연결선
- 작업 반경
- 흐름 방향

우측 패널
- 선택 설비 속성
- 길이
- 폭
- 속도
- 용량
- 방향
- 처리시간
- 분기 규칙
- 로봇 사이클타임

하단 패널
- 시뮬레이션 실행
- 재생 속도
- 시간 슬라이더
- 결과 요약
```

### 9.3 필수 편집 기능

```text
- 컴포넌트 추가
- 컴포넌트 선택
- 이동
- 회전
- 컨베이어 길이 수정
- 속성 수정
- 포트 연결
- 연결 삭제
- Layout JSON 저장
- Layout JSON 불러오기
- 도면 이미지 배경으로 불러오기
- 도면 스케일 보정
```

### 9.4 도면 배경 활용

복잡한 CAD 이미지를 바로 자동 변환하기보다, 1차 개발에서는 도면 이미지를 바닥 배경으로 깔고 그 위에 컴포넌트를 수동 배치한다.

```text
도면 이미지를 업로드한다.
도면 배경을 캔버스에 표시한다.
기준 거리 2점을 찍어 스케일을 보정한다.
사용자가 도면 위에 컨베이어와 로봇을 배치한다.
배치 결과는 Layout JSON으로 저장한다.
```

---

## 10. Network Routing 설계

### 10.1 개념

Network Routing은 설비 간 흐름을 그래프 구조로 관리하는 기능이다.

```text
Node = 설비의 포트
Edge = 포트 간 연결
Route = 아이템이 이동할 경로
```

### 10.2 포트 기반 연결

각 컴포넌트는 입구와 출구 포트를 가진다.

예시:

```text
CV_001_IN
CV_001_OUT
DIV_001_IN
DIV_001_OUT_A
DIV_001_OUT_B
```

연결은 다음처럼 표현한다.

```json
{
  "from": "CV_001_OUT",
  "to": "DIV_001_IN"
}
```

### 10.3 라우팅 방식

1차 MVP에서는 명시적 경로를 사용한다.

```text
BOX_A 경로:
SRC_001 → CV_001 → DIV_001 → CV_002 → RB_001 → PALLET_001
```

2차 고도화에서는 목적지만 입력하면 그래프에서 자동으로 경로를 찾도록 한다.

```text
BOX_A destination = PALLET_001
Router가 연결 그래프에서 가능한 경로 탐색
```

### 10.4 분기 규칙

```text
byDestination
- 아이템 목적지 기준으로 출구 선택

shortestQueue
- 가장 대기열이 짧은 출구 선택

priority
- 우선순위가 높은 출구 선택

roundRobin
- 출구를 순서대로 교대 선택
```

### 10.5 합류 규칙

```text
fifo
- 먼저 도착한 아이템 우선

roundRobin
- 입력 라인별 교대 통과

mainLinePriority
- 메인 라인 우선 통과

priority
- 아이템 우선순위 기준 통과
```

---

## 11. 입력 시나리오 설계

### 11.1 입력 데이터 종류

```text
Item Master
- 제품 또는 박스 종류
- 크기
- 중량
- 목적지
- 팔레타이징 조건

Arrival Schedule
- 시간대별 투입량
- 투입 Source
- 투입 간격
- 투입 패턴

Process Time
- 로봇 사이클타임
- 검사 시간
- 포장 시간
- 라벨링 시간
- 리프트 이동 시간

Rule Set
- 분기 규칙
- 합류 규칙
- 버퍼 규칙
- 우선순위 규칙
```

### 11.2 투입 패턴

```text
uniform
- 일정 간격으로 투입

batch
- 특정 시간에 묶음 투입

poisson
- 확률적 도착

csv
- CSV 파일에 정의된 시간표 기반 투입
```

### 11.3 예시

```json
{
  "arrivalSchedule": [
    {
      "source": "SRC_001",
      "itemType": "BOX_A",
      "startTime": 0,
      "endTime": 3600,
      "quantity": 600,
      "arrivalPattern": "uniform"
    },
    {
      "source": "SRC_002",
      "itemType": "BOX_B",
      "startTime": 1800,
      "endTime": 3600,
      "quantity": 300,
      "arrivalPattern": "uniform"
    }
  ]
}
```

---

## 12. Event Log 및 Timeline 설계

### 12.1 목적

Event Log는 이산 사건 시뮬레이션 결과를 시간순으로 저장하는 데이터이다.
Three.js Animation Player는 이 로그를 읽어서 화면을 재생한다.

### 12.2 아이템별 Timeline

```json
{
  "itemId": "BOX_A_001",
  "timeline": [
    {
      "time": 0,
      "event": "ITEM_CREATED",
      "component": "SRC_001"
    },
    {
      "time": 5,
      "event": "ITEM_ENTER_COMPONENT",
      "component": "CV_001"
    },
    {
      "time": 15,
      "event": "ITEM_EXIT_COMPONENT",
      "component": "CV_001"
    },
    {
      "time": 20,
      "event": "ROBOT_PICK_START",
      "component": "RB_001"
    },
    {
      "time": 28,
      "event": "PALLETIZE_COMPLETE",
      "component": "PALLET_001"
    }
  ]
}
```

### 12.3 설비별 State Timeline

```json
{
  "componentId": "RB_001",
  "states": [
    {
      "start": 0,
      "end": 20,
      "state": "idle"
    },
    {
      "start": 20,
      "end": 28,
      "state": "busy"
    },
    {
      "start": 28,
      "end": 35,
      "state": "idle"
    }
  ]
}
```

### 12.4 Timeline 활용

```text
- 애니메이션 재생
- 시간 슬라이더 이동
- 특정 시간 상태 조회
- 설비별 가동률 계산
- 병목 구간 분석
- 대기시간 분석
```

---

## 13. Three.js Animation Player 설계

### 13.1 기본 목표

시뮬레이션 엔진이 만든 Event Log와 State Timeline을 기반으로 화면을 재생한다.

### 13.2 주요 기능

```text
- 박스 이동 애니메이션
- 컨베이어 위 대기 표시
- 분기 방향 시각화
- 합류 대기 표시
- 로봇 Pick & Place 애니메이션
- 팔레트 적재 표시
- 설비 상태 색상 표시
- 병목 구간 강조
- 배속 조절
- 일시정지
- 특정 시간 이동
```

### 13.3 재생 속도

```text
1x
5x
10x
30x
60x
```

시뮬레이션 시간과 실제 재생 시간은 분리한다.

```text
시뮬레이션 1시간 결과를 화면에서는 1분으로 빠르게 재생할 수 있어야 한다.
```

### 13.4 상태 색상 규칙

```text
정상 흐름: 녹색
대기 발생: 노란색
병목 또는 정체: 빨간색
idle: 회색
busy: 파란색
blocked: 주황색
```

---

## 14. 분석 리포트 설계

### 14.1 기본 KPI

```text
총 투입 수량
총 완료 수량
미처리 WIP
시간당 처리량
평균 리드타임
평균 대기시간
설비별 가동률
컨베이어 정체 시간
버퍼 점유율
로봇 가동률
팔레트 완성 시간
병목 설비 순위
```

### 14.2 리포트 예시

```text
시뮬레이션 시간: 8시간
투입 수량: 4,800 box
완료 수량: 4,120 box
미처리 WIP: 680 box

시간당 처리량: 515 box/hour
평균 리드타임: 11.4분
평균 대기시간: 4.2분

병목 설비:
1위 RB_001 로봇, 가동률 96%
2위 MERGE_003 합류부, 누적 대기시간 2.8시간
3위 CV_017, downstream blocking 1.9시간
```

### 14.3 개선 제안 기능

고도화 단계에서는 결과를 기반으로 간단한 개선 제안을 표시한다.

```text
- 로봇 가동률이 95% 이상이면 로봇 추가 또는 사이클타임 단축 검토
- 특정 Merge에서 대기시간이 높으면 합류 규칙 변경 검토
- Buffer 점유율이 90% 이상이면 버퍼 용량 증설 검토
- Conveyor blocking 시간이 높으면 하류 설비 병목 확인
```

---

## 15. 개발 단계 요약

### Phase 1. 단순 라인 MVP

목표:

```text
Source 1개
Conveyor 2개
Diverter 1개
Robot 1개
Pallet Station 1개
Sink 1개
```

구현 기능:

```text
박스 생성
컨베이어 이동
분기
로봇 픽업
팔레트 적재
기본 처리량 계산
```

완료 기준:

```text
1시간 동안 600개 투입 시 시뮬레이션 결과가 생성된다.
Three.js 화면에서 박스 이동과 로봇 적재가 재생된다.
완료 수량과 로봇 가동률이 출력된다.
```

### Phase 2. Layout Editor

목표:

```text
웹 화면에서 설비를 추가, 이동, 회전, 길이 수정할 수 있다.
```

구현 기능:

```text
컴포넌트 라이브러리
속성 패널
포트 연결
JSON 저장 및 불러오기
```

완료 기준:

```text
사용자가 만든 레이아웃이 layout.json으로 저장된다.
저장한 layout.json을 다시 불러와 동일한 화면을 복원한다.
```

### Phase 3. DES Engine

목표:

```text
Layout JSON과 Scenario JSON을 읽어 이벤트 기반 시뮬레이션을 실행한다.
```

구현 기능:

```text
Event Queue
Item State
Component State
Conveyor 이동
Buffer 대기
Diverter 분기
Merge 합류
Robot 점유 및 해제
```

완료 기준:

```text
시뮬레이션 결과로 eventLog와 componentStats가 생성된다.
병목이 발생하는 구간을 수치로 확인할 수 있다.
```

### Phase 4. Animation Player

목표:

```text
Event Log를 기반으로 Three.js 화면에서 결과를 재생한다.
```

구현 기능:

```text
배속 조절
일시정지
시간 슬라이더
박스 위치 보간
설비 상태 색상 표시
```

완료 기준:

```text
시뮬레이션 결과를 1x, 10x, 60x 속도로 재생할 수 있다.
시간 슬라이더를 이동하면 해당 시간의 설비 상태가 표시된다.
```

### Phase 5. Report

목표:

```text
설계 검토에 필요한 KPI를 리포트로 출력한다.
```

구현 기능:

```text
처리량
대기시간
리드타임
가동률
병목 순위
팔레트 완성 시간
```

완료 기준:

```text
시뮬레이션 종료 후 Summary Report가 화면에 표시된다.
결과를 JSON 또는 CSV로 export할 수 있다.
```

### Phase 6. Drawing Background

목표:

```text
도면 이미지를 배경으로 깔고 그 위에 설비 컴포넌트를 배치한다.
```

구현 기능:

```text
이미지 업로드
스케일 보정
도면 투명도 조절
도면 잠금
컴포넌트 스냅
```

완료 기준:

```text
첨부된 CAD 이미지 위에 컨베이어와 로봇을 배치할 수 있다.
스케일 보정 후 컨베이어 길이가 실제 단위와 맞게 표시된다.
```

---

## 16. Plan 문서 구성

이 MasterPlan을 기준으로 다음 10개 실행 문서를 작성한다.

```text
docs/plans/Plan_01_ProjectScope.md
docs/plans/Plan_02_ComponentModel.md
docs/plans/Plan_03_LayoutJsonSchema.md
docs/plans/Plan_04_ThreeJsLayoutEditor.md
docs/plans/Plan_05_NetworkRouting.md
docs/plans/Plan_06_InputScenarioData.md
docs/plans/Plan_07_DesSimulationEngine.md
docs/plans/Plan_08_EventLogTimeline.md
docs/plans/Plan_09_ThreeJsAnimationPlayer.md
docs/plans/Plan_10_ReportAndRoadmap.md
```

각 Plan 문서는 다음 형식을 사용한다.

```text
# Plan_XX. 제목

## 1. 목적
이 단계가 왜 필요한지 설명한다.

## 2. 구현 범위
이번 단계에서 만들 기능을 정의한다.

## 3. 입력 데이터
필요한 데이터, JSON, 설정값을 정의한다.

## 4. 출력 결과
이 단계가 끝났을 때 생성되는 산출물을 정의한다.

## 5. 핵심 설계
구현 구조, 클래스, 컴포넌트, 데이터 흐름을 설명한다.

## 6. 작업 목록
개발 작업을 체크리스트로 정리한다.

## 7. 완료 기준
테스트 가능한 완료 조건을 정의한다.

## 8. 주의사항
나중에 문제가 될 수 있는 설계 리스크를 정리한다.
```

---

## 17. Plan 문서별 역할

### Plan_01_ProjectScope.md

```text
시뮬레이션 목적
MVP 범위
고객 설명용, 설계 검토용, 운영 분석용 구분
1차 개발 목표
성공 기준
```

### Plan_02_ComponentModel.md

```text
Source
Conveyor
Diverter
Merge
Buffer
Process Station
Robot
Pallet Station
Sink
포트 구조
공통 속성
컴포넌트별 상태 모델
```

### Plan_03_LayoutJsonSchema.md

```text
Layout JSON 구조
components
connections
ports
routes
rules
validation
sample layout
```

### Plan_04_ThreeJsLayoutEditor.md

```text
Three.js 화면 구성
컴포넌트 배치
이동
회전
길이 조정
속성 패널
도면 배경
저장 및 불러오기
```

### Plan_05_NetworkRouting.md

```text
포트 기반 연결 그래프
Route 정의
목적지 기반 경로
분기 규칙
합류 규칙
자동 경로 탐색
```

### Plan_06_InputScenarioData.md

```text
Item Master
Arrival Schedule
Process Time
Rule Set
CSV 입력
JSON 입력
샘플 시나리오
```

### Plan_07_DesSimulationEngine.md

```text
Event Queue
Simulation Clock
Item State
Component State
Conveyor Logic
Diverter Logic
Merge Logic
Robot Logic
Statistics Collector
```

### Plan_08_EventLogTimeline.md

```text
Event Log 구조
Item Timeline
Component State Timeline
KPI 계산용 로그
시간 슬라이더 데이터 구조
```

### Plan_09_ThreeJsAnimationPlayer.md

```text
Event Log 기반 재생
박스 이동 보간
로봇 Pick and Place
병목 색상 표시
배속 조절
일시정지
시간 이동
```

### Plan_10_ReportAndRoadmap.md

```text
처리량 리포트
대기시간 리포트
가동률 리포트
병목 분석
대안 레이아웃 비교
DXF 반자동 인식
고급 3D 모델
향후 확장 계획
```

---

## 18. 개발 우선순위

### 우선순위 1

```text
Layout JSON Schema
Component Model
DES Engine Basic
Simple Three.js Viewer
```

이 네 가지가 먼저다.
보기 좋은 화면보다 계산 가능한 구조가 우선이다.

### 우선순위 2

```text
Layout Editor
Network Routing
Event Log
Animation Player
```

사용자가 배치하고, 연결하고, 결과를 재생할 수 있어야 한다.

### 우선순위 3

```text
Report
Drawing Background
CSV Import
Case Compare
```

설계 검토 도구로 쓰기 위한 기능이다.

### 우선순위 4

```text
DXF Import
Advanced 3D Model
Physics Engine
Optimization
AI Layout Recommendation
```

이것들은 고도화 단계에서 검토한다.

---

## 19. MVP 구현 범위

### 포함

```text
- Source
- Conveyor
- Diverter
- Merge
- Buffer
- Robot
- Pallet Station
- Sink
- Layout JSON
- Scenario JSON
- Event Queue
- Basic Routing
- Event Log
- Three.js 2.5D 재생
- Summary Report
```

### 제외

```text
- 실제 CAD 자동 인식
- 정밀 물리 충돌
- 실제 로봇 역기구학
- 고급 3D 모델링
- 실시간 PLC 연동
- 자동 최적 배치 추천
```

MVP에서는 정밀한 물리 엔진보다 논리적 흐름과 병목 계산이 중요하다.

---

## 20. 주요 리스크와 대응

### 20.1 Three.js와 시뮬레이션 로직이 섞이는 문제

위험:

```text
화면 오브젝트가 곧 시뮬레이션 데이터가 되면 유지보수가 어려워진다.
```

대응:

```text
Layout JSON을 단일 기준 데이터로 사용한다.
Three.js는 Layout JSON을 렌더링한다.
DES Engine도 Layout JSON을 읽는다.
```

### 20.2 도면 자동 인식 난이도

위험:

```text
CAD 이미지에서 컨베이어와 설비를 완전 자동 인식하기 어렵다.
```

대응:

```text
1차는 도면 배경 위 수동 배치로 구현한다.
2차에서 DXF Layer 기반 반자동 인식을 검토한다.
```

### 20.3 컨베이어 누적 로직 복잡도

위험:

```text
박스 간 간격, 누적, blocking, release 조건이 복잡해질 수 있다.
```

대응:

```text
MVP에서는 capacity 기반 단순 모델을 사용한다.
고도화 단계에서 zone conveyor 모델을 추가한다.
```

### 20.4 로봇 동작 정밀도

위험:

```text
실제 6축 로봇의 역기구학과 충돌까지 구현하면 개발 범위가 커진다.
```

대응:

```text
MVP에서는 cycle time 기반 resource model로 처리한다.
Three.js에서는 단순 Pick and Place 애니메이션으로 표현한다.
```

### 20.5 성능 문제

위험:

```text
아이템 수가 많고 이벤트가 많으면 웹 브라우저에서 느려질 수 있다.
```

대응:

```text
시뮬레이션 계산과 애니메이션 렌더링을 분리한다.
필요하면 Web Worker에서 DES Engine을 실행한다.
화면에는 모든 박스를 그리지 않고 샘플링 또는 집계 표현을 사용할 수 있다.
```

---

## 21. 테스트 전략

### 21.1 단위 테스트

```text
Conveyor 이동 시간 계산
Diverter 출구 선택
Merge 우선순위 선택
Robot resource 점유 및 해제
Buffer capacity 계산
Route 탐색
```

### 21.2 통합 테스트

```text
Source → Conveyor → Sink
Source → Conveyor → Robot → Pallet
Source → Diverter → 2개 목적지
2개 Source → Merge → Sink
Buffer full 상황
Robot busy 상황
```

### 21.3 시각화 테스트

```text
Layout JSON을 Three.js 화면에 정확히 표시하는지 확인
Event Log 기반 박스 이동이 시간과 맞는지 확인
설비 상태 색상이 State Timeline과 일치하는지 확인
시간 슬라이더 이동 시 상태가 복원되는지 확인
```

### 21.4 결과 검증

```text
투입 수량과 완료 수량 비교
로봇 cycle time 기반 최대 처리량 비교
컨베이어 travel time 계산값 검증
대기시간 합계 검증
가동률 계산 검증
```

---

## 22. 향후 확장 방향

### 22.1 DXF 기반 반자동 모델링

```text
DXF Layer를 읽어 컨베이어 후보 선분을 추출한다.
사용자가 후보를 확인하고 컴포넌트로 변환한다.
설비명 Text를 인식하여 id와 name에 반영한다.
```

### 22.2 고급 컨베이어 모델

```text
Zone Conveyor
Accumulation Conveyor
Chain Transfer
Turntable
Lift
Shuttle
Sorter
```

### 22.3 로봇 고도화

```text
로봇 작업반경 체크
Pick and Place 위치 간 거리 기반 cycle time 계산
팔레타이징 패턴 자동 생성
다중 로봇 작업 분배
```

### 22.4 물류 최적화

```text
로봇 대수 추천
버퍼 용량 추천
컨베이어 속도 추천
분기 규칙 비교
대안 레이아웃 비교
```

### 22.5 AI 기능

```text
도면 기반 컴포넌트 후보 제안
병목 원인 설명
개선안 자동 작성
시뮬레이션 결과 리포트 자동 생성
고객 발표용 설명문 생성
```

---

## 23. Codex 작업 지시 방식

Codex 또는 AI 개발 도구에 작업을 줄 때는 전체 프로젝트를 한 번에 맡기지 않는다.

권장 지시 방식:

```text
MasterPlan.md를 먼저 읽고 전체 구조를 이해해.
그다음 Plan_03_LayoutJsonSchema.md만 기준으로 Layout JSON 타입과 샘플 데이터를 구현해.
다른 단계는 수정하지 마.
```

또는:

```text
Plan_07_DesSimulationEngine.md를 기준으로 EventQueue와 기본 Conveyor 로직만 구현해.
Three.js 화면은 건드리지 마.
```

작업 단위는 작게 나눈다.

```text
1회 작업 = 하나의 모듈 또는 하나의 기능
```

피해야 할 지시:

```text
이 프로젝트 전체를 만들어줘.
```

좋은 지시:

```text
src/simulation/engine/EventQueue.ts를 만들고,
이벤트를 시간순으로 push, pop, peek 할 수 있게 구현해.
테스트 코드도 함께 작성해.
```

---

## 24. 최종 목표 이미지

최종 시스템은 다음과 같은 흐름을 가진다.

```text
사용자는 도면 이미지를 업로드한다.
도면 위에 컨베이어, 로봇, 버퍼, 분기기를 배치한다.
설비 간 포트를 연결한다.
시간대별 물동량을 입력한다.
시뮬레이션을 실행한다.
박스 흐름이 Three.js 화면에서 재생된다.
병목 구간이 색상으로 표시된다.
로봇 가동률과 처리량이 리포트로 나온다.
대안 레이아웃을 복사해서 비교한다.
가장 좋은 설계안을 선택한다.
```

---

## 25. 결론

이 프로젝트의 본질은 3D 시각화가 아니라 **시뮬레이션 가능한 설비 데이터 구조**를 만드는 것이다.

Three.js는 도면에 생명을 불어넣는 무대이고,
DES Engine은 박스와 로봇의 운명을 계산하는 두뇌이며,
Layout JSON은 둘을 연결하는 설비 세계의 문법이다.

따라서 개발 순서는 다음 원칙을 따른다.

```text
1. 먼저 데이터 구조를 만든다.
2. 그다음 시뮬레이션 엔진을 만든다.
3. 그다음 Three.js로 보여준다.
4. 마지막으로 리포트와 고도화 기능을 붙인다.
```

이 순서를 지키면 첨부된 복잡한 물류 도면도 한 번에 삼키지 않고, 설비 컴포넌트 단위로 나누어 안정적으로 시뮬레이션할 수 있다.
