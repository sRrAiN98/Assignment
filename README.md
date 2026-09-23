# Kubernetes on Resume Presentation (오픈소스컨설팅 기술인터뷰)

Kubernetes 기반 온프레미스 인프라와 선언적 GitOps(Gitea + Argo CD)를 활용한 클라우드 네이티브 이력서 웹 애플리케이션 배포 프로젝트입니다.

---

## 📌 주요 특징 (Key Highlights)

1. **3노드 Kubernetes 클러스터 (v1.36.4)**
   - Control Plane 1대(`k8s-cp1: 192.168.56.10`)와 Worker 2대(`k8s-w1: 192.168.56.21`, `k8s-w2: 192.168.56.22`)
   - Flannel CNI VXLAN 터널링 (`--iface=k8s0`) 및 containerd 2.2.1 런타임
2. **사설 GitOps 파이프라인 (Gitea & Argo CD)**
   - 프라이빗 Git 서버인 **Gitea(NodePort 30082)**를 클러스터 내 구동
   - **Argo CD**가 Gitea 리포지토리를 감시하여 `resume` 네임스페이스에 선언적 자동 동기화(Self-Heal & Prune)
3. **요구사항 준수 워크로드 설계**
   - 전용 ServiceAccount: `service-resume` (AutomountServiceAccountToken: true, 비특권 실행)
   - 고가용성 분산 배포: Deployment `resume-web` (2 Replicas, k8s-w1/w2 균등 스케줄링)
   - 외부 노출: NodePort `30080:80/TCP`
4. **Helm 차트 표준화 및 볼륨 마운트 리팩토링**
   - 선언적 18KB 경량 HTML ConfigMap과 CSS/JS 정적 에셋 ConfigMap을 분리
   - `subPath` 볼륨 마운트로 etcd 부하 및 256KB 어노테이션 한도를 원천 차단

---

## 📂 디렉토리 구조 (Repository Layout)

```text
osc-k8s-resume/
├── ansible/                      # 클러스터 프로비저닝 & GitOps IaC 플레이북 모음
│   ├── ansible.cfg               # Ansible 최적화 설정
│   ├── inventory.yml             # 3노드 인벤토리 정의
│   ├── group_vars/               # 버전 및 네트워크 설정 변수
│   ├── playbooks/                # 01-prepare ~ 04-deploy-gitops, site.yml
│   └── README.md                 # Ansible 자동화 매뉴얼
├── argocd/
│   └── application-resume.yaml   # Gitea 연동 Argo CD Application
├── charts/
│   └── resume-web/               # 이력서 웹 애플리케이션 표준 Helm 차트
│       ├── Chart.yaml            # 차트 메타데이터 (v0.1.0)
│       ├── values.yaml           # 환경 설정값 (2 replicas, SA, NodePort)
│       ├── files/                # 웹 에셋 (index.html, bootstrap)
│       └── templates/            # k8s 매니페스트 템플릿
│           ├── deployment.yaml   # subPath 볼륨 마운트 & service-resume 바인딩
│           ├── service.yaml      # NodePort 30080 서비스
│           ├── serviceaccount.yaml # service-resume 계정 생성
│           ├── configmap.yaml    # HTML (18KB) ConfigMap
│           └── configmap-assets.yaml # CSS/JS 에셋 ConfigMap
├── docs/                         # 아키텍처 다이어그램 및 정식 제출 문서
│   ├── ARCHITECTURE.md           # 물리/논리 구성도 상세 명세서
│   ├── Kubernetes_구축결과서_장재희.pdf # 정식 제출용 고품질 결과보고서 PDF
│   └── diagrams/                 # 독립 Mermaid 원본 소스
│       ├── physical-topology.mmd # 물리 구성도 다이어그램 소스
│       ├── logical-topology.mmd  # 논리 구성도 다이어그램 소스
│       └── helm-architecture.mmd # Helm 패키징 흐름 다이어그램 소스
├── manifests/
│   └── gitea.yaml                # 사설 Gitea Git 서버 배포 매니페스트
├── submission.md                 # 과제 최종 결과 보고서 (Markdown)
├── submission.pdf                # 과제 최종 결과 보고서 (PDF)
└── README.md
```

---

## 🌐 서비스 접속 URL

| 서비스 | 주소 | 비고 |
| :--- | :--- | :--- |
| **이력서 웹서비스** | `http://192.168.56.21:30080` | NodePort 30080 (워커 노드 분산 서빙) |
| **과제 제출 공식 저장소 (GitHub)** | `https://github.com/sRrAiN98/Assignment` | 공개 소스코드 및 문서 저장소 |
| **상시 확인용 미러 (GitHub Pages)** | `https://sRrAiN98.github.io/Assignment/` | 로컬 PC 오프라인 시 상시 확인 가능한 웹 미러 |
| **Gitea 사설 콘솔** | `http://192.168.56.21:30082` | 클러스터 내부 GitOps 저장소 (면접 시연 시 접속) |
| **Argo CD 관리 콘솔** | `http://192.168.56.21:30081` | GitOps 컨트롤러 (면접 시연 시 접속) |

자세한 물리/논리 구성도와 엔지니어링 시행착오 및 트러블슈팅 사례는 [submission.md](submission.md), [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) 및 [submission.pdf](submission.pdf)를 참고해 주시기 바랍니다.
