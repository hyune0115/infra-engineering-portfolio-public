# kube-proxy 모드 IPVS Deprecated 대응 — IPVS에서 iptables로 재전환

## 1. 배경

### 가. 계기

1) Kubernetes v1.35 업그레이드를 준비하며 공식 릴리즈 노트를 검토하던 중, kube-proxy의 IPVS
   모드가 deprecated로 발표된 것을 확인 — 향후 릴리즈에서 제거 예정이며, 공식 권장 대체 모드는
   nftables

### 나. 기존 경과

1) 과거 IPVS가 alpha로 처음 올라왔을 당시, iptables 모드에서는 서비스가 많아질수록 체인/규칙
   수가 급증해 라우팅 정책 전체를 파악하며 모니터링하기 어려웠던 반면, IPVS는 서비스별 라우팅
   테이블을 직접 조회할 수 있어 가시성 확보 목적으로 IPVS로 전환했던 이력이 있음

## 2. 기대효과

### 가. 안정적인 모드로 선제 전환

1) 향후 릴리즈에서의 강제 제거 이전에, 검증되지 않은 신규 모드(nftables) 대신 기존에 운영
   이력이 있는 iptables로 미리 안정적으로 전환

### 나. 업그레이드 점검 프로세스 확립

1) 공식 릴리즈 노트를 업그레이드 전에 사전 검토해 영향받는 설정을 미리 파악하고 대응하는
   절차를 확보

### 다. AS-IS / TO-BE 비교

| 항목 | AS-IS | TO-BE |
|---|---|---|
| kube-proxy 모드 | IPVS | iptables |
| 전환 계기 | (과거) iptables 라우팅 가시성 부족 → IPVS 채택 | (이번) IPVS deprecated 발표 → 검증된 모드로 재전환 |
| 대체 후보 | - | nftables(신규, 검증 부족)는 보류, iptables(운영 이력 있음)로 결정 |
| 점검 방식 | - | 공식 릴리즈 노트 사전 검토 후 대응 |

## 3. 조치 흐름

```
STEP 1  k8s 1.35 릴리즈 노트에서 kube-proxy IPVS deprecated 확인
  │
  ▼
STEP 2  nftables(신규) vs iptables(운영 이력 있음) 검토 → iptables로 결정
  │
  ▼
STEP 3  kube-proxy ConfigMap mode 필드 수정 및 재기동
  │
  ▼
STEP 4  IPVS/iptables 라우팅 테이블 직접 조회로 정상 전환 확인
```

## 4. 성과

1) k8s 1.35의 IPVS deprecated 발표에 선제 대응해 안정적인 모드로 전환 완료
2) 릴리즈 노트 사전 검토 기반의 업그레이드 점검 프로세스 확립

---

## 상세 문서

ConfigMap 변경 세부 내용, 전환 후 검증 절차 등을 포함한 상세 문서는 비공개 리포지토리에 별도
보관 중입니다. 필요 시 요청해주시면 공유드리겠습니다.
