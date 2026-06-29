# Plan_10.md

# Plan_10. Report, Export, Comparison & Roadmap Definition

## 1. 목적

이 문서는 웹 기반 물류 라인 이산 사건 시뮬레이션 프로젝트의 마지막 실행 계획서로, **시뮬레이션 결과 리포트, 결과 내보내기, 시나리오 비교, 레이아웃 대안 비교, 향후 고도화 방향**을 정의한다.

Plan_01부터 Plan_09까지는 다음 흐름을 만들었다.

```text
설비 정의
→ Layout JSON
→ Three.js Layout Editor
→ Network Routing
→ Scenario JSON
→ DES Simulation Engine
→ Event Log Timeline
→ Three.js Animation Player
```

Plan_10은 이 흐름의 결과를 사용자가 의사결정에 활용할 수 있도록 정리하는 단계다.

---

## 2. Report의 기본 원칙

리포트는 단순 숫자 나열이 아니라 설계자가 바로 판단할 수 있는 구조여야 한다.

```text
1. 전체 성능을 먼저 보여준다.
2. 병목을 빠르게 찾게 한다.
3. 설비별 원인을 확인할 수 있게 한다.
4. 아이템 흐름의 체류시간과 대기시간을 보여준다.
5. 조건을 바꾼 대안끼리 비교할 수 있게 한다.
6. 결과를 외부 문서나 발표자료로 가져갈 수 있게 한다.
```

리포트는 다음 3단 구조가 좋다.

```text
Executive Summary
- 한눈에 보는 결과

Engineering Detail
- 설비별 상세 분석

Raw Data Export
- CSV, JSON 원천 데이터
```

---

## 3. Summary Report

Summary Report는 시뮬레이션 전체 결과를 한눈에 보여준다.

주요 KPI:

```text
simulationStartTime
simulationEndTime
simulationDuration
createdItems
completedItems
wip
throughputPerHour
averageLeadTime
averageWaitingTime
averageProcessingTime
deadlockDetected
bottleneckComponentIds
```

KPI 카드 구성:

```text
총 투입 수량
- createdItems

총 완료 수량
- completedItems

미완료 WIP
- wip

시간당 처리량
- throughputPerHour

평균 리드타임
- averageLeadTime

평균 대기시간
- averageWaitingTime

병목 1순위
- bottleneckComponentIds[0]

시뮬레이션 상태
- completed, deadlock, error, stopped
```

예시:

```text
시뮬레이션 시간: 3,600초
투입 수량: 600 box
완료 수량: 520 box
WIP: 80 box
시간당 처리량: 520 box/hour
평균 리드타임: 310초
평균 대기시간: 95초
병목 후보: RB_001
상태: completed
```

---

## 4. Bottleneck Report

Bottleneck Report는 어떤 설비가 전체 흐름을 제한하는지 찾는 리포트다.

MVP에서는 다음 점수식을 사용한다.

```text
bottleneckScore =
  blockedTimeWeight * normalizedBlockedTime
+ waitingTimeWeight * normalizedWaitingTime
+ utilizationWeight * normalizedUtilization
+ queueLengthWeight * normalizedMaxQueueLength
```

MVP 기본 가중치:

```text
blockedTimeWeight = 0.35
waitingTimeWeight = 0.30
utilizationWeight = 0.25
queueLengthWeight = 0.10
```

초기 구현에서는 단순 정렬을 먼저 사용해도 된다.

```text
1. blockedTime 내림차순
2. waitingTime 내림차순
3. utilization 내림차순
4. maxQueueLength 내림차순
```

Bottleneck Table:

| Rank | Component | Type     | Utilization | Waiting Time | Blocked Time | Max Queue | Score |
| ---- | --------- | -------- | ----------: | -----------: | -----------: | --------: | ----: |
| 1    | RB_001    | robot    |         96% |       1,240s |           0s |        42 |  0.91 |
| 2    | CV_003    | conveyor |         65% |         300s |         820s |        10 |  0.73 |
| 3    | MRG_001   | merge    |         72% |         610s |         200s |        18 |  0.68 |

---

## 5. Component Report

Component Report는 설비별 성능을 상세히 보여준다.

