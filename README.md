# Local Kubernetes GitOps CI/CD Demo

Tai lieu nay ghi lai toan bo qua trinh xay dung CI/CD theo mo hinh GitOps:

```text
App repo -> GitHub Actions -> Docker Hub -> GitOps repo -> Argo CD -> Kubernetes local
```

Trong demo nay co 2 repo:

```text
tuan-app-repo
  Chua source code ung dung Next.js, Dockerfile va GitHub Actions workflow.

k8scicd / tuan-gitops-repo
  Chua Kubernetes manifests va Argo CD Application.
```

## 1. Muc tieu

Khi developer push code len app repo, pipeline se tu dong:

1. Build Docker image tu source code.
2. Push image len Docker Hub.
3. Checkout GitOps repo.
4. Sua image tag trong `apps/demo-app/deployment.yaml`.
5. Commit va push thay doi vao GitOps repo.
6. Argo CD phat hien thay doi va sync vao Kubernetes.
7. Kubernetes rollout pod moi.

Ket qua cuoi cung: chi can push code, app tren Kubernetes local se duoc cap nhat tu dong.

## 2. Cau truc GitOps repo

```text
k8scicd
  argocd
    demo-app.yaml
  apps
    demo-app
      namespace.yaml
      deployment.yaml
      service.yaml
  README.md
```

Y nghia:

- `argocd/demo-app.yaml`: khai bao Argo CD Application.
- `apps/demo-app/namespace.yaml`: tao namespace cho app.
- `apps/demo-app/deployment.yaml`: khai bao Deployment chay container app.
- `apps/demo-app/service.yaml`: khai bao Service de truy cap app trong cluster.

## 3. Kubernetes local

Co the dung Docker Desktop Kubernetes, kind, hoac minikube. Kiem tra cluster:

```powershell
kubectl get nodes
```

Neu cluster hoat dong, lenh tren se tra ve node dang `Ready`.

## 4. Cai Argo CD

Tao namespace:

```powershell
kubectl create namespace argocd
```

Cai Argo CD:

