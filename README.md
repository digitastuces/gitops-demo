
# 006 — GitOps Pipeline (Backend + Frontend) pour EKS

## 🎯 Objectif
Mettre en place un pipeline **GitOps** complet :
- Build d’images Docker (backend Flask + frontend Nginx)
- Push dans **Amazon ECR**
- Déploiement automatique via **ArgoCD** sur le cluster **EKS** (namespace `demo`)

## 📂 Structure
```
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

## ⚙️ Pré-requis
- EKS + ArgoCD opérationnels
- ECR dans `eu-west-3`
- Secrets GitHub : `AWS_OIDC_ROLE_ARN` (assume-role OIDC)

## 🚀 Déploiement
1. Push ce dossier dans ton repo Git (ex. `digitastuces/gitops-demo`).
2. Applique le projet et les apps ArgoCD :
   ```bash
   kubectl apply -n argocd -f 006-gitops-pipeline/argocd-apps/project.yaml
   kubectl apply -n argocd -f 006-gitops-pipeline/argocd-apps/backend-app.yaml
   kubectl apply -n argocd -f 006-gitops-pipeline/argocd-apps/frontend-app.yaml
   ```
3. Attends la synchro automatique (namespace `demo`).

## 🔍 Vérifications
```bash
kubectl -n demo get deploy,svc
kubectl -n demo port-forward svc/backend 8080:80 &
curl -s localhost:8080
```

## 🔒 Suite DevSecOps
- Scan d’images (**Trivy/Grype**) dans la CI
- Signature d’images (**Cosign**) + **Kyverno** (policy)
- Gatekeeper/OPA pour la conformité
- NetworkPolicies par microservice