대상 컴포넌트:

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
```

Component Statistics Table:

| Component | Type     | Processed |   Busy |   Idle | Waiting | Blocked | Utilization |
| --------- | -------- | --------: | -----: | -----: | ------: | ------: | ----------: |
| CV_001    | conveyor |       600 | 3,000s |   600s |      0s |      0s |       83.3% |
| RB_001    | robot    |       520 | 3,400s |   200s |      0s |      0s |       94.4% |
| BUF_001   | buffer   |       500 |     0s | 2,000s |  1,600s |      0s |          0% |

설비별 주요 항목:

```text
Source
- createdCount
- releaseCount
- sourceQueueLength
- blockedByDownstreamTime

Conveyor
- enteredCount
- exitedCount
- busyTime
- blockedTime
- averageOccupancy
- capacity
- speed

Robot
- processedCount
- cycleTime
- busyTime
- idleTime
- utilization
- queueLength
- averageRobotWaitingTime

Pallet Station
- palletizedBoxCount
- completedPalletCount
- currentBoxCount
- averagePalletCompletionTime
```

---

## 6. Item Report

Item Report는 아이템별 흐름을 분석한다.

주요 질문:

```text
어떤 아이템이 오래 걸렸는가?
어떤 아이템이 많이 기다렸는가?
완료되지 못한 아이템은 어디에 남아 있는가?
제품별 Lead Time 차이가 있는가?
```

Item Statistics Table:

| Item ID      | Type  | Created | Completed | Lead Time | Waiting Time | Processing Time | Final State |
| ------------ | ----- | ------: | --------: | --------: | -----------: | --------------: | ----------- |
| BOX_A_000001 | BOX_A |      0s |       18s |       18s |           0s |              8s | completed   |
| BOX_A_000120 | BOX_A |    714s |      840s |      126s |          80s |              8s | completed   |
| BOX_A_000600 | BOX_A |   3594s |         - |         - |           0s |              0s | moving      |

---

## 7. Scenario Comparison

Scenario Comparison은 같은 Layout에서 시나리오 조건만 바꿔 결과를 비교하는 기능이다.

예시:

```text
Case A: 로봇 cycleTime 8초
Case B: 로봇 cycleTime 6초
Case C: 투입량 600 box/hour
Case D: 투입량 800 box/hour
```

비교 KPI:

| Case | Completed | Throughput/h | Avg Lead Time | Avg Waiting | Bottleneck | Robot Utilization |
| ---- | --------: | -----------: | ------------: | ----------: | ---------- | ----------------: |
| A    |       520 |          520 |          310s |         95s | RB_001     |               94% |
| B    |       590 |          590 |          160s |         30s | CV_001     |               78% |

해석 예시:

```text
Case B는 Case A 대비 완료 수량이 13.5% 증가했고 평균 대기시간이 68.4% 감소했습니다.
로봇 cycleTime 단축이 현재 레이아웃의 처리량 개선에 직접적인 효과가 있는 것으로 보입니다.
```

---

## 8. Layout Comparison

Layout Comparison은 서로 다른 설비 배치 또는 설비 수량을 비교하는 기능이다.

예시:

```text
Layout A: 로봇 1대
Layout B: 로봇 2대
Layout C: Buffer 추가
Layout D: Merge 위치 변경
```

비교 KPI:

```text
completedItems
throughputPerHour
averageLeadTime
averageWaitingTime
totalBlockedTime
robotUtilization
requiredBufferCapacity
bottleneckComponentIds
```

설계 판단 예시:

```text
Layout B는 로봇 2대를 사용하여 완료 수량은 증가했지만, Merge 구간의 blocked time이 증가했습니다.
로봇 추가만으로는 충분하지 않고 합류부 또는 하류 컨베이어 capacity 보완이 필요합니다.
```

---

## 9. Export 기능

지원할 export 형식:

```text
JSON
CSV
PNG
PDF
Markdown
```

CSV로 내보낼 데이터:

```text
summary.csv
component_stats.csv
item_stats.csv
event_log.csv
bottleneck_ranking.csv
timeline_bucket_stats.csv
```

component_stats.csv 예시:

```csv
componentId,componentType,processedCount,busyTime,idleTime,waitingTime,blockedTime,utilization,maxQueueLength
RB_001,robot,520,3400,200,0,0,0.944,42
```

item_stats.csv 예시:

```csv
itemId,itemType,createdAt,completedAt,leadTime,totalWaitingTime,totalProcessingTime,completed
BOX_A_000001,BOX_A,0,18,18,0,8,true
```

Markdown Report 예시:

```md
# Simulation Report

## Summary
- Completed Items: 520
- Throughput: 520 box/hour
- Bottleneck: RB_001

