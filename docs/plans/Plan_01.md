# Plan_01.md

# Plan_01. Project Scope & MVP Definition

## 1. 목적

이 문서는 웹 기반 물류 라인 이산 사건 시뮬레이션 프로젝트의 **개발 범위, 1차 MVP 목표, 성공 기준, 제외 범위**를 정의하기 위한 실행 계획서이다.

MasterPlan.md가 전체 지도를 보여주는 문서라면, Plan_01.md는 실제 개발을 시작하기 전에 프로젝트의 울타리를 세우는 문서이다.
이 단계에서 범위를 명확히 정하지 않으면 Three.js 화면, 시뮬레이션 엔진, 도면 인식, 로봇 애니메이션, 리포트 기능이 한꺼번에 섞여 프로젝트가 커질 수 있다.

따라서 Plan_01의 핵심 목적은 다음과 같다.

```text
1. 이 프로젝트가 무엇을 만들고, 무엇을 만들지 않을지 정한다.
2. 1차 MVP에서 반드시 구현할 기능을 정한다.
3. 개발 우선순위를 정한다.
4. 완료 기준을 테스트 가능한 형태로 정의한다.
5. Codex 또는 개발자가 작업할 때 흔들리지 않을 기준선을 만든다.
```

이 문서는 프로젝트의 첫 번째 안전펜스다.
여기서 정한 범위를 기준으로 Plan_02부터 Plan_10까지의 세부 설계 문서가 이어진다.

---

## 2. 프로젝트 최종 목표

최종 목표는 복잡한 물류 자동화 라인을 웹에서 구성하고, 시간대별 물동량을 입력하여 이산 사건 시뮬레이션을 실행한 뒤, Three.js 화면에서 설비 흐름을 재생하고 병목을 분석할 수 있는 시스템을 만드는 것이다.

최종 시스템은 다음 흐름을 가진다.

```text
도면 이미지 또는 수동 배치
        ↓
Three.js Layout Editor
        ↓
Layout JSON
        ↓
DES Simulation Engine
        ↓
Event Log / State Timeline
        ↓
Three.js Animation Player
        ↓
Analysis Report
```

핵심 원칙은 다음과 같다.

```text
Three.js는 화면과 편집기를 담당한다.
DES Simulation Engine은 계산과 판단을 담당한다.
Layout JSON은 둘 사이의 공통 언어가 된다.
```

즉, Three.js가 시뮬레이션을 직접 수행하는 것이 아니다.
사용자가 Three.js 화면에서 설비를 배치하면 그 결과가 Layout JSON으로 저장되고, DES Simulation Engine이 그 데이터를 읽어 시뮬레이션을 수행한다.

---

## 3. 프로젝트 사용 목적

이 프로젝트는 크게 세 가지 목적을 가진다.

### 3.1 고객 설명용

정적인 CAD 도면이나 2D 레이아웃만으로는 물류 흐름을 설명하기 어렵다.
고객 설명용 모드에서는 컨베이어, 박스, 로봇, 팔레트의 흐름을 시각적으로 보여주는 것이 중요하다.

주요 목적:

```text
- 박스가 어느 방향으로 이동하는지 설명
- 로봇이 어느 위치에서 픽업하고 어느 위치에 적재하는지 설명
- 분기와 합류 흐름을 직관적으로 표현
- 설비 배치의 전체 흐름을 비전문가도 이해할 수 있게 시각화
```

이 목적에서는 정밀한 계산보다 이해하기 쉬운 화면이 중요하다.

### 3.2 설계 검토용

설계 검토용 모드에서는 단순한 애니메이션이 아니라 실제 설계 판단에 도움이 되는 결과를 제공해야 한다.

주요 목적:

```text
- 로봇 대수 검토
- 컨베이어 속도 검토
- 버퍼 용량 검토
- 분기 및 합류 병목 확인
- 팔레타이징 처리량 검토
- 설비별 가동률 확인
```

이 프로젝트의 1차 MVP는 고객 설명용과 설계 검토용의 중간 지점을 목표로 한다.
즉, 화면으로도 이해 가능하고, 기본적인 수치 검토도 가능한 수준을 만든다.

### 3.3 운영 분석용

운영 분석용은 가장 고도화된 목적이다.
시간대별 물동량, 제품 Mix, 설비 정지, 작업자 투입, 고장 시나리오 등을 반영해야 한다.

주요 목적:

