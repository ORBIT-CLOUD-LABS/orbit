# CDN 캐시·Edge replica·Edge 복구 실험 결과

- 진행 관리: [orbit#24](https://github.com/ORBIT-CLOUD-LABS/orbit/issues/24) (측정마다 코멘트로 원본 기록)
- 상태: 측정 완료 (2026-10-17 20:42~22:08 UTC). Edge probe·CPU limit 조정은 판단 대기 (8 절)
- 측정 도구: [orbit-observability `k6/`](https://github.com/ORBIT-CLOUD-LABS/orbit-observability/tree/develop/k6) `cdn-download` 시나리오 (orbit-observability#16), Grafana `load-recovery-experiment` 대시보드 CDN 패널

run ID 별 조건·결과·판정을 남긴다. run ID 는 `20261017T<시각>Z-cdn-download` 이고 아래 표에서는 앞의 날짜를 줄여 쓴다.

## 1. 확인할 것

| 질문 | 판정 방법 |
| --- | --- |
| Edge 캐시가 Origin 요청을 줄이는가 | k6 `cache` label(HIT/MISS) 비율, Loki Origin access log 요청 수 |
| cold 와 warm 이 다른가 | Edge 재시작 직후와 그 다음 실행의 MISS 수·P95 |
| Edge 처리 한계는 얼마이고 replica 1개와 2개가 다른가 | 다운로드 P95 500ms, dropped iterations, 오류율 1% 를 넘지 않는 최대 요청률 |
| Edge Pod 가 삭제돼도 다운로드가 이어지는가 | 실패 다운로드 수, 대체 Edge 시작 시간, 새 Edge 의 MISS → HIT 전환 |

서버 실험(orbit#23)이 끝난 뒤 실행했다. 두 실험은 동시에 돌리지 않는다.

## 2. 공통 조건

| 항목 | 값 |
| --- | --- |
| GitOps revision | `6f478aa` (측정 중 Edge replica 는 `kubectl` 임시 변경, 측정 후 git 값으로 복구) |
| Edge | `cdn-edge` 2 replica, `nginx`, CPU request 100m / limit 500m, 메모리 256Mi, `proxy_cache` `max_size=1g` `inactive=30m`, liveness `/healthz` (period 10초, timeout 1초, failureThreshold 3) |
| Origin | `cdn-origin` 1 replica, `nginx`, PVC `origin-files`(NFS 읽기 전용) |
| 접근 | Edge NodePort `192.168.100.102:30081` (worker-1). NodePort 는 연결 단위로 Edge Pod 를 고른다 |
| 시험 파일 | `K6-CDN-20MB` (20,971,520 byte, SHA-256 `a80ed4b4…`), `/firmware/K6-CDN-20MB/2.0.0/firmware.bin` |
| 부하 발생기 | ASTRA 호스트 k6 v2.3.0. E1·E2·E2b 는 `taskset -c 10,11` (2코어), E2c·E3·E4 는 `taskset -c 8-11` (4코어) + 해시 검증 끔(`HASH_RATIO=0`) |
| k6 | `cdn-download`. 차량 1대로 매니페스트를 받아 Edge `downloadUrl` 을 `ramping-arrival-rate` 로 반복 다운로드. 응답 크기는 매번 검증 |
| Origin 요청 수 | Loki access log (`role=origin`) |

측정 시간에는 orbit-gitops 머지를 멈춘다. main 에 새 커밋이 들어온 뒤 첫 `kubectl` 변경은 Argo CD auto-sync 가 되돌린다 (아래 E2b 에서 실제로 되돌려졌다).

## 3. 캐시 효과: cold·warm

| run | 내용 | 결과 |
| --- | --- | --- |
| `204223Z` | 시험 (2 req/s, 50초) | 100건, Edge 별 MISS 1·Origin 2건, P95 55ms, 크기·해시 100% |
| `204407Z` | **E1 cold**: Edge `rollout restart` 직후 10 req/s 2분 | 1,288건, MISS 2 (Edge 당 1), Origin 2건·42MB, P95 48ms, 실패 0, 210MB/s |
| `204620Z` | **E1 warm**: 이어서 같은 부하 | 1,296건 전부 HIT, Origin 0, P95 49ms |

- **캐시가 Origin 요청을 줄인다.** Edge 마다 첫 요청 1건만 MISS 로 Origin 에 가고 이후는 전부 HIT 다. E1 cold 2분(약 27GB 전송) 동안 Origin 은 2건·42MB 로 약 99.8% 를 줄였고, warm 은 Origin 0 이다
- cold 와 warm 의 P95 는 같다 (48ms, 49ms). 파일이 1개라 차이는 Edge 당 첫 요청 1건뿐이다

## 4. Edge 처리 한계와 replica 1개·2개 비교

| run | 내용 | 결과 |
| --- | --- | --- |
| `204901Z` | **E2** Edge 2개, 10 → 100 req/s 단계, k6 2코어 | 60 req/s(1.26GB/s)까지 P95 45~89ms. 80 req/s 목표에서 1.15~1.24GB/s 에 막혀 P95 3.5~3.8초, dropped 7~45/s. Edge CPU 0.24 core, k6 코어 약 78% → **부하 발생기 한계로 판단해 k6 를 4코어로 늘렸다** |
| `205815Z` | **E2b** Edge 1개, 10 → 60 req/s | 60 req/s·1.24GB/s 에서 P95 53ms, Edge CPU 0.25. 21:00:17 Argo CD auto-sync 로 replica 2 로 돌아갔지만 기존 연결이 유지돼 요청은 원래 Edge 1개로만 갔다 (15,110건 모두 같은 Edge). **비교 run 으로 쓰지 않는다** |
| `210700Z` | **E2c** Edge 2개, 40 → 190 req/s, k6 4코어 | 약 70 req/s·1.44~1.47GB/s 에서 막힘. 요청 약 80% 가 한 Edge 로 몰려(23,028 vs 5,546건) CPU limit 500m 도달(throttling 95~98%), P95 최대 9.6초, dropped 최대 120/s. k6 코어 약 55%. 21:10 다른 Edge 가 liveness probe timeout 으로 재시작돼 남은 Edge 로 더 몰렸다 |
| `215053Z` | **E4-1** Edge 1개, 20 → 100 req/s, k6 새로 시작 | **60 req/s · 1.26GB/s, P95 64ms**. 80 req/s 에서 Edge throttling 100%, 처리량이 오히려 1.0~1.05GB/s 로 떨어지고 P95 9~14초 |
| `215821Z` | **E4-2** Edge 2개, 같은 부하, k6 새로 시작 | **60 req/s · 1.26GB/s, P95 63ms**. 80 req/s 에서 22:03:17 한 Edge 가 liveness probe timeout 으로 재시작 → 남은 Edge 에 몰려 CPU limit 도달, P95 6~8초. 요청 분산 11,607 vs 6,082건 |

E4 는 E2b 가 비교가 되지 않아 다시 잰 것이다. Argo CD 가 21:00 에 CDN 앱을 한 번 sync 한 뒤라 scale 변경이 유지됐고, run 마다 k6 를 새로 시작해 연결을 새로 맺었다.

- **20MB 파일 기준 한계는 Edge 1개든 2개든 약 60 req/s (1.26GB/s, 약 10Gbps) 다.** 그 이상에서는 Edge CPU limit 500m 에 걸린다
- **Edge 를 2개로 늘려도 한계가 늘지 않았다.** 이유는 두 가지다
  - NodePort 분산이 연결 단위라 k6 처럼 연결을 재사용하면 한 Edge 로 몰린다
  - 한계 근처에서 한 Edge 가 liveness probe(`/healthz`, timeout 1초) 실패로 재시작되면 부하가 다른 Edge 로 몰리고, 그 Edge 도 CPU limit 에 걸린다. E2c(21:10)와 E4-2(22:03)에서 두 번 재현됐다

## 5. Edge 삭제 복구

| run | 내용 | 결과 |
| --- | --- | --- |
| `212058Z` | **E3** Edge 2개, 20 req/s(420MB/s) 유지 중 Edge 1개 삭제 | 21:23:20 정상 삭제 → 실패 0, 대체 Edge 21:23:22 시작. 21:25:49 강제 삭제(`--force`) → 실패 1 (21:26:20 i/o timeout), 대체 Edge 21:25:51 시작. 전체 8,200건 중 실패 1, P95 56ms |

- 대체 Edge 는 2초 안에 시작했다. 새 Edge 는 캐시가 비어 첫 요청 1건만 MISS 이고 이후 HIT 다 (구간 Origin 요청 4건)
- 정상 삭제는 실패 없이 넘어갔다. 강제 삭제는 진행 중이던 연결 1건이 30초 뒤 timeout 으로 실패했다

## 6. Origin·NFS 사용량

NFS VM(`192.168.100.105`) node_exporter 의 nfsd 지표로 17일 14:00~22:10 UTC(서버·CDN 측정 전체)를 봤다.

| 항목 | 값 |
| --- | --- |
| NFS 서버가 읽은 양 | 20,975,087 byte, **20:43 UTC 한 번** (시험 run `204223Z` 의 첫 MISS. 20MB 파일 1개 크기) |
| NFS 클라이언트 요청 | worker-3 에서만 Create 3·Write 40 (시험 파일 등록) · Read 160 (위 읽기). worker-1·2 는 0 |
| 그 밖의 구간 | NFS 서버 디스크 읽기 0 |

- 업로드(서버 Pod → PVC `origin-files-rw` → NFS)와 첫 다운로드(Edge MISS → Origin → NFS)는 실제로 NFS 를 거쳤다
- 이후 cold run 의 Origin 요청(E1 2건, E3 4건)은 NFS 까지 내려가지 않았다. Origin 쪽 page cache 가 응답한 것으로 본다
- 따라서 이번 결과는 **NFS 성능과 무관하다.** cache miss 가 많은 상황(파일 종류가 많거나 page cache 가 비었을 때)의 NFS·Origin 한계는 재지 않았다

## 7. 결과 해석 범위

- 파일 1개(20MB)로 잰 결과다. 실제 펌웨어는 차종·버전마다 다르므로 cold 비율은 파일 수에 따라 커진다
- 부하 발생기 1대가 연결을 재사용하는 조건이다. 차량 수천 대가 각자 연결하면 NodePort 분산이 이보다 고르게 된다. 반대로 Edge 수를 늘린 효과는 이 실험으로 주장하지 않는다
- 단일 ASTRA 호스트 안의 VM 들이라 네트워크가 물리 NIC 를 거치지 않는다. 처리량(1.26GB/s)은 이 환경의 값이다
- Origin·NFS 는 1개라 고가용성(HA)을 주장하지 않는다

## 8. 판정과 후속 작업

- **CDN HPA 는 지금 규모에서 필요 없다.** 한계는 Edge 수가 아니라 Edge CPU limit(500m)·liveness probe 재시작·연결 단위 분산에서 온다. Pod 를 늘려도 기존 연결은 옮겨지지 않는다
- CDN 확장 전에 먼저 볼 것 (판단 대기, 정하면 orbit-gitops 에 반영)
  - Edge liveness probe 완화: 과부하에서 `/healthz` 응답이 늦어도 Pod 를 죽이지 않도록 `timeoutSeconds`·`failureThreshold` 여유
  - Edge CPU limit 상향 (500m 에서 throttling 95~100%)
- 아직 재지 않은 것: 파일 여러 개로 cold 캐시 비교, page cache 를 비운 cold Origin(NFS 읽기 포함)
- 대시보드 "Origin 요청률 / 전송 byte" 패널은 요청률과 byte 가 같은 축이라 단위가 섞여 보인다 (orbit-observability 에서 수정)
- 측정 후 Edge 는 git 값(2 replica)으로 돌아갔고 Argo CD `Synced`·`Healthy` 를 확인했다 (22:08 UTC)
- Grafana 스크린샷은 [`발표 자료/측정-스크린샷/`](<../발표 자료/측정-스크린샷/>) 의 `CDN-*` 2장이다 (orbit#38 PR 에서 추가)
