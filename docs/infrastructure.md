# 인프라 구성 (Phase2)

- 근거: Notion `서버 별 노드 구성 및 정보`
- 작성일: 2026-10-28

## 1. 전체 배치

```text
                       인터넷
                         │
          ┌──────────────┴──────────────┐
          │ astra (게이트웨이·모니터링 호스트) │  10.0.0.201
          │  - NAT (iptables)            │
          │  - nginx (*.orbit-labs.space)│
          │  - Vault                     │
          │  - VM: efk / lgtm / wazuh    │
          └──────────────┬──────────────┘
                         │ 내부망 10.0.0.0/24
     ┌──────────┬────────┼─────────┬──────────┐
  polaris      sol      terra     luna        io
  control     worker1  worker2   worker3   부하 발생기
  plane      └──────── k8s 워커 (앱) ──────┘

AWS (관리형): S3 (펌웨어 Origin), RDS (DB) — flow.md 기준, §5 참고
```

- k8s 클러스터: control plane `polaris` 1대 + worker `sol`, `terra`, `luna` 3대.
- `astra`는 클러스터 밖에 두고 게이트웨이(NAT, nginx), Vault, 모니터링 VM을 맡는다.
- `io`는 부하 발생기 전용. 클러스터에 넣지 않는다.

## 2. k8s 노드 스펙

| 역할 | 호스트 | 기종 | CPU | RAM | 저장장치 | 내부망 IP |
|---|---|---|---|---|---|---|
| Control Plane | `polaris` | ODROID-M1S | RK3566 (Cortex-A55 4C/4T) | 8 GB | eMMC 64 GB (OS), NVMe WD SN530 256 GB (데이터) | `10.0.0.200` |
| Worker1 | `sol` | 갤럭시북2 550XED | i5-1235U (10C/12T) | 32 GB | 210 GB | `10.0.0.202` |
| Worker2 | `terra` | 갤럭시북 | i5-1235U (10C/12T) | 32 GB | 210 GB | `10.0.0.203` |
| Worker3 | `luna` | 갤럭시북 | i5-1235U (10C/12T) | 32 GB | 210 GB | `10.0.0.204` |

- `polaris`는 ARM(aarch64)이고 워커 3대는 x86_64다. control plane에는 앱 Pod를 올리지 않으므로(`NoSchedule`) 앱 이미지는 amd64만 있어도 된다. 단, DaemonSet(node-exporter, Fluent Bit, Cilium)은 `polaris`에도 뜨므로 arm64 이미지가 필요하다.

### k8s 위에서 도는 것

| 서비스 | 배치 | 권장 리소스 | 상태 |
|---|---|---|---|
| OTA 서버 등 앱 | 워커 3대 | - | 계획 |
| CDN Edge (nginx) | 워커 3대 | - | 계획 ([flow.md](flow.md) §0) |
| 프론트엔드 대시보드 | 워커 3대 | - | 계획 ([flow.md](flow.md) §8) |
| Argo CD | 클러스터 | 1 CPU / 2Gi | 계획 |
| Cilium (Hubble 포함) | DaemonSet, 모든 노드 | 0.5 CPU / 512Mi (노드당) | 계획 |
| node-exporter | DaemonSet, 모든 노드 | 0.2 CPU / 128Mi (노드당) | 계획 |
| Fluent Bit (수집) | DaemonSet, 모든 노드 | 0.1 CPU / 128Mi (노드당) | 계획 |
| kube-state-metrics | 클러스터 | 0.2 CPU / 256Mi | 계획 |

- 앱 외 DaemonSet만 노드당 약 0.8 CPU / 768Mi를 쓴다. `polaris`(4코어, 8 GB)는 control plane 구성요소와 이 DaemonSet을 함께 버텨야 한다.

## 3. astra (게이트웨이·모니터링 호스트, 클러스터 밖)

| 항목 | 값 |
|---|---|
| 기종 | 삼성 데스크톱 DM500TEA (보드 H670S1) |
| CPU | i5-12400 (6C/12T) |
| RAM | 66 GB (OS 기준 62 GiB) |
| 저장장치 | NVMe 삼성 980 500 GB (루트 457G), 사용 가능 423 GB (VM 디스크 240 GB 할당 전) |
| 내부망 IP | `10.0.0.201` (`br0`) — 내부망 게이트웨이 |

### 모니터링 VM

| 용도 | VM | vCPU | RAM | Disk | IP |
|---|---|---:|---:|---:|---|
| Logs (EFK) | `efk` | 4 | 20 GB | 100 GB | `10.0.0.205` |
| Metrics (Prometheus, Grafana) | `lgtm` | 4 | 14 GB | 80 GB | `10.0.0.206` |
| Security (Wazuh) | `wazuh` | 4 | 14 GB | 60 GB | `10.0.0.207` |
| **합계** | | **12** | **48 GB** | **240 GB** | |
| astra 호스트 몫 | - | 공유 | 약 14 GB | 나머지 | `10.0.0.201` |

