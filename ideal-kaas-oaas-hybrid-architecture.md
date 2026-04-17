# Ideal KaaS OaaS Hybrid Architecture

## 1. Purpose

이 문서는 현재 단일 혼합 클러스터 구조를 넘어, 실무에서 많이 채택하는 이상적인 형태의 다음 세 가지 모델을 설명한다.

1. KaaS: Kubernetes as a Service
2. OaaS: OpenStack as a Service
3. OpenStack + Kubernetes Hybrid

목표는 "한 플랫폼에 모든 것을 몰아넣는 구조"를 피하고, 역할별 책임 경계를 분리하여 운영 안정성, 확장성, 보안성, 장애 대응력을 높이는 것이다.

## 2. Design Principles

실무형 아키텍처는 보통 아래 원칙을 따른다.

### 2.1 Control Plane Is Sacred

- Kubernetes control-plane은 앱 실행 장소가 아님
- OpenStack control services도 tenant workload와 분리
- 관측/배치/보안 부가 워크로드가 control-plane을 침범하지 않게 해야 함

### 2.2 Platform And Tenant Separation

- 플랫폼 운영 컴포넌트와 사용자 워크로드 분리
- 운영자 영역, 인프라 영역, 테넌트 영역을 다른 노드 풀과 정책으로 나눔

### 2.3 Resource Governance By Default

- requests/limits, quotas, limitranges 기본 적용
- 무제한 파드는 예외로만 허용

### 2.4 Storage And Network Are First-Class

- 상태 저장 워크로드는 의도적으로 PVC 사용
- 외부 노출은 Ingress/LB를 기본으로 사용
- NodePort는 디버그/예외 용도에 한정

### 2.5 Blast Radius Reduction

- 장애가 control-plane, storage, monitoring, tenant workload 전체로 전이되지 않도록 설계

## 3. Ideal KaaS Architecture

## 3.1 What KaaS Means

KaaS는 "사용자는 Kubernetes만 사용하고, underlying VM/network/storage 운영은 플랫폼이 관리"하는 모델이다.

### 3.2 Recommended Node Pools

- `control-plane` pool
  - API server, etcd, scheduler, controller-manager 전용
  - taint 유지
  - 일반 앱 배치 금지

- `system-infra` pool
  - ingress controller
  - monitoring
  - logging
  - service mesh control plane
  - backup operator
  - policy / security agents

- `general-worker` pool
  - 일반 stateless 서비스
  - API, web, batch

- `stateful-worker` pool
  - DB proxy
  - queue consumer
  - stateful operator-managed app

- `gpu-worker` pool
  - AI/ML
  - Jupyter
  - batch inference / training

## 3.3 Core Platform Services

- Ingress or Gateway API
- CNI with network policy support
- CSI with dynamic provisioning
- metrics-server
- Prometheus + Alertmanager + Grafana
- centralized logging
- cert-manager
- external-dns or equivalent
- admission / policy engine

## 3.4 Exposure Pattern

- Public: LB or Ingress
- Internal: ClusterIP + internal gateway
- NodePort: 원칙적으로 금지 또는 제한

## 3.5 Multi-Tenancy Controls

- namespace per team / service group
- ResourceQuota
- LimitRange
- NetworkPolicy
- RBAC
- optional Pod Security Admission or Kyverno / Gatekeeper

## 4. Ideal OaaS Architecture

## 4.1 What OaaS Means

OaaS는 OpenStack을 통해 VM, network, block storage, image, load balancer 같은 IaaS 기능을 내부 고객에게 서비스 형태로 제공하는 모델이다.

### 4.2 Recommended OpenStack Separation

- `OpenStack control plane`
  - Keystone
  - Nova API / Scheduler / Conductor
  - Neutron server
  - Glance
  - Cinder API / Scheduler
  - Horizon
  - DB / MQ / cache

- `OpenStack infra services`
  - message queue
  - DB clusters
  - memcached
  - LB
  - monitoring / logging

- `compute nodes`
  - tenant VM 실행 전용

- `storage nodes`
  - Ceph or dedicated storage backend

### 4.3 OaaS Operational Model

- 프로젝트/테넌트 단위 분리
- flavor / quota / security group / network segmentation 제공
- LBaaS, volume, image lifecycle 표준화

### 4.4 Common Best Practice

- OpenStack control plane과 Kubernetes control plane은 같은 노드에 섞지 않음
- observability, backup, bastion, CI runners도 tenant compute와 분리

## 5. Ideal OpenStack + Kubernetes Hybrid

## 5.1 Real-World Preferred Pattern

실무에서 가장 현실적이고 안정적인 형태는 보통 다음이다.

> OpenStack은 IaaS substrate, Kubernetes는 application runtime

즉:

- OpenStack이 VM, network, volume, LB를 제공
- Kubernetes 클러스터는 OpenStack 위 VM 또는 bare metal로 구성
- 앱은 Kubernetes에서 운영
- VM 중심 워크로드와 container 중심 워크로드를 혼합 사용

## 5.2 Recommended Layering

