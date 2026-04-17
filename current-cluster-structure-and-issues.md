# Current Cluster Structure And Issues

## 1. Executive Summary

이 Kubernetes 클러스터는 기능적으로는 동작하고 있지만, 운영 구조는 아직 "실험/확장 중인 랩 클러스터"에 가깝다. 가장 큰 문제는 다음 네 가지다.

1. control-plane 오염
2. 리소스 정책 부재
3. 운영 컴포넌트의 HA/배치 정책 부족
4. NodePort 중심의 단순 노출 구조

특히 최근 `control-3` 장애는 단순 우발 장애라기보다, control-plane에 monitoring과 기타 부가 컴포넌트를 같이 얹는 구조가 만든 운영 리스크가 실제로 표면화된 사례로 보는 것이 맞다.

## 2. Current Topology

### 2.1 Node Layout

- Control plane: `control-1`, `control-2`, `control-3`
- GPU worker: `gpu-1`, `gpu-2`
- Infra worker: `infra-1`, `infra-2`, `infra-3`
- General worker: `worker-1`, `worker-2`, `worker-3`

총 11노드 구성이다.

### 2.2 Kubernetes Roles And Taints

- `control-1`, `control-2`, `control-3`만 `control-plane`
- control-plane taint: `node-role.kubernetes.io/control-plane:NoSchedule`
- 나머지 노드는 별도 taint 없음

즉 기본적으로는 control-plane 보호가 걸려 있지만, 여러 운영/부가 워크로드가 toleration으로 이를 우회한다.

### 2.3 Namespaces

- `default`
- `ebpf-obs`
- `gpu-operator`
- `kube-system`
- `longhorn-system`
- `monitoring`

## 3. Workload Inventory

### 3.1 Default Namespace

- Jupyter 계열 서비스 4개
- 모두 `Deployment + NodePort` 형태
- GPU 노드에 직접 배치

### 3.2 Monitoring

- `kube-prometheus-stack`
- Grafana
- Prometheus
- Alertmanager
- kube-state-metrics
- prometheus-operator
- node-exporter

현재 Prometheus의 메모리 기반 TSDB 문제는 수정되었고, 디스크 기반 `emptyDir`와 메모리 제한이 반영된 상태다.

### 3.3 Storage

- Longhorn 사용
- 기본 StorageClass: `longhorn`
- PVC는 일부 애플리케이션과 `ebpf-obs`에만 사용

### 3.4 Networking

- CNI: `kube-ovn`
- Core services:
  - `kube-ovn-cni`
  - `ovs-ovn`
  - `ovn-central`
  - `kube-ovn-controller`
  - `kube-ovn-monitor`
  - `kube-ovn-pinger`

### 3.5 GPU Stack

- NVIDIA GPU Operator
- GPU toolkit
- driver daemonset
- device plugin
- dcgm-exporter
- node-feature-discovery

### 3.6 Observability / Security

- Tetragon
- `ebpf-obs`
- 자체 analyzer / collector 배포

## 4. Confirmed Structural Problems

## 4.1 Control-Plane Pollution

다음 워크로드가 control-plane 위에 직접 배치된다.

- `monitoring/prometheus-my-monitoring-kube-prometh-prometheus-0`
- `monitoring/alertmanager-my-monitoring-kube-prometh-alertmanager-0`
- `monitoring/my-monitoring-grafana`
- `gpu-operator` 일부 컴포넌트
- `gpu-operator node-feature-discovery worker`
- `ovn-central`
- `tetragon`
- `node-exporter`

이 구조는 control-plane을 "클러스터 제어용 전용 영역"이 아니라 "운영용 shared compute"처럼 쓰고 있다. 따라서 다음 문제가 생긴다.

- API server, scheduler, controller-manager와 부가 워크로드가 자원 경쟁
- 장애 시 control-plane 안정성과 운영 툴 안정성이 동시에 무너짐
- 재부팅 후 CNI/OVN 복구 이전에 상위 워크로드가 재기동을 시도하면서 복구가 거칠어짐

## 4.2 Resource Governance Is Missing

현재 확인된 상태:

- `ResourceQuota`: 없음
- `LimitRange`: 없음

그리고 다수 워크로드가 `BestEffort`다.

대표 사례:

- `monitoring`: Grafana, node-exporter, prometheus-operator, kube-state-metrics
- `ebpf-obs`: `ebpf-agent`, `ebpf-ml-mao-collector`
- `kube-system`: `kube-proxy`, `tetragon`
- `longhorn-system`: 다수 CSI 및 manager/UI 컴포넌트

이 상태에서는 메모리 압박 시 어떤 파드가 먼저 정리될지 예측성이 떨어지고, 운영 장애가 "원인 불명"처럼 보이기 쉬워진다.

## 4.3 Monitoring Placement Is Still Operationally Weak

