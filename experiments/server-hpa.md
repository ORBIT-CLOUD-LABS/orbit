# 서버 부하·HPA·복구 실험 결과

- 진행 관리: [orbit#23](https://github.com/ORBIT-CLOUD-LABS/orbit/issues/23) (측정마다 코멘트로 원본 기록)
- 상태: 사전 측정 완료, 본 실험 진행 중
- 측정 도구: [orbit-observability `k6/`](https://github.com/ORBIT-CLOUD-LABS/orbit-observability/tree/develop/k6) `server-hpa` 시나리오, Grafana `load-recovery-experiment` 대시보드

run ID 별 조건·결과·판정을 남긴다. 값은 Prometheus 30초 간격·2분 창 평균이다 (서버·cAdvisor 수집 간격이 30초라 1분 창은 값이 튄다).

## 1. 판정 기준

- 한계(`T_pod`, `T_db`): 아래 중 하나라도 해당하는 첫 단계의 바로 앞 단계
  - 차량 API P95 500ms 초과
  - 서버가 받은 요청률이 목표의 95% 미만이거나 dropped iterations 발생
  - 오류율 1% 초과
- "replica 가 늘었다"와 "처리량·P95 가 개선됐다"는 따로 판정한다. 개선이 없으면 원인을 기록하고 성공으로 바꾸지 않는다

## 2. 공통 조건

| 항목 | 값 |
| --- | --- |
| 서버 이미지 | `orbit-server@sha256:adafcf90f80839710febbf9c880477c30bc87cb62a33830a1496a7eba28ae358` |
| GitOps revision | `844b202` (측정 중 HPA·자원값은 `kubectl` 임시 패치, 측정 후 git 값으로 복구) |
| 서버 Pod | 메모리 request=limit `768Mi`, Hikari pool 5/Pod, JVM Serial GC (메모리 768Mi 라 CPU 와 무관하게 Serial) |
| 클러스터 | worker 3대 × 2 vCPU. ASTRA 물리 12 스레드를 VM vCPU 14개가 나눠 쓴다 |
| DB VM | `astra-vehicle-db` 1 vCPU, 메모리 3.9GiB. 데이터 디스크는 ASTRA 호스트 NVMe(Samsung SSD 980, 모든 VM 공유) 위 raw 파일 |
| 부하 발생기 | ASTRA 호스트에서 `taskset -c 10,11` 로 k6 v2.3.0 실행 (Tailscale 경유 장비는 중계 지연이 섞여 비교 run 에 쓰지 않음) |
| k6 | `server-hpa`, `VEHICLES=200`(VU 상한), `UPDATE_RATIO=0.1` (체크인 + 10% 매니페스트, 요청 수 ≈ iteration × 1.1) |

측정 시간에는 orbit-gitops 머지를 멈춘다. main 에 새 커밋이 들어온 뒤 첫 변경은 Argo CD auto-sync 가 git 값으로 되돌린다 (2026-10-17 HPA 패치가 3초 만에 되돌려진 것으로 확인).

## 3. 사전 측정 (2026-10-17)

| run ID | 목적 | 서버 조건 | 부하 |
| --- | --- | --- | --- |
| `20261017T063247Z-server-hpa` | 0단계 범위 탐색 | Pod 1, 500m | 5 → 200 iteration/s 연속 증가 |
| `20261017T064602Z-server-hpa` | `T_db` | Pod 2, CPU limit 없음 | 80~240, 단계 유지 |
| `20261017T070020Z-server-hpa` | `T_db` 연장 | Pod 2, CPU limit 없음 | 240~360, 단계 유지 |
| `20261017T071843Z-server-hpa` | `T_pod` 300m (기동 직후) | Pod 1, 300m | 5~130, 단계 유지 |
| `20261017T073533Z-server-hpa` | `T_pod` 300m (예열 후) | Pod 1, 300m | 10~90, 단계 유지 |
| `20261017T075745Z-server-hpa` | `T_pod` 500m | Pod 1, 500m | 10~170, 단계 유지 |
| `20261017T081620Z-server-hpa` | 고정 1개 (본 실험 부하 형태) | Pod 1, 500m, 예열된 상태 | 60 → 170 (6분 유지) → 60 |

### 3.1 `T_db` ≈ 290 req/s

| 서버 req/s | P95 (check-in) | DB user+system CPU | DB iowait | DB 디스크 쓰기/s | Hikari 활성 / 대기 |
| --- | --- | --- | --- | --- | --- |
| 245 | 12ms | 45% | 약 43% | 2011 | 1 / 0 |
| 274 | 12ms | 52% | 약 44% | 2307 | 1 / 0 |
| 300 | 337ms | 54% | 약 44% | 1743 | 3 / 0 |
| 300 이상 | 0.6~1.3초 | 54~58% | 약 42% | 640~970 | 5 / 60~124 |

- **DB 는 CPU 계산이 아니라 commit 디스크 동기화에서 막힌다.** 체크인마다 `vehicle_state` 갱신 트랜잭션이 commit 되고, InnoDB 는 commit 마다 디스크에 확정(fsync)한다. 디스크 쓰기가 약 2300회/s 에서 꺾였고 user+system CPU 는 55~58% 에서 멈췄다
- 대시보드의 `100 - idle` 은 iowait 를 사용량으로 세어 "DB CPU 96~100%" 로 보였다 (orbit-observability#25 에서 패널을 user·system·iowait 로 분리)
- DB VM vCPU·메모리를 늘려도 이 한계는 거의 바뀌지 않는다. 올리려면 commit 동기화 완화(`innodb_flush_log_at_trx_commit=2` 등, 장애 시 최대 약 1초 유실)나 디스크 변경이 필요해 이번 실험에서는 하지 않는다
- 본 실험 부하 상한: `0.75 × T_db ≈ 220 req/s` (디스크가 정상일 때. 아래 3.2)

### 3.2 DB 디스크는 높은 쓰기 부하가 이어지면 느려진다

DB 데이터 디스크(`vdb`) 쓰기 1회 평균 시간 (`node_disk_write_time_seconds_total / node_disk_writes_completed_total`, 1분 창):

| 시각 (UTC) | 상태 | 쓰기 지연 | 쓰기/s | 서버 P95 |
| --- | --- | --- | --- | --- |
| 06:49~07:03 | 정상 | 0.35ms | 최대 2300 | 11~12ms (265 req/s 까지) |
| 07:04~07:10 | 느려짐 | 1.1~1.5ms | 640~880 | 0.8~1.3초 |
| 07:29~08:24 | 회복 | 0.35~0.45ms | - | - |
| 08:25~08:34 | 느려짐 | 1.1~1.35ms | 600~880 | 간헐 최대 1.75초 |

- 높은 쓰기 부하가 몇 분 이어지면 지연이 3~4배로 뛰고, 부하를 줄여도 한동안 유지된다 (67 req/s 에서도 1.35ms). 20분가량 쉬면 회복한다
- 같은 시간 다른 VM 쓰기는 거의 없었고 NVMe 온도도 정상(47℃). DRAM 없는 소비자용 SSD 가 동기화 쓰기를 계속 받으면서 내부에서 느려지는 것으로 본다
- 따라서 DB 한계는 고정값이 아니다. 디스크가 정상일 때 약 290 req/s, 느려지면 약 180 req/s 수준

### 3.3 `T_pod`

| CPU limit | `T_pod` | 기동 시간 (Ready) | 비고 |
| --- | --- | --- | --- |
| limit 없음 | - | 약 7.5초 | Pod 2개로 264 req/s 에서 Pod 당 CPU 약 0.25 core |
| 500m | **약 140 req/s** (기동 7분), **약 180~190 req/s** (20분 이상 예열) | 약 55~60초 | 한계에서도 CPU 는 request 의 50~55%. 기동 7분 Pod 는 약 175 req/s 에서 무너졌고, 예열 전 41~68 req/s 구간에서 P95 최대 4초로 튀었다가 2분 뒤 회복. 예열된 Pod 는 187 req/s 를 6분 유지 (P95 상승은 DB 디스크 저하 구간) |
| 300m | **약 40~45 req/s** | **151초** (startupProbe 120초 초과) | 기동 직후에는 3~9 req/s 에서 무너지고 회복하지 못함. 예열 후에도 한계를 넘으면 P95 7초가 이어짐 |

- 한계는 평균 CPU 부족이 아니라 순간 throttling 으로 응답이 늦어져 DB 연결을 오래 잡고 Hikari 대기가 쌓이는 형태로 온다. limit 을 낮추면 처리량이 비례보다 크게 떨어진다 (300m 는 500m 대비 CPU 60%, 처리량 1/3)
- HPA 목표 사용률은 한계에서의 CPU 사용률(500m 에서 50~55%)보다 낮아야 무너지기 전에 늘어난다

## 4. HPA 범위 결정

HPA 효과가 보이려면 본 실험 부하 L 이 `최소 replica × T_pod < L ≤ 0.75 × T_db < 최대 replica × T_pod` 를 만족해야 한다.

| 조건 | 최소 replica 한계 | L 구간 | 판단 |
| --- | --- | --- | --- |
| 500m, HPA 2~4 | 약 280 | 없음 | 고정 2개도 DB 에 먼저 막힌다 |
| 300m, HPA 2~4 | 약 85 | 숫자상 약 90~170 | 새 Pod 가 Ready 까지 2분 반, 예열 전 무너짐 위험이 커서 HPA 효과로 해석하기 어렵다 |
| **500m, HPA 1~3** | 약 140 | **약 160~210** | limit 을 낮추지 않고 구간이 넓다. 목표 사용률 50% |

phase1 §4 의 HPA "최소 2, 최대 4" 는 확정값이므로 1~3 실험이 유의미하면 배포값 변경을 검토한다. 최소 1 은 부하가 없을 때 Pod 1개라 가용성이 떨어진다. Pod 삭제 복구 실험은 Pod 2개로 따로 한다.

**고정 1개 비교 결과 1~3 도 구간이 거의 없다.** 예열된 500m Pod 1개가 약 187 req/s 를 처리했고 (`20261017T081620Z`), 오래 버틸 수 있는 DB 한계도 디스크 저하 때문에 약 180~200 이다. 서버가 DB 보다 먼저 막히는 구간이 없어 HPA 1~3 측정은 하지 않았다.

| 구간 | 서버 req/s | P95 | Pod CPU | throttling |
| --- | --- | --- | --- | --- |
| 60 iteration/s | 66~85 | 12~14ms | 40% | 0% |
| 170 유지 처음 3분 | 186~187 | 28~33ms | 65~66% | 2~3% |
| 170 유지 뒤 3분 (DB 디스크 저하) | 180~187 | 190ms~1.2초 (간헐) | 64~71% | 4~7% |

그래서 DB commit 동기화를 줄여 DB 한계를 올렸다 (아래 5). 그 결과 500m·HPA 2~4 로 본 실험을 할 수 있게 됐다.

## 5. DB commit 동기화 완화

### 5.1 변경

2026-10-17 08:45:42 UTC 에 DB 런타임 설정을 바꿨다 (`SET GLOBAL`). 설정 파일 반영은 [orbit-infrastructure#63](https://github.com/ORBIT-CLOUD-LABS/orbit-infrastructure/issues/63) (PR #64).

| 설정 | 전 | 후 | 의미 |
| --- | --- | --- | --- |
| `innodb_flush_log_at_trx_commit` | 1 | 2 | redo log 디스크 동기화를 commit 마다 → 1초마다 |
| `sync_binlog` | 1 | 0 | binlog 동기화를 commit 마다 → OS 에 맡김 (`log_bin` ON) |

기존에는 commit 한 번에 디스크 동기화가 두 번(redo log, binlog) 일어났다. 대가는 DB VM·ASTRA 호스트가 갑자기 꺼질 때 최대 약 1초치 commit 유실이다 (MySQL 프로세스만 죽으면 유실 없음). 차량 체크인은 다음 주기에 다시 들어온다.

### 5.2 `T_db` 재측정 ≈ 700~750 req/s

| run ID | 서버 조건 | 부하 |
| --- | --- | --- |
| `20261017T084619Z-server-hpa` | Pod 2, CPU limit 없음 | 10~440 iteration/s, 단계 유지 |
| `20261017T090228Z-server-hpa` | Pod 2, CPU limit 없음 | 440~680, 단계 유지 (연장) |

| 서버 req/s | P95 (check-in) | DB user+system | DB iowait | DB 디스크 쓰기/s | dropped |
| --- | --- | --- | --- | --- | --- |
| 220 | 6.3ms | 39% | 거의 0 | 24 | 0 |
| 309 | 6.6ms | 47% | 거의 0 | 34 | 0 |
| 406 | 7.9ms | 57% | 거의 0 | 42 | 0 |
| 491 | 12ms | 70% | 0.6% | 약 50 | 0 |
| 572~588 | 19~62ms | 80~83% | 0.3% | - | 약 4.7/s (일부 구간) |
| 652~664 | 46~49ms | 88~91% | 0.3% | - | 0 |
| 690~748 | 128~161ms | 93~98% | 0.2% | - | 0 |

- `T_db` ≈ 700~750 req/s (기존 약 290). 748 req/s 에서 P95 161ms 로 기준 안이지만 DB CPU 98% 라 사실상 한계. 실패 0
- 막힌 곳이 디스크 동기화에서 **DB CPU (1 vCPU)** 로 옮겨졌다. 디스크 쓰기는 2300 → 약 50회/s 로 줄었고, 3.2 의 쓰기 지연 저하는 다시 나타나지 않았다 (측정 중 0.53~0.86ms)
- 748 req/s 시점 다른 자원: worker-1·3 CPU 46~48%, ASTRA 호스트 CPU 43% (k6 포함), steal 0

### 5.3 본 실험 조건

| 항목 | 값 |
| --- | --- |
| DB 여유 기준 `0.75 × T_db` | 약 525 req/s |
| 고정 2개 한계 `2 × T_pod` (500m) | 약 280 (덜 예열) ~ 370 (충분히 예열) |
| HPA 4개 한계 | 약 560~740 |
| 본 실험 부하 L | 약 430~480 req/s |

phase1 확정값(HPA 2~4)과 CPU limit 500m 를 바꾸지 않고 서버가 DB 보다 먼저 막히는 조건이 만들어졌다. 4절의 HPA 1~3 검토는 필요 없어졌다. HPA 목표 사용률은 500m Pod 한계에서의 CPU 사용률(50~55%)을 고려해 50% 로 시험한다.

## 6. 본 실험: 고정 replica vs HPA

### 6.1 1차 (2026-10-17, DB 동기화 완화·DB 1 vCPU)

| 항목 | 고정 2개 | HPA 2~4 |
| --- | --- | --- |
| run | `20261017T093350Z-server-hpa` | `20261017T095434Z-server-hpa` |
| 서버 | 500m, HPA 2~2 | 500m, HPA 2~4, 목표 50% |
| 시작 상태 | 측정 직전 `rollout restart` (Pod 기동 약 4분 뒤 부하) | 같음 |
| 부하 | warm-up 10 → 150 iteration/s → 400 iteration/s(약 440 req/s) 6분 유지 → 150 | 같음 |

| 구간 | 지표 | 고정 2개 | HPA 2~4 |
| --- | --- | --- | --- |
| 예열 구간 (10→150 상승) | P95 | 2.0~2.4초, 약 3분 뒤 회복 | 1.8~6.5초, 약 8분 |
| | Hikari 대기 / throttling | 최대 95 / 최대 87% | 91~109 / 80~96% |
| | replica | 2 | 3 → 4 (CPU 100% 로 바로 확장, 새 Pod 2개 모두 worker-3) |
| 440 req/s 유지 (예열 후) | P95 | 55~300ms (흔들림) | **16~64ms** |
| | throttling | 20~29% | 2~3% |
| | Pod CPU (request 대비) | 75~86% | 50~55% |
| | dropped | 간헐 2.5~6/s | 0~1.3/s |
| 전체 | 실패 / dropped / P95 | 0 / 1,149 / 199ms | 0 / 40,736 / 2.38초 |

- 예열 후 같은 부하에서는 HPA 4개가 P95·throttling·dropped 모두 낫다. 처리량은 둘 다 부하만큼 받았다
- 전체 수치는 HPA 쪽이 나빴다. 측정 직전 Pod 재시작으로 생긴 JVM 예열(JIT) 구간이 결과를 좌우했다. HPA 는 예열 CPU 에 반응해 바로 4개로 늘렸지만 새 Pod 도 예열 전이라 함께 밀렸다
- **판정 보류**: 예열을 분리해 다시 잰다 (Pod 재시작 뒤 낮은 부하로 충분히 예열한 다음 비교 구간 시작, 판정은 예열 이후 구간). scale-out 직후 새 Pod 가 예열 전이라 지연이 일시적으로 커질 수 있다는 점은 결과로 함께 기록한다

## 7. Pod 삭제 복구 실험

(예정)

## 8. 결과 해석 범위

- "DB 한계(commit 동기화 완화 후 약 700~750 req/s, DB 1 vCPU) 아래에서 서버 계층이 수평 확장된다"는 범위로만 주장한다
- DB commit 동기화를 완화한 조건(최대 약 1초치 유실 허용)의 결과다. 기본값에서는 DB 한계가 약 290 req/s 라 같은 결론이 나오지 않는다
- 단일 ASTRA 호스트·DB 1대·NFS 1대라 HA 를 주장하지 않는다
- DB 한계는 ASTRA 호스트의 소비자용 NVMe 를 모든 VM 이 함께 쓰는 환경의 값이다
