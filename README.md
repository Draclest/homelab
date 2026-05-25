# Homelab — Infrastructure GitOps

Ce dépôt gère l'ensemble des applications déployées sur le cluster k3s personnel via **ArgoCD** en mode GitOps. Tout changement poussé sur la branche `master` est automatiquement appliqué sur le cluster.

---

## Architecture générale

```
homelab/                        ← Root chart (App of Apps)
├── Chart.yaml                  ← Déclare le chart racine
├── values.yaml                 ← URL du repo et cluster cible
├── templates/                  ← Une Application ArgoCD par service
│   ├── argocd.yaml
│   ├── sealed-secrets.yaml
│   ├── nextcloud.yaml
│   ├── eck-operator.yaml
│   ├── elasticsearch.yaml
│   ├── kibana.yaml
│   └── otel-collector.yaml
└── charts/                     ← Un sous-chart Helm par service
    ├── argocd/
    ├── sealed-secrets/
    ├── nextcloud/
    ├── eck-operator/
    ├── elasticsearch/
    ├── kibana/
    └── otel-collector/
```

### Principe de fonctionnement

```
GitHub (master)
      │
      ▼
ArgoCD (root App of Apps)
      │  lit templates/*.yaml
      ▼
ArgoCD crée une Application par service
      │  chaque Application pointe vers charts/<service>/
      ▼
k3s applique les manifests Kubernetes
```

Le **root chart** (`Chart.yaml` + `templates/`) est un **App of Apps** : il déclare une `Application` ArgoCD pour chaque service. Chaque Application pointe ensuite vers son propre sous-chart dans `charts/`.

---

## Services déployés

| Service | Namespace | Description |
|---|---|---|
| ArgoCD | `argocd` | Moteur GitOps — synchronise le repo vers le cluster |
| Sealed Secrets | `sealed-secrets` | Chiffrement des secrets Kubernetes en Git |
| Nextcloud | `nextcloud` | Cloud personnel (fichiers, calendrier, contacts) |
| ECK Operator | `elastic-system` | Opérateur Elastic Cloud on Kubernetes |
| Elasticsearch | `elastic` | Moteur de recherche et stockage de logs |
| Kibana | `kibana` | Interface de visualisation des logs |
| OTel Collector | `monitoring` | Collecte et export de traces/métriques/logs |

---

## Prérequis

- Cluster k3s fonctionnel
- `kubectl` configuré avec accès au cluster
- `helm` v3 installé
- `kubeseal` installé (pour créer des secrets chiffrés)
- Accès push au dépôt GitHub

---

## Ajouter une nouvelle application

### Étape 1 — Créer le sous-chart

```bash
mkdir -p charts/<nom-app>/templates
```

Créer `charts/<nom-app>/Chart.yaml` :

```yaml
apiVersion: v2
name: <nom-app>
description: Description courte
type: application
version: 0.1.0
appVersion: "<version-upstream>"

# Si tu utilises un chart Helm public (optionnel) :
dependencies:
  - name: <chart-name>
    version: "<version>"
    repository: "https://<repo-helm>"
```

Créer `charts/<nom-app>/values.yaml` (peut rester vide ou contenir la configuration).

Si tu n'utilises pas de chart public, crée tes manifests directement dans `charts/<nom-app>/templates/`.

### Étape 2 — Déclarer l'Application ArgoCD

Créer `templates/<nom-app>.yaml` :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <nom-app>
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: {{ .Values.spec.source.repoURL }}
    targetRevision: {{ .Values.spec.source.targetRevision }}
    path: charts/<nom-app>
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: {{ .Values.spec.destination.server }}
    namespace: <nom-app>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        maxDuration: 3m0s
        factor: 2
```

### Étape 3 — Gérer les secrets (si nécessaire)

Ne jamais stocker de secrets en clair dans Git. Utiliser **Sealed Secrets** :

```bash
# 1. Créer le secret Kubernetes en local (jamais commité)
kubectl create secret generic <nom-secret> \
  --namespace <nom-app> \
  --from-literal=ma-cle=ma-valeur \
  --dry-run=client -o yaml > /tmp/secret.yaml

# 2. Le chiffrer avec kubeseal
kubeseal --format yaml < /tmp/secret.yaml > charts/<nom-app>/templates/sealed-secret.yaml

# 3. Supprimer le fichier temporaire
rm /tmp/secret.yaml
```

Le fichier `sealed-secret.yaml` peut être commité sans risque — seul le cluster peut le déchiffrer.

### Étape 4 — Pousser sur GitHub

```bash
git add charts/<nom-app>/ templates/<nom-app>.yaml
git commit -m "feat: add <nom-app>"
git push origin master
```

ArgoCD détecte le commit et déploie automatiquement la nouvelle application dans les secondes qui suivent.

---

## Mettre à jour une application existante

```bash
# Modifier les fichiers concernés dans charts/<nom-app>/
# Puis bumper la version dans charts/<nom-app>/Chart.yaml

git add charts/<nom-app>/
git commit -m "fix: update <nom-app> to <version>"
git push origin master
```

ArgoCD applique les changements automatiquement (`selfHeal: true`).

---

## Commandes utiles

```bash
# Voir l'état de toutes les applications ArgoCD
kubectl get applications -n argocd

# Forcer une synchronisation manuelle
kubectl -n argocd patch app <nom-app> \
  --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'

# Voir les logs d'un pod
kubectl logs -n <namespace> deployment/<nom-app> -f

# Accéder à l'interface ArgoCD
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Puis ouvrir https://localhost:8080

# Récupérer le mot de passe ArgoCD initial
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

---

## Accès aux services

| Service | Accès |
|---|---|
| Nextcloud | [https://nextcloud.draclest.ovh](https://nextcloud.draclest.ovh) via Nginx Proxy Manager |
| ArgoCD | `kubectl port-forward` ou via NPM |
| Kibana | `kubectl port-forward` ou via NPM |

L'exposition externe passe par **Nginx Proxy Manager (NPM)** qui tourne sur le réseau local et termine le TLS avant de forwarder vers les NodePorts k3s.
