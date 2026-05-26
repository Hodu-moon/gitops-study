# GitOps Study with ArgoCD 🚀

이 프로젝트는 Kubernetes 환경에서 ArgoCD를 활용하여 GitOps 방식으로 애플리케이션을 배포하고 관리하는 실습 저장소입니다.

---

## 📂 프로젝트 구조

```text
gitops-study/
├── README.md           # 프로젝트 가이드 및 문제 해결 문서
└── my-app/             # 배포용 쿠버네티스 매니페스트 폴더
    ├── deployment.yaml # Nginx Deployment 정의
    └── service.yaml    # Nginx Service 정의
```

---

## 1. ArgoCD 서비스 외부 노출 (접속 방법) 🌐

설치 직후 ArgoCD Server의 `LoadBalancer`가 `<pending>` 상태이거나 `http://공인IP` 접속 시 `404 Not Found`가 발생할 때 아래 방법으로 접속할 수 있습니다.

### 방법 A. NodePort와 HTTPS로 직접 접속 (권장)
ArgoCD는 자체 TLS(HTTPS) 보안이 켜져 있으므로 **HTTPS용 NodePort(443 매핑 포트)**를 통해 직접 접속합니다.
* **접속 주소:** `https://<공인IP>:<NodePort_443>` (예: `https://공인IP:30290`)
* *참고: 브라우저 인증서 경고 화면에서 `고급 > 이동(안전하지 않음)`을 클릭하여 접속합니다.*

### 방법 B. Port-Forwarding 임시 접속
로컬 PC 터미널에서 클러스터의 ArgoCD 서비스 포트를 포워딩하여 접속합니다.
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
* **접속 주소:** `https://localhost:8080`

### 방법 C. Ingress 설정 및 insecure 모드 활성화 (도메인/IP 접속)
포트 번호 없이 `http://공인IP`로만 접속하려면 ArgoCD를 HTTP(insecure) 모드로 바꾸고 Ingress를 생성해야 합니다.

1. **insecure 모드 활성화:**
   ```bash
   kubectl edit cm argocd-cmd-params-cm -n argocd
   # data 항목 하단에 server.insecure: "true" 추가 후 저장
   ```
2. **ArgoCD 서버 재시작:**
   ```bash
   kubectl rollout restart deployment argocd-server -n argocd
   ```
3. **Ingress 리소스 배포:**
   `my-app/` 폴더에 Ingress YAML을 만들어 적용하거나 직접 배포합니다.

---

## 2. Nginx 애플리케이션 매니페스트 (Manifests) 📄

ArgoCD를 통해 배포할 Nginx 웹 서버의 설정 파일 예시입니다. `my-app/` 폴더 안에 각각 저장합니다.

### 1) `my-app/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

### 2) `my-app/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

---

## 3. ArgoCD Application 생성 단계 🛠️

ArgoCD UI에서 **`+ NEW APP`**을 누르고 아래 설정을 입력하여 배포 애플리케이션을 생성합니다.

| 설정 구분 | 설정 항목 | 입력 값 | 설명 |
| :--- | :--- | :--- | :--- |
| **General** | Application Name | `nginx-app` | 생성할 애플리케이션 이름 |
| | Project Name | `default` | 기본 프로젝트 분류 |
| | Sync Policy | `Manual` 또는 `Automatic` | Git 변경사항 반영 방식 |
| **Source** | Repository URL | `https://github.com/<유저ID>/gitops-study.git` | 본인의 GitHub 레포 주소 |
| | Revision | `HEAD` 또는 `main` | 배포할 Git 브랜치 |
| | Path | `my-app` | 매니페스트가 들어있는 폴더명 |
| **Destination** | Cluster URL | `https://kubernetes.default.svc` | ArgoCD가 돌아가는 로컬 클러스터 |
| | Namespace | `default` | 배포할 쿠버네티스 네임스페이스 |

### 배포 반영하기 (Git Push)
매니페스트 작성이 완료되면 GitHub에 푸시하여 ArgoCD가 변경 사항을 인지하고 자동으로 클러스터 상태와 동기화(Sync)하도록 만듭니다.
```bash
git add my-app/
git commit -m "feat: add nginx manifests"
git push origin main
```
