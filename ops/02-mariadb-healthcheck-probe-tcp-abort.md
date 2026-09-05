# MariaDB Pod Healthcheck Probe 개선 — TCP → Exec 전환

## 1. 배경

### 가. 증상

1) error 로그 파일 용량이 750MB 이상 누적, `Aborted connect` 관련 에러가 13만 건 이상 발생
2) 고객사 문의를 통해 문제를 인지

### 나. 원인

1) Pod의 healthcheck probe가 TCP 방식으로 구성되어 있었음
2) TCP probe는 MySQL 프로토콜 핸드셰이크 없이 TCP 커넥션만 열고 닫는 방식이라, MariaDB 입장에서는
   probe 주기마다 이를 비정상 종료 연결(`Aborted connect`)로 기록
3) probe 주기가 짧아 이 로그가 반복적으로 누적되면서 error 로그 파일 용량이 비정상적으로 커짐

## 2. 기대효과

### 가. 불필요한 에러 로그 제거

1) 실제 MySQL 프로토콜 레벨에서 상태를 확인하는 exec 방식으로 전환해, probe로 인한 비정상
   연결 기록 자체가 발생하지 않도록 개선

### 나. 최소권한 원칙 적용

1) probe라는 저위험 반복 작업에 root 계정을 상시 사용하던 관행을 개선, 전용 최소권한 계정으로 분리

### 다. AS-IS / TO-BE 비교

| 항목 | AS-IS | TO-BE |
|---|---|---|
| Probe 방식 | TCP (핸드셰이크 없이 연결만 열고 닫음) | Exec (`mysqladmin ping`/`status`, 소켓 기반) |
| MariaDB 로그 영향 | probe 주기마다 Aborted connect 기록 누적 | 정상 프로토콜 확인이라 비정상 연결 기록 없음 |
| Probe 인증 계정 | 별도 계정 없음 / 관리성 작업에 root 단일 사용 관행 | probe 전용 최소권한 계정(`stopuser`) 분리 |
| 인증정보 관리 | 해당 없음 | 별도 Secret으로 분리 관리 (애플리케이션 계정과 분리) |
| 적용 범위 | 특정 고객사 이슈 대응 | 전체 MariaDB 배포 표준으로 반영 |

## 3. 조치 흐름

```
STEP 1  고객사 문의로 error 로그 용량 급증(750MB+) 및 Aborted connect(13만 건+) 확인
  │
  ▼
STEP 2  원인 분석 — TCP probe가 핸드셰이크 없이 연결만 열고 닫아 MariaDB가 비정상 종료로 기록
  │
  ▼
STEP 3  Probe 방식을 TCP → Exec(mysqladmin ping/status, 소켓 기반)로 전환
  │
  ▼
STEP 4  Probe 전용 최소권한 계정 신설 + 인증정보를 별도 Secret으로 분리
  │
  ▼
STEP 5  검증 후 전체 MariaDB 배포 표준으로 반영
```

## 4. 성과

1) `Aborted connect` 로그가 완전히 사라짐 (750MB 이상/13만 건 이상 → 0)
2) 특정 고객사 문의로 시작된 조치를 전체 MariaDB 배포 표준으로 반영해 재발 방지
3) probe 인증을 root 단일 계정에서 최소권한 전용 계정으로 분리해 보안성 개선

---

## 상세 문서

Probe 설정값과 Secret 매니페스트 등 세부 구현을 포함한 상세 문서는 비공개 리포지토리에 별도
보관 중입니다. 필요 시 요청해주시면 공유드리겠습니다.
