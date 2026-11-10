# 구현 API 목록

- 근거: [flow.md](flow.md) (Phase2, 펌웨어 서명은 #63), Notion `API 설계` 데이터베이스
- 상태: 설계중 (v2)
- 작성일: 2026-10-10
- 수정일: 2026-11-11 (보조 API 구현 반영, #78)
- 구현 추적: [orbit-server#25](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/25)

orbit-server가 구현할 API를 한 곳에 모은 목록이다. API별 요청·응답 형식은 [api-spec.md](api-spec.md)에, 처리 순서는 flow.md의 각 단계에 있다.
Notion과 flow.md가 다르면 flow.md를 기준으로 한다 (§5 참고).

구현 상태 표기

| 표기 | 의미 |
|---|---|
| 완료 | Phase1에서 구현했고 Phase2에서 바뀌는 것이 없음 |
| Phase2 변경 | Phase1에서 구현했지만 Phase2 요구사항을 반영해야 함 (§2) |
| 미구현 | 아직 구현하지 않음 |

## 1. 핵심 흐름 API

OTA 흐름에 반드시 필요한 API. 표의 순서는 데이터가 생기는 순서이며, 구현도 이 순서로 진행한다.

| 단계 | 이름 | Method | Endpoint | 인증 | 성공 | 상태 | 구현 Issue |
|---|---|---|---|---|---|---|---|
| ① | Origin 파일 등록 | POST | `/api/v1/admin/artifacts` | admin-token | 201 | Phase2 변경 | [server#7](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/7) |
| ① | Origin 파일 목록 | GET | `/api/v1/admin/artifacts?model=` | admin-token | 200 | Phase2 변경 | [server#8](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/8) |
| ② | 캠페인 등록 | POST | `/api/v1/admin/campaigns` | admin-token | 201 | 완료 | [server#9](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/9) |
| ③ | 차량 등록 · 토큰 발급 | POST | `/api/v1/vehicles/register` | enrollment-key | 201 (재등록 200) | 완료 | [server#10](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/10) |
| ④ | 체크인 · 대상 판정 | POST | `/api/v1/vehicles/{vehicleId}/check-in` | vehicle-token | 200 | Phase2 변경 | [server#12](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/12) |
| ④ | 체크인 결과 보고 (`lastUpdate`) | - | 체크인 요청에 포함 | vehicle-token | 200 | Phase2 변경 | [server#13](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/13) |
| ⑤ | 매니페스트 요청 · Edge 선택 | GET | `/api/v1/vehicles/{vehicleId}/campaigns/{campaignId}/manifest` | vehicle-token | 200 | Phase2 변경 | [server#14](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/14), [server#38](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/38) |
| ⑧ | 차량 상태 스냅샷 | GET | `/api/v1/dashboard/vehicles` | admin-token | 200 | Phase2 변경 | [server#16](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/16) |
| ⑧ | 차량 상태 스트림 (SSE) | GET | `/api/v1/dashboard/vehicles/stream` | admin-token (fetch 기반 SSE) | 200 | Phase2 변경 | [server#17](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/17) |
| ⑧ | **[Phase2]** CDN Edge 상태 | GET | `/api/v1/dashboard/cdn-edges` | admin-token | 200 | 미구현 | 생성 예정 |

- ⑥ 업데이트 파일 다운로드(`GET {downloadUrl}`)는 CDN Edge가 처리하며 OTA 서버 API가 아니다. OTA 서버는 ⑤에서 가장 가까운 Edge의 URL(URL 서명 `md5`, `expires` 포함)과 `fallbackUrls`를 만든다.
  - Edge는 URL 서명 불일치면 `403`, 만료면 `410`을 준다. 서버 API 오류는 아니지만 차량이 매니페스트를 다시 요청하는 원인이 된다 (flow.md ⑥).
- **[Phase2]** 펌웨어 서명(`signature`, `keyId`)은 관리자가 Vault로 등록 전에 만든다. OTA 서버는 서명하지 않고 등록 때 검증만 하며, 차량은 설치 전에 다시 검증한다 (flow.md §0 펌웨어 서명, ⑦).
- 업데이트 결과 보고는 별도 API 없이 체크인의 `lastUpdate`로 받는다.
- CDN Edge 상태 응답: Edge별 `edgeId`, `name`, `latitude`, `longitude`, `healthy`(요청을 받은 Pod 기준), 배정 차량 수(`vehicle_state.assigned_edge_id` 기준), 캠페인별 다운로드 성공·실패 수(`update_result.edge_id` 기준).

### 공통 인증

API는 아니지만 위 API가 의존하므로 따로 구현한다.

| 이름 | 적용 대상 | 실패 응답 | 상태 | 구현 Issue |
|---|---|---|---|---|
| 차량 토큰 인증 | 체크인, 매니페스트 | `401 UNAUTHORIZED`, `403 VEHICLE_MISMATCH` | 완료 | [server#11](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/11) |
| 관리자 토큰 인증 | `/api/v1/admin/**`, `/api/v1/dashboard/**` | `401` | 완료 | [server#15](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/15) |

### [Phase2] API가 아닌 서버 작업

위 API가 의존하는 서버 내부 작업. API보다 먼저 구현한다.

| 이름 | 내용 | 의존하는 API | 구현 Issue |
|---|---|---|---|
| Edge 목록 설정 | `orbit.cdn.edges` 로딩. `id` 중복, 좌표 범위, `default-edge-id` 누락이면 서버 기동 실패 (flow.md §0) | 매니페스트, CDN Edge 상태 | 생성 예정 |
| Edge 헬스체크 | `{baseUrl}/healthz`를 10초마다 호출, 연속 3회 실패면 선택에서 제외, `orbit_cdn_edge_healthy{edge_id}` 메트릭 | 매니페스트, CDN Edge 상태 | 생성 예정 |
| URL 서명 발급 | `md5 = base64url(MD5("{expires}{path} {secret}"))`, 비밀 값은 Secret으로 주입 | 매니페스트 | [server#38](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/38) |
| 펌웨어 서명 검증 | `orbit.signing.public-keys`(`keyId → 공개 키`) 로딩, 서명 문자열(`orbit-firmware-v1`, `model`, `version`, `sha256`, `fileSize`) 생성, Ed25519 검증 (flow.md §0) | Origin 파일 등록 | 생성 예정 |
| DB 마이그레이션 | `vehicle_state`에 `latitude`, `longitude`, `location_updated_at`, `assigned_edge_id`, `update_result`에 `edge_id`, `artifact`에 `signature`, `key_id`(NOT NULL) 추가, 대시보드 조회 뷰 반영 (flow.md §4) | 체크인, 매니페스트, 대시보드, Origin 파일 등록 | 생성 예정 |
| Origin AWS S3 전환 | S3 업로드에 `ChecksumAlgorithm=SHA256`, 큰 파일은 multipart upload, 자격 증명은 Secret 또는 Vault | Origin 파일 등록 | 생성 예정 |
| 서명·업로드 스크립트 | Vault 서명(`vault:v1:...` → `keyId`, `signature` 분리) 후 등록 API 호출. 관리자 토큰과 Vault 인증은 실행 시 따로 입력 (위치는 flow.md §7 협의) | Origin 파일 등록 | 생성 예정 |

## 2. [Phase2] 기존 API 변경

API 경로와 인증은 그대로이고 요청·응답 필드만 바뀐다. 상세는 flow.md의 각 단계를 따른다.

| API | 변경 | flow.md |
|---|---|---|
| Origin 파일 등록 | 저장 위치가 S3(`firmware/{model}/{version}/firmware.bin`)로 바뀜. 요청에 `signature`(base64, 64바이트), `keyId` 필수 추가, 응답에 `keyId` 추가 | ① |
| Origin 파일 목록 · 상세 | 응답에 `keyId` 추가 (`signature`는 내려주지 않음) | ① |
| 체크인 | 요청에 `location{latitude, longitude}`(선택, 없으면 이전 값 유지), `lastUpdate.edgeId` 추가. `lastUpdate.failureReason`에 `SIGNATURE_INVALID`, `DOWNGRADE_REJECTED` 추가 | ④ |
| 매니페스트 요청 | 응답에 `edge{edgeId, distanceKm}`, `fallbackUrls[{edgeId, url}]`, `urlExpiresAt`, `signature`, `keyId` 추가. `downloadUrl`은 가장 가까운 정상 Edge의 URL 서명이 붙은 URL. `vehicle_state.assigned_edge_id` 기록 | ⑤ |
| 차량 상태 스냅샷 · 스트림 | 차량 필드에 `location{latitude, longitude, updatedAt}`, `assignedEdgeId`, `downloadedEdgeId` 추가 | ⑧ |
| 차량 상세 (보조, §4) | `location`, `assignedEdgeId`, `lastUpdate.edgeId` 추가 (반영 완료, [server#23](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/23)) | ⑧ |

## 3. API별 오류 코드와 요구사항

| API | 오류 | 요구사항 |
|---|---|---|
| Origin 파일 등록 | `409 ARTIFACT_VERSION_EXISTS`, `422 HASH_MISMATCH`, `422 SIGNATURE_INVALID` (**[Phase2]** 서명·`keyId` 누락, 모르는 `keyId`, 검증 실패), `502 ORIGIN_UPLOAD_FAILED` | ORIGIN-REQ-001 / ORIGIN-POL-001 |
| Origin 파일 목록 | - | ORIGIN-REQ-001 |
| 캠페인 등록 | `400 INVALID_INPUT`, `404 ARTIFACT_NOT_FOUND`, `409 ACTIVE_CAMPAIGN_EXISTS` | CAMPAIGN-REQ-001 / CAMPAIGN-POL-001 |
| 차량 등록 · 토큰 발급 | `401 INVALID_ENROLLMENT_KEY` | VEHICLE-REQ-002 / VEHICLE-POL-003 |
| 체크인 · 대상 판정 · 결과 보고 | `400 INVALID_INPUT` (**[Phase2]** 좌표 범위 오류), `401 UNAUTHORIZED`, `403 VEHICLE_MISMATCH` | VEHICLE-REQ-001, UPDATE-REQ-001 / VEHICLE-POL-001, VEHICLE-POL-002, UPDATE-POL-001 |
| 매니페스트 요청 | `404 CAMPAIGN_NOT_FOUND`, `409 NOT_UPDATE_TARGET`, `410 CAMPAIGN_INACTIVE`, `503 NO_AVAILABLE_EDGE` (**[Phase2]** 정상 Edge 없음) | MANIFEST-REQ-001 / MANIFEST-POL-001, DOWNLOAD-REQ-001 / DOWNLOAD-POL-001 |
| 차량 상태 스냅샷 | - | MONITORING-REQ-004 / MONITORING-POL-001, MONITORING-POL-003 |
| 차량 상태 스트림 (SSE) | `400 INVALID_INPUT` (형식이 틀린 `Last-Event-ID`) | MONITORING-REQ-001 / MONITORING-POL-001 |
| **[Phase2]** CDN Edge 상태 | - | - |

## 4. 보조 API

흐름을 돕는 API. Notion에 `[제안]`으로 등록되어 있었고 핵심 흐름 이후에 구현했다. 요청·응답 형식은 [api-spec.md §4](api-spec.md#4-보조-api).

| 이름 | Method | Endpoint | 설명 | 상태 | 구현 Issue |
|---|---|---|---|---|---|
| 캠페인 목록 | GET | `/api/v1/admin/campaigns` | 상태·차종 필터 | 완료 | [server#18](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/18) |
| 캠페인 상세 | GET | `/api/v1/admin/campaigns/{campaignId}` | 대상 조건·기간·상태 | 완료 | [server#19](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/19) |
| 캠페인 비활성화 | PATCH | `/api/v1/admin/campaigns/{campaignId}` | ACTIVE → INACTIVE, 배포 중지. 재활성화 없음 | 완료 | [server#20](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/20) |
| 캠페인 진행 현황 | GET | `/api/v1/admin/campaigns/{campaignId}/stats` | 대상·진행·성공·실패(사유별) 집계 | 완료 | [server#21](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/21) |
| Origin 파일 상세 | GET | `/api/v1/admin/artifacts/{artifactId}` | 해시·경로, **[Phase2]** `keyId` | Phase2 변경 | [server#22](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/22) |
| 차량 상세 | GET | `/api/v1/dashboard/vehicles/{vehicleId}` | 상세 + 마지막 업데이트 결과, **[Phase2]** 위치·배정 Edge·다운로드 Edge | 완료 | [server#23](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/23) |
| 차량 업데이트 이력 | GET | `/api/v1/dashboard/vehicles/{vehicleId}/events` | 최근 50건. 결과 보고마다 `vehicle_event`에 기록 | 완료 | [server#24](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/24) |

헬스체크(`GET /actuator/health`)는 orbit-server 기초 설정([server#5](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/5))에서 구성했으므로 이 목록에서 제외한다.

## 5. Notion과 flow.md의 차이

| 항목 | Notion | 기준 (flow.md) |
|---|---|---|
| Origin 파일 등록 | 버전·크기·해시·서명 저장 | 같음. 서명은 Ed25519 `signature`와 `keyId`로 나눠 저장하고, 서명 대상 형식은 flow.md §0 펌웨어 서명을 따른다 |
| Origin 파일 상세 | 해시·서명·경로 | 해시·경로·`keyId`. `signature`는 저장만 하고 상세 응답에는 넣지 않는다 (매니페스트로만 내려준다) |
| 차량 업데이트 이력 | `vehicle_event` 필요 | `vehicle_event` 추가. 결과 보고마다 한 행 기록, 체크인은 기록하지 않음 ([server#24](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/24)) |

Notion 쪽 설명도 flow.md에 맞춰 수정해야 한다.

## 6. 결정이 필요한 항목

| 항목 | 관련 API | 구현 Issue |
|---|---|---|
| URL 서명 유효 기간, URL 서명 비밀 값 교체 절차 | 매니페스트 요청 | [server#38](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/38) |
| 펌웨어 서명 권한 범위, 서명·업로드 스크립트 위치 | Origin 파일 등록 | 생성 예정 |

결정된 항목

| 항목 | 결정 | 구현 Issue |
|---|---|---|
| 캠페인 비활성화·수정 API 필요 여부 | 비활성화만 둔다. 수정·재활성화는 하지 않고 새로 등록한다 | [server#20](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/20) |
| `vehicle_event` 테이블 설계 | 결과 보고마다 한 행. `(vehicle_id, campaign_id, finished_at)` 중복은 한 번만 기록 | [server#24](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/24) |

Edge 개수·좌표, 헬스체크 주기, `fallback-count`, `location-max-age` 등 Phase2 협의 항목은 [flow.md §7](flow.md#7-협의-필요-항목)을 따른다.
