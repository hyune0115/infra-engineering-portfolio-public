# ArgoCD-GitLab 연쇄 장애 대응

TA Unit · 2025.10

## 1. 배경

### 가. 현상
1) 서비스 업데이트 시 ArgoCD Sync 실패로 인한 자동 업데이트 오류가 반복 발생, 고객사 서비스 배포가 정상 반영되지 않는 문제 발생
2) 하루 단위로 4-5개 고객사에서 관련 장애 발생

### 나. 문제의 어려움
1) 단순 네트워크 일시 장애처럼 보였으나 재시도 설정만으로는 해결되지 않고, 특정 시점 이후 장애가 반복 재현됨
2) 표면적 증상(Sync 실패)과 근본 원인(대량 트래픽 → 인증정보 소실) 사이 거리가 멀어 진단에 다단계 로그 교차 분석이 필요했음

## 2. 장애 전파 구조 (Cascading Failure)

```
STEP 1  수백 개 ArgoCD Application, reconciliation 주기 과도하게 짧음
             │  (+ 존재하지 않는 Repo URL 설정으로 불필요 조회 추가 가중)
             ▼
STEP 2  중앙 GitLab 서버 요청 폭주
             │
             ▼
STEP 3  HTTP 502 응답
             │
             ▼
STEP 4  git ls-remote 가 502 수신 → 인증 없이 재요청 → 서버 재인증 요구
             │
             ▼
STEP 5  credential-store 에서 저장된 인증정보로 재요청
             │  (502 상황을 인증 실패로 오처리)
             ▼
STEP 6  ★ 로컬 credential 파일 자동 삭제(erase) ★
             │
             ▼
STEP 7  이후 모든 Git 인증 지속 실패
             │
             ▼
STEP 8  ArgoCD Sync 장기 중단 → 고객사 서비스 배포 반영 안 됨
```

**핵심 발견**: 단순 "GitLab이 느려서 Sync가 안 됨"이 아니라, 부하로 인한 502 응답이 git 클라이언트의
인증 실패 처리 로직을 잘못 타면서 **인증정보 자체가 삭제**되는 연쇄 구조였음. 이 지점을 규명하지 못하면
재시도 횟수를 아무리 늘려도 credential이 없으니 계속 실패하는 상태였음.

## 3. AS-IS / TO-BE 비교

| 항목 | AS-IS | TO-BE |
|---|---|---|
| Reconciliation 주기 | 초 단위로 매우 짧음, Application 수백 개가 동시 폴링 | Manual Sync 전환 + 폴링 주기 최소화 |
| Repository 설정 | 일부 Application에 존재하지 않는 Repo URL 설정 | 오설정 전수 점검·수정 |
| 실패 시 재시도 | 없음 (또는 미흡) | `syncPolicy.retry`(backoff) + Git remote 재시도 파라미터 적용 |
| Credential 저장 방식 | `store` (파일 영속) — 인증 오류 시 파일 자체가 삭제됨 | `cache` (메모리, TTL) — 삭제 없이 자연 만료만 발생 |
| 불필요 트리거 | GitLab CI가 불필요하게 추가 트리거됨 | 불필요 Pipeline skip 처리 |

## 4. 재현 및 검증
- Repository cache를 의도적으로 비활성화해 Git 원격 조회·fetch가 반복되는 환경을 구성, 장애를 재현
- ArgoCD/GitLab 로그를 상호 대조해 부하 → 502 → 인증 실패 → credential 삭제 → Sync 실패의 흐름을 단계별로 검증
- `argocd sync` / `argocd diff`로 개선 전후 Sync 및 배포 동작을 직접 검증

## 5. 성과
- 일일 4-5개 고객사에서 발생하던 장애를 **일일 1개 고객사 수준으로 감소** (약 75-80% 감소)
- 단순 Retry 설정 추가가 아닌, Git 조회 부하 → HTTP 오류 → credential 소실 → Sync 실패로 이어지는 연쇄 장애 구조를 로그 기반으로 규명하고 개선
- 장애 원인 분석, 재현 환경 구성, 개선안 설계, 테스트 및 운영 적용까지 전 과정 단독 수행

---

## 상세 문서

원인 분석 과정에서의 시행착오와 세부 파라미터(ArgoCD 버전별 reconciliation 설정 등)를 포함한 상세 문서는
비공개 리포지토리에 별도 보관 중입니다. 필요 시 요청해주시면 공유드리겠습니다.