```text
- 시간대별 물동량 변화 반영
- 제품별 경로 차이 반영
- 설비 고장 시나리오 반영
- 일 처리량 예측
- 대안 운영 조건 비교
```

운영 분석용 기능은 MVP 범위에는 포함하지 않고, 고도화 단계에서 검토한다.

---

## 4. 1차 MVP 정의

### 4.1 MVP 목표

1차 MVP의 목표는 **단순한 물류 라인을 Layout JSON으로 정의하고, 이산 사건 시뮬레이션으로 박스 이동과 로봇 팔레타이징을 계산한 뒤, Three.js 화면에서 결과를 재생하는 것**이다.

MVP는 완성형 제품이 아니라, 전체 구조가 옳은지 검증하는 첫 번째 작동 모델이다.

### 4.2 MVP 기본 시나리오

1차 MVP에서는 다음과 같은 단순 라인을 기준으로 한다.

```text
Source
  ↓
Conveyor 1
  ↓
Diverter
  ↓              ↓
Conveyor 2      Conveyor 3
  ↓              ↓
Robot 1         Robot 2
  ↓              ↓
Pallet 1        Pallet 2
```

또는 더 단순한 첫 테스트는 다음과 같다.

```text
Source
  ↓
Conveyor
  ↓
Robot
  ↓
Pallet Station
  ↓
Sink
```

단순 라인부터 시작하는 이유는 다음과 같다.

```text
1. Layout JSON 구조를 검증하기 쉽다.
2. Event Queue 동작을 검증하기 쉽다.
3. 컨베이어 이동 시간 계산을 확인하기 쉽다.
4. 로봇 cycle time 기반 처리량을 검증하기 쉽다.
5. Three.js 재생 구조를 만들기 쉽다.
```

첨부 도면처럼 복잡한 라인은 MVP 이후 확장 단계에서 다룬다.
처음부터 큰 도면을 통째로 다루면 바다에 컨베이어를 깔고 길을 묻는 꼴이 된다. 먼저 작은 라인에서 물류의 문법을 검증해야 한다.

---

## 5. MVP 포함 범위

1차 MVP에 포함할 기능은 다음과 같다.

### 5.1 데이터 구조

```text
- Layout JSON
- Scenario JSON
- Simulation Result JSON
- Component Type 정의
- Port 정의
- Connection 정의
- Event Log 정의
```

### 5.2 컴포넌트

MVP에 포함할 표준 컴포넌트는 다음과 같다.

```text
Source
- 박스를 생성하는 시작점

Conveyor
- 박스를 일정 속도로 이동시키는 설비

Diverter
- 목적지 기준으로 박스를 분기시키는 설비

Merge
- 여러 라인을 하나로 합치는 설비

Buffer
- 박스가 대기하는 공간

Robot
- 박스를 픽업하고 팔레트에 적재하는 설비

Pallet Station
- 로봇이 박스를 적재하는 위치

Sink
- 완료 지점
```

### 5.3 시뮬레이션 로직

MVP에 포함할 기본 로직은 다음과 같다.

```text
- 시간대별 박스 생성
- 컨베이어 이동 시간 계산
- 다음 설비 진입 가능 여부 확인
- downstream blocking 처리
- 분기기 목적지 기반 출구 선택
- 합류부 FIFO 처리
- 버퍼 capacity 기반 대기 처리
- 로봇 resource 점유 및 해제
- 로봇 cycle time 기반 작업 처리
- 팔레트 적재 수량 계산
- Sink 도착 시 완료 처리
```

### 5.4 Three.js 시각화

MVP의 Three.js 화면은 고급 3D가 아니라 **2.5D Top View 기반 시각화**로 시작한다.

포함 기능:

```text
- 설비 배치 표시
- 컨베이어 방향 표시
- 박스 이동 표시
- 로봇 작업 상태 표시
- 팔레트 적재 수량 표시
- 병목 또는 대기 상태 색상 표시
- 시뮬레이션 결과 재생
- 배속 조절
- 일시정지
- 시간 슬라이더
```

### 5.5 리포트

MVP에서 출력할 기본 지표는 다음과 같다.

```text
- 총 투입 수량
- 총 완료 수량
- 미처리 WIP
- 시간당 처리량
- 평균 리드타임
- 평균 대기시간
- 로봇 가동률
- 설비별 busy / idle / blocked 시간
- 병목 설비 후보
```

---

## 6. MVP 제외 범위

다음 기능은 1차 MVP에서는 제외한다.