## Bottleneck Analysis
RB_001 utilization is 94.4%.
```

---

## 10. 개선 제안 Rules

### Robot 병목

조건:

```text
componentType = robot
utilization >= 0.9
averageWaitingTime > 0
```

제안:

```text
- robot cycleTime 단축 검토
- robot 추가 검토
- upstream 투입 간격 조정 검토
- pick point 앞 buffer 추가 검토
```

### Conveyor Blocking

조건:

```text
componentType = conveyor
blockedTime > threshold
```

제안:

```text
- 하류 설비 capacity 확인
- 하류 merge rule 확인
- downstream robot 또는 process station 처리시간 확인
- buffer 추가 검토
```

### Buffer Full

조건:

```text
componentType = buffer
maxQueueLength >= capacity
```

제안:

```text
- buffer capacity 증설 검토
- 하류 설비 처리시간 단축 검토
- 투입량 평준화 검토
```

### Merge 병목

조건:

```text
componentType = merge
waitingTime high
blockedTime high
```

제안:

```text
- merge rule 변경 검토
- roundRobin 또는 mainLinePriority 비교
- 합류 전 buffer 추가 검토
- output conveyor speed 검토
```

---

## 11. Roadmap 개요

로드맵은 다음 단계로 나눈다.

```text
Phase 1. MVP 안정화
Phase 2. Layout Editor 고도화
Phase 3. DES Engine 고도화
Phase 4. Animation 고도화
Phase 5. Report와 Comparison 고도화
Phase 6. CAD/DXF 반자동 인식
Phase 7. Optimization과 AI 개선안
Phase 8. Enterprise 기능
```

---

## 12. Phase 1. MVP 안정화

목표:

```text
기본 구조가 실제로 작동하는지 검증한다.
```

포함 기능:

```text
- Source
- Conveyor
- Robot
- Pallet Station
- Sink
- Layout JSON
- Scenario JSON
- DES Engine
- Event Log
- Animation Player
- Summary Report
```

완료 기준:

```text
- Source → Conveyor → Robot → Pallet → Sink 라인이 정상 실행된다.
- 1시간 600개 투입 시 결과가 생성된다.
- 박스 이동이 화면에 재생된다.
- Summary KPI가 표시된다.
- robot utilization이 계산된다.
```

---

## 13. Phase 2. Layout Editor 고도화

포함 기능:

```text
- 도면 배경 업로드
- 스케일 보정
- 컴포넌트 snap
- 포트 연결 UX 개선
- Diverter, Merge, Buffer 추가
- Undo / Redo
- Layout validation 강화
```

추가 기능:

```text
- 컴포넌트 복사/붙여넣기
- 다중 선택
- 정렬 기능
- 그룹화
- 레이어 표시
- 설비 검색
```

---

## 14. Phase 3. DES Engine 고도화

추가 기능:

```text
- Zone Conveyor
- Accumulation Conveyor
- Merge roundRobin
- Merge mainLinePriority
- Diverter shortestQueue
- Buffer priority queue
- Process Station
- 설비 고장
- Shift와 Break
- 재작업 루프
- 확률적 도착
```

고급 통계:

```text
- p95 lead time
- p95 waiting time
- time bucket throughput
- queue length trend
- utilization trend
- deadlock diagnosis
```

---

## 15. Phase 4. Animation 고도화

추가 기능:

```text
- InstancedMesh
- 대용량 item 최적화
- GLB 로봇 모델
- 컨베이어 롤러 애니메이션
- 로봇 단순 관절 애니메이션
- 병목 heatmap
- flow density overlay
- camera preset
- 발표 모드
```

발표 모드:

```text
- 자동 카메라 이동
- 주요 병목 지점 순서대로 확대
- KPI 카드 자동 표시
- 시나리오 설명 자막
```

---

## 16. Phase 5. Report와 Comparison 고도화

추가 기능:

```text
- Scenario Comparison
- Layout Comparison
- 자동 개선 제안
- Markdown Report
- PDF Report
- Excel Export
- Chart Export
- Case 관리
```

Case 관리 구조:

```text
Case A
- Layout A
- Scenario A
- Result A

Case B
- Layout A
- Scenario B
- Result B

