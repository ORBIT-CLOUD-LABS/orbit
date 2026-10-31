# 서버 HPA·복구 시연 시나리오 (1차 발표)

- 진행 관리: [orbit#38](https://github.com/ORBIT-CLOUD-LABS/orbit/issues/38)
- 실험 결과 원본: [`experiments/server-hpa.md`](../experiments/server-hpa.md), [orbit#23](https://github.com/ORBIT-CLOUD-LABS/orbit/issues/23)
- 상태: orbit#23 결과 반영 (서버 request `500m` / limit `750m`, HPA 2~4·목표 70%)

발표에서 보여줄 흐름은 **부하 → HPA 확장 → 모니터링으로 판정 → Pod 삭제 복구** 다. 라이브 시연은 실패할 수 있으므로 같은 장면을 재측정 run 의 스크린샷·녹화로 준비해 두고, 라이브는 짧게 한다.

## 1. 화면 준비

| 화면 | 내용 |
| --- | --- |
| Grafana | 대시보드 "부하·복구 실험 (orbit-observability#6)" (`http://192.168.100.107:3000/d/orbit-load-recovery`). 시간 범위는 시연 시작 10분 전부터, 새로고침 30초 (라이브) 또는 재측정 run 구간 고정 (녹화·캡처) |
| 터미널 1 | `kubectl -n orbit get hpa orbit-server -w` |
| 터미널 2 | `kubectl -n orbit get pods -l app=orbit-server -o wide -w` |
| 터미널 3 | k6 실행 (ASTRA 호스트) |

- Grafana 위쪽 annotation 토글 2개(HPA scale, orbit-server Pod 생성·삭제·probe 실패)를 켜 둔다. 모든 그래프에 세로선으로 scale 시점과 Pod 교체 시점이 표시된다
- `k6 run_id` 변수는 시연 run 하나만 고른다 (k6 패널이 그 run 만 보이게)

## 2. 보여줄 패널과 순서

| 순서 | 패널 | 말할 것 |
| --- | --- | --- |
| 1 | k6 보낸 요청률 / 서버 차량 API 요청률 | 부하가 목표만큼 들어가고 서버가 그만큼 받는지 (발생기가 병목이 아님을 먼저 보인다) |
| 2 | HPA 현재 / 목표 사용률 | 사용률이 목표를 넘는 순간 |
| 3 | Replica desired / available / ready + HPA scale annotation | desired 가 먼저 오르고 ready 가 뒤따르는 간격 = 새 Pod 기동 시간(limit 750m 약 25~28초) |
| 4 | 차량 API P95 (uri 별) | scale-out 뒤 P95 가 내려가는 시점. 새 Pod 는 예열(JIT) 전에는 약해 바로 좋아지지 않는다는 점도 짚는다 |
| 5 | Pod 별 CPU throttling | 고정 2개에서는 throttling 이 계속 높고, HPA 에서는 Pod 가 늘며 낮아진다 |
| 6 | DB VM CPU (user·system·iowait) | DB 는 아직 여유가 있다 = 서버 계층에서 확장 효과를 본 것 |
| 7 | 서버 WARN·ERROR 로그 | (붕괴 사례를 보여줄 때) DB 연결 획득 지연 같은 원인이 로그로 보인다 |
| 8 | Pod 나이 + Pod annotation | (복구 시연) 삭제 → 새 Pod 생성 → Ready 까지 |

## 3. 라이브 시연 절차 (약 10분)

1. 시작 전: 서버 Pod 2개, HPA `2~4`, `kubectl get hpa` 사용률 한 자릿수 확인. Pod 는 미리 예열해 둔다 (HPA 를 건드리지 않도록 10 → 150 iteration/s 5분, 별도 k6 run). 막 기동한 Pod 에 바로 높은 부하를 주면 JIT 예열 때문에 P95 가 크게 튄다
2. 부하 시작 (ASTRA 호스트):
   ```bash
   cd ~/orbit-observability
   JUMP_HOST= taskset -c 10,11 k6/scripts/run.sh server-hpa \
     -e VEHICLES=200 -e UPDATE_RATIO=0.1 -e STAGES=150:30s,150:1m,650:2m,650:6m,150:1m
   ```
   부하 `650` iteration/s 는 약 715 req/s 다 (iteration 하나가 체크인 1회 + 10% 확률로 매니페스트 1회). 고정 2개로는 처리하지 못하고 Pod 4개면 처리하는 값이며, DB 한계(약 1,000 req/s) 아래다. scale-out 뒤 회복까지 4~6분 걸리므로 `650` 유지는 6분보다 줄이지 않는다
3. 터미널 1 에서 `TARGETS` 가 목표를 넘고 `REPLICAS` 가 2 → 3 → 4 로 바뀌는 것을 보여준다. 같은 시점이 Grafana 세로선으로 찍힌다
4. Grafana 패널 2 → 3 → 4 → 5 → 6 순서로 설명
5. 부하가 내려가면 HPA 가 약 5분 뒤(scale-down 안정화) Pod 를 줄인다. 시간 관계상 녹화로 보여준다
6. 복구: `kubectl -n orbit delete pod <하나>` → 터미널 2 에서 새 Pod 가 뜨고 Ready 가 되는 것, Grafana 패널 8 과 k6 실패 요청률

## 4. 백업 자료 (재측정 run 에서 준비)

run ID 는 `20261017T` 뒤 시각이다 (예: `194408Z` = `20261017T194408Z-server-hpa`). Grafana 에서 run 구간으로 시간을 고정해 캡처한다.

| 장면 | 출처 run | 형식 |
| --- | --- | --- |
| 고정 2개 vs HPA 2~4 P95·throttling 비교 | B1 `194408Z` (고정 2개, L=650), B2 `200549Z` (HPA 2~4, 목표 70%) | 같은 부하의 스크린샷 2장 |
| HPA scale-out 순간 (사용률, replica, annotation) | B2 `200549Z` (20:10:28 replica 3, 20:10:58 replica 4) | 녹화 (1분) 또는 스크린샷 |
| scale-out 직후 새 Pod 예열 구간 | B2 `200549Z` (20:10:30~20:15:30, P95 최대 2.2~2.3초) | 스크린샷 |
| DB 여유 (user·system·iowait) | B2 `200549Z` (전환 뒤 DB CPU 약 64%, iowait 1% 미만) | 스크린샷 |
| Pod 삭제 → 복구 | C `213444Z` (limit 750m) | 스크린샷 |
| 설정 비교: 붕괴 사례 | C `183046Z` (request=limit 500m, 삭제 뒤 붕괴), `185946Z` (750m 2Pod 한계) | 스크린샷 |
| DB 병목 해결 전후 (`T_db` 290 → 700~750 → 약 1,000) | 사전 측정, A `170253Z` | 표 |

### 준비된 스크린샷 (`측정-스크린샷/`)

| 장면 | 파일 |
| --- | --- |
| 고정 2개 L=600 (버팀) | [B1-고정2-750m-L600-요청률·P95](<측정-스크린샷/B1-고정2-750m-L600-요청률·P95.jpg>) |
| 고정 2개 L=650 (못 받음) | [B1-고정2-750m-L650-붕괴-요청률·P95·Hikari](<측정-스크린샷/B1-고정2-750m-L650-붕괴-요청률·P95·Hikari.jpg>) |
| HPA 2~4 L=650 요청률·P95 | [B2-HPA70-750m-L650-요청률·P95·Hikari](<측정-스크린샷/B2-HPA70-750m-L650-요청률·P95·Hikari.jpg>) |
| HPA 2~4 scale-out (replica·사용률·CPU) | [B2-HPA70-750m-L650-replica·HPA·CPU](<측정-스크린샷/B2-HPA70-750m-L650-replica·HPA·CPU.jpg>) |
| 750m Pod 2개 한계 (throttling·Hikari 대기) | [limit750m-2Pod한계-throttling·Hikari대기](<측정-스크린샷/limit750m-2Pod한계-throttling·Hikari대기.jpg>) |
| Pod 삭제 복구 (limit 1000m) | [C1-limit1000m-pod삭제-replica·CPU](<측정-스크린샷/C1-limit1000m-pod삭제-replica·CPU.jpg>) |
| Pod 삭제 뒤 붕괴 (request=limit 500m) | [C2-limit500m-pod삭제-붕괴·probe실패](<측정-스크린샷/C2-limit500m-pod삭제-붕괴·probe실패.jpg>), [C2-limit500m-k6-VU·dropped](<측정-스크린샷/C2-limit500m-k6-VU·dropped.jpg>) |
| 300m 붕괴 원인 로그 (패널 7) | [300m-붕괴-WARN로그](<측정-스크린샷/300m-붕괴-WARN로그.jpg>) |
| 본 실험 1차 replica·annotation | [본실험1차-HPA2-4-replica·annotation](<측정-스크린샷/본실험1차-HPA2-4-replica·annotation.jpg>) |
| CDN 캐시 (Edge HIT·전송량·Origin) | [CDN-E1~E3-Edge HIT·전송량·Origin](<측정-스크린샷/CDN-E1~E3-Edge HIT·전송량·Origin.jpg>) |
| CDN Edge 1개 vs 2개 처리량 한계 | [CDN-Edge1vs2-처리량한계](<측정-스크린샷/CDN-Edge1vs2-처리량한계.jpg>) |

권고 설정(750m)의 Pod 삭제 복구(`213444Z`) 스크린샷은 아직 없다. 녹화 백업과 함께 준비한다.

## 5. 결과 표 (발표용)

같은 부하 L=650 iteration/s (약 715 req/s), 서버 request `500m` / limit `750m`, 예열 뒤 측정. HPA 2~4 값은 전환 구간이 끝난 뒤다.

| 항목 | 고정 2개 (`194408Z`) | HPA 2~4, 목표 70% (`200549Z`) |
| --- | --- | --- |
| 부하 L 에서 서버 처리량 | 520~600 req/s (부하를 다 받지 못함) | 705~720 req/s (전부 처리) |
| P95 | 1.1~1.4초 | 약 20ms |
| 오류율 / dropped | 실패 요청 없음, dropped 100~180/s | 실패 요청 없음, dropped 0 |
| throttling | 80~100% | 2~10% |
| scale-out 시각 → P95 개선 시각 | - | L 도달 약 10초 뒤 replica 3, 40초 뒤 4 (부하 상승 시작부터 30초 안에 scale-up 시작) → 새 Pod 예열로 4~6분 P95 최대 2.2~2.3초 → 20:15:30 회복 |
| 병목 위치 | 서버 Pod CPU (limit 750m 에서 throttling, Hikari 대기 150~190). DB CPU 약 58% | 없음 (DB CPU 약 64%, DB 한계 약 1,000 req/s 아래) |

HPA 목표 50% (`202727Z`) 도 scale 시점이 같고 전환 뒤 P95 21~37ms 로 70% 보다 낫지 않았다.

Pod 2개 고정, 150 iteration/s (약 165 req/s, Pod 1개로도 처리 가능한 양) 유지 중 Pod 1개 삭제 (`213444Z`, limit 750m).

| 복구 | 정상 종료 (`kubectl delete pod`) | 강제 종료 (`--force`) |
| --- | --- | --- |
| 실패 요청 수 | 2 (EOF) | 113 (2초 동안 connection reset) |
| available 복귀 시간 | 약 45초 | 약 45초 |
| 복귀 뒤 | 새 Pod 예열 약 2분 P95 455~793ms, 이후 회복 | P95 10~16ms 유지 |

- 전체 실패율 0.09%, P95 105ms, dropped 0
- 강제 종료 실패는 진행 중이던 연결만큼 생긴다. 설정으로 줄일 수 있는 값이 아니다
- 비교: request=limit `500m` 은 삭제 뒤 남은 Pod 와 새 Pod 가 함께 무너져 측정 끝까지 P95 8~10초 (`183046Z`)

## 6. 실험 과정에서 말할 것 (사전 측정)

- 처음에는 DB 가 서버보다 먼저 막혀 HPA 효과를 볼 수 없었다 (DB 한계 약 290 req/s)
- 원인은 CPU 가 아니라 commit 마다의 디스크 동기화였다. 대시보드가 iowait 를 CPU 사용으로 세고 있어 "DB CPU 100%" 로 보였다 → 패널을 고쳐서 찾았다
- DB vCPU 를 늘려도 해결되지 않는 문제라 commit 동기화를 먼저 완화했고 (DB 한계 약 700~750 req/s, 대가는 장애 시 최대 약 1초치 유실), 그 뒤 DB VM 을 2 vCPU 로 늘려 약 1,000 req/s 가 됐다
- 서버 CPU limit 을 낮춰 HPA 를 억지로 보이게 하는 방법도 시험했지만, 300m 에서는 기동 151초·예열 전 붕괴로 결과를 해석할 수 없어 쓰지 않았다
- request=limit `500m` 은 막 기동한 Pod 가 JIT 컴파일에 CPU 를 쓰다 상한에 막혀 낮은 부하에서도 무너졌다. limit `1000m` 은 Pod 2개가 DB 한계 근처까지 처리해 HPA 효과가 드러나지 않았다. 그래서 request `500m` / limit `750m` 으로 정했다
- k6 는 VU(차량) 200 을 상한으로 둔다. 정상 응답(약 20ms)에서는 VU 가 13개 정도만 필요해 병목이 아니고, 응답이 약 300ms 를 넘으면 VU 가 모자라 dropped 로 한계가 드러난다

## 7. 한계 (발표에 명시)

- ASTRA 호스트 1대 위의 VM 들이라 HA 를 주장하지 않는다. DB·NFS 도 1대
- DB 디스크는 모든 VM 이 함께 쓰는 소비자용 NVMe 이고, 결과는 DB commit 동기화 완화 조건이다
- "DB 한계 아래에서 서버 계층이 수평 확장된다" 는 범위로만 주장한다
- scale-out 직후 4~6분 전환 구간이 있으므로 "HPA 가 즉시 복구한다" 고 말하지 않는다
- Phase2 범위: CDN HPA, 서명 URL, Argo CD 자동 Sync 확장
