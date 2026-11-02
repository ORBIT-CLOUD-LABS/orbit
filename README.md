# ORBIT Docs

> 차량 OTA(Over-The-Air) 업데이트 서비스의 요구사항·설계 문서와 실험 결과

다른 저장소의 AI 문서는 이 저장소의 `docs/` 를 요구사항 원본으로 읽는다.

## 레포지토리

| 레포 | 내용 |
|---|---|
| [orbit](https://github.com/ORBIT-CLOUD-LABS/orbit) | 요구사항·설계 문서, 실험 결과 (이 레포) |
| [orbit-server](https://github.com/ORBIT-CLOUD-LABS/orbit-server) | OTA 서버 (Java) |
| [orbit-vehicle-agent](https://github.com/ORBIT-CLOUD-LABS/orbit-vehicle-agent) | 차량 에이전트 (Go) |
| [orbit-visualization](https://github.com/ORBIT-CLOUD-LABS/orbit-visualization) | 대시보드 프론트엔드 (React, Vite, TypeScript) |
| [orbit-infrastructure](https://github.com/ORBIT-CLOUD-LABS/orbit-infrastructure) | 클러스터 구성 자동화 (Ansible) |
| [orbit-gitops](https://github.com/ORBIT-CLOUD-LABS/orbit-gitops) | 클러스터 배포 선언 (Argo CD) |
| [orbit-observability](https://github.com/ORBIT-CLOUD-LABS/orbit-observability) | 모니터링 VM 구성, k6 부하 실험 |

## Phase2 (`docs/`)

| 문서 | 내용 |
|---|---|
| [전체구성.md](docs/전체구성.md) | 클러스터 안팎에 무엇을 두는지 한눈에 보는 요약 |
| [flow.md](docs/flow.md) | OTA 전체 흐름 (Edge 구성, 단계별 흐름, 인증, 협의 필요 항목) |
| [api.md](docs/api.md) | 구현 API 목록 |
| [infrastructure.md](docs/infrastructure.md) | 노드 스펙, 모니터링 VM, Vault, nginx 게이트웨이, 네트워크, OS |

## Phase1 기록 (`phase1(old)/`)

Phase1 문서는 내용을 바꾸지 않고 보관한다. Phase2 작업의 기준으로 쓰지 않는다.

### 문서 (`phase1(old)/docs/`)

| 문서 | 내용 |
|---|---|
| [phase1.md](phase1(old)/docs/phase1.md) | Phase1 범위와 공통 값 |
| [flow.md](phase1(old)/docs/flow.md) | OTA 전체 흐름 (Phase1) |
| [api.md](phase1(old)/docs/api.md) | 구현 API 목록 (Phase1) |
| [infrastructure-plan.md](phase1(old)/docs/infrastructure-plan.md) | 호스트별 VM 사양 |
| [runbook-alerts.md](phase1(old)/docs/runbook-alerts.md) | 알림 대응 Runbook |

### 실험 결과 (`phase1(old)/experiments/`)

| 문서 | 내용 |
|---|---|
| [server-hpa.md](phase1(old)/experiments/server-hpa.md) | 서버 부하·HPA·복구 실험 결과 |
| [cdn.md](phase1(old)/experiments/cdn.md) | CDN 캐시·Edge replica·Edge 복구 실험 결과 |

### 발표 자료 (`phase1(old)/발표 자료/`)

[1차 발표](phase1(old)/발표%20자료/4조-ORBIT-1차.pdf) · [2차 발표](phase1(old)/발표%20자료/4조-ORBIT-2차.pdf) · [서버 HPA 시연 시나리오](phase1(old)/발표%20자료/서버-HPA-시연-시나리오.md) · [측정 스크린샷](phase1(old)/발표%20자료/측정-스크린샷)
