# Kubernetes 클러스터 구성 자동화 4-Tier 리팩토링

TA Unit

## 1. 배경

### 가. 기존 구조의 한계
1) 단일 `install.sh`에서 하나의 대형 Ansible Playbook을 실행하는 구조로, Kubernetes 설치·설정 과정이 하나의 실행 흐름으로 강하게 결합되어 있었음
2) 설치 중 특정 단계에서 실패할 경우 실패 지점과 완료된 작업을 명확하게 구분하기 어려워 전체 Playbook을 처음부터 재실행해야 했음
3) Ansible Task 자체의 idempotency와 별개로, 전체 구축 프로세스 관점에서는 이미 완료된 작업까지 반복 실행되는 문제가 발생
4) 반복 실행 과정에서 불필요한 작업 수행 및 추가 오류가 발생하고, 고객 환경별 커스텀 요구사항이 증가하면서 Playbook의 분기 및 유지보수 복잡도가 지속적으로 증가

## 2. 기대효과

### 가. 재실행 안정성
1) 설치 중간 실패 시 처음부터 재실행하지 않고, 완료된 Task/Role은 Skip하고 실패 지점 이후부터 재개
2) 중간 장애가 발생해도 직전까지의 완료 이력이 State 파일에 보존되어 손실되지 않음

### 나. 구조적 표준화
1) Playbook을 오케스트레이션 전용으로 한정하고, 실제 설치 작업과 상태 관리 로직을 계층별로 분리
2) 장애 발생 지점을 Role/Task 단위로 명확하게 특정 가능

### 다. 커스터마이징 확장성
1) 노드 구성 템플릿을 분리해 All-Master / Master-Worker 분리 / External Load Balancer 등 다양한 토폴로지를 선택적으로 지원
2) 반복적으로 발생하는 요구사항만 표준 템플릿으로 승격해 무분별한 분기 증가 방지

### 라. AS-IS / TO-BE 비교

| 항목 | AS-IS | TO-BE |
|---|---|---|
| 실행 구조 | 단일 install.sh + 단일 대형 Playbook | Initialization / Orchestration / Implementation / State Management 4-Tier 분리 |
| 재실행 시 동작 | 실패 시 처음부터 전체 재실행 | 완료된 Task/Role Skip, 실패 지점 이후부터 재개 |
| 완료 이력 관리 | 없음 (재실행 시 중복 수행) | JSON State 파일에 Merge 방식으로 Checkpoint 기록 |
| 토폴로지 지원 | 표준화된 단일 구성만 지원 | All-Master / Master-Worker 분리 / External LB 등 템플릿 기반 지원 |
| 노드 수별 구성 | 수동 대응 | 1~5대 노드 수에 따라 DB/검색엔진 클러스터링, Workload 배치 자동 분기 |

## 3. 변경 구성도 (Topology)

### 가. AS-IS

```
install.sh 실행
     │
     ▼
단일 대형 Playbook 실행 (Role/Task 구분 없이 순차 진행)
     │
     ├─ 중간 실패 발생 시 → 실패 지점 특정 불가
     │
     ▼
실패 시 처음부터 전체 재실행
     │  (완료된 작업도 중복 재실행 → 시간 낭비, 추가 오류)
     ▼
설치 완료
```

### 나. TO-BE

```
Tier 1  Initialization (install.sh / Inventory)
             │  환경변수·설정 로드, 토폴로지 템플릿 기반 Inventory 생성
             ▼
Tier 2  Orchestration (Playbook)
             │  Role 실행 여부를 State 조회 결과로 판단
             ▼
Tier 3  Implementation (Roles / Tasks)
             │  ├─ 이미 완료된 Task → Skip
             │  └─ 미완료 Task → 실행 → Checkpoint 기록
             ▼
Tier 4  State Management (JSON State 파일)
             │  기존 State + 신규 완료 항목을 Merge 저장 (이력 손실 방지)
             │  Task 완료 누적 → Role 단위 완료 처리 (2단계 Checkpoint)
             ▼
설치 완료 (중단 시 다음 실행에서 실패 지점 이후부터 재개)
```

## 4. State Management 동작 흐름

```
START
  │
  ▼
STEP 1  State 파일 조회 (없으면 빈 상태로 초기화 — 최초 실행 대응)
  │
  ▼
STEP 2  Role 진입 → 하위 Task 순회
  │
  ▼
STEP 3  Task 완료 여부 확인
          ├─ 완료(true) → Skip
          └─ 미완료 → Task 실행 → Checkpoint 기록 (기존 State와 Merge)
  │
  ▼
STEP 4  Role 내 모든 Task 완료 → Role 자체 완료 처리 (2단계 Checkpoint)
  │
  ▼
END  (다음 실행 시 STEP 1부터 재조회 → 완료 항목은 자동 Skip)
```

## 5. 성과

1) 기존 단일 Playbook 기반 구축 구조를 4-Tier 아키텍처로 표준화하고, 설치 프로세스에 State Management를 도입하여 구축 단위의 멱등성 확보
2) 일주일 평균 기준 Kubernetes 설치 소요 시간 약 20% 단축
3) All-Master / Master-Worker 분리 / External Load Balancer 등 다양한 토폴로지를 템플릿 기반으로 지원하도록 확장
4) 자동화 구조가 실제 신규 Kubernetes 구축 표준으로 적용되어 운영에 활용

---

## 상세 문서

State Management 로직의 세부 구현(Checkpoint Merge 방식, Task Dispatcher 조건부 Skip 처리 등)과 노드 수별 커스터마이징 분기 로직을 포함한 상세 문서는 비공개 리포지토리에 별도 보관 중입니다. 필요 시 요청해주시면 공유드리겠습니다.
