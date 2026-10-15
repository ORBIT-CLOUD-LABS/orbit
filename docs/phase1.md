# Phase1 범위와 공통 값

- 근거: Notion `99-공통 용어집` (팀 공통 용어와 Phase1 설계 배경)
- 진행 관리: [orbit#25 Phase1 배포·관측·부하 실험 로드맵](https://github.com/ORBIT-CLOUD-LABS/orbit/issues/25)
- 상태: 구성 확정, 공통 값 일부 미확정 (아래 §4)

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
- 서버 HPA 실험은 서버 Pod 가 DB 보다 먼저 병목이 되는 조건에서 한다. 서버 CPU limit 은 사전 측정으로 정한다 (orbit#23).
- 단일 ASTRA·DB·NFS·Origin 을 남기므로 결과를 전체 시스템 고가용성(HA)으로 표현하지 않는다.

## 4. 공통 값

여러 저장소가 함께 쓰는 값이다. 바꾸면 이 표와 영향받는 저장소를 함께 고친다. **미확정** 값은 제안값이며, 확정되면 표시를 지운다.

| 값 | 값 | 상태 | 쓰는 곳 |
|---|---|---|---|
| 앱 namespace | `orbit` | 확정 | gitops, 수집기 |
| 수집기 namespace | `observability` | 미확정 | gitops |
| 서버 NodePort | `30080` | 미확정 | gitops, k6 |
| Edge NodePort | `30081` | 미확정 | gitops, `CDN_BASE_URL`, k6 |
| `CDN_BASE_URL` | `http://<worker IP>:30081` | 미확정 | gitops (서버 ConfigMap) |
| NFS 서버 | `192.168.100.105` | 확정 | infrastructure, gitops |
| NFS export 경로 | `/srv/nfs/origin` | 미확정 | infrastructure, gitops |
| 원본 파일 경로 규칙 | `/firmware/{model}/{version}/firmware.bin` | 확정 | 서버, Origin |
| 서버 원본 루트 환경 변수 | `ORIGIN_ROOT_PATH` | 미확정 | 서버, gitops |
| PVC (Origin, 읽기 전용) | `origin-files` | 미확정 | gitops |
| PVC (서버, 읽기·쓰기) | `origin-files-rw` | 미확정 | gitops |
| `DB_URL` | `jdbc:mysql://192.168.100.106:3306/vehicle` | 확정 | gitops (서버 ConfigMap) |
| `DB_USERNAME` | `vehicle_app` (`192.168.100.%` 에서 접속 허용) | 확정 | gitops (서버 ConfigMap) |
| 서버 메트릭 | `/actuator/prometheus`, `8080` | 미확정 | 서버, 수집기 |
| 서버 CPU request·limit, HPA 목표 | 사전 측정 후 결정 | 미확정 | gitops |
| 서버 HPA replica | 최소 2, 최대 4 | 확정 | gitops |
| 부하 발생기(k6) 위치 | ASTRA 밖 호스트 | 미확정 | 실험 |

## 5. 범위 밖

- Gateway API·Ingress·LoadBalancer 등 외부 진입점
- CDN HPA (고정 replica 비교 결과를 보고 판단)
- Argo CD 자동 Sync·drift 복구 실험 (CI/CD 파이프라인 [orbit-server#61](https://github.com/ORBIT-CLOUD-LABS/orbit-server/issues/61) 에서 결정)
- Secret 정식 관리 방식 (Sealed Secrets·External Secrets 등)
- CDN 서명 URL, Ed25519 파일 서명 (Phase2)
- 실차·Agent 설치 성공 검증
- SOL·luna 호스트 사용
