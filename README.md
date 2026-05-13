# POC Cloud Native Platform — Kubernetes, Helm, ArgoCD, Grafana

Un POC pour comprendre les briques d'une cloud native platform en local.

Pipeline mis en place : `git push` → ArgoCD détecte → Helm render → Kubernetes apply → pods déployés et observables via Grafana.

## Stack

| Outil                    | Rôle                                                |
| ------------------------ | --------------------------------------------------- |
| **Kind**                 | Cluster Kubernetes local dans des conteneurs Docker |
| **Kubernetes**           | Orchestrateur de conteneurs                         |
| **Helm 3**               | Packaging/templating des manifests K8s              |
| **ArgoCD**               | Déploiement continu en mode GitOps                  |
| **Grafana + Prometheus** | Observabilité (métriques + dashboards)              |

## Pré-requis

- WSL2 avec Debian (ou Linux, ou macOS)
- Docker installé et fonctionnel (`docker version` doit répondre)
- Un compte GitHub
- Un Personal Access Token GitHub avec scope `repo`

## Installation des outils CLI

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# argocd CLI (optionnel)
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd /usr/local/bin/argocd
rm argocd
```

Vérifie les versions :

```bash
kubectl version --client
kind version
helm version
argocd version --client
```

## Phase 1 — Cluster Kind

### Créer le dossier de travail

```bash
mkdir -p ~/cnp-poc && cd ~/cnp-poc
```

### Config du cluster

Crée `kind-config.yaml` :

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cnp-poc
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
  - role: worker
```

### Lancer le cluster

```bash
kind create cluster --config kind-config.yaml
kubectl get nodes
kubectl get pods -A
```

Tu dois voir 3 nœuds en `Ready` et tous les pods système en `Running`.

## Phase 2 — Chart Helm

### Structure

```bash
mkdir -p charts/nginx-demo/templates
```

### Chart.yaml

```yaml
apiVersion: v2
name: nginx-demo
description: Un chart Helm pour déployer une app nginx de démo
type: application
version: 0.1.0
appVersion: "1.27"
```

### values.yaml

```yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.27-alpine"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  requests:
    cpu: "50m"
    memory: "64Mi"
  limits:
    cpu: "200m"
    memory: "128Mi"
```

### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: { { .Release.Name } }
  namespace: { { .Release.Namespace } }
  labels:
    app: { { .Release.Name } }
spec:
  replicas: { { .Values.replicaCount } }
  selector:
    matchLabels:
      app: { { .Release.Name } }
  template:
    metadata:
      labels:
        app: { { .Release.Name } }
    spec:
      containers:
        - name: nginx
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: { { .Values.image.pullPolicy } }
          ports:
            - containerPort: { { .Values.service.port } }
          resources: { { - toYaml .Values.resources | nindent 12 } }
```

### templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: { { .Release.Name } }
  namespace: { { .Release.Namespace } }
spec:
  type: { { .Values.service.type } }
  selector:
    app: { { .Release.Name } }
  ports:
    - port: { { .Values.service.port } }
      targetPort: { { .Values.service.port } }
      protocol: TCP
```

### Tester localement (optionnel)

```bash
helm lint ./charts/nginx-demo
helm template my-nginx ./charts/nginx-demo --namespace demo
```

Pour installer en standalone (sans ArgoCD) :

```bash
helm install my-nginx ./charts/nginx-demo --namespace demo --create-namespace
helm list -n demo
helm uninstall my-nginx -n demo
kubectl delete namespace demo
```

## Phase 3 — Repo Git

Crée un repo public `cnp-poc-gitops` sur GitHub (vide, pas de README ni .gitignore).

```bash
cd ~/cnp-poc
git init

cat > .gitignore << 'EOF'
manifests/
EOF

git add charts/ .gitignore
git commit -m "Initial nginx-demo chart"
git remote add origin https://github.com/YOUR_USER/cnp-poc-gitops.git
git branch -M main
git push -u origin main
```

Au push, utilise ton Personal Access Token GitHub comme mot de passe.

## Phase 4 — ArgoCD

### Install

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Attends que les pods démarrent :

```bash
kubectl get pods -n argocd -w
```

Si `argocd-applicationset-controller` est en `CrashLoopBackOff` (problème de CRD), désactive-le car on n'en a pas besoin :

```bash
kubectl scale deployment argocd-applicationset-controller -n argocd --replicas=0
```

