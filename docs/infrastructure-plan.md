# 호스트별 VM 사양

Phase1 은 orbit-astra 한 대만 사용한다. 범위와 공통 값은 [phase1.md](phase1.md).

| 호스트 | 호스트 IP | 하드웨어 | VM 대역 | Phase1 역할 |
|---|---|---|---|---|
| orbit-astra | `192.168.200.200` | i5-12400 (6C/12T), 66 GB, SSD 500 GB | `192.168.100.0/24` | Kubernetes(OTA 서버·CDN), NFS, DB, 모니터링, Jenkins |
| orbit-sol | `192.168.200.201` | i5-1235U (10C/12T), 32 GB, 210 GB | `192.168.101.0/24` | 사용하지 않음 |
| orbit-terra | `192.168.200.202` | i5-1235U (10C/12T), 32 GB, 210 GB | - | develop 서버 (VM 없음, #13) |
| orbit-luna | 미정 | 미정 | `192.168.103.0/24` | 사용하지 않음 |

호스트는 `192.168.200.0/24` 대역에 고정 IP를 두고, VM은 호스트별 대역(`192.168.10x.0/24`)을 쓴다.

## orbit-astra

| VM | 역할 | vCPU | RAM | OS Disk | Data Disk | IP |
|---|---|---:|---:|---:|---:|---|
| `astra-jenkins` | Terraform plan/apply 파이프라인 | 2 | 6 GB | 35 GB | - | `192.168.100.100` |
| `astra-control-plane` | Kubernetes control plane | 2 | 4 GB | 25 GB | - | `192.168.100.101` |
| `astra-worker-1` | Kubernetes worker | 2 | 8 GB | 35 GB | - | `192.168.100.102` |
| `astra-worker-2` | Kubernetes worker | 2 | 8 GB | 35 GB | - | `192.168.100.103` |
| `astra-worker-3` | Kubernetes worker | 2 | 8 GB | 35 GB | - | `192.168.100.104` |
| `astra-nfs` | 펌웨어 원본 NFS export | 1 | 4 GB | 20 GB | 45 GB | `192.168.100.105` |
| `astra-vehicle-db` | MySQL 8 (OTA 서버 DB) | 1 | 4 GB | 20 GB | 15 GB | `192.168.100.106` |
| `astra-monitoring` | Prometheus, Loki, Grafana | 2 | 8 GB | 25 GB | 20 GB | `192.168.100.107` |
| **합계** | **8대** | **14** | **50 GB** | **230 GB** | **80 GB** |  |

- VM vCPU 합계(14)가 호스트 스레드(12)보다 많다. 부하 시험 중에는 VM 끼리 물리 CPU 를 나눠 쓰므로 호스트 CPU 사용률을 함께 기록한다.
- VM 을 나눠도 호스트의 전원·디스크·CPU 장애는 격리되지 않는다.

### Kubernetes

| 항목 | 값 |
|---|---|
| 버전 | v1.33 (kubeadm), containerd, Calico |
| 노드 | control plane 1 (`NoSchedule`), worker 3 |
| 배포 | Argo CD 가 `orbit-gitops` `main` 을 읽어 반영 (수동 Sync) |
| 이미지 | GHCR `ghcr.io/orbit-cloud-labs/*` (Public) |

worker 3대 모두 일반 Pod 를 받는다. `astra-worker-3` 의 예비 노드 taint 는 해제한다 (orbit-infrastructure#46).

| 워크로드 | namespace | 개수 | 노출 |
|---|---|---|---|
| OTA 서버 | `orbit` | 2~4 (CPU HPA) | NodePort |
| Origin (Nginx) | `orbit` | 1 | ClusterIP |
| Edge (Nginx) | `orbit` | 2 | NodePort |
| Argo CD | `argocd` | - | - |
| metrics-server | `kube-system` | 1 | - |
| 수집기 (Alloy, kube-state-metrics) | `observability` | - | - |

### 펌웨어 원본과 CDN

- 원본은 `astra-nfs` 의 데이터 디스크(45 GB)에 두고 worker 노드에 NFS 로 export 한다.
- OTA 서버는 업로드한 파일을 NFS 마운트 경로에 직접 쓴다. Origin Pod 는 같은 원본을 읽기 전용으로 마운트해 HTTP 로 제공한다.
- Edge Pod 는 Origin 에서 받은 응답을 Pod 별 `emptyDir` 캐시에 저장한다. Edge 는 NFS 를 마운트하지 않는다.
- 원본을 Pod 밖에 두어, Origin Pod 가 다른 worker 에 다시 생겨도 같은 파일을 읽는다. NFS 는 단일 장애점이며 백업은 Phase1 범위 밖이다.

### DB

- MySQL 8, 데이터 디스크 `/srv/db-data` (15 GB), `max_connections` 200.
- 서버 Pod 수 × 연결 pool 크기가 `max_connections` 안에 여유 있게 들어와야 한다. vCPU 가 1개라 서버 HPA 실험에서 먼저 병목이 될 수 있다.

### 모니터링

| 서비스 | 포트 | 수집 대상 |
|---|---|---|
| Prometheus | 9090 | 클러스터 수집기의 remote write, VM node_exporter |
| Loki | 3100 | 클러스터 수집기가 보낸 Pod 로그 |
| Grafana | 3000 | Prometheus·Loki 조회 |
| node_exporter | 9100 | ASTRA 호스트와 각 VM |

## 이전 구성 (Phase1 에서 사용하지 않음)

Phase1 을 Notion `99-공통 용어집` 기준으로 정하기 전에는 파일 전달 계층을 orbit-sol 에 두었다. 이전 문서는 태그 `pre-phase1` 에 남아 있다.

- `sol-cdn`: nginx 캐시·서명 URL 검증, `sol-origin`: SeaweedFS(S3 호환) 원본 저장소 (#10)
- Phase1 에서는 원본을 `astra-nfs` 로, CDN 을 클러스터 안 Origin·Edge 로 대체했다. 서버는 S3 API 대신 NFS 에 직접 쓴다.
- orbit-luna(차량 시뮬레이터)는 Phase1 에서 쓰지 않는다. 부하는 k6 로 건다.
