# Go 애플리케이션 재기동 시 좀비 프로세스/CPU 부하 대응

## 1. 배경

### 가. 증상

1) Go 애플리케이션 컨테이너가 재기동될 때마다 간헐적으로 좀비 프로세스가 남았고, 이 좀비
   프로세스가 계속 누적되면서 CPU I/O wait가 과점유되는 현상이 발생

### 나. 원인

1) 컨테이너의 PID 1이 단순 쉘 스크립트였고, `exec` 없이 Go 애플리케이션을 자식 프로세스로
   fork하는 구조였음
2) PID 1은 자식 프로세스 종료 시 좀비 상태가 된 프로세스를 reap(wait)해줄 책임이 있는데,
   해당 쉘 스크립트에는 이 처리가 없었음
3) 컨테이너 종료 시에도 PID 1이 받은 SIGTERM이 자식 프로세스(Go 앱)로 전파되지 않아,
   애플리케이션이 정상 종료 절차를 타지 못하고 강제 종료되는 경우가 있었음

## 2. 기대효과

### 가. 시그널 전파/좀비 회수 체계화

1) PID 1 역할을 `dumb-init`으로 대체해, 시그널 전파와 좀비 프로세스 reap을 컨테이너
   런타임 레벨에서 책임지도록 구조화

### 나. 애플리케이션 레벨 정상 종료

1) Go 애플리케이션에 SIGTERM 핸들러를 추가해, 강제 종료가 아닌 진행 중인 작업을 마무리하는
   graceful shutdown이 가능해짐

### 다. 종료 유예 시간 확보

1) preStop Hook과 `terminationGracePeriodSeconds` 조정으로, 애플리케이션이 정상 종료를
   끝마칠 시간을 충분히 확보한 뒤에만 강제 종료(SIGKILL)가 발생하도록 구조화

### 라. AS-IS / TO-BE 비교

| 항목 | AS-IS | TO-BE |
|---|---|---|
| PID 1 프로세스 | 단순 쉘 스크립트 (fork만 수행) | `dumb-init` (시그널 전파 + 좀비 reap) |
| 좀비 프로세스 처리 | 없음 (계속 누적) | PID 1이 자동 reap |
| 애플리케이션 종료 처리 | 시그널 핸들러 없음 (강제 종료) | `os/signal` 기반 SIGTERM 핸들러로 graceful shutdown |
| 종료 유예 시간 | 기본값(30초) | preStop Hook + `terminationGracePeriodSeconds: 60`으로 확대 |

## 3. 구성도 (종료 흐름)

```
STEP 1  kubelet이 Pod 종료 요청 → preStop Hook 실행
  │       preStop: 대상 프로세스에 SIGTERM 전송 후 5초 대기
  ▼
STEP 2  dumb-init(PID 1)이 SIGTERM 수신 → 자식 프로세스(Go 앱)에 전파
  ▼
STEP 3  Go 앱의 signal 핸들러가 SIGTERM 수신 → 진행 중인 요청 마무리, 리소스 정리 후 종료
  ▼
STEP 4  dumb-init이 자식 프로세스 종료를 reap (좀비 프로세스로 남지 않음)
  ▼
STEP 5  terminationGracePeriodSeconds(60초) 이내에 종료되면 정상 완료,
        초과 시에만 kubelet이 SIGKILL로 강제 종료
```

## 4. 성과

1) 좀비 프로세스 발생 빈도가 줄어들어 CPU 부하가 완화되고 시스템 성능이 개선됨
2) 컨테이너 재시작 시 발생할 수 있는 예상치 못한 장애를 미연에 방지하여 서비스 가용성 향상

---

## 상세 문서

`dumb-init` 도입 배경, Go 애플리케이션의 SIGTERM 핸들러 구현, preStop Hook 커맨드 등 세부
구현을 포함한 상세 문서는 비공개 리포지토리에 별도 보관 중입니다. 필요 시 요청해주시면
공유드리겠습니다.
