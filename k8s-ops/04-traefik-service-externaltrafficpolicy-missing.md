# Traefik 서비스 설정 누락으로 인한 클라이언트 IP 미보존

TA Unit

## 1. 배경

### 가. 증상

1) Kubernetes 버전 업그레이드(1.21.5 → 1.26.5) 과정에서 Traefik helm chart도 함께 업그레이드
2) 업그레이드 이후 "시스템설정 > 시스템통계 > 로그인내역 / 메뉴사용내역" 화면에서 접속자의
   클라이언트 IP가 실제 클라이언트 IP가 아닌 Pod IP로 기록되는 현상 발견

### 나. 원인

1) 새 Traefik helm chart의 기본 서비스 매니페스트에 `externalTrafficPolicy` 설정이 누락되어
   기본값인 `Cluster`로 동작
2) `Cluster` 모드는 트래픽을 받은 노드가 아닌 다른 노드의 파드로도 전달할 수 있는 대신, In/Out
   경로 유지를 위해 kube-proxy가 클라이언트 IP를 노드 IP로 SNAT 처리 — 원본 클라이언트 IP 유실
3) 애플리케이션이 SNAT된 이후의 IP를 그대로 로그인내역에 기록해 Pod IP로 표시되는 결과로 이어짐

## 2. 기대효과

### 가. 근본 원인 해소를 통한 클라이언트 IP 보존

1) 서비스 설정에 `externalTrafficPolicy: Local`을 명시해 SNAT 없이 클라이언트 IP가 그대로
   유지되도록 개선

### 나. 표준화를 통한 재발 방지 확대

1) helm chart 업그레이드 시 서비스 매니페스트의 필수 필드 누락 여부를 diff로 점검하는 절차를
   확보해 이후 chart 버전업 시에도 동일 유형의 설정 누락을 조기에 발견 가능

### 다. AS-IS / TO-BE 비교

| 항목 | AS-IS | TO-BE |
|---|---|---|
| externalTrafficPolicy | 미지정(기본값 Cluster) | Local로 명시 |
| 트래픽 경로 | 임의 노드로 전달, kube-proxy가 SNAT 처리 | 트래픽을 받은 노드의 로컬 파드로만 전달, SNAT 없음 |
| 로그인내역 화면 IP | Pod IP로 기록 | 실제 클라이언트 IP로 기록 |
| 원인 규명 방식 | - | 업그레이드 전/후 서비스 리소스 diff + 공식 문서 검토 |

## 3. 조치 흐름

```
변경 전 — externalTrafficPolicy 미지정(기본값 Cluster)
Client ──▶ Node(임의) ──SNAT(클라이언트 IP → 노드 IP)──▶ 다른 노드의 Pod
                                                          → 로그인내역에 Pod IP 기록

변경 후 — externalTrafficPolicy: Local
Client ──▶ Node(트래픽 수신 노드) ──(SNAT 없음)──▶ 동일 노드의 로컬 Pod
                                                    → 로그인내역에 실제 클라이언트 IP 기록
```

```
STEP 1  로그인내역 화면에서 클라이언트 IP가 Pod IP로 기록되는 현상 발견
  │
  ▼
STEP 2  Traefik 서비스 리소스를 업그레이드 전/후로 diff
  │
  ▼
STEP 3  externalTrafficPolicy 필드가 새 chart 기본값에서 누락된 것을 확인
  │
  ▼
STEP 4  공식 문서로 Local 설정 시 클라이언트 IP 보존 동작 원리 검증
  │
  ▼
STEP 5  externalTrafficPolicy: Local을 helm values에 명시적으로 반영
```

## 4. 성과

1) 로그인내역/메뉴사용내역 화면에서 실제 클라이언트 IP가 정상적으로 표시되도록 이슈 해결
2) helm chart 업그레이드 시 서비스 매니페스트 필드 누락을 diff로 점검하는 절차의 필요성 확인

---

## 상세 문서

원인 규명에 사용한 서비스 리소스 diff 결과 등 세부 내용을 포함한 상세 문서는 비공개 리포지토리에
별도 보관 중입니다. 필요 시 요청해주시면 공유드리겠습니다.