Prometheus 자체의 즉시 원인은 수정했지만 구조적으로는 여전히 좋지 않다.

- Prometheus replica: `1`
- Alertmanager replica: `1`
- Grafana replica: `1`
- kube-state-metrics replica: `1`
- prometheus-operator replica: `1`

문제:

- 노드 장애 또는 drain 시 단절점이 많음
- control-plane 의존도가 여전히 큼
- PDB 적용 범위가 부족함

## 4.4 GPU Operator Scope Is Too Broad

`gpu-operator-...-node-feature-discovery-worker`가 11노드 전체에 배치되어 있다.

즉:

- GPU 노드만이 아니라
- control-plane
- infra
- 일반 worker

모두에서 worker가 돈다.

이건 GPU 관련 기능 탐지가 GPU 노드만이 아니라 전체 노드 오버헤드가 되는 구조다. 특히 control-plane까지 포함되는 점이 좋지 않다.

## 4.5 HA Policy Is Inconsistent

복제 워크로드를 보면 topology spread가 전혀 없다.

- `topologySpreadConstraints`: 사실상 전부 `0`
- anti-affinity는 일부만 존재

영향:

- 복제본이 특정 노드군으로 쏠릴 수 있음
- 장애 도메인 분리가 약함
- 재스케줄링 시 기대와 다른 쏠림이 발생할 수 있음

## 4.6 NodePort-Heavy Exposure

현재 외부 노출 예:

- Grafana: `30000`
- Longhorn UI: `30001`
- ebpf-ml-mao UI: `30002`
- Jupyter 계열: `30128~30131`

NodePort를 과도하게 직접 쓰는 구조는 다음 문제가 있다.

- 서비스 진입점이 난립
- 인증, TLS, 접근 통제, 감사 일관성이 약함
- 추후 도메인/경로 기반 운영이 어려움

## 4.7 DNS Resolver Configuration Is Noisy

최근 이벤트에 `DNSConfigForming` 경고가 반복된다.

적용 nameserver line 예:

- `169.254.25.10 172.30.0.118 8.8.8.8`

이는 노드 resolver 구성이 이미 limit에 걸린 상태임을 의미한다. 당장 치명적 장애는 아니더라도, 장기적으로는 DNS 문제를 디버깅하기 어렵게 만든다.

## 4.8 CNI / OVN Recovery Is Sensitive During Node Restart

최근 이벤트에서 다음이 확인됐다.

- `FailedCreatePodSandBox`
- `dial unix /run/openvswitch/kube-ovn-daemon.sock: connect: no such file or directory`
- `ovn-central readiness failed`

즉 노드 재기동 시:

1. OVS/OVN 복구
2. kube-ovn CNI 복구
3. 상위 파드 네트워크 생성

순서가 매끄럽지 않다.

이 문제는 control-plane 위에 monitoring/GPU 관련 워크로드가 같이 존재할 때 더 크게 드러난다.

## 5. Current Operational Risk Assessment

### High

- control-plane 위에 monitoring/부가 워크로드 다수 상주
- quota/limitrange 부재
- BestEffort 파드 과다
- Prometheus/운영 컴포넌트 단일 replica

### Medium

- GPU operator 범위 과다
- topology spread 부재
- NodePort 과다 사용
- OVN/CNI 복구 민감도

### Low But Persistent

- DNSConfigForming 경고
- 일부 오래된 restart count 누적

## 6. Recommended Improvements

## 6.1 Immediate

1. monitoring 스택을 control-plane에서 infra 노드로 이동
2. `default`, `monitoring`, `ebpf-obs`에 `LimitRange`와 `ResourceQuota` 추가
3. BestEffort 워크로드 제거
4. GPU operator NFD worker를 GPU 노드 전용으로 제한

## 6.2 Short Term

1. Grafana / kube-state-metrics / prometheus-operator 최소 2 replica 검토
2. PDB 추가
3. NodePort 직접 노출 축소, Ingress 또는 LB로 통합
4. DNS resolver 정리

## 6.3 Mid Term

1. control / infra / gpu / worker 역할을 스케줄링 정책으로 명확히 분리
2. topology spread와 anti-affinity 기본화
3. 운영 서비스와 사용자 서비스의 네트워크/접근 경계 분리

## 7. Bottom Line

현재 클러스터는 "돌아가는" 상태이지 "운영에 강한" 상태는 아니다.

핵심 구조 문제는 다음 한 문장으로 요약할 수 있다.

> control-plane 보호, 리소스 거버넌스, HA 배치 원칙이 약한 상태에서 운영/관측/실험 워크로드가 한 클러스터에 혼합되어 있다.

이 구조를 그대로 유지하면, 앞으로도 장애는 개별 파드 문제가 아니라 "구조상 예견된 방식"으로 반복될 가능성이 높다.