### Layer 1: Infrastructure

- Physical servers
- Underlay network
- shared storage or distributed storage

### Layer 2: OpenStack

- VM provisioning
- Neutron networking
- Cinder / Ceph volume
- Octavia LB
- tenant/project isolation

### Layer 3: Kubernetes Clusters

- separate clusters by purpose
  - production apps cluster
  - GPU/AI cluster
  - internal platform cluster
  - dev/test cluster

### Layer 4: Platform Services

- CI/CD
- observability
- secrets and PKI
- backup / DR
- service catalog

### Layer 5: Tenant Workloads

- web services
- APIs
- jobs
- AI/ML
- notebooks
- VM-only legacy apps on OpenStack

## 5.3 Why This Hybrid Model Wins

- VM이 필요한 워크로드와 container가 잘 맞는 워크로드를 동시에 수용 가능
- Kubernetes 장애가 OpenStack 전체 장애로 번지지 않음
- OpenStack compute와 Kubernetes worker를 역할별로 최적화 가능
- GPU, stateful, legacy VM workload를 섞되 경계를 유지 가능

## 6. Ideal Role Separation For Your Environment

현재 환경을 기준으로 하면 이상적인 모델은 아래처럼 재구성하는 것이 좋다.

### 6.1 OpenStack / Infra Layer

- OpenStack control services 전용 노드 또는 전용 클러스터
- storage backend 전용 영역
- tenant VM compute 전용 영역

### 6.2 Kubernetes Platform Cluster

- control-plane 3대 전용
- infra worker pool 3대 이상
- monitoring, logging, ingress, policy, backup, CI runner는 infra pool에만 배치

### 6.3 Kubernetes Workload Cluster

- general worker pool
- GPU worker pool
- notebook / research / batch / AI는 별도 pool

### 6.4 Optional Split

운영 성숙도를 높이려면 다음과 같은 이중화가 더 좋다.

- `platform cluster`
  - monitoring
  - logging
  - security
  - ingress
  - internal tools

- `tenant cluster`
  - 실제 사용자 서비스
  - 실험성 workload
  - notebooks / AI jobs

## 7. Recommended Target Architecture For This Case

## 7.1 Short-Term Target

- 단일 Kubernetes 클러스터 유지
- 하지만 node pool 정책을 분명히 함

권장 역할:

- `control-*`: control-plane only
- `infra-*`: monitoring, ingress, logging, longhorn control, operators
- `worker-*`: 일반 서비스
- `gpu-*`: AI/ML/GPU 전용

## 7.2 Mid-Term Target

- OpenStack은 VM/IaaS 계층
- Kubernetes는 앱 계층
- monitoring 및 internal platform은 별도 infra cluster 또는 infra node pool

## 7.3 Long-Term Target

- 최소 2개 Kubernetes cluster
  - platform cluster
  - tenant cluster
- GPU는 별도 cluster 또는 최소 별도 node group
- OpenStack과 Kubernetes를 하나의 "서비스 카탈로그" 아래 묶음

## 8. Concrete Policy Recommendations

### 8.1 Scheduling Policy

- control-plane toleration 금지 기본화
- infra workloads는 `infra` nodeSelector/affinity 강제
- GPU workloads는 `gpu` nodeSelector/taint 기반 강제

### 8.2 Resource Policy

- namespace별 LimitRange
- namespace별 ResourceQuota
- BestEffort 금지
- VPA/HPA는 선택적으로 적용

### 8.3 Exposure Policy

- NodePort 직접 사용 금지
- Ingress/Gateway 또는 LB 통합
- 인증과 TLS를 ingress 계층에서 표준화

### 8.4 Storage Policy

- 상태 저장 서비스는 PVC 우선
- `emptyDir`는 캐시/임시 데이터 전용
- `medium: Memory`는 명시적 승인 없이는 금지

### 8.5 Reliability Policy

- PDB 기본 적용
- topology spread 기본 적용
- anti-affinity 기본 적용
- control-plane 장애와 worker 장애의 blast radius 분리

## 9. Anti-Patterns To Avoid

- control-plane에 monitoring/observability를 상주시킴
- GPU operator 컴포넌트를 전체 노드에 무차별 배포
- NodePort로 서비스 직접 노출
- quota/limitrange 없이 운영
- single replica 운영 컴포넌트를 다수 유지
- 디스크 기반이어야 할 저장소를 메모리 기반으로 둠
- 실험성 workload와 플랫폼 workload를 같은 장애 도메인에 둠

## 10. Bottom Line

가장 이상적인 실무형 방향은 다음이다.

> OpenStack은 인프라 제공자, Kubernetes는 애플리케이션 실행 플랫폼, 그리고 두 플랫폼의 운영 서비스는 tenant workload와 분리된 전용 영역에서 운영한다.

즉:

- control-plane은 control-plane답게
- infra는 infra답게
- tenant workload는 tenant 영역답게
- GPU/AI는 별도 자원군답게

이렇게 역할을 분리해야 장애가 "재미있는 현상"이 아니라 "제어 가능한 사건"이 된다.
