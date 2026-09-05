# infra-portfolio-summary

인프라/SRE 엔지니어링 프로젝트를 기획서 형태로 간단히 정리한 요약 포트폴리오입니다.
각 문서는 배경 → 기대효과/AS-IS-TO-BE → 구성도 → 성과 순으로 정리되어 있습니다.
(상세 기술 문서는 비공개로 별도 관리 중이며, 필요 시 요청하시면 공유드립니다.)

## Summary

### 핵심 성과

- **GitLab PAT 기반 인증 재설계** — 토큰 유출 시 영향범위를 전체 750개 프로젝트에서 1개
  프로젝트로 축소 ([자세히](cicd/01-gitlab-pat-redesign.md))
- **ArgoCD-GitLab 연쇄 장애 대응** — 일일 장애 발생 고객사 75-80% 감소
  ([자세히](cicd/02-argocd-gitlab-cascading-failure.md))
- **Kubernetes 클러스터 구성 자동화 4-Tier 리팩토링** — 설치 소요시간 약 20% 단축, 전체 신규
  구축 표준으로 적용 ([자세히](iaas/01-k8s-install-automation-refactor.md))
- **대규모 고객사 솔루션 신규 구축 프로젝트 리딩** — 6개 이상 클러스터 규모의 구축 프로젝트를
  설치계획부터 오픈 지원까지 전 주기 단독 리딩, 정상 서비스 오픈으로 완료
  ([자세히](project/01-solution-installation-project-lead.md))

### 기술적 문제 해결 사례

- **네트워크 계층** — 고객사 NAC 장비의 주기적 ARP 갱신과 Ingress의 노드별 개별 MAC 광고가
  충돌해 발생한 서비스 접속 장애를, 패킷 덤프 기반으로 여러 원인 후보를 배제해가며 근본 원인을
  특정하고 MetalLB 도입으로 해결 ([자세히](k8s-ops/01-nac-arp-mac-floating-conflict.md))
- **애플리케이션-인프라 연계 계층** — 대량 reconciliation이 유발한 GitLab 부하 → HTTP 502 →
  git credential 자동 삭제로 이어지는 연쇄 장애 구조를 ArgoCD/GitLab 로그 교차 분석으로 규명
  ([자세히](cicd/02-argocd-gitlab-cascading-failure.md))
- **OS/프로세스 계층** — TCP probe가 MySQL 핸드셰이크 없이 연결을 열고 닫아 발생하던 Aborted
  connect 로그 폭증의 원인을 규명하고, exec 기반 probe로 전환해 근본적으로 해결
  ([자세히](k8s-ops/06-mariadb-healthcheck-probe-tcp-abort.md))

## 목차

### [k8s-ops/](k8s-ops/) - Kubernetes 인프라 운영, 장애대응

- [고객사 NAC 장비 ARP 갱신 × Ingress 개별 IP 광고 충돌 대응](k8s-ops/01-nac-arp-mac-floating-conflict.md)
- [Traefik 서비스 설정 누락으로 인한 클라이언트 IP 미보존](k8s-ops/02-traefik-service-externaltrafficpolicy-missing.md)
- [RHEL9 전환에 따른 cgroup v2 미인식으로 인한 JVM 애플리케이션 OOM](k8s-ops/03-redhat9-cgroupv2-jdk-oom.md)
- [kube-proxy 모드 IPVS Deprecated 대응 — IPVS에서 iptables로 재전환](k8s-ops/04-kubeproxy-mode-ipvs-to-iptables.md)
- [Go 애플리케이션 재기동 시 좀비 프로세스/CPU 부하 대응](k8s-ops/05-go-zombie-process-graceful-shutdown.md)
- [MariaDB Pod Healthcheck Probe 개선 — TCP → Exec 전환](k8s-ops/06-mariadb-healthcheck-probe-tcp-abort.md)

### [cicd/](cicd/) - CI/CD, GitOps

- [GitLab Project Access Token 기반 인증 재설계](cicd/01-gitlab-pat-redesign.md)
- [ArgoCD-GitLab 연쇄 장애 대응](cicd/02-argocd-gitlab-cascading-failure.md)

### [iaas/](iaas/) - IaaS, 구축 자동화

- [Kubernetes 클러스터 구성 자동화 4-Tier 리팩토링](iaas/01-k8s-install-automation-refactor.md)

### [ops/](ops/) - 운영, 장애대응

- [정전/유지보수 대응 서비스 중지·재기동 자동화](ops/01-service-maintenance-automation.md)
- [컨테이너 레지스트리 CDN 매니페스트 캐싱 이슈 대응](ops/02-registry-cdn-manifest-cache-bypass.md)

### [monitoring/](monitoring/) - 모니터링, 관측성

- [Prometheus Stack + Grafana 커스텀 모니터링 대시보드 구축](monitoring/01-grafana-dashboard-custom-build.md)

### [cloud/](cloud/) - 퍼블릭 클라우드, 아키텍처 설계

- [온프레미스 솔루션의 퍼블릭 클라우드 공통 아키텍처 설계](cloud/01-public-cloud-nlb-alb-architecture.md)

### [project/](project/) - 프로젝트 리딩, PM

- [대규모 고객사 솔루션 신규 구축 프로젝트 리딩](project/01-solution-installation-project-lead.md)
