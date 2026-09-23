# Kubernetes on Resume Presentation - 시스템 아키텍처 명세서

본 문서는 오픈소스컨설팅(OSC) Kubernetes 기술 인터뷰 과제 요건에 따라 구축된 시스템의 **물리 구성도(Physical Topology)**와 **논리 구성도(Logical Topology)**를 상세히 기술합니다.

---

## 1. Kubernetes 물리 구성 (Physical Topology)

> **[과제 요건] 자신이 구축한 시스템의 물리 구성도를 기술하시오.**

### 1.1 물리/가상 인프라 토폴로지 다이어그램

```mermaid
flowchart TB
    subgraph HostPC ["호스트 물리 시스템 (Windows 11 Pro 64-bit)"]
        subgraph Hardware ["물리 하드웨어 리소스"]
            cpu["CPU: 8 Cores 16 Threads"]
            mem["RAM: 32GB DDR4/DDR5"]
            disk["Storage: 1TB NVMe SSD"]
        end

        subgraph VirtualSwitches ["Hyper-V 가상 스위치 계층"]
            vSwitchExt["Default Switch (NAT / DHCP)\n외부 패키지 및 런타임 다운로드 전용 (172.19.x.x)"]
            vSwitchInt["K8s-Internal Switch (Internal Private)\n클러스터 제어 및 파드 VXLAN 통신 전용 (192.168.56.0/24)"]
        end

        subgraph VirtualMachines ["Hyper-V 가상 머신 계층 (Ubuntu 24.04 LTS)"]
            subgraph CP1 ["k8s-cp1 (Control Plane)"]
                cp_spec["2 vCPU / 4GB RAM / 30GB VHDX"]
                cp_nic1["internet0: 172.19.x.x (DHCP)"]
                cp_nic2["k8s0: 192.168.56.10 (Static 고정 IP)"]
            end

            subgraph W1 ["k8s-w1 (Worker Node 1)"]
                w1_spec["2 vCPU / 4GB RAM / 30GB VHDX"]
                w1_nic1["internet0: 172.19.x.x (DHCP)"]
                w1_nic2["k8s0: 192.168.56.21 (Static 고정 IP)"]
            end

            subgraph W2 ["k8s-w2 (Worker Node 2)"]
                w2_spec["2 vCPU / 4GB RAM / 30GB VHDX"]
                w2_nic1["internet0: 172.19.x.x (DHCP)"]
                w2_nic2["k8s0: 192.168.56.22 (Static 고정 IP)"]
            end
        end

        hostNic["Host Virtual Adapter: 192.168.56.1"]
    end

    evaluator["기술 인터뷰 평가관 / 웹 브라우저"]

    vSwitchExt -.->|"인터넷 아웃바운드"| cp_nic1
    vSwitchExt -.->|"인터넷 아웃바운드"| w1_nic1
    vSwitchExt -.->|"인터넷 아웃바운드"| w2_nic1

    vSwitchInt <===>|"클러스터 제어 & VXLAN 패킷"| cp_nic2
    vSwitchInt <===>|"클러스터 제어 & VXLAN 패킷"| w1_nic2
    vSwitchInt <===>|"클러스터 제어 & VXLAN 패킷"| w2_nic2
    vSwitchInt <===> hostNic

    evaluator -->|"웹서비스: http://192.168.56.21:30080"| w1_nic2
    evaluator -->|"Gitea Git: http://192.168.56.21:30082"| w1_nic2
    evaluator -->|"Argo CD: http://192.168.56.21:30081"| w1_nic2
```

### 1.2 물리 및 가상 인프라 상세 제원표

| 구분 | 호스트 / 노드명 | 역할 | 할당 사양 (vCPU / RAM / Disk) | 네트워크 구성 (IP / MAC) | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **물리 호스트** | Local Workstation | Hyper-V 호스트 | 8C 16T / 32GB RAM / 1TB NVMe | Windows 11 Pro, Host IP: `192.168.56.1` | 가상 스위치 관리 및 라우팅 |
| **VM 1** | **k8s-cp1** | Control Plane | 2 vCPU / 4GB RAM / 30GB SSD | • `internet0`: 172.19.x.x (DHCP)<br>• `k8s0`: `192.168.56.10` (`00:15:5d:56:00:10`) | etcd, API Server, Scheduler |
| **VM 2** | **k8s-w1** | Worker Node 1 | 2 vCPU / 4GB RAM / 30GB SSD | • `internet0`: 172.19.x.x (DHCP)<br>• `k8s0`: `192.168.56.21` (`00:15:5d:56:00:21`) | 워크로드 Pod 분산 배치 |
| **VM 3** | **k8s-w2** | Worker Node 2 | 2 vCPU / 4GB RAM / 30GB SSD | • `internet0`: 172.19.x.x (DHCP)<br>• `k8s0`: `192.168.56.22` (`00:15:5d:56:00:22`) | 워크로드 Pod 분산 배치 |

### 1.3 물리 네트워크 격리 및 이중화 원리
- **격리된 전용 스위치 (`K8s-Internal`)**: 호스트 머신의 무선랜(Wi-Fi) 접속 환경 변경이나 IP 변동에 구애받지 않고, 노드 간 통신, kube-apiserver 접근, etcd 클러스터링 및 Flannel CNI 오버레이 통신이 영구적으로 보장됩니다.
- **아웃바운드 전용 스위치 (`Default Switch`)**: OS 패키지, containerd 바이너리, Kubernetes 이미지 다운로드를 위해서만 사용되며, 클러스터 내부로는 인바운드 트래픽이 유입되지 않아 보안이 강화됩니다.