### 6.1 CAD 자동 인식 제외

MVP에서는 CAD 이미지나 도면을 자동으로 인식하여 컨베이어를 생성하지 않는다.

제외 기능:

```text
- 이미지 기반 컨베이어 자동 검출
- DXF 자동 파싱
- DWG 직접 해석
- 도면 텍스트 OCR 자동 매핑
- 설비명 자동 인식
```

MVP에서는 도면 이미지를 배경으로 까는 기능까지는 가능하지만, 실제 컴포넌트 배치는 사용자가 수동으로 한다.

### 6.2 정밀 물리 엔진 제외

MVP에서는 물리 충돌, 박스 낙하, 박스 흔들림, 마찰, 중력 기반 적재 안정성은 다루지 않는다.

제외 기능:

```text
- Rapier 등 물리엔진 연동
- 박스 충돌 계산
- 낙하 시뮬레이션
- 적재 안정성 계산
- 박스 변형
```

MVP에서는 박스가 지정된 경로를 따라 이동하고, 팔레트에는 계산된 좌표에 적재되는 방식으로 처리한다.

### 6.3 실제 로봇 역기구학 제외

MVP에서는 실제 6축 로봇의 관절각, 역기구학, 충돌 회피를 계산하지 않는다.

제외 기능:

```text
- 6축 로봇 IK 계산
- 관절 제한 계산
- 로봇 간 충돌 회피
- 실제 로봇 모션 플래닝
- PLC 연동
```

MVP에서는 로봇을 cycle time을 가진 resource로 모델링한다.
Three.js 화면에서는 단순 Pick & Place 애니메이션으로 표현한다.

### 6.4 최적화 기능 제외

MVP에서는 자동으로 최적 레이아웃이나 최적 속도를 추천하지 않는다.

제외 기능:

```text
- 로봇 대수 자동 추천
- 컨베이어 속도 최적화
- 버퍼 용량 최적화
- 경로 자동 최적화
- AI 개선안 자동 생성
```

MVP에서는 사용자가 조건을 입력하고 결과를 확인하는 수준까지만 구현한다.

---

## 7. 입력 데이터

MVP에서 필요한 입력 데이터는 크게 세 종류이다.

### 7.1 Layout JSON

Layout JSON은 설비 배치와 연결관계를 정의한다.

필수 항목:

```text
- project id
- project name
- unit
- components
- component position
- component rotation
- component dimensions
- component properties
- ports
- connections
```

예시:

```json
{
  "project": {
    "id": "mvp_project_001",
    "name": "MVP Palletizing Line",
    "unit": "meter"
  },
  "components": [
    {
      "id": "SRC_001",
      "type": "source",
      "position": { "x": 0, "y": 0, "z": 0 },
      "ports": {
        "in": [],
        "out": ["SRC_001_OUT"]
      }
    },
    {
      "id": "CV_001",
      "type": "conveyor",
      "position": { "x": 3, "y": 0, "z": 0 },
      "dimensions": {
        "length": 6,
        "width": 0.8,
        "height": 0.6
      },
      "properties": {
        "speed": 0.6,
        "capacity": 10
      },
      "ports": {
        "in": ["CV_001_IN"],
        "out": ["CV_001_OUT"]
      }
    }
  ],
  "connections": [
    {
      "from": "SRC_001_OUT",
      "to": "CV_001_IN"
    }
  ]
}
```

### 7.2 Scenario JSON

Scenario JSON은 물동량과 운영 조건을 정의한다.

필수 항목:

```text
- simulation start time
- simulation end time
- item master
- arrival schedule
- routing rule
- process time
- robot cycle time
```

