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
├── argocd/                       # Argo CD 선언적 Application 배포 매니페스트
│   ├── application-traefik.yaml  # 공식 Traefik v41.6.0 Ingress 컨트롤러
│   ├── application-gitea.yaml    # 공식 Gitea v12.7.0 사설 GitOps 저장소
│   └── application-resume.yaml   # 이력서 웹 애플리케이션 Helm 차트 연동
├── charts/
│   └── resume-web/               # 이력서 웹 애플리케이션 표준 Helm 차트
│       ├── Chart.yaml            # 차트 메타데이터 (v0.1.0)
│       ├── values.yaml           # 환경 설정값 (2 replicas, SA, NodePort)
│       ├── files/                # 웹 에셋 (index.html, bootstrap)
│       └── templates/            # k8s 매니페스트 템플릿
│           ├── deployment.yaml   # subPath 볼륨 마운트 & service-resume 바인딩
│           ├── service.yaml      # NodePort 30080 서비스
│           ├── ingress.yaml      # Ingress 라우팅 (Traefik 연동)
│           ├── serviceaccount.yaml # service-resume 계정 생성
│           ├── configmap.yaml    # HTML (18KB) ConfigMap
│           └── configmap-assets.yaml # CSS/JS 에셋 ConfigMap
├── docs/                         # 과제 구축 결과서 및 공식 산출물 일원화
│   ├── submission.md             # 과제 구축 결과보고서 (Markdown 완전판)
│   ├── Kubernetes_구축결과서_장재희.pdf # 정식 제출용 고품질 결과보고서 PDF
│   ├── submission_print.html     # 인쇄 및 보고서 템플릿 소스
│   └── diagrams/                 # 아키텍처 Mermaid 다이어그램 소스
└── README.md
```

---

## 🌐 서비스 접속 URL 및 도메인 체계

외부 단일 인입 도메인(`*.srrain.kro.kr`)을 통해 Traefik Ingress가 서브도메인 기반으로 각 서비스를 라우팅합니다.

| 서비스 | 외부 접속 도메인 (80 Port) | 내부망 / NodePort 접속 | 비고 |
| :--- | :--- | :--- | :--- |
| **이력서 웹서비스 (메인)** | `http://srrain.kro.kr`<br>`http://resume.srrain.kro.kr` | `http://192.168.56.21:30080`<br>`http://192.168.56.21:30000` | 이력서 웹 애플리케이션 (기본 Catch-All) |
| **Gitea 사설 GitOps 저장소** | `http://git.srrain.kro.kr` | `http://192.168.56.21:30082` | 클러스터 내부 사설 저장소 (면접 시연) |
| **Argo CD 관리 콘솔** | `http://argocd.srrain.kro.kr` | `http://192.168.56.21:30081` | GitOps 컨트롤러 웹 UI (면접 시연) |
| **과제 제출 공식 저장소 (GitHub)** | `https://github.com/sRrAiN98/Assignment` | - | 소스코드 및 문서 공식 저장소 |
| **상시 확인용 미러 (GitHub Pages)** | `https://sRrAiN98.github.io/Assignment/` | - | 오프라인 시 상시 열람 가능한 웹 미러 |

자세한 물리/논리 구성도와 엔지니어링 시행착오 및 트러블슈팅 사례는 [docs/submission.md](docs/submission.md) 및 정식 제출용 [docs/Kubernetes_구축결과서_장재희.pdf](docs/Kubernetes_구축결과서_장재희.pdf)를 참고해 주시기 바랍니다.
