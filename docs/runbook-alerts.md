# 알림 대응 Runbook

Alertmanager 알림이 왔을 때 무엇을 확인하고 어떻게 조치하는지다. 알림 규칙은
[orbit-observability `prometheus/rules/alerts.yml`](https://github.com/ORBIT-CLOUD-LABS/orbit-observability/blob/develop/prometheus/rules/alerts.yml),
통지 설정은 같은 레포 `alertmanager/alertmanager.yml` 이다
([orbit-observability#29](https://github.com/ORBIT-CLOUD-LABS/orbit-observability/issues/29)).
각 알림의 `runbook_url` 은 이 문서의 해당 절을 가리킨다.

알림은 디스코드 `#orbit-알림` 으로 온다. 통지가 끊긴 것은 `Watchdog` 으로 판단한다 (아래).
채널은 웹후크를 만들 때 고정되므로 바꿀 때는 설정이 아니라 웹후크를 다시 만든다
(orbit-observability `alertmanager/README.md`).

## 공통

### 접속 경로

| 대상 | 주소 | 접근 제한 |
| --- | --- | --- |
| Grafana | `http://192.168.100.107:3000` | 모든 출발지 |
| Prometheus | `http://192.168.100.107:9090` | k8s 노드 4대(`.101`~`.104`)와 `192.168.100.1` |
| Alertmanager | `http://192.168.100.107:9093` | `192.168.100.1` |
| Loki | `:3100` | k8s 노드 4대. 사람은 Grafana Explore 로 본다 |
| kubectl | `astra-control-plane` `192.168.100.101` | - |

ASTRA 밖 장비는 Tailscale subnet router(`orbit-astra`)를 지나며 출발지가 모두
`192.168.100.1` 로 보인다. 외부에서 Prometheus·Alertmanager 가 열리지 않으면 먼저 Tailscale
연결을 확인한다. 9093·9090 은 수동 ufw 규칙이고 코드화는
[orbit-infrastructure#62](https://github.com/ORBIT-CLOUD-LABS/orbit-infrastructure/issues/62)
에서 다룬다.

### 알림 상태 확인·끄기

```bash
# 지금 발화 중인 알림
curl -s localhost:9093/api/v2/alerts | jq -r '.[] | "\(.labels.alertname) \(.labels.instance // "") \(.status.state)"'

# 규칙 평가 상태 (pending / firing)
curl -s localhost:9090/api/v1/rules | jq -r '.data.groups[].rules[] | select(.type=="alerting") | "\(.name) \(.state)"'

# 작업 중 통지만 멈춘다 (조치 중·실험 중에 쓴다)
amtool silence add --alertmanager.url=http://localhost:9093 --duration=2h \
  --comment="부하 실험 중 (orbit#23)" alertname=ContainerCpuThrottlingHigh
amtool silence query --alertmanager.url=http://localhost:9093
```

**부하 실험·발표 시연 전에는 silence 를 먼저 걸고 시작한다.** 실험 중에는
`ContainerCpuThrottlingHigh`·`VehicleApiLatencyP95High`·`DeploymentReplicasUnavailable` 이
의도된 상태로 발화한다.

### 임계값에 대한 전제

임계값과 `for` 는 1차 제안값이다. 실측 기준선은
[`experiments/server-hpa.md`](../experiments/server-hpa.md) 에 있다. 아래 절의 "정상 범위" 는
그 문서 8·11 절의 확정 조건(서버 request `500m` / limit `750m`, HPA 2~4 목표 70%, DB 2 vCPU)
기준이다.

---

## Watchdog

**무엇이 깨졌나** — 깨진 게 없다. 항상 발화하는 생존 신호이고 사람에게 보내지 않는다
(`severity: none` → `orbit-null` 수신처). **이 알림이 사라진 것이 이상 신호다.**

**어디를 보나** — 디스코드에는 오지 않으므로 Alertmanager 에서 직접 본다.

```bash
curl -s localhost:9093/api/v2/alerts | jq -r '.[].labels.alertname' | grep Watchdog
```

없으면 순서대로 가른다.

```bash
systemctl status prometheus prometheus-alertmanager
curl -s localhost:9090/api/v1/alertmanagers | jq -c '.data.activeAlertmanagers'   # 비어 있으면 연결이 끊겼다
curl -s localhost:9090/api/v1/rules | jq -r '.data.groups[].name'                 # 규칙이 안 읽혔으면 비어 있다
journalctl -u prometheus -n 50 --no-pager
```

**1차 조치** — Prometheus 가 죽었으면 재시작한다. 규칙이 비어 있으면
`promtool check rules /etc/prometheus/rules/alerts.yml` 로 확인한 뒤 `systemctl reload prometheus`.
`activeAlertmanagers` 가 비었으면 Alertmanager 를 재시작한다.

**오탐 가능성** — 통지 경로가 멀쩡해도 Watchdog 은 디스코드에 오지 않는다(의도된 동작).
디스코드가 조용한 것 자체는 신호가 아니다. 반대로 **디스코드 통지만 끊긴 경우**는
Watchdog 으로 잡히지 않으므로, 아래 통지 실패 확인을 쓴다.

```bash
journalctl -u prometheus-alertmanager --since '-1h' --no-pager | grep -iE 'notify|error'
```

통지 실패는 로그에만 남는다. `amtool check-config` 도 Watchdog 도 잡지 못하고,
`unrecoverable` 이 붙은 실패는 재시도도 하지 않으므로 **알림이 조용히 사라진다.**

| 로그 | 원인 | 조치 |
|---|---|---|
| `open /etc/alertmanager/discord-webhook-url: permission denied` | webhook 파일 소유자. Alertmanager 는 `prometheus` 계정으로 돈다 | `chown prometheus:prometheus`, `chmod 0400` |
| `unexpected status code 400: {"attachments": ["0"]}` | attachment 안의 URL 이 거부된 것이다. 디스코드는 TLD 없는 호스트명 URL 을 받지 않는다 | 설정의 `title_link` 를 본다 (아래) |
| `unexpected status code 401`·`404` | 웹후크가 지워졌거나 URL 이 틀렸다 | 디스코드에서 웹후크 재생성 |

`400 {"attachments": ["0"]}` 이 나오면 로드된 설정의 `title_link` 를 확인한다.

```bash
curl -s localhost:9093/api/v2/status | jq -r .config.original | grep title_link
```

`{{ template "slack.default.titlelink" . }}` 로 보이면 그게 원인이다. 기본 템플릿은
`{{ .ExternalURL }}/#/alerts?receiver=...` 로 펼쳐지고, `--web.external-url` 이 없으면
`.ExternalURL` 이 호스트명(`http://astra-monitoring:9093/...`)이 된다. 설정에 IP 로 명시된
값(`http://192.168.100.107:9093/...`)이 보여야 정상이다. 2026-10-18·10-19 에 두 번 발생했다.

절차와 원인 분석은 orbit-observability `alertmanager/README.md` 와 같은 레포 #35 에 있다.

---

## TargetDown

**무엇이 깨졌나** — `up == 0` 이 10분 이상이다. node_exporter 중단, Alloy 의 remote write
중단, Prometheus 수신 거부, Alertmanager 자체 중단이 모두 이 알림으로 온다. 알림 label 의
`job`·`instance` 가 어디인지 알려준다.

**어디를 보나**

```bash
# 어떤 대상이 내려갔는지
curl -s 'localhost:9090/api/v1/query?query=up==0' | jq -r '.data.result[] | "\(.metric.job) \(.metric.instance)"'
```

`job` 에 따라 갈라진다.

| `job` | 의미 | 확인 |
| --- | --- | --- |
| `node` | node_exporter(5s scrape) | 해당 VM 에서 `systemctl status prometheus-node-exporter`, 모니터링 VM 에서 `curl -s <IP>:9100/metrics \| head -1` |
| `alertmanager` | 같은 VM 의 Alertmanager | `systemctl status prometheus-alertmanager` |
| `prometheus` | Prometheus 자신 | 이 값이 0 이면 Watchdog 절로 간다 |
| Alloy 가 붙인 job (kubelet·cAdvisor·kube-state-metrics·orbit-server) | 클러스터에서 remote write 로 들어오는 `up` | 아래 Alloy 절차 |

Alloy 쪽이면 클러스터에서 본다.

```bash
kubectl -n monitoring get pods -l app.kubernetes.io/name=alloy -o wide
kubectl -n monitoring logs -l app.kubernetes.io/name=alloy --tail=100 | grep -iE 'remote write|error|401|400'
```

**1차 조치**

- node_exporter 중단 → 해당 VM 에서 `sudo systemctl restart prometheus-node-exporter`
- VM 자체가 꺼짐 → ASTRA 호스트에서 `virsh list --all` 로 확인하고 기동
- Alloy 가 보내지 못함 → Alloy Pod 로그의 remote write 오류를 본다. 400 이면
  `RemoteWriteRejected` 절로 간다
- 9090 이 막힌 경우(새 출발지 추가 직후 등) → `sudo ufw status numbered | grep 9090` 으로
  출발지 allowlist 확인. Alloy 는 노드 IP 로 나가고, ASTRA 밖 장비는 `192.168.100.1` 로 보인다

**오탐 가능성** — VM 재부팅·노드 재시작 직후 10분 안에 저절로 복구되는 경우. `for: 10m` 으로
짧은 흔들림은 걸러지지만, 계획된 재부팅 전에는 silence 를 건다. `sol`·`terra`·`luna` 는 아직
수집 대상이 아니라 이 알림에 나타나지 않는다.

---

## RemoteWriteRejected

**무엇이 깨졌나** — Prometheus 가 remote write 요청을 4xx·5xx 로 거부하고 있다. **이 레포군이
반복해서 겪은 장애 유형이다.** label 허용목록에서 구분 label 이 빠지면 서로 다른 시계열이 같은
label set 으로 붕괴하고, `duplicate sample` 400 이 나면 Prometheus 2.45 는 그 write 배치를
통째로 Rollback 한다 — 정상 샘플까지 사라져 패널이 빈다
([orbit-gitops#35](https://github.com/ORBIT-CLOUD-LABS/orbit-gitops/issues/35),
[#38](https://github.com/ORBIT-CLOUD-LABS/orbit-gitops/issues/38)).

**어디를 보나**

```bash
# 어떤 코드로 거부되는지
curl -s 'localhost:9090/api/v1/query?query=sum by (code) (rate(prometheus_http_requests_total{handler="/api/v1/write",code=~"4..|5.."}[5m]))' | jq -c '.data.result[]'
journalctl -u prometheus --since '-30m' --no-pager | grep -iE 'duplicate|out of order|out of bounds'
kubectl -n monitoring logs -l app.kubernetes.io/name=alloy --tail=200 | grep -iE 'status=400|duplicate'
```

**1차 조치** — 최근 변경을 먼저 의심한다.

1. orbit-gitops `platform/alloy/config.alloy` 의 `labelkeep` 허용목록을 바꿨는가
   (`orbit_server_labels` / `limit_labels` / `cadvisor_select`)
2. 서버가 새 지표를 내보냈는가 — `/actuator/prometheus` 출력에서 시계열을 구분하는 label 을
   확인하고 `orbit_server_labels` 에 추가한다
3. k6 를 새 위치에서 돌렸는가 — `409`·`400` 이 아니라 거부가 아니면 ufw 출발지 문제다

허용목록 변경을 되돌리거나 빠진 label 을 추가해 PR 을 올린다. 복구 확인은
`up` 이 모든 대상에서 1 이고 Alloy 로그에 400 이 없는 것이다.

**오탐 가능성** — 재시작 직후 짧은 `out of order sample`, k6 run 종료 시점의 마지막 배치.
`for: 10m` 으로 대부분 걸러진다. 거부율이 낮아도(초당 1건 미만) 지속되면 label 붕괴가 특정
시계열에서만 일어나는 중일 수 있어 무시하지 않는다.

---

## PodNotReady

**무엇이 깨졌나** — `orbit` namespace 의 Pod 가 10분 이상 Ready 가 아니다.

**어디를 보나**

```bash
kubectl -n orbit get pods -o wide
kubectl -n orbit describe pod <pod>        # Events: 이미지 pull, probe 실패, 스케줄 실패
kubectl -n orbit logs <pod> --tail=100
kubectl -n orbit logs <pod> --previous --tail=100   # 재시작했다면
```

로그는 Loki 에도 있다. Grafana Explore → Loki:

```logql
{namespace="orbit", app="orbit-server"} |= "ERROR"
{namespace="orbit", pod="<pod>"}
```

배포 상태는 Argo CD 를 본다 (Application `orbit-astra-orbit-server`, namespace `argocd`).

```bash
kubectl -n argocd get applications.argoproj.io orbit-astra-orbit-server \
  -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'
```

**1차 조치**

- `startupProbe` 초과로 기동이 안 끝남 → 기동 시간을 본다. 권고 설정(`750m`)의 기동은
  25~28초이고, `500m`(request=limit)은 약 52초, `300m` 은 151초로 probe 를 넘긴다
  (`experiments/server-hpa.md` 7.3). CPU limit 이 의도한 값인지 확인한다
- 이미지 pull 실패 → GHCR 접근과 태그를 확인한다
- 스케줄 실패(`Insufficient cpu`) → `kubectl describe node` 로 노드 여유를 본다. 권고 설정은
  `requests ≠ limits`(Burstable)라 노드가 꽉 차면 limit 까지 못 올라간다
- 설정이 의도와 다르면 **클러스터에 직접 패치하지 않고** orbit-gitops 에 PR 을 올린다. 직접
  패치는 Argo CD 자동 sync 가 되돌린다 (`automated.enabled: true`, `selfHeal: false`)

**오탐 가능성** — 배포·`rollout restart` 직후 새 Pod 가 Ready 로 가는 동안. `for: 10m` 보다
오래 걸리는 경우는 위 기동 시간 문제로 본다. 복구 실험에서 Pod 를 일부러 삭제할 때도 뜨므로
실험 전에는 silence 를 건다 (`available` 2 → 1 → 2 복귀는 약 45초).

---

## DeploymentReplicasUnavailable

**무엇이 깨졌나** — `available` replica 가 `spec` 보다 적은 상태가 15분 이상이다.

**어디를 보나**

```bash
kubectl -n orbit get deploy,hpa
kubectl -n orbit describe hpa orbit-server     # 왜 그 숫자인지, scale 이벤트
kubectl -n orbit get events --sort-by=.lastTimestamp | tail -20
```

**1차 조치** — 개별 Pod 가 Ready 가 아니면 `PodNotReady` 절을 따른다. HPA 가 늘렸는데 새 Pod
가 안 뜨면 노드 여유(`Insufficient cpu`)를 본다. HPA 범위는 2~4, 목표 CPU 70% 가 확정값이므로
숫자가 다르면 drift 를 의심한다 (Argo CD sync 상태 확인).

**오탐 가능성** — HPA scale-out 중에는 정상이다. 그래서 `for` 를 15분으로 뒀다. scale-out
직후에는 새 Pod 예열 때문에 4~6분 전환 구간이 있다. 부하 실험 중에는 대부분 오탐이다.

---

## ContainerRestartingRepeatedly

**무엇이 깨졌나** — 15분 동안 컨테이너 재시작이 2회를 넘었다. CrashLoop 를 재시작 횟수로 잡는
규칙이다.

**어디를 보나**

```bash
kubectl -n orbit get pods            # RESTARTS 열
kubectl -n orbit logs <pod> --previous --tail=200    # 죽기 직전 로그가 여기 있다
kubectl -n orbit describe pod <pod> | sed -n '/Last State/,/Ready/p'   # OOMKilled / Error / exit code
```

Loki 에서 죽은 Pod 의 로그를 시간 범위로 본다 (`--previous` 가 없는 경우에도 남아 있다).

```logql
{namespace="orbit", app="orbit-server"} |~ "ERROR|Exception|OutOfMemory"
```

**1차 조치**

- `OOMKilled` → 메모리 limit 과 JVM heap 설정을 함께 본다. 서버는 CPU limit 이 2 CPU 미만일
  때 Serial GC 를 고르므로 limit 을 바꾸면 GC 종류와 메모리 거동이 함께 바뀐다
- 애플리케이션 예외로 종료 → 로그의 스택을 orbit-server 이슈로 올린다
- 설정 원인이면 orbit-gitops PR 로 고친다 (직접 패치 금지)

**오탐 가능성** — 이미지 태그를 바꿔 연속 배포한 직후, 또는 노드 재시작으로 Pod 가 함께
재생성된 경우. 강제 삭제(`--grace-period=0 --force`) 실험에서도 올라간다.

---

## ContainerCpuThrottlingHigh

**무엇이 깨졌나** — CFS throttling 비율이 25% 를 15분 이상 넘었다. CPU limit 이 작아 컨테이너가
주기마다 멈추고 있다는 뜻이다.

**어디를 보나** — Grafana `load-recovery-experiment` 대시보드의 **Pod CPU·throttling** 패널.
또는:

```promql
rate(container_cpu_cfs_throttled_periods_total{namespace="orbit",container!=""}[5m])
  / rate(container_cpu_cfs_periods_total{namespace="orbit",container!=""}[5m])
```

```bash
kubectl -n orbit get pod <pod> -o jsonpath='{.spec.containers[0].resources}{"\n"}'
```

**정상 범위** — 확정 설정은 request `500m` / limit `750m` 다. 예열된 Pod 는 한계 근처에서도
throttling 이 지속되지 않는다. 고정 2개가 약 715 req/s 를 받아 무너진 run 에서는 throttling
80~99% 였다 (`experiments/server-hpa.md` 8 절 `194408Z`).

**1차 조치** — 부하가 평상시 수준인데도 지속되면 limit 이 작다는 신호다. 처리량·P95 와 함께
읽고, 바꿀 때는 orbit-gitops PR 로 올린다. 확정값을 바꾸는 판단은 `experiments/server-hpa.md`
7.3 의 비교(500m 기각, 1000m 기각, 750m 채택) 근거를 다시 본다.

**오탐 가능성 — 높다.**

- **부하 실험 중에는 의도된 상태다.** 실험 전에 silence 를 건다
- **기동 직후 Pod 는 JIT 컴파일로 CPU 를 많이 쓴다.** `750m` 에서도 기동 직후 P95 가 최대
  442ms 까지 올라갔다가 2분 안에 회복한다. scale-out·Pod 복구 직후 몇 분은 정상 거동이다

---

## VehicleApiErrorRateHigh

**무엇이 깨졌나** — 차량 API(`/api/v1/vehicles/{vehicleId}/...`) 5xx 비율이 1% 를 10분 이상
넘었다. health probe 와 `/actuator/prometheus` 는 제외된 기준이다.

**어디를 보나** — Grafana **차량 API 요청률·오류율** 패널. 그리고 어떤 `uri`·`status` 인지:

```promql
sum by (uri, status) (rate(http_server_requests_seconds_count{uri=~"/api/v1/vehicles/[{]vehicleId[}]/.*",status=~"5.."}[5m]))
```

```logql
{namespace="orbit", app="orbit-server"} |= "ERROR" | json | line_format "{{.message}}"
```

**1차 조치**

1. Pod 상태를 먼저 본다 (`PodNotReady`·재시작 여부). Pod 가 빠지는 중이면 그게 원인이다
2. DB 쪽을 본다 — `HikariConnectionsPending` 절의 분기를 따른다
3. 특정 `uri` 에만 몰리면 서버 코드 문제로 보고 orbit-server 이슈로 올린다

**오탐 가능성** — Pod 정상 삭제·강제 삭제 구간의 연결 실패. 강제 종료는 진행 중이던 연결만큼
실패하고(약 100건, 2초) 설정으로 줄일 수 있는 값이 아니다. k6 가 fixture 를 만들다 낸 4xx 는
이 규칙에 들어오지 않는다(5xx 만 본다).

---

## VehicleApiLatencyP95High

**무엇이 깨졌나** — 차량 API P95 가 1초를 10분 이상 넘었다. 임계값 1초는 1차 제안값이고,
실험 판정 기준은 500ms 였다 (`experiments/server-hpa.md` 1 절).

**어디를 보나** — Grafana **차량 API P95** 패널.

```promql
histogram_quantile(0.95, sum by (le) (rate(http_server_requests_seconds_bucket{uri=~"/api/v1/vehicles/[{]vehicleId[}]/.*"}[5m])))
```

**정상 범위** — 예열된 상태에서 낮은 부하면 6~9ms, 500 req/s 부근까지 한 자리 ms 다. 한계
근처(약 715 req/s, 고정 2개)에서 1.1~1.4초로 올라간다. scale-out 전환 구간에서는 4~6분 동안
최대 2.2~2.3초까지 흔들린 뒤 회복한다.

**1차 조치** — 세 갈래로 가른다.

| 함께 보이는 것 | 해석 | 다음 |
| --- | --- | --- |
| throttling 높음, Hikari 대기 0 | 서버 CPU 가 병목 | `ContainerCpuThrottlingHigh` 절 |
| Hikari 대기 쌓임 | DB 또는 커넥션 풀이 병목 | `HikariConnectionsPending` 절 |
| 둘 다 정상인데 느림 | 외부 요인(네트워크·NFS·Origin) | 노드·VM 디스크 I/O 패널 |

**오탐 가능성** — 부하 실험 중, 배포·scale-out 직후 전환 구간. 요청량이 아주 적을 때는 몇 건의
느린 요청이 P95 를 끌어올린다.

---

## HikariConnectionsPending

**무엇이 깨졌나** — DB 커넥션 풀 대기가 10분 이상 0 보다 크다. 요청이 커넥션을 기다리고 있다.

**어디를 보나** — Grafana **Hikari**, **DB VM CPU·디스크 쓰기** 패널.

```promql
hikaricp_connections_pending
hikaricp_connections_active
```

DB VM(`192.168.100.106`)은 **user·system 과 iowait 을 나눠서** 본다. `100 - idle` 은 디스크
대기까지 사용 중으로 세기 때문에 "CPU 96~100%" 로 보일 수 있다.

**정상 범위·판정 기준** (`experiments/server-hpa.md` 7.1, DB 2 vCPU)

| 서버 처리량 | DB CPU | Hikari 대기 |
| --- | --- | --- |
| ~820 req/s | 62% 이하 | 0 |
| 916~996 req/s | 78~93% | 0~9 |
| 1,025 req/s 이상 | 94% 이상 | 89~132 (여기가 `T_db` ≈ 1,000) |

**1차 조치**

1. 처리량이 1,000 req/s 근처인가 → DB CPU 한계다. 부하를 줄이거나 DB vCPU 를 늘리는 판단으로
   간다 (orbit-infrastructure)
2. 처리량이 낮은데 대기가 쌓였나 → **기동 직후 Pod** 를 의심한다. JIT 컴파일로 응답이 늦어져
   Pod 당 5개뿐인 커넥션을 오래 잡으면 대기가 쌓이고, 부하가 유지되는 동안 풀리지 않는다.
   `kubectl -n orbit get pods` 로 Pod 나이를 본다
3. iowait 가 큰가 → DB 디스크 쓰기 대기다. 현재 설정
   (`innodb_flush_log_at_trx_commit=2`, `sync_binlog=0`)에서는 iowait 가 1% 미만이어야 한다.
   크면 DB 설정이 되돌아갔는지 확인한다

**오탐 가능성** — scale-out 직후·배포 직후 새 Pod 예열 구간(2분 내 회복). 부하 실험의 한계
측정 구간에서는 의도된 상태다.

---

## NodeDiskSpaceLow

**무엇이 깨졌나** — 어느 VM 의 파일시스템 여유가 15% 아래로 30분 이상 유지됐다. 알림 label 의
`instance`·`mountpoint` 가 어디인지 알려준다.

**어디를 보나**

```bash
df -h
du -sh /var/lib/prometheus /var/lib/loki /srv/monitoring/* 2>/dev/null | sort -h | tail
```

모니터링 VM 은 Prometheus TSDB(보존 15일, `--storage.tsdb.retention.time=15d`)와 Loki 가
차지한다. 두 경로의 실제 위치를 먼저 확인한다 — 데이터 디스크(`/srv/monitoring`, 20GiB)가
아니라 루트에 쌓여 있으면 보존 기간을 잡은 전제가 깨진다.

```bash
readlink -f /var/lib/prometheus
systemctl cat prometheus | grep -i 'storage.tsdb.path\|ARGS'
grep -nE 'path_prefix|working_directory|filesystem' /etc/loki/config.yml
```

**1차 조치**

- 로그·TSDB 가 아닌 것이 찼으면(`/var/log`, apt 캐시, k6 결과 파일) 그것을 지운다
- TSDB 가 큰 경우 보존 기간을 줄인다. 실험 기간을 감안해 15일로 잡았으므로 줄이기 전에 실험
  일정을 확인한다
- Loki 가 큰 경우 retention 설정을 확인한다
- 다른 VM(worker·DB·NFS)이면 이미지·컨테이너 로그·업데이트 파일을 본다

**오탐 가능성** — k6 run 결과나 백업을 잠깐 올려둔 경우. `/boot` 는 용량이 작아(881M) 비율
기준으로 쉽게 걸린다 — 커널 패키지 정리로 해결되고 서비스 영향은 없다.

---

## 알림을 바꿀 때

- 규칙·임계값 변경은 orbit-observability `prometheus/rules/alerts.yml` (PR 필요)
- 이 문서의 절 제목은 알림 이름과 1:1 이고 `runbook_url` 이 그 anchor 를 가리킨다. **절 제목을
  바꾸면 알림의 링크가 깨진다**
- 규칙을 추가하면 이 문서에도 절을 추가한다. 네 항목(무엇이 깨졌나 / 어디를 보나 / 1차 조치 /
  오탐 가능성)을 채운다
- 쓸 수 있는 지표·label 은 Alloy 허용목록에 제한된다. orbit-observability
  `docs/monitoring-targets.md` §알림 규칙 을 본다