예시:

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
      "source": "SRC_001",
      "itemType": "BOX_A",
      "startTime": 0,
      "endTime": 3600,
      "quantity": 600,
      "arrivalPattern": "uniform"
    }
  ]
}
```

### 7.3 Simulation Config

Simulation Config는 실행 옵션을 정의한다.

예시:

```json
{
  "randomSeed": 1,
  "enableAnimationLog": true,
  "maxEvents": 100000,
  "defaultMergeRule": "fifo",
  "defaultBlockingRule": "downstreamBlocking"
}
```

---

## 8. 출력 결과

MVP에서 생성해야 하는 출력 결과는 다음과 같다.

### 8.1 Simulation Result JSON

시뮬레이션 결과 요약과 상세 로그를 저장한다.

포함 항목:

```text
- summary
- componentStats
- itemStats
- eventLog
- stateTimeline
```

예시:

```json
{
  "summary": {
    "simulationTime": 3600,
    "createdItems": 600,
    "completedItems": 520,
    "wip": 80,
    "throughputPerHour": 520,
    "averageLeadTime": 310,
    "averageWaitingTime": 95
  }
}
```

### 8.2 Event Log

각 아이템과 설비의 주요 사건을 시간순으로 기록한다.

주요 이벤트:

```text
ITEM_CREATED
ITEM_ENTER_COMPONENT
ITEM_EXIT_COMPONENT
ITEM_WAIT_START
ITEM_WAIT_END
ROBOT_PICK_START
ROBOT_PICK_END
PALLETIZE_COMPLETE
ITEM_COMPLETED
BLOCKING_START
BLOCKING_END
```

### 8.3 Component Statistics

설비별 상태와 성능을 기록한다.

포함 항목:

```text
- busy time
- idle time
- blocked time
- waiting item count
- utilization
- throughput
```

### 8.4 화면 출력

Three.js 화면에서 다음을 표시한다.

```text
- 설비 배치
- 박스 위치
- 컨베이어 이동 방향
- 로봇 작업 상태
- 팔레트 적재 현황
- 대기 또는 병목 구간
- 시뮬레이션 현재 시간
```

### 8.5 리포트 출력

화면 또는 파일로 다음 요약을 출력한다.

```text
- 총 투입 수량
- 총 완료 수량
- 시간당 처리량
- 평균 리드타임
- 평균 대기시간
- 설비별 가동률
- 병목 설비 Top 3
```

---

## 9. 핵심 설계 원칙

### 9.1 데이터 우선 설계

MVP에서 가장 중요한 것은 멋진 화면이 아니라 **정확한 데이터 구조**이다.

우선순위는 다음과 같다.

```text
1. Layout JSON
2. Scenario JSON
3. DES Engine
4. Event Log
5. Three.js Viewer
6. Report
```

화면이 먼저가 아니다.
데이터와 엔진이 먼저이고, 화면은 그 결과를 보여주는 창문이다.

### 9.2 화면과 계산 분리

Three.js는 편집과 재생을 담당한다.
DES Engine은 계산을 담당한다.

금지할 방식:

```text
Three.js Mesh의 position을 직접 읽어서 컨베이어 흐름을 판단한다.
```

권장 방식:

```text
Layout JSON의 component, port, connection 정보를 기준으로 흐름을 판단한다.
Three.js는 Layout JSON을 렌더링한다.
```

### 9.3 MVP에서는 단순 모델을 사용

컨베이어는 실제 박스 간 미세 간격을 모두 계산하지 않고, capacity와 travel time 기반으로 단순화한다.

로봇은 실제 6축 운동이 아니라 cycle time 기반 resource로 처리한다.

팔레트 적재는 물리 안정성보다 적재 수량과 위치 계산 중심으로 처리한다.

### 9.4 테스트 가능한 기능부터 구현

각 기능은 반드시 테스트 가능한 단위로 쪼갠다.

예시:

```text
EventQueue는 시간순으로 이벤트를 꺼낼 수 있어야 한다.
Conveyor는 길이와 속도를 기준으로 이동 시간을 계산해야 한다.
Robot은 busy 상태일 때 다음 아이템을 대기시켜야 한다.
Buffer는 capacity가 가득 차면 상류를 blocking해야 한다.
```

---

## 10. 개발 우선순위

### 10.1 1순위

```text
Component Type 정의
Layout JSON Schema 정의
Scenario JSON Schema 정의
EventQueue 구현
기본 Conveyor 이동 로직 구현
```

이 단계는 프로젝트의 뼈대다.

### 10.2 2순위

```text
Source 생성 로직
Robot resource 로직
Pallet Station 적재 로직
Event Log 생성
기본 Summary KPI 계산
```

이 단계에서 단순 팔레타이징 라인이 계산 가능해진다.

### 10.3 3순위

```text
Three.js 기본 화면
Layout JSON 렌더링
Event Log 기반 박스 이동 재생
시뮬레이션 시간 슬라이더
```

이 단계에서 계산 결과가 눈에 보이기 시작한다.

### 10.4 4순위

```text
Diverter
Merge
Buffer
Blocking
병목 표시
Report
```

이 단계에서 물류 라인다운 성격이 강해진다.

---

## 11. 작업 목록

### 11.1 문서 작업

* [ ] MasterPlan.md와 Plan_01.md의 범위 일치 여부 확인
* [ ] MVP 포함 범위 확정
* [ ] MVP 제외 범위 확정
* [ ] Plan_02 ~ Plan_10 문서 목차 확정

### 11.2 데이터 설계 작업

* [ ] Component 공통 속성 정의
* [ ] Port 구조 정의
* [ ] Connection 구조 정의
* [ ] Layout JSON 초안 작성
* [ ] Scenario JSON 초안 작성
* [ ] Simulation Result JSON 초안 작성

### 11.3 엔진 설계 작업

* [ ] Event 타입 목록 정의
* [ ] EventQueue 기능 정의
* [ ] Item 상태 정의
* [ ] Component 상태 정의
* [ ] Conveyor 기본 로직 정의
* [ ] Robot 기본 로직 정의
* [ ] Statistics 계산 항목 정의

### 11.4 화면 설계 작업

* [ ] MVP 화면 구성 정의
* [ ] Layout Viewer 화면 정의
* [ ] Animation Player 화면 정의
* [ ] Report 화면 정의

### 11.5 테스트 설계 작업

* [ ] 단순 Source → Conveyor → Sink 테스트 케이스 정의
* [ ] Source → Conveyor → Robot → Pallet 테스트 케이스 정의
* [ ] Robot busy 대기 테스트 케이스 정의
* [ ] Buffer full blocking 테스트 케이스 정의
* [ ] Diverter 목적지 분기 테스트 케이스 정의

---

## 12. 완료 기준

Plan_01 단계는 실제 코드를 구현하는 단계가 아니라, 프로젝트 범위를 고정하는 단계이다.
따라서 완료 기준은 다음과 같다.

### 12.1 문서 완료 기준

```text
- 프로젝트 최종 목표가 명확히 작성되어 있다.
- MVP 포함 범위가 명확히 작성되어 있다.
- MVP 제외 범위가 명확히 작성되어 있다.
- 입력 데이터 종류가 정의되어 있다.
- 출력 결과 종류가 정의되어 있다.
- 개발 우선순위가 정리되어 있다.
- 테스트 가능한 완료 기준이 작성되어 있다.
```

### 12.2 개발 착수 가능 기준

Plan_01이 완료되면 다음 질문에 답할 수 있어야 한다.

```text
1. 1차 MVP에서 어떤 설비 컴포넌트를 만들 것인가?
2. 1차 MVP에서 어떤 기능은 만들지 않을 것인가?
3. Three.js와 DES Engine은 어떤 역할로 분리되는가?
4. Layout JSON은 왜 필요한가?
5. 첫 번째 테스트 라인은 어떤 구조인가?
6. 시뮬레이션 결과로 어떤 지표를 볼 것인가?
7. Codex에게 첫 작업을 어떻게 나눠 줄 것인가?
```

이 질문에 답할 수 있으면 Plan_02_ComponentModel.md로 넘어간다.

---

## 13. 첫 번째 테스트 시나리오

MVP 검증을 위한 첫 번째 테스트 시나리오는 다음과 같다.

### 13.1 라인 구성

```text
SRC_001
  ↓
