
# 🚀 006 — GitOps Pipeline (Backend + Frontend) pour EKS

## 🎯 Objectif

Mettre en place un pipeline **GitOps** complet et sécurisé pour le cluster **EKS**, comprenant :
- **Build & push** d’images Docker (Backend Flask + Frontend Nginx)
- **Publication dans Amazon ECR**
- **Déploiement automatique via ArgoCD** (GitOps)
- **Évolutions DevSecOps** (Trivy, Cosign, Kyverno…)

---

## 📂 Structure du projet

```bash
006-gitops-pipeline/
├── backend/
│   ├── app.py
│   ├── Dockerfile
│   ├── requirements.txt
│   └── manifests/
│       └── k8s.yaml
├── frontend/
│   ├── index.html
│   ├── Dockerfile
│   └── manifests/
│       └── k8s.yaml
├── argocd-apps/
│   ├── project.yaml
│   ├── backend-app.yaml
│   └── frontend-app.yaml
└── .github/workflows/
    └── build-and-push-ecr.yml
```

---

## ⚙️ Pré-requis

- ✅ Cluster **EKS** déjà fonctionnel (ex : `eks-devsecops`)
- ✅ **ArgoCD** déployé sur le namespace `argocd`
- ✅ **AWS SSO** connecté :
  ```bash
  aws sso login --profile formation
  export AWS_PROFILE=formation AWS_REGION=eu-west-3 AWS_SDK_LOAD_CONFIG=1
  ```
- ✅ Docker installé et connecté à ton compte AWS
- ✅ ECR créé ou autorisé dans la région `eu-west-3`
- ✅ Secret GitHub `AWS_OIDC_ROLE_ARN` (si tu veux utiliser GitHub Actions)

---

## 🧱 Étape 1 — Construction locale des images Docker

### 🧩 Backend (Flask)
```bash
cd 006-gitops-pipeline/backend
docker build -t gitops-demo-backend:latest .
```

### 🧩 Frontend (Nginx)
```bash
cd 006-gitops-pipeline/frontend
docker build -t gitops-demo-frontend:latest .
```

---

## 🏗️ Étape 2 — Création des dépôts ECR

### 🔹 Backend
```bash
aws ecr describe-repositories   --repository-names gitops-demo-backend   --region eu-west-3 --profile formation >/dev/null 2>&1 || aws ecr create-repository   --repository-name gitops-demo-backend   --region eu-west-3 --profile formation
```

### 🔹 Frontend
```bash
aws ecr describe-repositories   --repository-names gitops-demo-frontend   --region eu-west-3 --profile formation >/dev/null 2>&1 || aws ecr create-repository   --repository-name gitops-demo-frontend   --region eu-west-3 --profile formation
```

---

## 🔐 Étape 3 — Connexion Docker à ECR

```bash
aws ecr get-login-password --region eu-west-3 --profile formation | docker login --username AWS --password-stdin 065967698083.dkr.ecr.eu-west-3.amazonaws.com
```

> ⚠️ Si tu obtiens une erreur “no basic auth credentials” : relance `aws sso login --profile formation`.

---

## 🚢 Étape 4 — Tag & Push des images vers ECR

### Backend
```bash
docker tag gitops-demo-backend:latest   065967698083.dkr.ecr.eu-west-3.amazonaws.com/gitops-demo-backend:latest

docker push 065967698083.dkr.ecr.eu-west-3.amazonaws.com/gitops-demo-backend:latest
```

### Frontend
```bash
docker tag gitops-demo-frontend:latest   065967698083.dkr.ecr.eu-west-3.amazonaws.com/gitops-demo-frontend:latest

docker push 065967698083.dkr.ecr.eu-west-3.amazonaws.com/gitops-demo-frontend:latest
```

---

## ⚙️ Étape 5 — Mise à jour des manifests Kubernetes

Remplace dans :
```
backend/manifests/k8s.yaml
frontend/manifests/k8s.yaml
```

👉 `<ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com/...`
par :
```
065967698083.dkr.ecr.eu-west-3.amazonaws.com/gitops-demo-backend:latest
065967698083.dkr.ecr.eu-west-3.amazonaws.com/gitops-demo-frontend:latest
```

---

## 🌀 Étape 6 — Déploiement GitOps avec ArgoCD

1. Pousse ce dossier dans ton repo GitHub, ex. :
   ```
   github.com/digitastuces/gitops-demo
   ```

2. Applique le projet et les applications :
   ```bash
   kubectl apply -n argocd -f 006-gitops-pipeline/argocd-apps/project.yaml
   kubectl apply -n argocd -f 006-gitops-pipeline/argocd-apps/backend-app.yaml
   kubectl apply -n argocd -f 006-gitops-pipeline/argocd-apps/frontend-app.yaml
   ```

3. Vérifie la synchronisation automatique dans ArgoCD UI.

---

## 🔍 Vérifications

### Kubernetes
```bash
kubectl -n demo get all
```

### Accès local au backend
```bash
kubectl -n demo port-forward svc/backend 8080:80 &
curl -s localhost:8080
```

### Accès via ALB (si Ingress décommenté)
```bash
curl -I https://backend.eks.digitastuces.com
```

---

## 🧰 Étape 7 — (Optionnel) Automatisation CI/CD via GitHub Actions

Le workflow `.github/workflows/build-and-push-ecr.yml` :
- Build automatiquement `backend` et `frontend`
- Pousse les images dans ECR
- Déclenche la mise à jour ArgoCD (GitOps)

Configure :
- Secret GitHub : `AWS_OIDC_ROLE_ARN` (assume-role IAM pour ton repo)
- Branche `main` comme cible

---

## 🧩 Suite DevSecOps — Sécurisation et conformité

### 1️⃣ Analyse des vulnérabilités
- Ajoute **Trivy** ou **Grype** dans ton workflow GitHub Actions :
  ```bash
  trivy image $IMAGE_URI
  ```
  ou
  ```bash
  grype $IMAGE_URI
  ```

### 2️⃣ Signature des images (Cosign)
- Génère une clé :
  ```bash
  cosign generate-key-pair
  cosign sign $IMAGE_URI
  cosign verify $IMAGE_URI
  ```

### 3️⃣ Politiques Kyverno
- Interdire le déploiement d’images non signées :
  ```yaml
  validationFailureAction: enforce
  rules:
    - name: verify-signed-images
      match:
        resources:
          kinds: ["Pod"]
      verifyImages:
        - image: "065967698083.dkr.ecr.eu-west-3.amazonaws.com/*"
          key: "cosign.pub"
  ```

### 4️⃣ Contrôle d’admission (OPA / Gatekeeper)
- Crée des policies de conformité : labels obligatoires, namespaces restreints, etc.

### 5️⃣ Sécurité réseau
- Définis des **NetworkPolicies** par microservice pour limiter les flux inter-pods.

---

## 👨‍💻 Auteur
**Adama DIENG**
📧 [formation@digitastuces.com](mailto:formation@digitastuces.com)
🌐 [digitastuces.com](https://digitastuces.com)

🧠 _Ce module s’inscrit dans le plan de formation DevSecOps complet :_
**AWS SSO → Terraform → EKS → ALB (HTTPS) → ArgoCD → GitOps → Trivy/Cosign/Kyverno**
