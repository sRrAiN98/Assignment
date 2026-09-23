# Ansible 기반 Kubernetes 클러스터 & GitOps 파이프라인 자동화

본 디렉터리는 오픈소스컨설팅(OSC) Kubernetes 기술 인터뷰 과제를 위해 제작된 **인프라 자동화(IaC: Infrastructure as Code) 플레이북 모음**입니다.  
Ubuntu 24.04 LTS 노드 3대(`k8s-cp1`, `k8s-w1`, `k8s-w2`)를 대상으로 OS 최적화, CRI(containerd) 및 Kubernetes 패키지 설치, Flannel CNI 구성, Worker 노드 조인, 사설 Gitea 및 Argo CD GitOps 파이프라인 배포까지 전 과정을 **100% 코드 기반으로 멱등하게(Idempotent) 자동화**합니다.

---

## 1. 아키텍처 및 역할 분담

```text
ansible/
├── ansible.cfg              # SSH 커넥션 최적화 및 타임아웃/become 설정
├── inventory.yml            # 3노드 인벤토리 정의 (Control Plane 1 + Worker 2)
├── group_vars/
│   └── all.yml              # K8s 버전(v1.36.4), Pod CIDR, Gitea/Argo CD 전역 변수
├── playbooks/
│   ├── site.yml             # 전체 과정을 순차 실행하는 마스터 플레이북
│   ├── 01-prepare-nodes.yml # OS 최적화(swapoff, sysctl, 모듈) 및 containerd/k8s 설치
│   ├── 02-init-control-plane.yml # kubeadm init 및 Flannel CNI(--iface=k8s0) 패치
│   ├── 03-join-workers.yml  # Worker 노드 클러스터 조인 자동화
│   └── 04-deploy-gitops.yml # Gitea 사설 저장소 및 Argo CD GitOps 파이프라인 배포
└── README.md
```

---

## 2. 노드 인벤토리 사양

| 노드명 | 역할 | IP (클러스터 전용망) | vCPU / RAM | OS |
| :--- | :--- | :--- | :--- | :--- |
| **k8s-cp1** | Control Plane | `192.168.56.10` | 2 vCPU / 4GB | Ubuntu 24.04 LTS |
| **k8s-w1** | Worker 1 | `192.168.56.21` | 2 vCPU / 4GB | Ubuntu 24.04 LTS |
| **k8s-w2** | Worker 2 | `192.168.56.22` | 2 vCPU / 4GB | Ubuntu 24.04 LTS |

---

## 3. 핵심 자동화 단계

### Step 1: OS 최적화 및 런타임 설치 (`01-prepare-nodes.yml`)
- `/etc/hosts`에 노드 간 FQDN 및 IP 자동 등록
- `swapoff -a` 및 `/etc/fstab` 주석 처리로 Swap 메모리 영구 비활성화
- `overlay`, `br_netfilter` 커널 모듈 적재 및 `/etc/modules-load.d/k8s.conf` 등록
- 브리지 패킷 필터링(`net.bridge.bridge-nf-call-iptables = 1`) 및 IP 포워딩(`net.ipv4.ip_forward = 1`) sysctl 적용
- `containerd` 2.2.1 설치 및 `SystemdCgroup = true` 설정
- Kubernetes 공식 패키지 저장소 등록, `kubelet`/`kubeadm`/`kubectl` 설치 및 `apt-mark hold` 고정
- 클러스터 전용 NIC(`k8s0`) 바인딩: `KUBELET_EXTRA_ARGS=--node-ip={{ k8s_node_ip }}`

### Step 2: Control Plane 초기화 및 CNI 배포 (`02-init-control-plane.yml`)
- `kubeadm init`을 통해 API Server, Controller Manager, Scheduler, etcd 기동
- 관리자 권한 kubeconfig (`/home/ubuntu/.kube/config`) 설정
- **Flannel CNI Dual-NIC 패치**: Hyper-V 다중 인터페이스 환경에서 VXLAN 트래픽이 올바른 내부망(`192.168.56.0/24`)을 타도록 `--iface=k8s0` 인자 자동 삽입 및 배포
- 워커 노드 조인을 위한 토큰 및 인증서 해시 추출

### Step 3: Worker 노드 클러스터 조인 (`03-join-workers.yml`)
- 추출된 join 명령어로 `k8s-w1`, `k8s-w2`를 클러스터에 안전하게 편입
- 이미 조인된 노드는 `/etc/kubernetes/kubelet.conf` 검사를 통해 건너뛰도록 멱등성 보장

### Step 4: Gitea & Argo CD GitOps 배포 (`04-deploy-gitops.yml`)
- Helm 3 바이너리 다운로드 및 설치
- 경량 프라이빗 Git 서버 Gitea 배포 (NodePort `30082`)
- Argo CD 공식 Helm 차트 배포 (NodePort `30081`)
- 이력서 웹 애플리케이션(`resume-web`) GitOps Application 리소스 등록 및 롤아웃 상태 모니터링

---

## 4. 플레이북 실행 방법

### 사전 요구사항
Ansible 제어 노드(WSL 또는 관리자 터미널)에서 패키지를 설치합니다:
```bash
pip install ansible
ansible-galaxy collection install ansible.posix community.general
```

### 전체 클러스터 및 GitOps 파이프라인 일괄 프로비저닝
```bash
cd ansible
ansible-playbook -i inventory.yml playbooks/site.yml
```

### 특정 단계별 개별 실행
```bash
# 1. OS 설정 및 런타임만 배포
ansible-playbook -i inventory.yml playbooks/01-prepare-nodes.yml

# 2. Control Plane 초기화
ansible-playbook -i inventory.yml playbooks/02-init-control-plane.yml

# 3. Worker 노드 조인
ansible-playbook -i inventory.yml playbooks/03-join-workers.yml

# 4. GitOps 파이프라인 배포
ansible-playbook -i inventory.yml playbooks/04-deploy-gitops.yml
```
