# resume-web Helm Chart

개인정보와 특정 회사명을 포함하지 않는 공개용 기술 이력서를 배포한다.

```bash
helm lint .
helm template resume-web . --namespace resume
helm upgrade --install resume-web . --namespace resume --create-namespace
```

기본값:

- Deployment: `resume-web`, replicas 2
- Service: `resume-web`, NodePort 30080
- ServiceAccount: `service-resume`, token automount enabled (`helm create` default)
- Container: unprivileged NGINX on port 8080
- UI: Bootstrap 5.3.8, bundled locally without a CDN dependency

웹 콘텐츠는 `files/index.html`에서 수정한다. 파일이 변경되면 checksum annotation이
달라져 Deployment가 자동으로 rolling update된다.

`helm create resume-web`가 만드는 기본 구조를 유지했다. `serviceAccount.create`와
`serviceAccount.automount`의 기본값은 모두 `true`이며, Deployment는 생성된
ServiceAccount를 `serviceAccountName`으로 사용한다. 최신 Kubernetes에서는 이 인증
정보가 짧은 수명의 projected token volume으로 Pod에 제공된다.