```powershell
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Kiem tra pod:

```powershell
kubectl get pods -n argocd
```

Mo Argo CD UI local:

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Lay password admin ban dau:

```powershell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
```

Gia tri tra ve dang base64, can decode neu dung truc tiep qua PowerShell:

```powershell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | %{ [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
```

Dang nhap:

```text
https://localhost:8080
username: admin
password: password vua lay
```

## 5. Argo CD Application

File `argocd/demo-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/tuanpm2003/tuan-gitops-repo.git
    targetRevision: main
    path: apps/demo-app
  destination:
    server: https://kubernetes.default.svc
    namespace: demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Giai thich:

- `repoURL`: URL cua GitOps repo. Nen dung day du `https://...git`.
- `targetRevision`: branch Argo CD theo doi, o day la `main`.
- `path`: thu muc chua Kubernetes manifests.
- `destination.namespace`: namespace app se duoc deploy vao.
- `automated.prune`: neu resource bi xoa khoi Git, Argo CD se xoa trong cluster.
- `automated.selfHeal`: neu resource trong cluster bi sua lech voi Git, Argo CD se sua lai theo Git.
- `CreateNamespace=true`: cho phep tao namespace neu chua co.

Apply Application:

```powershell
kubectl apply -f D:\testDeploy\learncicd\k8scicd\argocd\demo-app.yaml
```

Kiem tra:

```powershell
kubectl get applications -n argocd
kubectl describe application demo-app -n argocd
```

Trang thai mong muon:

```text
SYNC STATUS: Synced
HEALTH STATUS: Healthy
```

## 6. Kubernetes manifests

### Namespace

File `apps/demo-app/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
```

Namespace `demo` la noi app chay.

### Deployment

File `apps/demo-app/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
        - name: demo-app
          image: <dockerhub-username>/tuan-app-repo:latest
          ports:
            - containerPort: 80
```

Giai thich:

- `replicas: 2`: chay 2 pod.
- `selector.matchLabels`: Deployment quan ly pod co label `app: demo-app`.
- `image`: Docker image cua app. Pipeline se tu dong thay tag image trong dong nay.
- `containerPort: 80`: container dung nginx, nen app lang nghe port 80.

Sau khi CI/CD chay, image se thanh dang:

```yaml
image: <dockerhub-username>/tuan-app-repo:<commit-sha>
```

Vi du:

```yaml
image: tuanpm2003/tuan-app-repo:5049dfd2f6b0e84ddd9e143cfae124d439f23b3a
```

### Service

File `apps/demo-app/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-app
  namespace: demo
spec:
  type: NodePort
  selector:
    app: demo-app
  ports:
    - port: 80
      targetPort: 80
```

Giai thich:

- `selector`: Service route traffic den pod co label `app: demo-app`.
- `port: 80`: port cua Service.
- `targetPort: 80`: port container.

Truy cap app bang port-forward:

```powershell
kubectl port-forward svc/demo-app -n demo 8080:80
```

Mo bang HTTP:

```text
http://localhost:8080
```

Khong dung `https://localhost:8080` vi Service nay khong cau hinh TLS.

## 7. App repo va Dockerfile

App repo la `tuan-app-repo`. Ung dung Next.js duoc build thanh static files bang cau hinh:

```js
const nextConfig = {
  output: "export",
  trailingSlash: true,
  images: {
    unoptimized: true,
  },
};

export default nextConfig;
```

Trong `package.json`, script build nen la:

```json
"build": "next build"
```

Dockerfile mau:

```Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM nginx:1.27-alpine
COPY --from=builder /app/out /usr/share/nginx/html
EXPOSE 80
```

Giai thich:

- Stage `builder` cai dependency va build app.
- `npm ci` dung cho moi truong CI vi cai dung theo `package-lock.json`.
- `npm run build` tao thu muc `out/`.
- Stage nginx chi chua file static va serve qua port 80.

Test local:

```powershell
cd D:\testDeploy\learncicd\tuan-app-repo
npm run build
docker build -t tuan-app-repo:test .
docker run --rm -p 8080:80 tuan-app-repo:test
```

Mo:

```text
http://localhost:8080
```

## 8. Docker Hub

Image tren Docker Hub co dang:

```text
<dockerhub-username>/tuan-app-repo:<tag>
```

Vi du:

```text
tuanpm2003/tuan-app-repo:latest
tuanpm2003/tuan-app-repo:5049dfd2f6b0e84ddd9e143cfae124d439f23b3a
```

Can tao Docker Hub access token:

```text
Docker Hub -> Account Settings -> Personal access tokens -> Generate new token
```

Quyen can co:

```text
Read & Write
```

Nen dung access token thay vi password account.

## 9. GitHub Secrets trong app repo

Trong repo `tuan-app-repo`, vao:

```text
Settings -> Secrets and variables -> Actions -> New repository secret
```

Tao cac secret:

```text
DOCKERHUB_USERNAME
DOCKERHUB_TOKEN
GITOPS_TOKEN
```

Y nghia:

- `DOCKERHUB_USERNAME`: username Docker Hub, khong phai email.
- `DOCKERHUB_TOKEN`: Docker Hub access token co quyen push image.
- `GITOPS_TOKEN`: GitHub token co quyen push vao repo `tuan-gitops-repo`.

`GITOPS_TOKEN` can co quyen `Contents: Read and write` voi GitOps repo.

## 10. GitHub Actions workflow

File trong app repo:

```text
tuan-app-repo/.github/workflows/deploy.yml
```

Workflow mau:

```yaml
name: Build and Update GitOps

on:
  push:
    branches: [main]

env:
  IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/tuan-app-repo

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout app repo
        uses: actions/checkout@v4

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push image
        run: |
          docker build -t $IMAGE_NAME:${{ github.sha }} -t $IMAGE_NAME:latest .
          docker push $IMAGE_NAME:${{ github.sha }}
          docker push $IMAGE_NAME:latest

      - name: Checkout GitOps repo
        uses: actions/checkout@v4
        with:
          repository: tuanpm2003/tuan-gitops-repo
          token: ${{ secrets.GITOPS_TOKEN }}
          path: gitops

      - name: Update image tag in GitOps repo
        run: |
          cd gitops
          sed -i "s|image: .*|image: $IMAGE_NAME:${{ github.sha }}|" apps/demo-app/deployment.yaml
          git config user.name "github-actions"
          git config user.email "github-actions@github.com"
          git add apps/demo-app/deployment.yaml
          git commit -m "deploy: update image to $IMAGE_NAME:${{ github.sha }}" || exit 0
          git push
```

Giai thich cac buoc:

1. `Checkout app repo`: lay source code app de build.
2. `Login to Docker Hub`: dang nhap Docker Hub bang secret.
3. `Build and push image`: build image va push 2 tag: commit SHA va `latest`.
4. `Checkout GitOps repo`: lay repo chua Kubernetes manifests.
5. `Update image tag`: sua dong `image:` trong Deployment sang tag moi.
6. `git commit` va `git push`: day thay doi len GitOps repo.
7. Argo CD tu phat hien commit moi va sync.

## 11. Vi sao dung commit SHA thay vi chi dung latest

Tag `latest` tien cho test nhanh, nhung khong ro version nao dang chay.

Tag commit SHA tot hon vi:

- Moi deployment gan voi mot commit cu the.
- De rollback.
- De debug khi co loi.
- GitOps repo ghi lai lich su image da deploy.

Vi vay workflow push ca 2 tag:

```text
latest
<commit-sha>
```

Nhung Deployment duoc update sang tag SHA.

## 12. Quy trinh deploy moi lan sua code

Trong app repo:

```powershell
cd D:\testDeploy\learncicd\tuan-app-repo
git add .
git commit -m "feat: update app"
git push
```

Sau do GitHub Actions se tu chay.

Kiem tra tren cluster:

```powershell
kubectl get applications -n argocd
kubectl get pods -n demo
kubectl get deployment demo-app -n demo -o jsonpath="{.spec.template.spec.containers[0].image}"
```

Neu pod running:

```powershell
kubectl port-forward svc/demo-app -n demo 8080:80
```

Mo:

```text
http://localhost:8080
```

## 13. Cac loi thuong gap

### Argo CD bao repository not found

Loi:

```text
failed to list refs: repository not found
```

Nguyen nhan:

- `repoURL` sai.
- Thieu `https://`.
- Repo private nhung Argo CD khong co credential.

Cach sua:

```yaml
repoURL: https://github.com/tuanpm2003/tuan-gitops-repo.git
```

### Pod ImagePullBackOff voi denied

Loi:

```text
error from registry: denied
```

Nguyen nhan:

- Registry tu choi quyen pull.
- Image private.
- Kubernetes chua co imagePullSecret.

Cach sua:

- Public image tren registry, hoac
- Tao `imagePullSecret`.

### Pod ImagePullBackOff voi not found

Loi:

```text
image:latest: not found
```

Nguyen nhan:

- Deployment dang dung tag chua ton tai.
- Pipeline chua push tag do.
- Image name sai.

Cach sua:

- Push them tag `latest`, hoac
- De workflow update Deployment sang tag SHA da push.

### Docker build fail vi thieu build-static.mjs

Loi:

```text
Cannot find module '/app/scripts/build-static.mjs'
```

Nguyen nhan:

`package.json` tro den script khong ton tai:

```json
"build": "node scripts/build-static.mjs"
```

Cach sua:

```json
"build": "next build"
```

### Docker Hub login unauthorized

Loi:

```text
unauthorized: incorrect username or password
```

Nguyen nhan:

- `DOCKERHUB_USERNAME` sai.
- Dung email thay vi username.
- `DOCKERHUB_TOKEN` sai, het han, hoac thieu quyen.
- Secret tao nham repo.

Cach sua:

- Tao lai Docker Hub access token.
- Cap nhat secret trong repo `tuan-app-repo`.
- Dam bao username la Docker Hub username.

### Checkout GitOps repo bao thieu token

Loi:

```text
Input required and not supplied: token
```

Nguyen nhan:

- Secret `GITOPS_TOKEN` chua ton tai.
- Tao sai ten secret.
- Tao secret nham repo.

Cach sua:

Tao secret `GITOPS_TOKEN` trong repo app `tuan-app-repo`, khong phai chi trong GitOps repo.

### Truy cap HTTPS vao app local bi loi

Loi tren browser:

```text
This site can't provide a secure connection
```

Nguyen nhan:

Dang vao:

```text
https://localhost:8080
```

Nhung port-forward dang map den HTTP port 80:

```powershell
kubectl port-forward svc/demo-app -n demo 8080:80
```

Cach dung dung:

```text
http://localhost:8080
```

## 14. Lenh debug nhanh

Xem Application:

```powershell
kubectl get applications -n argocd
kubectl describe application demo-app -n argocd
```

Xem resource app:

```powershell
kubectl get all -n demo
```

Xem pod:

```powershell
kubectl get pods -n demo
kubectl describe pod -n demo <pod-name>
```

Xem image dang chay:

```powershell
kubectl get deployment demo-app -n demo -o jsonpath="{.spec.template.spec.containers[0].image}"
```

Restart deployment:

```powershell
kubectl rollout restart deployment demo-app -n demo
```

Theo doi rollout:

```powershell
kubectl get pods -n demo -w
```

Port-forward:

```powershell
kubectl port-forward svc/demo-app -n demo 8080:80
```

## 15. Tong ket

Pipeline hoan chinh co dang:

```text
Code change
  -> GitHub Actions
  -> Docker build
  -> Docker Hub push
  -> Update GitOps deployment.yaml
  -> Git commit/push
  -> Argo CD sync
  -> Kubernetes rollout
```

Diem cot loi cua GitOps la cluster khong deploy truc tiep tu app repo. Cluster chi nghe theo GitOps repo. App repo chi tao image va cap nhat desired state trong GitOps repo. Nho vay moi lan deploy deu co lich su ro rang, de debug, de rollback, va de quan ly thay doi bang Git.
