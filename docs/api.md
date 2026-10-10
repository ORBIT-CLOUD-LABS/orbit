# 구현 API 목록

- 근거: [flow.md](flow.md), Notion `API 설계` 데이터베이스
- 상태: 모든 API `설계중` (v1)
- 작성일: 2026-10-10
- 구현 추적: [orbit-server#25](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/25)

orbit-server가 구현할 API를 한 곳에 모은 목록이다. 요청·응답 필드와 처리 순서는 flow.md의 각 단계를 따른다.
Notion과 flow.md가 다르면 flow.md를 기준으로 한다 (§4 참고).

## 1. 핵심 흐름 API

Phase1 OTA 흐름에 반드시 필요한 API. 표의 순서는 데이터가 생기는 순서이며, 구현도 이 순서로 진행한다.

| 단계 | 이름 | Method | Endpoint | 인증 | 성공 | 구현 Issue |
|---|---|---|---|---|---|---|
| ① | Origin 파일 등록 | POST | `/api/v1/admin/artifacts` | admin-token | 201 | [server#7](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/7) |
| ① | Origin 파일 목록 | GET | `/api/v1/admin/artifacts?model=` | admin-token | 200 | [server#8](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/8) |
| ② | 캠페인 등록 | POST | `/api/v1/admin/campaigns` | admin-token | 201 | [server#9](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/9) |
| ③ | 차량 등록 · 토큰 발급 | POST | `/api/v1/vehicles/register` | enrollment-key | 201 (재등록 200) | [server#10](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/10) |
| ④ | 체크인 · 대상 판정 | POST | `/api/v1/vehicles/{vehicleId}/check-in` | vehicle-token | 200 | [server#12](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/12) |
| ④ | 체크인 결과 보고 (`lastUpdate`) | - | 체크인 요청에 포함 | vehicle-token | 200 | [server#13](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/13) |
| ⑤ | 매니페스트 요청 | GET | `/api/v1/vehicles/{vehicleId}/campaigns/{campaignId}/manifest` | vehicle-token | 200 | [server#14](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/14) |
| ⑧ | 차량 상태 스냅샷 | GET | `/api/v1/dashboard/vehicles` | admin-token | 200 | [server#16](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/16) |
| ⑧ | 차량 상태 스트림 (SSE) | GET | `/api/v1/dashboard/vehicles/stream` | 협의 (쿠키 또는 쿼리 토큰) | 200 | [server#17](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/17) |

- ⑥ 업데이트 파일 다운로드(`GET {downloadUrl}`)는 CDN이 처리하며 OTA 서버 API가 아니다. OTA 서버는 ⑤에서 서명 URL만 만든다.
- 업데이트 결과 보고는 별도 API 없이 체크인의 `lastUpdate`로 받는다.

### 공통 인증

API는 아니지만 위 API가 의존하므로 따로 구현한다.

| 이름 | 적용 대상 | 실패 응답 | 구현 Issue |
|---|---|---|---|
| 차량 토큰 인증 | 체크인, 매니페스트 | `401 UNAUTHORIZED`, `403 VEHICLE_MISMATCH` | [server#11](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/11) |
| 관리자 토큰 인증 | `/api/v1/admin/**`, `/api/v1/dashboard/**` | `401` | [server#15](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/15) |

## 2. API별 오류 코드와 요구사항

| API | 오류 | 요구사항 |
|---|---|---|
| Origin 파일 등록 | `409 ARTIFACT_VERSION_EXISTS`, `422 HASH_MISMATCH`, `502 ORIGIN_UPLOAD_FAILED` | ORIGIN-REQ-001 / ORIGIN-POL-001 |
| Origin 파일 목록 | - | ORIGIN-REQ-001 |
| 캠페인 등록 | `400 INVALID_REQUEST`, `404 ARTIFACT_NOT_FOUND`, `409 ACTIVE_CAMPAIGN_EXISTS` | CAMPAIGN-REQ-001 / CAMPAIGN-POL-001 |
| 차량 등록 · 토큰 발급 | `401 INVALID_ENROLLMENT_KEY` | VEHICLE-REQ-002 / VEHICLE-POL-003 |
| 체크인 · 대상 판정 · 결과 보고 | `401 UNAUTHORIZED`, `403 VEHICLE_MISMATCH` | VEHICLE-REQ-001, UPDATE-REQ-001 / VEHICLE-POL-001, VEHICLE-POL-002, UPDATE-POL-001 |
| 매니페스트 요청 | `404 CAMPAIGN_NOT_FOUND`, `409 NOT_UPDATE_TARGET`, `410 CAMPAIGN_INACTIVE` | MANIFEST-REQ-001 / MANIFEST-POL-001, DOWNLOAD-REQ-001 / DOWNLOAD-POL-001 |
| 차량 상태 스냅샷 | - | MONITORING-REQ-004 / MONITORING-POL-001, MONITORING-POL-003 |
| 차량 상태 스트림 (SSE) | - | MONITORING-REQ-001 / MONITORING-POL-001 |

## 3. 보조 API (제안)

흐름을 돕는 API. Notion에 `[제안]`으로 등록되어 있으며 핵심 흐름 이후에 구현한다.

| 이름 | Method | Endpoint | 설명 | 구현 Issue |
|---|---|---|---|---|
| 캠페인 목록 | GET | `/api/v1/admin/campaigns` | 상태·차종 필터 | [server#18](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/18) |
| 캠페인 상세 | GET | `/api/v1/admin/campaigns/{campaignId}` | 대상 조건·기간·상태 | [server#19](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/19) |
| 캠페인 비활성화 | PATCH | `/api/v1/admin/campaigns/{campaignId}` | ACTIVE → INACTIVE, 배포 중지 | [server#20](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/20) |
| 캠페인 진행 현황 | GET | `/api/v1/admin/campaigns/{campaignId}/stats` | 대상·진행·성공·실패(사유별) 집계 | [server#21](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/21) |
| Origin 파일 상세 | GET | `/api/v1/admin/artifacts/{artifactId}` | 해시·경로 | [server#22](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/22) |
| 차량 상세 | GET | `/api/v1/dashboard/vehicles/{vehicleId}` | 상세 + 마지막 업데이트 결과 | [server#23](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/23) |
| 차량 업데이트 이력 | GET | `/api/v1/dashboard/vehicles/{vehicleId}/events` | `vehicle_event` 테이블 필요 | [server#24](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/24) |

헬스체크(`GET /actuator/health`)는 orbit-server 기초 설정([server#5](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/5))에서 구성했으므로 이 목록에서 제외한다.

## 4. Notion과 flow.md의 차이

| 항목 | Notion | 기준 (flow.md) |
|---|---|---|
| Origin 파일 등록 | 버전·크기·해시·서명 저장 | Ed25519 서명은 Phase2. Phase1은 SHA-256만 저장 |
| Origin 파일 상세 | 해시·서명·경로 | 해시·경로 |
| 차량 업데이트 이력 | `vehicle_event` 필요 | 현재 스키마에 `vehicle_event` 없음. 추가하려면 테이블 설계 결정 필요 |

Notion 쪽 설명도 flow.md에 맞춰 수정해야 한다.

## 5. 결정이 필요한 항목

| 항목 | 관련 API | 구현 Issue |
|---|---|---|
| CDN URL 유효 기간, 서명 비밀 값 관리 | 매니페스트 요청 | [server#14](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/14) |
| 관리자 토큰 발급 방식 | 관리자·대시보드 API | [server#15](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/15) |
| SSE 인증 방식, keep-alive 간격 | 차량 상태 스트림 | [server#17](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/17) |
| 캠페인 비활성화·수정 API 필요 여부 | 캠페인 비활성화 | [server#20](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/20) |
| `vehicle_event` 테이블 설계 | 차량 업데이트 이력 | [server#24](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/24) |

그 밖의 협의 항목(차량 토큰 만료, 체크인 주기, 롤백 방식 등)은 [flow.md §7](flow.md#7-협의-필요-항목)을 따른다.