CV_001
  ↓
RB_001
  ↓
PALLET_001
  ↓
SINK_001
```

### 13.2 조건

```text
시뮬레이션 시간: 1시간
투입 수량: 600 box
투입 패턴: uniform
컨베이어 길이: 6m
컨베이어 속도: 0.6m/s
컨베이어 이동 시간: 10초
로봇 cycle time: 8초/box
팔레트 최대 적재 수량: 100 box
```

### 13.3 예상 검토 포인트

```text
- 600개가 정상 생성되는가?
- 컨베이어 이동 시간이 10초로 계산되는가?
- 로봇이 8초마다 1개씩 처리되는가?
- 로봇 처리 능력이 투입량보다 충분한가?
- 팔레트 적재 수량이 증가하는가?
- 완료 수량과 WIP가 계산되는가?
- Event Log가 시간순으로 생성되는가?
- Three.js 화면에서 박스 이동이 재생되는가?
```

---

## 14. 두 번째 테스트 시나리오

두 번째 테스트는 분기 로직을 확인하기 위한 시나리오이다.

### 14.1 라인 구성

```text
SRC_001
  ↓
CV_001
  ↓
DIV_001
  ↓           ↓
CV_002       CV_003
  ↓           ↓
RB_001       RB_002
  ↓           ↓