Case C
- Layout B
- Scenario A
- Result C
```

---

## 17. Phase 6. CAD/DXF 반자동 인식

DXF 반자동 인식 흐름:

```text
1. 사용자가 DXF 파일을 업로드한다.
2. DXF Layer 목록을 표시한다.
3. 사용자가 Conveyor Layer를 선택한다.
4. 선분 또는 polyline을 컨베이어 후보로 추출한다.
5. 사용자가 후보를 확인한다.
6. 후보를 Conveyor Component로 변환한다.
7. 설비명 Text를 name 또는 id 후보로 사용한다.
8. 연결 가능한 선분을 connection 후보로 표시한다.
```

반자동을 우선하는 이유:

```text
- 도면마다 layer 규칙이 다르다.
- 컨베이어와 구조물이 같은 색 또는 선 스타일일 수 있다.
- 자동 인식 오류가 설계 오류로 이어질 수 있다.
- 사용자가 확인하는 반자동 방식이 더 안전하다.
```

---

## 18. Phase 7. Optimization과 AI 개선안

가능 기능:

```text
- 로봇 대수 추천
- 버퍼 capacity 추천
- 컨베이어 speed 추천
- Merge rule 추천
- 투입량 평준화 추천
- 병목 원인 설명
- 개선안 보고서 자동 생성
```

Simulation Sweep 예시:

```text
Robot cycleTime: 6s, 8s, 10s
Buffer capacity: 10, 20, 30
Conveyor speed: 0.4, 0.6, 0.8 m/s
```

AI가 할 수 있는 일:

```text
- 병목 원인 자연어 설명
- 개선안 후보 작성
- 고객 보고서 문장 생성
- 발표 대본 생성
- 설계 비교 요약
```

AI가 직접 하면 안 되는 일:

```text
- 검증 없는 최종 설계 확정
- 안전성 판단 단독 수행
- 비용 추정 단독 확정
- 실제 PLC 제어값 생성
```

---

## 19. 개발 폴더 구조 제안

```text
src/report/
  summary/
    SummaryCards.tsx
    buildSummaryViewModel.ts

  bottleneck/
    BottleneckTable.tsx
    calculateBottleneckRanking.ts
    bottleneckRules.ts

  component/
    ComponentStatsTable.tsx
    ComponentDetailPanel.tsx

  item/
    ItemStatsTable.tsx
    ItemDetailPanel.tsx

  timeline/
    TimelineBucketChart.tsx
    buildTimelineBuckets.ts

  comparison/
    ComparisonDashboard.tsx
    compareSimulationResults.ts
    compareLayouts.ts
    compareScenarios.ts

  export/
    exportSummaryCsv.ts
    exportComponentStatsCsv.ts
    exportItemStatsCsv.ts
    exportEventLogCsv.ts
    exportReportMarkdown.ts
    exportCanvasPng.ts

  recommendations/
    buildRuleBasedRecommendations.ts
    recommendationRules.ts
```

---

## 20. TypeScript 타입 초안

```ts
export interface ReportModel {
  reportInfo: ReportInfo;
  summary: SimulationSummary;
  bottleneckReport: BottleneckReport;
  componentReport: ComponentReport;
  itemReport: ItemReport;
  timelineReport?: TimelineReport;
  comparisonReport?: ComparisonReport;
  recommendations: Recommendation[];
}

export interface ReportInfo {
  id: string;
  layoutId: string;
  scenarioId: string;
  resultId: string;
  name: string;
  createdAt: string;
  reportVersion: string;
}

export interface BottleneckReport {
  candidates: BottleneckCandidate[];
  primaryBottleneckId?: string;
  explanation?: string;
}

export interface BottleneckCandidate {
  rank: number;
  componentId: string;
  componentType: string;
  utilization: number;
  waitingTime: number;
  blockedTime: number;
  maxQueueLength?: number;
  score: number;
  reasonCodes: string[];
}

export interface Recommendation {
  id: string;
  severity: "info" | "warning" | "critical";
  targetComponentId?: string;
  title: string;
  description: string;
  reasonCodes: string[];
  suggestedActions: string[];
}
```

---

## 21. MVP 구현 범위

### 포함

```text
- Summary Report
- Bottleneck Ranking
- Component Statistics Table
- Item Statistics Table
- Rule-based Recommendation 기초
- JSON Export
- CSV Export
- Canvas PNG Export
- Markdown Report Export
```

### 선택 포함

```text
- Scenario Comparison 2개 case 비교
- Timeline Bucket Report
- Bottleneck Overlay와 Report 연동
- Component Detail Panel
```

### 제외

```text
- PDF Report 자동 생성
- Excel 고급 서식 export
- AI 개선안 자동 생성
- DXF 자동 인식
- Simulation Sweep 자동 실행
- Enterprise 계정 기능
```

---

## 22. 작업 목록

```text
타입 정의 작업
- src/types/report.ts 생성
- ReportModel 타입 정의
- BottleneckReport 타입 정의
- BottleneckCandidate 타입 정의
- ComponentReport 타입 정의
- ItemReport 타입 정의
- ComparisonReport 타입 정의
- Recommendation 타입 정의
- ExportOption 타입 정의

