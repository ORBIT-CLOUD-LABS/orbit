# API 명세 (요청·응답 형식)

- 근거: [flow.md](flow.md) (Phase2), [api.md](api.md) (API 목록·구현 상태), orbit-server `develop` 브랜치의 요청·응답 DTO, 공통 응답 형식([server#82](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/82), [server#83](https://github.com/ORBIT-CLOUD-LABS/orbit-server/pull/83))
- 상태: 설계중 (v2)
- 작성일: 2026-11-02
- 수정일: 2026-11-11 (버전 형식·비교 규칙, #77)

API별 요청·응답 형식을 정리한 문서다. API 목록과 구현 상태, 구현 Issue는 [api.md](api.md)에 둔다.

- Phase1에서 구현한 필드는 서버 코드 그대로 적었다. **[Phase2]** 표시는 아직 구현하지 않았거나 바뀌는 필드다.
- 처리 순서와 이유는 flow.md의 각 단계(①~⑧)를 따른다. 이 문서와 flow.md가 다르면 flow.md가 기준이다.
- 보조 API(§4) 중 미구현인 것은 **[제안]** 형식이다. 구현 Issue에서 확정한다.

## 0. 공통 규칙

### 기본

| 항목 | 규칙 |
|---|---|
| Base URL | `http://{ota-server}` (경로는 모두 `/api/v1`로 시작) |
| 본문 | `application/json` (파일 등록만 `multipart/form-data`, SSE는 `text/event-stream`) |
| 필드 이름 | camelCase |
| 시각 | ISO-8601 UTC 문자열 (예: `2026-10-28T10:00:00Z`) |
| ID | `artifactId`, `campaignId`는 숫자(Long). `vehicleId`는 차량이 정한 문자열 |
| 버전 | 점으로 나눈 숫자만 받는다 (예: `1.0.1`). `v1.0`, `1.0.1-rc1`, `1..0`은 `400 INVALID_INPUT`. 비교는 자리별 숫자 비교이며 빈 자리는 0, 앞자리 0은 무시한다 (`1.0` == `1.0.0` == `01.0.0`). 저장·응답은 입력한 표기 그대로 |
| 응답에서 빠지는 필드 | 값이 없으면 `null`. 예외는 각 API에 적었다 (예: 체크인 `campaignId`) |

### 인증 헤더

| 대상 | 헤더 | 실패 응답 |
|---|---|---|
| 차량 등록 | `Authorization: Bearer {enrollment-key}` | `401 INVALID_ENROLLMENT_KEY` |
| 차량 (체크인, 매니페스트) | `Authorization: Bearer {vehicle-token}` | `401 UNAUTHORIZED`(토큰 없음·폐기·틀림), `403 VEHICLE_MISMATCH`(토큰 차량 ≠ 경로 `vehicleId`) |
| 관리자, 대시보드 | `Authorization: Bearer {admin-token}` | `401 AUTHENTICATION_REQUIRED` |

### 응답 형식

모든 JSON 응답은 성공과 실패 모두 같은 형식으로 감싼다.

```json
{
  "isSuccess": true,
  "code": "OK",
  "message": "요청에 성공했습니다.",
  "data": { "...": "API별 응답 본문" },
  "error": null
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `isSuccess` | boolean | 요청 성공 여부 |
| `code` | string | 성공이면 성공 코드(아래 표), 실패면 오류 코드 |
| `message` | string | 사람이 읽는 설명. 분기는 `code`로 한다 (문구는 바뀔 수 있다) |
| `data` | object | 성공 시 API별 응답 본문. 실패 시 `null` |
| `error` | object | 오류 상세를 위한 자리. 지금은 항상 `null` |

- HTTP 상태는 응답 상태 코드로 전달한다. 본문에 따로 넣지 않는다.
- 이 문서의 API별 응답 필드 표는 `data` 안의 필드를 설명한다.
- 대시보드 SSE 스트림(§3.2, `text/event-stream`)은 감싸지 않는다. CDN Edge 파일 다운로드(§2.4)도 OTA 서버 응답이 아니라 해당하지 않는다.

성공 코드

| HTTP | code | message |
|---|---|---|
| 200 | `OK` | 요청에 성공했습니다. |
| 201 | `CREATED` | 리소스를 생성했습니다. |

### 오류 응답

오류도 같은 형식이다. `isSuccess`는 `false`, `data`와 `error`는 `null`이다.

```json
{
  "isSuccess": false,
  "code": "ARTIFACT_VERSION_EXISTS",
  "message": "같은 차종과 버전의 파일이 이미 등록되어 있습니다.",
  "data": null,
  "error": null
}
```

- 인증·인가 실패(401, 403)도 같은 형식이다.

모든 API에 공통으로 생길 수 있는 오류

| HTTP | code | 상황 |
|---|---|---|
| 400 | `INVALID_INPUT` | 필수 값 누락, 형식 오류, 정의되지 않은 enum 값 |
| 404 | `RESOURCE_NOT_FOUND` | 없는 경로 |
| 405 | `METHOD_NOT_ALLOWED` | 지원하지 않는 메서드 |
| 406 | `NOT_ACCEPTABLE` | 응답 미디어 타입을 맞출 수 없음 |
| 413 | `CONTENT_TOO_LARGE` | 요청 크기 초과 (파일 등록 최대 1GB) |
| 415 | `UNSUPPORTED_MEDIA_TYPE` | 지원하지 않는 `Content-Type` |
| 500 | `INTERNAL_SERVER_ERROR` | 서버 내부 오류 |

## 1. 관리자 API

### 1.1 Origin 파일 등록 (①)

`POST /api/v1/admin/artifacts` · `multipart/form-data` · admin-token · 성공 `201`

요청 (form 필드)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `file` | 파일 | O | 업데이트 파일 |
| `model` | string | O | 차종. `/` 불가 |
| `version` | string | O | 버전 (§0 버전 형식). 등록한 문자열 그대로 저장 경로와 서명 대상에 들어간다 |
| `sha256` | string | X | 관리자가 계산한 SHA-256 (hex 64자, 대소문자 무관). 있으면 서버 계산값과 비교 |
| **[Phase2]** `signature` | string | O | 펌웨어 서명 (base64, 64바이트). Vault 출력 `vault:v1:<base64>`의 `<base64>` 부분 |
| **[Phase2]** `keyId` | string | O | 서명 키 버전 (예: `v1`). Vault 출력의 `v1` 부분 |

```bash
curl -X POST http://{ota-server}/api/v1/admin/artifacts \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -F file=@firmware.bin \
  -F model=MODEL-A \
  -F version=1.1.0 \
  -F sha256=a80ed4b4... \
  -F signature=Kx8f2... \
  -F keyId=v1
```

응답 `201`

```json
{
  "isSuccess": true,
  "code": "CREATED",
  "message": "리소스를 생성했습니다.",
  "data": {
    "artifactId": 12,
    "fileSize": 20971520,
    "sha256": "a80ed4b4...",
    "path": "/firmware/MODEL-A/1.1.0/firmware.bin",
    "keyId": "v1"
  },
  "error": null
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `artifactId` | number | 파일 ID. 캠페인 등록에 쓴다 |
| `fileSize` | number | 바이트 |
| `sha256` | string | 서버가 계산한 SHA-256 (소문자 hex) |
| `path` | string | URL 경로. S3 키는 앞의 `/`를 뺀 값 |
| **[Phase2]** `keyId` | string | 검증에 쓴 키 버전 |

오류

| HTTP | code | 상황 |
|---|---|---|
| 400 | `INVALID_INPUT` | `file`·`model`·`version` 누락, `version` 형식 오류, `sha256` 형식 오류, `model`에 `/` |
| 409 | `ARTIFACT_VERSION_EXISTS` | 같은 `model`에 비교 규칙상 같은 `version`이 이미 있음 (`1.0`이 있으면 `1.0.0`도 거부) |
| 422 | `HASH_MISMATCH` | 요청 `sha256` ≠ 서버 계산값 |
| 422 | **[Phase2]** `SIGNATURE_INVALID` | `signature`·`keyId` 누락, 모르는 `keyId`, 서명 검증 실패 |
| 502 | `ORIGIN_UPLOAD_FAILED` | S3 업로드 실패·타임아웃 (재시도 가능) |

### 1.2 Origin 파일 목록 (①)

`GET /api/v1/admin/artifacts?model={model}` · admin-token · 성공 `200`

| 쿼리 | 필수 | 설명 |
|---|---|---|
| `model` | X | 차종 필터. 없으면 전체 |

응답 `200`

```json
{
  "isSuccess": true,
  "code": "OK",
  "message": "요청에 성공했습니다.",
  "data": {
    "artifacts": [
      {
        "artifactId": 12,
        "model": "MODEL-A",
        "version": "1.1.0",
        "fileSize": 20971520,
        "sha256": "a80ed4b4...",
        "path": "/firmware/MODEL-A/1.1.0/firmware.bin",
        "keyId": "v1",
        "createdAt": "2026-10-28T09:00:00Z"
      }
    ]
  },
  "error": null
}
```

- **[Phase2]** `keyId` 추가. `signature`는 넣지 않는다 (매니페스트로만 내려준다).

### 1.3 Origin 파일 상세 (보조)

`GET /api/v1/admin/artifacts/{artifactId}` · admin-token · 성공 `200`

응답 `200`: `data`는 목록의 항목 하나와 같은 형식 (**[Phase2]** `keyId` 포함, `signature` 없음).

```json
{
  "isSuccess": true,
  "code": "OK",
  "message": "요청에 성공했습니다.",
  "data": {
    "artifactId": 12,
    "model": "MODEL-A",
    "version": "1.1.0",
    "fileSize": 20971520,
    "sha256": "a80ed4b4...",
    "path": "/firmware/MODEL-A/1.1.0/firmware.bin",
    "keyId": "v1",
    "createdAt": "2026-10-28T09:00:00Z"
  },
  "error": null
}
```

오류: `404 ARTIFACT_NOT_FOUND`, `400 INVALID_INPUT`(`artifactId`가 숫자가 아님)

### 1.4 캠페인 등록 (②)

`POST /api/v1/admin/campaigns` · admin-token · 성공 `201`

요청

```json
{
  "name": "MODEL-A 1.1.0 배포",
  "artifactId": 12,
  "targetVersion": "1.1.0",
  "target": {
    "model": "MODEL-A",
    "hwVersions": ["HW1", "HW2"],
    "regions": ["KR"],
    "currentVersionMin": "1.0.0",
    "currentVersionMax": "1.0.9"
  },
  "startAt": "2026-10-28T00:00:00Z",
  "endAt": "2026-11-28T00:00:00Z"
}
```

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `name` | string | O | 캠페인 이름 |
| `artifactId` | number | O | 배포할 파일 |
| `targetVersion` | string | O | 버전 (§0 버전 형식). 비교 규칙상 `artifact.version`과 같아야 함 (파일 `1.1.0`에 `1.1.0.0` 가능) |
| `target.model` | string | O | `artifact.model`과 같아야 함 |
| `target.hwVersions` | string[] | X | 없거나 비면 모든 HW 버전 |
| `target.regions` | string[] | X | 배포 대상 지역(등록 속성). 차량 위치(좌표)와 다르다. 없거나 비면 모든 지역 |
| `target.currentVersionMin` | string | X | 현재 버전 하한(포함). 버전 형식. 생략하거나 비우면 제한 없음 |
| `target.currentVersionMax` | string | X | 현재 버전 상한(포함). 버전 형식. 생략하거나 비우면 제한 없음 |
| `startAt`, `endAt` | string | O | 적용 기간. `startAt < endAt` |

응답 `201`

```json
{
  "isSuccess": true,
  "code": "CREATED",
  "message": "리소스를 생성했습니다.",
  "data": { "campaignId": 7 },
  "error": null
}
```

오류

| HTTP | code | 상황 |
|---|---|---|
| 400 | `INVALID_INPUT` | 필수 값 누락, `targetVersion`·`target.currentVersionMin`·`target.currentVersionMax` 형식 오류, `target.model ≠ artifact.model`, 비교 규칙상 `targetVersion ≠ artifact.version`, `startAt ≥ endAt` |
| 404 | `ARTIFACT_NOT_FOUND` | 없는 `artifactId` |
| 409 | `ACTIVE_CAMPAIGN_EXISTS` | 같은 대상 조건 + 목표 버전의 활성 캠페인이 있음 |

### 1.5 캠페인 목록 (보조)

`GET /api/v1/admin/campaigns?status={status}&model={model}` · admin-token · 성공 `200`

| 쿼리 | 필수 | 설명 |
|---|---|---|
| `status` | X | `ACTIVE` \| `INACTIVE`. 없으면 전체. 그 밖의 값은 `400 INVALID_INPUT` |
| `model` | X | 차종 필터 |

응답 `200`

```json
{
  "isSuccess": true,
  "code": "OK",
  "message": "요청에 성공했습니다.",
  "data": {
    "campaigns": [
      {
        "campaignId": 7,
        "name": "MODEL-A 1.1.0 배포",
        "model": "MODEL-A",
        "targetVersion": "1.1.0",
        "status": "ACTIVE",
        "startAt": "2026-10-28T00:00:00Z",
        "endAt": "2026-11-28T00:00:00Z",
        "createdAt": "2026-10-27T09:00:00Z"
      }
    ]
  },
  "error": null
}
```

## 2. 차량 API

### 2.1 차량 등록 · 토큰 발급 (③)

`POST /api/v1/vehicles/register` · enrollment-key · 성공 `201` (재등록 `200`)

요청

```json
{
  "vehicleId": "VH-0001",
  "model": "MODEL-A",
  "hwVersion": "HW1",
  "region": "KR",
  "currentVersion": "1.0.0"
}
```

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `vehicleId` | string | O | 차량 식별자 |
| `model` | string | O | 차종. 차량이 펌웨어 서명을 검증할 때 이 값을 쓴다 (⑦) |
| `hwVersion` | string | O | HW 버전 |
| `region` | string | O | 판매·등록 지역 (예: `KR`). 위치(좌표)는 체크인 때 받는다 |
| `currentVersion` | string | O | 현재 설치된 버전 (§0 버전 형식) |

응답 `201` / `200`

```json
{
  "isSuccess": true,
  "code": "CREATED",
  "message": "리소스를 생성했습니다.",
  "data": {
    "vehicleId": "VH-0001",
    "vehicleToken": "vt_...",
    "issuedAt": "2026-10-28T09:00:00Z"
  },
  "error": null
}
```

- 처음 등록은 `201 CREATED`, 재등록은 `200 OK`다. `data` 형식은 같다.

- `vehicleToken` 원문은 이 응답에서 한 번만 내려준다. 이전 토큰은 폐기된다.

오류: `400 INVALID_INPUT` (필수 값 누락, `currentVersion` 형식 오류), `401 INVALID_ENROLLMENT_KEY` (차량은 재시도하지 않고 시작 실패로 종료)

### 2.2 체크인 · 대상 판정 · 결과 보고 (④)

`POST /api/v1/vehicles/{vehicleId}/check-in` · vehicle-token · 성공 `200`

요청

```json
{
  "currentVersion": "1.0.0",
  "location": { "latitude": 37.5665, "longitude": 126.9780 },
  "lastUpdate": {
    "campaignId": 7,
    "result": "FAILED",
    "failureReason": "SIGNATURE_INVALID",
    "failureDetail": "unknown keyId: v2",
    "edgeId": "edge-seoul",
    "finishedAt": "2026-10-28T10:00:00Z"
  }
}
```

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `currentVersion` | string | O | 현재 설치된 버전 (§0 버전 형식) |
| **[Phase2]** `location` | object | X | GPS 미수신이면 생략. 생략하면 서버는 이전 위치를 유지 |
| **[Phase2]** `location.latitude` | number | O (`location`이 있으면) | `-90 ~ 90` |
| **[Phase2]** `location.longitude` | number | O (`location`이 있으면) | `-180 ~ 180` |
| `lastUpdate` | object | X | 보고할 업데이트 결과가 있을 때만. 보고가 성공하면 차량은 지운다 |
| `lastUpdate.campaignId` | number | O | 결과를 보고할 캠페인 |
| `lastUpdate.result` | string | O | `SUCCEEDED` \| `FAILED` |
| `lastUpdate.failureReason` | string | X | 실패 사유 (아래 표). 성공이면 `null` |
| `lastUpdate.failureDetail` | string | X | 실패 상세. `failureReason` 없이 보낼 수 없음 |
| **[Phase2]** `lastUpdate.edgeId` | string | X | 실제로 파일을 받은 Edge. 다운로드 전에 실패했으면 `null` |
| `lastUpdate.finishedAt` | string | O | 끝난 시각 (ISO-8601) |

`failureReason`

| 값 | 상황 |
|---|---|
| `DOWNLOAD_FAILED` | 모든 Edge 실패, 크기 불일치, 재시도 초과 |
| `HASH_MISMATCH` | SHA-256 불일치 |
| **[Phase2]** `SIGNATURE_INVALID` | 펌웨어 서명 검증 실패, 모르는 `keyId` |
| **[Phase2]** `DOWNGRADE_REJECTED` | `targetVersion`이 현재 버전보다 낮거나 읽을 수 없음 |
| `INSTALL_FAILED` | 설치 중 실패 |

응답 `200`

```json
{
  "isSuccess": true,
  "code": "OK",
  "message": "요청에 성공했습니다.",
  "data": { "updateTarget": false, "nextCheckInSeconds": 30 },
  "error": null
}
```

```json
{
  "isSuccess": true,
  "code": "OK",
  "message": "요청에 성공했습니다.",
  "data": { "updateTarget": true, "campaignId": 7, "nextCheckInSeconds": 30 },
  "error": null
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `updateTarget` | boolean | 업데이트 대상 여부 |
| `campaignId` | number | 대상일 때만 들어 있다 (대상이 아니면 필드 자체가 없음) |
| `nextCheckInSeconds` | number | 다음 체크인까지 대기 초 |

- `lastUpdate`는 `(vehicleId, campaignId)` 기준으로 덮어쓴다(멱등). 같은 결과를 여러 번 보내도 된다.
- `lastUpdate`의 형식이 틀리거나(값 누락, 정의되지 않은 enum, 시각 형식 오류) 없는 캠페인이면 서버는 그 보고만 버리고 로그를 남긴다. 체크인 자체는 `200`이다.
- 현재 버전이 캠페인 목표 버전 이상이면(비교 규칙 기준) 대상이 아니다. 차량이 `DOWNGRADE_REJECTED`로 거부할 파일을 체크인마다 내려주지 않기 위해서다.
- 위치는 대상 판정에 쓰지 않는다. 매니페스트의 Edge 선택에만 쓴다.

오류

| HTTP | code | 상황 | 차량 동작 |
|---|---|---|---|
| 400 | `INVALID_INPUT` | `currentVersion` 누락·형식 오류, **[Phase2]** 좌표 범위 오류 | 차량 버그. 로그 남기고 다음 주기 |
| 401 | `UNAUTHORIZED` | 토큰 없음·폐기·틀림 | 등록부터 다시 |
| 403 | `VEHICLE_MISMATCH` | 토큰 차량 ≠ 경로 `vehicleId` | 설정 오류. 재등록으로 해결되지 않음 |
| 500 | `INTERNAL_SERVER_ERROR` | 서버 오류 | 다음 주기에 재시도 |

### 2.3 매니페스트 요청 · Edge 선택 (⑤)

`GET /api/v1/vehicles/{vehicleId}/campaigns/{campaignId}/manifest` · vehicle-token · 성공 `200`

요청 본문 없음. 경로의 `campaignId`는 체크인 응답의 값.

응답 `200`

```json
{
  "isSuccess": true,
  "code": "OK",
  "message": "요청에 성공했습니다.",
  "data": {
    "campaignId": 7,
    "targetVersion": "1.1.0",
    "fileSize": 20971520,
    "sha256": "a80ed4b4...",
    "edge": { "edgeId": "edge-seoul", "distanceKm": 3.2 },
    "downloadUrl": "http://192.168.100.102:30081/firmware/MODEL-A/1.1.0/firmware.bin?md5=Xb3kQ9...&expires=1761649200",
    "fallbackUrls": [
      { "edgeId": "edge-daejeon", "url": "http://192.168.100.102:30082/firmware/MODEL-A/1.1.0/firmware.bin?md5=Xb3kQ9...&expires=1761649200" },
      { "edgeId": "edge-busan", "url": "http://192.168.100.102:30083/firmware/MODEL-A/1.1.0/firmware.bin?md5=Xb3kQ9...&expires=1761649200" }
    ],
    "urlExpiresAt": "2026-10-28T11:00:00Z",
    "signature": "Kx8f2...",
    "keyId": "v1"
  },
  "error": null
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `campaignId` | number | 캠페인 ID |
| `targetVersion` | string | 설치할 버전. 차량은 다운그레이드 판정과 서명 문자열에 쓴다 |
| `fileSize` | number | 바이트 |
| `sha256` | string | 소문자 hex 64자 |
| **[Phase2]** `edge` | object | 고른 Edge. `edgeId`, `distanceKm`(위치를 모르면 `null`) |
| `downloadUrl` | string | 가장 가까운 정상 Edge의 URL. **[Phase2]** URL 서명(`md5`, `expires`) 포함 |
| **[Phase2]** `fallbackUrls` | object[] | 다음으로 가까운 정상 Edge들(최대 `fallback-count`). 없으면 빈 배열 |
| **[Phase2]** `urlExpiresAt` | string | URL 서명 만료 시각. 지났으면 다운로드 전에 매니페스트를 다시 요청 |
| **[Phase2]** `signature` | string | 펌웨어 서명 (base64, 64바이트). 요청마다 바뀌지 않는다 |
| **[Phase2]** `keyId` | string | 서명 키 버전. 차량은 이 값으로 내장 공개 키를 고른다 |

오류 (판정 순서대로)

| HTTP | code | 상황 | 차량 동작 |
|---|---|---|---|
| 401 | `UNAUTHORIZED` | 토큰 오류 | 등록부터 다시 |
| 403 | `VEHICLE_MISMATCH` | 토큰 차량 ≠ 경로 `vehicleId` | 설정 오류 |
| 404 | `CAMPAIGN_NOT_FOUND` | 없는 캠페인 | 체크인으로 돌아감 |
| 410 | `CAMPAIGN_INACTIVE` | 비활성이거나 적용 기간이 아님 | 체크인으로 돌아감 |
| 409 | `NOT_UPDATE_TARGET` | 이 차량이 대상이 아님 (현재 버전이 목표 버전 이상인 경우 포함) | 체크인으로 돌아감 |
| 503 | **[Phase2]** `NO_AVAILABLE_EDGE` | 정상 Edge가 하나도 없음 | 다음 체크인에 다시 시도 |

### 2.4 업데이트 파일 다운로드 (⑥, OTA 서버 API 아님)

`GET {downloadUrl}` · CDN Edge가 처리 · 인증 헤더 없음 (**[Phase2]** URL 서명으로 검증)

- 응답 `200`, `application/octet-stream`, 헤더 `X-Cache-Status: HIT | MISS`.
- 오류는 nginx 응답이며 위 공통 응답 형식(JSON)이 아니다. 상태 코드로만 판단한다.

| HTTP | 상황 | 차량 동작 |
|---|---|---|
| 403 | **[Phase2]** URL 서명 불일치 | 매니페스트를 다시 요청 (1회) |
| 410 | **[Phase2]** URL 만료 | 매니페스트를 다시 요청 (1회) |
| 404 | 파일 없음 | `DOWNLOAD_FAILED` |
| 502, 504 | Edge → S3 오류 | 같은 Edge에 재시도 |
| 연결 실패·타임아웃, 5xx 재시도 초과 | Edge 장애 | **[Phase2]** `fallbackUrls`의 다음 Edge |

## 3. 대시보드 API

### 차량 상태 객체

스냅샷, 스트림, 차량 상세가 같은 차량 객체를 쓴다. 공통 응답 형식으로 감싸기 전의 객체 하나다.

```json
{
  "vehicleId": "VH-0001",
  "model": "MODEL-A",
  "currentVersion": "1.0.0",
  "lastSeenAt": "2026-10-28T10:00:30Z",
  "updatedAt": "2026-10-28T10:00:30Z",
  "campaignId": 7,
  "lastResult": "FAILED",
  "failureReason": "SIGNATURE_INVALID",
  "location": { "latitude": 37.5665, "longitude": 126.9780, "updatedAt": "2026-10-28T10:00:30Z" },
  "assignedEdgeId": "edge-seoul",
  "downloadedEdgeId": "edge-seoul"
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `vehicleId` | string | 차량 식별자 |
| `model` | string | 차종 |
| `currentVersion` | string | 마지막 체크인 버전 |
| `lastSeenAt` | string | 마지막 체크인 시각 |
| `updatedAt` | string | 상태가 마지막으로 바뀐 시각. 같은 차량은 이 값이 늦은 쪽을 쓴다 |
| `campaignId` | number | 최근 업데이트 결과의 캠페인. 결과가 없으면 `null` |
| `lastResult` | string | `SUCCEEDED` \| `FAILED` \| `null` |
| `failureReason` | string | 실패 사유 \| `null` |
| **[Phase2]** `location` | object | 마지막 위치. 받은 적 없으면 `null` |
| **[Phase2]** `assignedEdgeId` | string | 매니페스트에서 배정한 Edge \| `null` |
| **[Phase2]** `downloadedEdgeId` | string | 최근 결과 보고의 실제 다운로드 Edge \| `null` |

### 3.1 차량 상태 스냅샷 (⑧)

`GET /api/v1/dashboard/vehicles` · admin-token · 성공 `200`

```json
{
  "isSuccess": true,
  "code": "OK",
  "message": "요청에 성공했습니다.",
  "data": {
    "asOf": "2026-10-28T10:00:31.123Z",
    "vehicles": [ { "vehicleId": "VH-0001", "...": "차량 상태 객체" } ]
  },
  "error": null
}
```

- `asOf`: 조회 기준 시각(DB 시각). `vehicleId` 순으로 정렬.
- 대시보드는 SSE를 먼저 연결한 뒤 스냅샷을 받는다.

### 3.2 차량 상태 스트림 (SSE, ⑧)

`GET /api/v1/dashboard/vehicles/stream` · admin-token · `text/event-stream` · 성공 `200`

| 요청 헤더 | 필수 | 설명 |
|---|---|---|
| `Authorization` | O | `Bearer {admin-token}`. `EventSource`는 헤더를 못 붙이므로 fetch 기반 SSE로 연결 |
| `Last-Event-ID` | X | 마지막으로 받은 이벤트 `id`. 있으면 그 뒤 변경분부터 먼저 보낸다 |

이벤트

```text
event: vehicles
id: 2026-10-28T10:00:31.123Z
data: [{"vehicleId":"VH-0001","model":"MODEL-A", ... }]

: keep-alive
```

- SSE 이벤트는 공통 응답 형식으로 감싸지 않는다. 연결 전에 실패한 오류 응답(아래)만 공통 형식이다.
- `event: vehicles`의 `data`는 바뀐 차량 상태 객체 배열이다 (전체가 아니라 변경분).
- `id`는 조회 기준 시각이다. 재연결할 때 `Last-Event-ID`로 보낸다.
- 같은 변경이 두 번 올 수 있다. 차량별로 `updatedAt`이 늦은 쪽을 쓴다.
- `: keep-alive` 주석은 연결 유지용이다. 무시한다.

오류: `400 INVALID_INPUT` (`Last-Event-ID`가 ISO-8601 시각이 아님), `401 AUTHENTICATION_REQUIRED`

### 3.3 **[Phase2]** CDN Edge 상태

`GET /api/v1/dashboard/cdn-edges` · admin-token · 성공 `200` · 미구현

```json
{
  "isSuccess": true,
  "code": "OK",
  "message": "요청에 성공했습니다.",
  "data": {
    "edges": [
      {
        "edgeId": "edge-seoul",
        "name": "서울",
        "latitude": 37.5665,
        "longitude": 126.9780,
        "healthy": true,
        "assignedVehicles": 42,
        "downloads": [
          { "campaignId": 7, "succeeded": 40, "failed": 2 }
        ]
      }
    ]
  },
  "error": null
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `edgeId`, `name`, `latitude`, `longitude` | | 설정의 Edge 목록 값. 설정 순서대로 |
| `healthy` | boolean | 이 요청을 받은 서버 Pod의 헬스체크 결과 (Pod마다 잠깐 다를 수 있음) |
| `assignedVehicles` | number | `vehicle_state.assigned_edge_id`가 이 Edge인 차량 수 |
| `downloads` | object[] | 캠페인별 `update_result.edge_id` 기준 성공(`succeeded`)·실패(`failed`) 수 |

- 필드 이름(`assignedVehicles`, `downloads`)은 **[제안]** 이다. 구현 Issue에서 확정한다.
- `baseUrl`은 내부 주소라 응답에 넣지 않는다 (**[제안]**).

## 4. 보조 API

### 4.1 캠페인 상세 — **[제안]**, 미구현

`GET /api/v1/admin/campaigns/{campaignId}` · admin-token · 성공 `200`

```json
{
  "isSuccess": true,
  "code": "OK",
  "message": "요청에 성공했습니다.",
  "data": {
    "campaignId": 7,
    "name": "MODEL-A 1.1.0 배포",
    "artifactId": 12,
    "targetVersion": "1.1.0",
    "target": {
      "model": "MODEL-A",
      "hwVersions": ["HW1", "HW2"],
      "regions": ["KR"],
      "currentVersionMin": "1.0.0",
      "currentVersionMax": "1.0.9"
    },
    "status": "ACTIVE",
    "startAt": "2026-10-28T00:00:00Z",
    "endAt": "2026-11-28T00:00:00Z",
    "createdAt": "2026-10-27T09:00:00Z"
  },
  "error": null
}
```

- `data`는 등록 요청과 같은 모양에 `campaignId`, `status`, `createdAt`을 더한다.
- 오류: `404 CAMPAIGN_NOT_FOUND`

### 4.2 캠페인 비활성화 — **[제안]**, 미구현

`PATCH /api/v1/admin/campaigns/{campaignId}` · admin-token · 성공 `200`

요청

```json
{ "status": "INACTIVE" }
```

- 응답은 캠페인 상세와 같은 형식 (`data`에 캠페인 상세).
- `INACTIVE → ACTIVE`를 허용할지는 [server#20](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/20)에서 정한다.
- 오류: `400 INVALID_INPUT`(정의되지 않은 `status`), `404 CAMPAIGN_NOT_FOUND`

### 4.3 캠페인 진행 현황 — **[제안]**, 미구현

`GET /api/v1/admin/campaigns/{campaignId}/stats` · admin-token · 성공 `200`

```json
{
  "isSuccess": true,
  "code": "OK",
  "message": "요청에 성공했습니다.",
  "data": {
    "campaignId": 7,
    "targetVehicles": 100,
    "succeeded": 80,
    "failed": 5,
    "failureReasons": {
      "DOWNLOAD_FAILED": 2,
      "HASH_MISMATCH": 0,
      "SIGNATURE_INVALID": 1,
      "DOWNGRADE_REJECTED": 0,
      "INSTALL_FAILED": 2
    },
    "inProgress": 15
  },
  "error": null
}
```

- `inProgress` = `targetVehicles - succeeded - failed`. "진행 중"의 기준(대상 판정만 받은 차량 포함 여부)은 [server#21](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/21)에서 정한다.
- 오류: `404 CAMPAIGN_NOT_FOUND`

### 4.4 차량 상세 — **[제안]**, 미구현

`GET /api/v1/dashboard/vehicles/{vehicleId}` · admin-token · 성공 `200`

```json
{
  "isSuccess": true,
  "code": "OK",
  "message": "요청에 성공했습니다.",
  "data": {
    "vehicleId": "VH-0001",
    "model": "MODEL-A",
    "hwVersion": "HW1",
    "region": "KR",
    "currentVersion": "1.0.0",
    "lastSeenAt": "2026-10-28T10:00:30Z",
    "updatedAt": "2026-10-28T10:00:30Z",
    "location": { "latitude": 37.5665, "longitude": 126.9780, "updatedAt": "2026-10-28T10:00:30Z" },
    "assignedEdgeId": "edge-seoul",
    "lastUpdate": {
      "campaignId": 7,
      "result": "FAILED",
      "failureReason": "SIGNATURE_INVALID",
      "failureDetail": "unknown keyId: v2",
      "edgeId": "edge-seoul",
      "finishedAt": "2026-10-28T10:00:00Z"
    }
  },
  "error": null
}
```

- `data`는 차량 상태 객체에 `hwVersion`, `region`과 마지막 결과의 `failureDetail`, `finishedAt`을 더한다. 결과가 없으면 `lastUpdate`는 `null`.
- 오류: `404` (code는 구현 Issue에서 정한다. 예: `VEHICLE_NOT_FOUND`)

### 4.5 차량 업데이트 이력 — 미정

`GET /api/v1/dashboard/vehicles/{vehicleId}/events` · admin-token

- `vehicle_event` 테이블 설계가 정해지지 않아 형식을 두지 않는다 ([server#24](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/24)).

## 5. 문서와 구현의 차이

| 항목 | 내용 |
|---|---|
| `campaignId` 타입 | 서버는 숫자(Long)다. flow.md 예시의 `"cmp-20261028-001"`은 문자열이라 맞지 않는다. 이 문서는 서버 기준으로 숫자를 쓴다 |
| 차량 에이전트 `campaignId` | orbit-vehicle-agent는 `campaignId`를 문자열로 받는다. 서버가 숫자를 보내므로 JSON 해석이 실패할 수 있다 (확인 필요) |
| 차량 에이전트 응답 형식 | 서버는 응답 본문을 `data` 아래에 담는다(server#83). orbit-vehicle-agent는 아직 본문을 최상위에서 읽으므로 `data`에서 읽도록 바꿔야 한다 (별도 Issue) |
