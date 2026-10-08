# 호스트별 VM 사양

| 호스트 | 호스트 IP | 하드웨어 | VM 대역 | 역할 |
|---|---|---|---|---|
| orbit-astra | `192.168.200.200` | i5-12400 (6C/12T), 66 GB, SSD 500 GB | `192.168.100.0/24` | 운영 환경 (k8s, Jenkins, DB, 모니터링) |
| orbit-sol | `192.168.200.201` | i5-1235U (10C/12T), 32 GB, 210 GB | `192.168.101.0/24` | 파일 전달 계층 (CDN + Origin) |
| orbit-terra | `192.168.200.202` | i5-1235U (10C/12T), 32 GB, 210 GB | `192.168.102.0/24` | develop 서버 (astra 하위 버전) |
| orbit-luna | 미정 | 미정 | `192.168.103.0/24` | 차량 시뮬레이터 |

호스트는 `192.168.200.0/24` 대역에 고정 IP를 두고, VM은 호스트별 대역(`192.168.10x.0/24`)을 쓴다.

각 호스트는 OS·하이퍼바이저용으로 메모리 약 6 GB 이상, 디스크 약 30 GB 이상을 남긴다 (astra는 메모리 12~16 GB, 디스크 100 GB 이상).

## orbit-astra

| VM | 역할 | vCPU | RAM | OS Disk | Data Disk | IP |
|---|---|---:|---:|---:|---:|---|
| `astra-jenkins` | CI/CD, 이미지 레지스트리 | 2 | 6 GB | 35 GB | - | `192.168.100.100` |
| `astra-control-plane` | Kubernetes control plane | 2 | 4 GB | 25 GB | - | `192.168.100.101` |
| `astra-worker-1` | Kubernetes worker | 2 | 8 GB | 35 GB | - | `192.168.100.102` |
| `astra-worker-2` | Kubernetes worker | 2 | 8 GB | 35 GB | - | `192.168.100.103` |
| `astra-worker-3` | Kubernetes worker | 2 | 8 GB | 35 GB | - | `192.168.100.104` |
| `astra-nfs` | k8s 공유 PV | 1 | 4 GB | 20 GB | 45 GB | `192.168.100.105` |
| `astra-vehicle-db` | MySQL 8 (`ota` DB: 캠페인 + 차량) | 1 | 4 GB | 20 GB | 15 GB | `192.168.100.106` |
| `astra-monitoring` | Prometheus, Loki, Grafana | 2 | 8 GB | 25 GB | 20 GB | `192.168.100.107` |
| **합계** | **8대** | **14** | **50 GB** | **230 GB** | **80 GB** |  |

## orbit-sol

| VM | 역할 | vCPU | RAM | OS Disk | Data Disk | IP |
|---|---|---:|---:|---:|---:|---|
| `sol-cdn` | nginx 캐시·서명 URL 검증·펌웨어 전달 | 4 | 6 GB | 20 GB | 50 GB (캐시) | `192.168.101.100` |
| `sol-origin` | MinIO 원본 펌웨어 저장소 | 2 | 4 GB | 20 GB | 80 GB | `192.168.101.101` |
| **합계** | **2대** | **6** | **10 GB** | **40 GB** | **130 GB** |  |

- 디스크 210 GB 중 VM 170 GB, 호스트 여유 40 GB.
- Origin 데이터 디스크가 80 GB로 제한된다. 펌웨어 보관 버전 수를 이 안에서 정하고, 오래된 버전은 sol 밖 백업으로 옮긴다.
- CDN은 차량 수백 대의 동시 다운로드를 받으므로 vCPU·RAM을 Origin보다 많이 준다. 메모리가 남으므로(약 16 GB) 부하 시험 결과에 따라 `sol-cdn`을 늘린다.

## orbit-terra (아직 vm 을 어떻게 배치할지 합의해야함)

astra와 같은 구성을 사양만 줄여 둔다. Jenkins는 두지 않고 `astra-jenkins`가 terra에도 배포한다. IP 끝자리는 astra와 맞춘다.

| VM | 역할 | vCPU | RAM | OS Disk | Data Disk | IP |
|---|---|---:|---:|---:|---:|---|
| `terra-control-plane` | Kubernetes control plane | 2 | 4 GB | 20 GB | - | `192.168.102.101` |
| `terra-worker-1` | Kubernetes worker | 2 | 6 GB | 25 GB | - | `192.168.102.102` |
| `terra-worker-2` | Kubernetes worker | 2 | 6 GB | 25 GB | - | `192.168.102.103` |
| `terra-nfs` | k8s 공유 PV | 1 | 2 GB | 20 GB | 20 GB | `192.168.102.105` |
| `terra-vehicle-db` | MySQL 8 (`ota` DB: 캠페인 + 차량) | 1 | 3 GB | 20 GB | 10 GB | `192.168.102.106` |
| `terra-monitoring` | Prometheus, Loki, Grafana | 2 | 4 GB | 20 GB | 10 GB | `192.168.102.107` |
| **합계** | **6대** | **10** | **25 GB** | **130 GB** | **40 GB** |  |

- 메모리 32 GB 중 VM 25 GB, 호스트 여유 7 GB.
- 디스크 210 GB 중 VM 170 GB, 호스트 여유 40 GB.
- worker는 2대로 줄였다. 여러 노드에 걸친 스케줄링과 NFS PV 마운트를 develop에서도 확인하기 위해 1대가 아니라 2대로 둔다.

## orbit-luna

차량 수백 대 기준으로 재산정 중이다. 목표 차량 수, 펌웨어 최대 크기, luna 하드웨어 사양이 정해지면 채운다.

| VM | 역할 | vCPU | RAM | Disk | IP |
|---|---|---:|---:|---:|---|
| `luna-vehicle-sim` | 차량 시뮬레이터 컨테이너 실행 | 미정 | 미정 | 미정 | `192.168.103.100` |

- 산정 기준(안): 차량당 RAM 32~64 MB, CPU는 다운로드·검증 때만 사용. 디스크는 이미지·로그 30 GB + 펌웨어 크기 × 동시 다운로드 수.
- 예: 300대 → RAM 10~20 GB, vCPU 8 내외.