Summary Report 작업
- SummaryCards 컴포넌트 구현
- simulation summary 표시
- status 표시
- deadlock 여부 표시
- bottleneck 1순위 표시

Bottleneck Report 작업
- calculateBottleneckRanking 구현
- BottleneckTable 구현
- bottleneck score 계산
- reasonCodes 생성
- bottleneck 클릭 시 Animation Player 강조 연동

Component Report 작업
- ComponentStatsTable 구현
- component type 필터 구현
- utilization 정렬
- blockedTime 정렬
- ComponentDetailPanel 구현

Item Report 작업
- ItemStatsTable 구현
- completed / WIP 필터
- itemType 필터
- leadTime 정렬
- waitingTime 정렬

Export 작업
- exportSummaryCsv 구현
- exportComponentStatsCsv 구현
- exportItemStatsCsv 구현
- exportEventLogCsv 구현
- exportReportMarkdown 구현
- exportCanvasPng 구현
- exportJson 구현
```

---

## 23. 완료 기준

```text
- Summary Report 구조가 정의되어 있다.
- Bottleneck Report 계산 기준이 정의되어 있다.
- Component Report 구조가 정의되어 있다.
- Item Report 구조가 정의되어 있다.
- Timeline Report 방향이 정의되어 있다.
- Scenario Comparison 구조가 정의되어 있다.
- Layout Comparison 구조가 정의되어 있다.
- Export 형식과 파일 구조가 정의되어 있다.
- Rule-based Recommendation 기준이 정의되어 있다.
- 고도화 Roadmap이 Phase별로 정의되어 있다.
- TypeScript 타입 초안이 작성되어 있다.
- Codex가 Report와 Export MVP를 구현할 수 있는 수준이다.
```

---

## 24. Codex 작업 지시 예시

```text
MasterPlan.md부터 Plan_10.md까지 읽고,
src/types/report.ts 파일을 만들어줘.

Plan_10.md의 TypeScript 타입 초안을 기준으로 다음 타입을 구현해.
- ReportModel
- ReportInfo
- BottleneckReport
- BottleneckCandidate
- ComponentReport
- ComponentReportRow
- ItemReport
- ItemReportRow
- ItemTypeSummary
- ComparisonReport
- Recommendation
- ExportOption

아직 UI는 만들지 마.
```

---

## 25. 전체 문서 세트 정리

```text
MasterPlan.md
- 전체 전략과 시스템 구조

Plan_01.md
- 프로젝트 범위와 MVP 정의

Plan_02.md
- 컴포넌트 모델 정의

Plan_03.md
- Layout JSON Schema 정의

Plan_04.md
- Three.js Layout Editor 정의

Plan_05.md
- Network Routing 정의

Plan_06.md
- Input Scenario Data 정의

Plan_07.md
- DES Simulation Engine 정의

Plan_08.md
- Event Log Timeline 정의

Plan_09.md
- Three.js Animation Player 정의

Plan_10.md
- Report, Export, Comparison, Roadmap 정의
```

---

## 26. 최종 결론

이 프로젝트는 단순히 Three.js로 움직이는 컨베이어를 만드는 프로젝트가 아니다.

정확한 구조는 다음과 같다.

```text
Layout JSON
- 설비 세계의 지도

Scenario JSON
- 물동량의 시간표

DES Engine
- 사건을 계산하는 두뇌

Event Timeline
- 지나간 사건의 필름

Animation Player
- 필름을 재생하는 화면

Report
- 결과를 판단으로 바꾸는 해석기
```

Plan_10의 역할은 마지막 해석기다.
박스가 어디서 멈췄는지, 로봇이 얼마나 바빴는지, 컨베이어가 왜 막혔는지, 어떤 대안이 더 나은지 보여준다.

이 단계까지 구현되면 프로젝트는 단순한 시각화 도구를 넘어, 웹 기반 물류 설계 검토 도구로 성장할 수 있다.
::: 
