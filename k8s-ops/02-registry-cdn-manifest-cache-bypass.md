# 컨테이너 레지스트리 CDN 매니페스트 캐싱 이슈 대응

## 1. 배경

### 가. 증상

1) QA팀이 동일 태그(`latest`)로 이미지를 재빌드한 뒤 pull했을 때, 새로 추가·변경된 내용이
   반영되지 않는다고 보고

### 나. 원인

1) 컨테이너 레지스트리 앞단에 CDN이 붙어 있는 구조였고, 이 CDN이 manifest 응답을 캐싱하고
   있었음
2) origin(레지스트리)에서는 이미지가 정상적으로 갱신되었지만, 동일 태그로 재요청 시 CDN이
   캐싱된 과거 manifest를 그대로 응답해 클라이언트는 실제로 이전 이미지를 pull하게 되는
   구조였음
3) 운영 방식상 이미지 태그를 `latest` 하나로 고정해서 쓰고 있어 이 문제가 재빌드 때마다
   반복적으로 재현됨

## 2. 기대효과

### 가. 원인 특정을 통한 근본 조치

1) Registry API를 직접 호출해 CDN 응답과 origin 응답의 manifest digest가 다름을 확인,
   문제 지점을 CDN 캐싱으로 명확히 특정한 뒤 조치

### 나. 캐시 퍼지 작업 불필요화

1) 문제가 되는 경로(`/v2/<image>/manifests/latest`)만 CDN 캐시 바이패스로 설정해, 이미지
   재빌드 후 별도의 캐시 퍼지 작업 없이도 항상 최신 이미지를 받을 수 있도록 개선

### 다. AS-IS / TO-BE 비교

| 항목 | AS-IS | TO-BE |
|---|---|---|
| Manifest 응답 | CDN이 캐싱된 과거 manifest 응답 | `/manifests/latest` 경로 캐시 바이패스로 항상 origin 응답 |
| 재빌드 후 반영 | 반영 안 됨 (별도 조치 필요) | 별도 캐시 퍼지 없이 즉시 반영 |
| 문제 재현 범위 | `latest` 태그를 쓰는 모든 이미지 | 바이패스 설정을 전체 이미지에 공통 적용해 재발 방지 |

## 3. 구성도 (조회 흐름, Pull 기준)

```
STEP 1  HEAD /manifests/<tag> → digest 비교
  │
  ├─ 로컬과 동일 → 캐시된 이미지 그대로 사용
  │
  ▼ (다름)
STEP 2  GET /manifests/<manifest_digest> → 새 manifest 다운로드
  │
  ▼
STEP 3  각 레이어에 대해 HEAD /blobs/<layer_digest>로 존재 확인
  │
  ▼
STEP 4  변경된 레이어만 GET /blobs/<layer_digest>로 다운로드 → 압축 해제 → 레이어 결합
```

## 4. 성과

1) 이미지 재빌드 후 origin으로의 불필요한 재요청 없이, CDN 캐시 바이패스만으로 항상 최신
   이미지가 즉시 반영되도록 개선
2) CDN-origin 간 manifest 캐시 불일치로 인한 배포 신뢰도 저하(재빌드했는데 이전 이미지가
   그대로 떠 있는 문제) 재발 방지

---

## 상세 문서

원인 조사에 사용한 Registry API 호출 절차(Bearer 토큰 발급, manifest/tag 조회)와 Docker
Registry V2 API 기준 Pull/Push 전체 동작 방식 정리를 포함한 상세 문서는 비공개 리포지토리에
별도 보관 중입니다. 필요 시 요청해주시면 공유드리겠습니다.