PALLET_001   PALLET_002
```

### 14.2 조건

```text
BOX_A destination: PALLET_001
BOX_B destination: PALLET_002
BOX_A 수량: 300
BOX_B 수량: 300
분기 규칙: byDestination
로봇 1 cycle time: 8초
로봇 2 cycle time: 10초
```

### 14.3 예상 검토 포인트

```text
- BOX_A가 PALLET_001 방향으로 이동하는가?
- BOX_B가 PALLET_002 방향으로 이동하는가?
- 로봇별 가동률이 다르게 계산되는가?
- 느린 로봇 쪽에서 대기 또는 WIP가 증가하는가?
- 병목 후보가 올바르게 표시되는가?
```

---

## 15. Codex 작업 지시 예시

Plan_01 완료 후 Codex에게 줄 수 있는 첫 번째 지시는 다음과 같다.

```text
MasterPlan.md와 Plan_01.md를 읽고, 프로젝트의 MVP 범위를 이해해.
아직 Three.js 화면은 만들지 말고, src/types 폴더에 Layout, Scenario, SimulationResult 타입 정의만 작성해.
컴포넌트 타입은 Source, Conveyor, Diverter, Merge, Buffer, Robot, PalletStation, Sink를 포함해.
```

두 번째 지시는 다음과 같다.

```text
Plan_01.md의 첫 번째 테스트 시나리오를 기준으로,
src/simulation/engine/EventQueue.ts를 구현해.
이벤트는 time 기준 오름차순으로 pop 되어야 하고,
같은 time이면 sequence 값 기준으로 정렬되게 해.
단위 테스트도 함께 작성해.
```

세 번째 지시는 다음과 같다.

```text
Source → Conveyor → Robot → PalletStation → Sink 테스트 라인을 하드코딩 샘플로 만들고,
1시간 동안 600개 box가 uniform하게 투입되는 시뮬레이션을 실행해.
결과로 createdItems, completedItems, robotUtilization, eventLog를 출력해.
```

---

## 16. 주의사항

### 16.1 프로젝트를 처음부터 크게 만들지 말 것

첨부 도면처럼 복잡한 라인을 바로 구현하려고 하면 범위가 급격히 커진다.

초기에는 다음 순서를 지켜야 한다.

```text
1. 단순 라인
2. 분기 라인
3. 합류 라인
4. 버퍼 포함 라인
5. 로봇 2대 라인
6. 복잡한 도면 기반 라인
```

### 16.2 Three.js 화면에 집착하지 말 것

멋진 화면은 중요하지만, 시뮬레이션 계산 구조가 먼저다.
데이터 구조가 부실하면 화면이 아무리 예뻐도 나중에 병목 분석이 불가능하다.

### 16.3 실제 물리와 논리 시뮬레이션을 혼동하지 말 것

이 프로젝트는 1차적으로 물리 시뮬레이터가 아니라 물류 흐름 시뮬레이터이다.

따라서 MVP에서는 다음처럼 단순화한다.

```text
박스 이동 = 거리 / 속도
로봇 작업 = cycle time
버퍼 대기 = capacity
분기 = rule
합류 = queue discipline
```

### 16.4 모든 기능은 JSON으로 재현 가능해야 함

사용자가 화면에서 만든 레이아웃은 반드시 JSON으로 저장되고, 다시 불러왔을 때 동일하게 복원되어야 한다.

이 원칙을 지키지 않으면 시뮬레이션, 리포트, 대안 비교가 어려워진다.

---

## 17. 다음 단계

Plan_01이 완료되면 다음 문서로 이동한다.

```text
Plan_02_ComponentModel.md
```

Plan_02에서는 MVP에 포함되는 설비 컴포넌트를 더 세밀하게 정의한다.

다룰 내용:

```text
- Component 공통 필드
- Source 상세 정의
- Conveyor 상세 정의
- Diverter 상세 정의
- Merge 상세 정의
- Buffer 상세 정의
- Robot 상세 정의
- Pallet Station 상세 정의
- Sink 상세 정의
- Port 구조
- Component State 구조
- Component별 이벤트 처리 책임
```

Plan_01은 프로젝트의 경계선이고, Plan_02는 설비 세계의 부품 카탈로그다.
이 두 문서가 단단하면 이후 Layout JSON과 DES Engine이 흔들리지 않는다.
::: 
