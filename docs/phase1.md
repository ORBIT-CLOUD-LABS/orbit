# Phase1 범위와 공통 값

- 근거: Notion `99-공통 용어집` (팀 공통 용어와 Phase1 설계 배경)
- 진행 관리: [orbit#25 Phase1 배포·관측·부하 실험 로드맵](https://github.com/ORBIT-CLOUD-LABS/orbit/issues/25)
- 상태: 구성 확정, 공통 값 확정 (아래 §4)
- 최종 수정: 2026-10-18 (orbit#23 측정 결과와 머지된 PR 반영)

Phase1 작업은 이 문서를 먼저 읽는다. 다른 문서(`flow.md`, `api.md`, `infrastructure-plan.md`)와 내용이 다르면 이 문서를 따르고, 다른 문서를 고치는 Issue를 만든다.

Notion 은 사람이 읽는 배경 자료다. 구현이 따를 값과 범위는 이 문서에 둔다.

## 1. 목표

ASTRA 한 대 위의 Kubernetes 에 OTA 서버와 펌웨어 전달 계층(CDN)을 배포하고, 다음을 검증한다.

- 서버 Pod 가 CPU 부하에 따라 HPA 로 늘고 줄어드는지, 늘어난 만큼 처리량이 개선되는지
- 서버·Edge Pod 가 삭제됐을 때 서비스가 복구되는지
- Edge 캐시가 Origin 요청을 줄이는지

## 2. 구성

```text
관리자 ──업로드──▶ OTA 서버 Pod (2~4, HPA) ──쓰기──▶ astra-nfs (원본)
                        │                                  │ 읽기 전용
차량·k6 ──체크인·매니페스트──┘                         Origin Pod (1, Nginx)
                                                           ▲ MISS
차량·k6 ──────────────다운로드──────────────▶ Edge Pod (2, Nginx, Pod별 캐시)
```

| 구성요소 | 위치 | 개수 | 노출 |
|---|---|---|---|
| OTA 서버 | Kubernetes `orbit` namespace | 2~4 (CPU HPA) | NodePort |
| Origin | Kubernetes `orbit` namespace, Nginx | 1 (고정) | ClusterIP |
| Edge | Kubernetes `orbit` namespace, Nginx | 2 (고정) | NodePort |
| 펌웨어 원본 | `astra-nfs` VM, NFS export | 1 | worker 노드에만 공유 |
| DB | `astra-vehicle-db` VM, MySQL 8 | 1 | 3306 |
| 메트릭·로그 저장 | `astra-monitoring` VM, Prometheus·Loki·Grafana | 1 | 9090, 3100, 3000 |
| 메트릭·로그 수집 | Kubernetes, Alloy·kube-state-metrics | - | remote write·push |

- 서버는 펌웨어를 NFS 마운트 경로에 직접 쓴다. S3 API(SeaweedFS)는 쓰지 않는다.
- Origin 은 같은 NFS 원본을 읽기 전용으로 마운트해 HTTP 로 제공한다.
- Edge 는 NFS 를 마운트하지 않는다. Origin 의 HTTP 응답을 Pod 별 `emptyDir` 캐시에 저장하며, 두 Edge 는 캐시를 공유하지 않는다.
- 같은 클러스터 안의 캐시 계층을 모사한 것이며 지리 분산 CDN 이 아니다.
- 외부 진입점(Gateway API·Ingress)은 두지 않는다. 클라이언트가 서버 주소와 파일 주소를 따로 사용하므로 경로 분기가 필요 없다.
- 배포는 Argo CD 가 `orbit-gitops` 를 읽어 반영한다. Phase1 실험 동안은 수동 Sync 를 유지하고 이미지·설정을 고정한다.
- worker 3대 모두 일반 Pod 를 받는다. 노드를 늘려도 ASTRA 의 물리 CPU(12 스레드)는 늘지 않는다.

VM·IP·사양은 [infrastructure-plan.md](infrastructure-plan.md).

## 3. 완료 기준

| 단계 | 기준 |
|---|---|
| 기본 배포 | 서버 2·Origin 1·Edge 2 Ready, 실제 API 호출과 파일 다운로드 확인 |
| 관측 연결 | 서버·Pod·VM 메트릭과 서버·Origin·Edge 로그가 실제 수집됨 |
| 서버 실험 | 고정 replica 와 HPA 비교, 서버 Pod 삭제 후 복구 결과와 판정 기록 |
| CDN 실험 | cold·warm 캐시, Edge 1·2개 비교, Edge 삭제 후 복구 결과와 판정 기록 |

- "replica 가 늘었다"와 "처리량·P95 가 개선됐다"는 따로 판정한다. 개선이 없으면 원인을 기록하고 성공으로 바꾸지 않는다.
- 서버 HPA 실험은 서버 Pod 가 DB 보다 먼저 병목이 되는 조건에서 한다. 사전 측정으로 `T_db`·`T_pod` 를 재고 서버 CPU 값을 정했다 (orbit#23, [experiments/server-hpa.md](../experiments/server-hpa.md)).
- 단일 ASTRA·DB·NFS·Origin 을 남기므로 결과를 전체 시스템 고가용성(HA)으로 표현하지 않는다.

## 4. 공통 값

여러 저장소가 함께 쓰는 값이다. 바꾸면 이 표와 영향받는 저장소를 함께 고친다. **미확정** 값은 제안값이며, 확정되면 표시를 지운다.

| 값 | 값 | 상태 | 쓰는 곳 |
|---|---|---|---|
| 앱 namespace | `orbit` | 확정 | gitops, 수집기 |
| 수집기 namespace | `observability` | 확정 | gitops |
| 서버 NodePort | `30080` | 확정 | gitops, k6 |
| Edge NodePort | `30081` | 확정 | gitops, `CDN_BASE_URL`, k6 |
| `CDN_BASE_URL` | `http://192.168.100.102:30081` (worker-1) | 확정 | gitops (서버 ConfigMap) |
| NFS 서버 | `192.168.100.105` | 확정 | infrastructure, gitops |
| NFS export 경로 | `/srv/nfs/origin` | 확정 | infrastructure, gitops |
| 원본 파일 경로 규칙 | `/firmware/{model}/{version}/firmware.bin` | 확정 | 서버, Origin |
| 서버 원본 루트 환경 변수 | `ORIGIN_ROOT_PATH` (Pod 안 마운트 경로 `/data/origin`) | 확정 | 서버, gitops |
| PVC (Origin, 읽기 전용) | `origin-files` | 확정 | gitops |
| PVC (서버, 읽기·쓰기) | `origin-files-rw` | 확정 | gitops |
| `DB_URL` | `jdbc:mysql://192.168.100.106:3306/vehicle` | 확정 | gitops (서버 ConfigMap) |
| `DB_USERNAME` | `vehicle_app` (worker `192.168.100.102~104` 에서만 접속 허용, ORBIT-CLOUD-LABS/orbit-infrastructure#44) | 확정 | infrastructure, gitops (서버 ConfigMap) |
| 서버 메트릭 | `/actuator/prometheus`, `8080` (클러스터 안 Alloy 가 Pod IP 를 30초마다 scrape 해 모니터링 VM 으로 remote write) | 확정 | 서버, 수집기 |
| 서버 메모리 request·limit | `768Mi` (이 크기에서는 CPU 와 무관하게 JVM Serial GC) | 확정 | gitops |
| 서버 CPU request·limit | request `500m` / limit `750m` (orbit#23 측정, ORBIT-CLOUD-LABS/orbit-gitops#42) | 확정 | gitops |
| HPA 목표 CPU 사용률 | `70%` (request 500m 기준) | 확정 | gitops |
| 서버 HPA replica | 최소 2, 최대 4 | 확정 | gitops |
| Hikari 연결 pool | `5`/Pod | 확정 | gitops (서버 ConfigMap) |
| Vehicle DB VM vCPU | `2` (2026-10-17 증설, ORBIT-CLOUD-LABS/orbit-infrastructure#65) | 확정 | infrastructure, 실험 |
| DB commit 디스크 동기화 | `innodb_flush_log_at_trx_commit=2`, `sync_binlog=0` (ORBIT-CLOUD-LABS/orbit-infrastructure#63) | 확정 | infrastructure, 실험 |
| 부하 발생기(k6) 위치 | ASTRA 호스트, 서버 실험 `taskset -c 10,11`, CDN 실험 `taskset -c 8-11` | 확정 | 실험 |

- **서버 CPU**: orbit#23 측정으로 request `500m` / limit `750m` 로 확정했다 (ORBIT-CLOUD-LABS/orbit-gitops#42, PR ORBIT-CLOUD-LABS/orbit-gitops#43). request=limit `500m` 는 막 기동한 Pod 가 Pod 당 약 55 req/s 에서도 무너져 회복하지 못하고(JIT 컴파일이 CPU 상한에 막혀 DB 연결을 오래 잡는다), limit `1000m` 은 Pod 2개가 DB 한계 근처까지 처리해 HPA 효과가 드러나지 않는다. requests ≠ limits 라 Pod QoS 는 Burstable 이다.
- **HPA 목표 70%**: 목표 50% 와 비교해 scale-out 시점(부하 상승 약 30초 뒤)이 같고 전환 구간이 더 짧지 않아 배포값을 유지한다.
- **DB commit 동기화 완화**는 DB VM·ASTRA 호스트가 갑자기 꺼질 때 최대 약 1초치 commit 유실을 허용한다는 뜻이다 (MySQL 프로세스만 죽으면 유실 없음). 실험 결과는 이 조건에서만 주장한다.
- **측정 중에는 orbit-gitops `main` 머지를 멈춘다.** Argo CD auto-sync(selfHeal 꺼짐)는 앱과 무관한 커밋이라도 `main` 에 새 커밋이 들어온 뒤 첫 `kubectl` 임시 변경을 되돌린다. 임시 패치 뒤 Argo CD 로그에 `Skipping auto-sync` 가 찍히는 것을 확인하고 측정을 시작한다.

## 5. 범위 밖

- Gateway API·Ingress·LoadBalancer 등 외부 진입점
- CDN HPA (고정 replica 비교 결과를 보고 판단)
- Argo CD 자동 Sync·drift 복구 실험 (CI/CD 파이프라인 [orbit-server#61](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/61) 에서 결정)
- Secret 정식 관리 방식 (Sealed Secrets·External Secrets 등)
- CDN 서명 URL, Ed25519 파일 서명 (Phase2)
- 실차·Agent 설치 성공 검증
- SOL·luna 호스트 사용