---

## 2. Kubernetes 논리 구성 (Logical Topology)

> **[과제 요건] 자신이 구축한 시스템의 논리 구성도를 기술하시오.**

### 2.1 논리 아키텍처 다이어그램

```mermaid
flowchart TB
    subgraph GitOpsLayer ["형상 관리 및 선언형 GitOps 계층"]
        engineer["DevOps 엔지니어 (장재희)"]
        gitea["사설 Git 저장소 (Gitea)\nhttp://192.168.56.21:30082\nrepo: jaehee/osc-k8s-resume.git"]
        argo["Argo CD Controller & Repo Server\nhttp://192.168.56.21:30081\nApp: resume-web (Auto Sync / Self-Heal)"]
    end

    subgraph ControlPlaneLayer ["Control Plane 노드 (k8s-cp1, 192.168.56.10)"]
        apiserver["kube-apiserver (v1.36.4)"]
        etcd[("etcd v3.5\n(클러스터 상태 분산 저장소)")]
        sched["kube-scheduler"]
        cm["kube-controller-manager"]
        flannelMaster["kube-flannel (CNI DaemonSet)"]
    end

    subgraph WorkerLayer ["Worker 노드 계층 (k8s-w1: .21 / k8s-w2: .22)"]
        subgraph Node1 ["k8s-w1 노드"]
            kubelet1["kubelet"]
            proxy1["kube-proxy"]
            cri1["containerd 2.2.1"]
            subgraph Pods1 ["파드 오케스트레이션 (10.244.1.0/24)"]
                pod1["resume-web-xxx-1\n(Nginx Web Server)"]
            end
        end

        subgraph Node2 ["k8s-w2 노드"]
            kubelet2["kubelet"]
            proxy2["kube-proxy"]
            cri2["containerd 2.2.1"]
            subgraph Pods2 ["파드 오케스트레이션 (10.244.2.0/24)"]
                pod2["resume-web-xxx-2\n(Nginx Web Server)"]
            end
        end
    end

    subgraph WorkloadSpec ["워크로드 보안 및 리소스 정책 (Namespace: resume)"]
        sa["ServiceAccount: service-resume\n(과제 필수 요구조건 바인딩)"]
        cmHtml["ConfigMap: resume-web-html (18KB)\nsubPath: index.html"]
        cmAsset["ConfigMap: resume-web-assets\nsubPath: bootstrap css/js"]
        svc["NodePort Service: resume-web-svc\nNodePort: 30080 -> Port 80"]
    end

    engineer -->|"Git Commit & Push"| gitea
    gitea -->|"1. Poll & Detect Diff"| argo
    argo -->|"2. Server-Side Apply"| apiserver
    apiserver <--> etcd
    apiserver --> sched
    sched -->|"파드 스케줄링"| kubelet1
    sched -->|"파드 스케줄링"| kubelet2

    pod1 -.-> sa
    pod2 -.-> sa
    cmHtml -.->|"Volume Mount (subPath)"| pod1
    cmAsset -.->|"Volume Mount (subPath)"| pod1
    cmHtml -.->|"Volume Mount (subPath)"| pod2
    cmAsset -.->|"Volume Mount (subPath)"| pod2

    svc -->|"트래픽 로드밸런싱"| pod1
    svc -->|"트래픽 로드밸런싱"| pod2

    client["웹 브라우저 (평가관)"] -->|"접속: 192.168.56.21:30080"| svc
```

### 2.2 논리 계층별 컴포넌트 매핑 및 역할

| 계층 | 컴포넌트 | 네임스페이스 / 버전 | 상세 역할 및 구현 내용 |
| :--- | :--- | :--- | :--- |
| **코어 제어 계층** | kube-apiserver, etcd, scheduler | `kube-system` / v1.36.4 | 선언적 상태 제어, 쿼럼 분산 저장 및 스케줄링 |
| **네트워크(CNI)** | Flannel CNI | `kube-flannel` / v0.28.8 | VXLAN 터널링 오버레이 (`10.244.0.0/16`), `--iface=k8s0` 바인딩 |
| **컨테이너 런타임** | containerd | v2.2.1 | `SystemdCgroup=true`, CRI 안정성 및 리소스 격리 |
| **형상 관리(Git)** | Gitea | `gitea` / 1.22 | 사설 Git 저장소 (NodePort `30082`, `jaehee/osc-k8s-resume`) |
| **선언형 GitOps** | Argo CD | `argocd` / v2.13.0 | Git 저장소 자동 추적, Self-Heal & Prune 자동 롤아웃 (NodePort `30081`) |
| **워크로드** | `resume-web` Deployment | `resume` (2 Replicas) | `k8s-w1`, `k8s-w2` 분산 스케줄링, 비특권(Non-root) 실행 |
| **서비스 계정** | **service-resume** | `resume` | **과제 필수 요구조건**: 파드에 연결된 단독 ServiceAccount |
| **설정/볼륨** | ConfigMap 2종 | `resume` | HTML(18KB)과 CSS/JS 에셋을 분리하고 `subPath` 볼륨 마운트 적용 |
| **서비스 노출** | NodePort Service | `resume` | 외부 접근 포트 `30080`을 통해 2개 워커 파드로 HTTP 로드밸런싱 |

---

## 3. 원본 다이어그램 소스 파일 링크

- [물리 구성도 Mermaid 소스](diagrams/physical-topology.mmd)
- [논리 구성도 Mermaid 소스](diagrams/logical-topology.mmd)
- [Helm 패키징 구조 Mermaid 소스](diagrams/helm-architecture.mmd)