### Mot de passe admin

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo ""
```

User : `admin`. Note le password.

### Accès UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Ouvre `https://localhost:8080`, accepte le certificat self-signed, connecte-toi.

### Créer l'Application

Crée `argocd/application.yaml` (remplace `YOUR_USER`) :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USER/cnp-poc-gitops.git
    targetRevision: main
    path: charts/nginx-demo
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

Note : on ne met PAS de bloc `helm.values` dans la spec, sinon ses valeurs override le `values.yaml` du chart.

Applique :

```bash
kubectl apply -f argocd/application.yaml
```

Dans l'UI, tu vois la carte `nginx-demo` passer en `Synced + Healthy`.

### Tester le pipeline GitOps

Modifie `values.yaml` (en local ou via l'interface GitHub) :

```yaml
replicaCount: 5
```

Commit + push. Dans ArgoCD, clique REFRESH (ou attends 3 min). Vérifie :

```bash
kubectl get pods -n demo
```

5 pods. Tu n'as jamais touché `kubectl apply` ou `helm install`.

### Tester le self-healing

```bash
kubectl scale deployment nginx-demo -n demo --replicas=10
kubectl get pods -n demo
```

10 pods, mais Git dit 5. Attends ou REFRESH dans l'UI :

```bash
kubectl get pods -n demo
```

ArgoCD a remis à 5.

### Rollback via Git

```bash
git revert HEAD --no-edit
git push
```

ArgoCD redescend à 3 pods.

## Phase 5 — Observabilité (à venir)

À venir : déployer la `kube-prometheus-stack` via ArgoCD pour avoir Prometheus + Grafana + Alertmanager + dashboards préconfigurés.

## Commandes utiles

### Kubernetes

```bash
kubectl get all -n demo                          # vue d'ensemble
kubectl get pods -n demo -o wide                 # avec les nœuds
kubectl describe pod <NOM> -n demo               # détails et events
kubectl logs <NOM-POD> -n demo                   # logs
kubectl exec -it <NOM-POD> -n demo -- sh         # shell dans un pod
kubectl port-forward svc/<SVC> -n demo 8080:80   # tunnel local
```

### Helm

```bash
helm list -n demo                                # releases dans un ns
helm history <RELEASE> -n demo                   # historique des revisions
helm get values <RELEASE> -n demo                # valeurs appliquées
helm rollback <RELEASE> <REV> -n demo            # rollback à une révision
```

### ArgoCD

```bash
kubectl get applications -n argocd               # liste des Apps
kubectl describe application <NOM> -n argocd     # détails
kubectl patch application <NOM> -n argocd \
  --type merge -p '{"operation":{"sync":{}}}'    # force le sync
```

### Kind

```bash
kind get clusters                                # liste les clusters
kind delete cluster --name cnp-poc               # supprime le cluster
docker ps                                        # voir les conteneurs-nœuds
docker exec -it cnp-poc-control-plane bash       # rentrer dans un nœud
```

## Nettoyage

Tout supprimer en une commande :

```bash
kind delete cluster --name cnp-poc
```

## Concepts clés

**Modèle déclaratif** : tu décris l'état souhaité, K8s/ArgoCD/Helm convergent en permanence vers cet état (boucle de réconciliation).

**Pod** : unité d'exécution K8s, contient 1+ conteneurs qui partagent le réseau. Éphémère.

**Deployment vs ReplicaSet** : le ReplicaSet maintient N pods d'une version. Le Deployment gère plusieurs ReplicaSets (un par version) pour faire des rolling updates et garder les anciennes versions pour les rollbacks instantanés.

**Service ClusterIP** : IP virtuelle stable interne au cluster, load-balance vers les pods qui matchent ses labels.

**Helm release** : instance déployée d'un chart. Versionnée, rollback-able.

**Application ArgoCD** : objet K8s (CRD) qui dit "surveille ce repo Git, déploie ce chemin dans ce namespace, garde sync".

**GitOps** : Git = source de vérité. Toute modif passe par un commit, un agent (ArgoCD) réconcilie le cluster vers cet état.

## Architecture finale

```
Toi (commit/push) → GitHub → ArgoCD (poll/webhook) → Helm (render) → K8s (apply) → Pods
                                                                                      ↓
                                                              Prometheus (scrape) → Grafana
```

## Liens utiles

- Kubernetes : https://kubernetes.io/docs/concepts/
- Helm : https://helm.sh/docs/
- ArgoCD : https://argo-cd.readthedocs.io/
- Kind : https://kind.sigs.k8s.io/
