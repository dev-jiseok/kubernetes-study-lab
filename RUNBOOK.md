# Week 3 재현 및 발표 시연

작성자: 이지석 / dev-jiseok. 공용 레포에는 3주차만 제출합니다.

## 최종 검증 결과

소스 저장소는 Private, GHCR 패키지는 Public입니다. 공개 전환 후 개인 자격 증명 없는 manifest 요청이 HTTP 200이고, registry Secret과 ServiceAccount/Pod imagePullSecrets 없이 새 SHA 이미지로 v1·v2 직접 배포와 v3 자동 GitOps 배포를 완료했습니다.

실측 전체 값은 `evidence/week3-public-verification.json`에 있습니다. 과거 파일의 Private/임시 인증 기록은 공개 전환 이전의 이력입니다. 현재 재현에는 이미지 pull 토큰이 필요하지 않습니다.

## 클러스터와 도구

kind 0.33.0, node image `kindest/node:v1.35.0`, Kubernetes 1.35.0, kubectl 1.34.1, Helm 4.3.2, Docker Desktop 4.55.0 / Engine 29.1.3, Argo CD 3.6.2.

```bash
kind create cluster --name jiseok-netlab --image kindest/node:v1.35.0 --config infra/kind.yaml
kubectl --context kind-jiseok-netlab get nodes
for ns in cicd-plain cicd-kustomize cicd-helm cicd-ci; do
  kubectl --context kind-jiseok-netlab create namespace "$ns" --dry-run=client -o yaml | kubectl --context kind-jiseok-netlab apply -f -
done
kubectl --context kind-jiseok-netlab -n cicd-plain apply -f deploy/plain
kubectl --context kind-jiseok-netlab -n cicd-kustomize apply -k deploy/kustomize/overlays/dev
helm --kube-context kind-jiseok-netlab upgrade --install nginx deploy/helm/nginx -n cicd-helm
kubectl kustomize deploy/kustomize/overlays/prod
helm template nginx deploy/helm/nginx --set replicaCount=2
```

이 작업 환경은 기본 kubeconfig를 바꾸지 않고 별도 `.runtime/kubeconfig`를 사용했습니다. 이 파일은 저장소 밖의 로컬 자격 증명입니다. 이 환경에서 아래 명령을 실행하려면 먼저 해당 kubeconfig 경로를 `KUBECONFIG`로 지정합니다.

## Direct CI

공개 이미지로 최종 검증한 과제 2 워크플로는 `examples/ci-direct-public.yml`에 보존했습니다. `examples/ci-direct.yml`은 공개 전환 전 임시 인증을 사용한 과거 기록입니다. self-hosted runner는 개인 비공개 레포에 등록하고 `jiseok-k8s` 라벨과 이 실습 클러스터 kubeconfig를 갖도록 실행합니다. main push만 배포하고 외부 PR은 실행하지 않습니다.

## GitOps

```bash
kubectl --context kind-jiseok-netlab create namespace argocd
kubectl --context kind-jiseok-netlab apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.6.2/manifests/install.yaml
# 비공개 저장소 전용 read-only deploy key를 등록하고 Argo CD repository Secret에 연결합니다.
# 개인키, 토큰, kubeconfig를 Git에 넣지 않습니다.
kubectl --context kind-jiseok-netlab apply -f infra/argocd-application.yaml
kubectl --context kind-jiseok-netlab -n argocd get application nginx-api
```

현재 `.github/workflows/ci.yml`은 GitHub-hosted runner에서 이미지 빌드와 Git 갱신만 합니다. Argo CD가 `main:deploy/gitops`를 읽고 자동 적용합니다. 과거의 `registry-session.yml`은 공개 전환 전 한시적 인증을 위한 파일이며, 공개 이미지 최종 검증에서는 실행하지 않았습니다. 현재 재현에는 필요하지 않습니다.

## 발표 시연 (3분)

터미널 1:
```bash
kubectl --context kind-jiseok-netlab -n cicd-gitops port-forward svc/nginx-api 18090:80
```
터미널 2:
```bash
curl http://localhost:18090/api/hello
kubectl --context kind-jiseok-netlab -n argocd get application nginx-api
kubectl --context kind-jiseok-netlab -n cicd-gitops get deploy,rs,pods
```

Argo CD UI를 직접 확인하려면 별도 터미널에서 `kubectl -n argocd port-forward svc/argocd-server 18443:443`를 실행하고 `https://localhost:18443`를 엽니다. 관리자 초기 비밀번호는 로컬 Secret으로만 확인하고 캡처하거나 보고서에 넣지 않습니다.

라이브 시연 전에는 로컬 kind와 Argo CD가 실행 중인지 확인합니다. 새 응답 버전을 시연하려면 app/nginx.conf를 수정해 main에 push하고 Actions 성공, 새 manifest commit, Argo CD Synced/Healthy, 실제 응답 순서로 확인합니다.