- VM vCPU 합계(12)가 호스트 스레드(12)와 같다. 호스트 직접 서비스(nginx, Vault)와 CPU를 나눠 쓴다.
- VM OS: Ubuntu 24.04 클라우드 이미지 (cloud-init), 계정 `orbit` (sudo 비밀번호, SSH는 키만).

| VM | 서비스 | 권장 리소스 | 접근 주소 |
|---|---|---|---|
| `lgtm` | Prometheus | 2 CPU / 4Gi | `10.0.0.206` |
| `lgtm` | Grafana | 1 CPU / 1Gi | `observe.orbit-labs.space` → `10.0.0.206:3000` |
| `lgtm` | Alertmanager | 0.5 CPU / 256Mi | `10.0.0.206` |
| `efk` | Elasticsearch | 2 CPU / 8Gi (힙은 절반) | `10.0.0.205` |
| `efk` | Kibana | 1 CPU / 2Gi | `10.0.0.205` |
| `efk` | Fluent Bit (수신) | 0.5 CPU / 512Mi | `10.0.0.205` |
| `wazuh` | Wazuh 매니저, 인덱서, 대시보드 | 4 CPU / 8Gi (공식 최소) | `10.0.0.207` |

- Wazuh agent는 모든 노드에 호스트 직접 설치한다 (계획).

## 4. Vault

| 항목 | 값 |
|---|---|
| 배치 | astra 호스트 직접 |
| 권장 리소스 | 1 CPU / 1Gi |
| 접근 | 내부만 |
| 상태 | 계획 |

- 보관 대상: AWS IAM 키(OTA 서버 → S3), RDS 비밀번호 ([flow.md](flow.md) §5, §7).

## 5. nginx (게이트웨이)

| 항목 | 값 |
|---|---|
| 배치 | astra 호스트 직접 |
| 권장 리소스 | 0.5 CPU / 512Mi |
| 접근 | `*.orbit-labs.space` |
| 상태 | 계획 |

- 외부에서 들어오는 요청을 클러스터 앱과 Grafana(`observe.orbit-labs.space`)로 넘기는 리버스 프록시다.
- 펌웨어를 배포하는 CDN Edge nginx와는 다르다. Edge는 k8s 안의 Pod다 ([flow.md](flow.md) §0).
- 같은 호스트에 NAT 게이트웨이(iptables)가 있다 (적용 완료).

## 6. 부하 발생기

| 호스트 | 기종 | CPU | RAM | 저장장치 |
|---|---|---|---|---|
| `io` | 홈서버 | i5-10400 (6C/12T) | 32 GB | 500 GB (RTX 3060) |

## 7. 네트워크

| 망 | 대역 | 게이트웨이 | DNS |
|---|---|---|---|
| 내부망 (유선) | `10.0.0.0/24` | `10.0.0.201` (astra) | 8.8.8.8, 1.1.1.1 |
| 와이파이 | `192.168.200.0/22` | `192.168.200.1` | 8.8.8.8, 1.1.1.1 |

| 장비 | 내부망 (유선) | 와이파이 | 와이파이 예약 |
|---|---|---|---|
| `polaris` | `10.0.0.200` | `192.168.200.200` | 예약 |
| `astra` | `10.0.0.201` (`br0`) | `192.168.200.201` | 예약 |
| `sol` | `10.0.0.202` | `192.168.200.202` | 예약 |
| `terra` | `10.0.0.203` | `192.168.200.203` | 예약 없음, 고정 |
| `luna` | `10.0.0.204` | `192.168.200.204` | 예약 없음, 고정 |
| `efk` | `10.0.0.205` | - | - |
| `lgtm` | `10.0.0.206` | - | - |
| `wazuh` | `10.0.0.207` | - | - |

- 원격 접속은 tailscale을 쓴다. tailscale 주소는 저장소에 두지 않고 Notion에서 확인한다.

## 8. 운영체제

| 장비 | OS | 커널 | 계정 |
|---|---|---|---|
| `polaris` | Ubuntu 24.04 Server (Hardkernel 공식 이미지, eMMC) | Rockchip 6.1 | `orbit` |
| `astra` | Ubuntu 24.04.5 LTS | 6.8.0-147-generic | `orbit` |
| `sol`, `terra`, `luna` | Ubuntu 24.04.5 LTS | 6.8.0-147-generic | `orbit` |
| `efk`, `lgtm`, `wazuh` | Ubuntu 24.04 클라우드 이미지 (cloud-init) | - | `orbit` |

