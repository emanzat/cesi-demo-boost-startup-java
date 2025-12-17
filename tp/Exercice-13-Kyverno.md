# Exercice 13 : Tester les Politiques Kyverno avec nginx

[⬅️ Exercice précédent](Exercice-12-ArgoCD.md) | [🏠 Sommaire](README.md)

---

## 🎯 Objectif

Comprendre et tester les politiques de sécurité Kyverno déjà déployées sur le cluster en utilisant nginx et votre application demo-java.

## ⏱️ Durée Estimée

45 minutes

---

## 📚 Qu'est-ce que Kyverno ?

**Kyverno** est un moteur de politiques natif Kubernetes qui permet de :

- ✅ **Valider** : Bloquer les ressources non conformes
- 🔧 **Muter** : Modifier automatiquement les ressources pour les rendre conformes
- 📝 **Générer** : Créer automatiquement des ressources associées
- 📊 **Vérifier** : Auditer les ressources existantes

### 📂 Organisation des politiques dans le projet

Les politiques Kyverno sont stockées dans `k8s/kyverno/admin/cluster-policies.yaml` et déployées automatiquement via ArgoCD par l'admin.

**Vous n'avez pas besoin de les déployer** - elles sont déjà actives sur le cluster !

**3 ClusterPolicies déployées** (scope: namespaces `cesi*`) :

| Politique | Type | Règles | Que fait-elle ? |
|-----------|------|--------|-----------------|
| `formation-security-policies` | Validation (Enforce) | 5 | Bloque les déploiements non conformes |
| `formation-mutation-policies` | Mutation | 3 | Ajoute automatiquement labels et securityContext |
| `formation-generate-policies` | Génération | 3 | Crée ResourceQuota, NetworkPolicy, ConfigMap |

**Dans cet exercice, vous allez :**
1. ✅ Vérifier que les politiques sont déployées
2. ✅ Tester avec nginx pour voir les blocages et modifications automatiques
3. ✅ Adapter votre application demo-java pour être conforme

### Kyverno vs OPA/Gatekeeper

| Aspect | Kyverno | OPA/Gatekeeper |
|--------|---------|----------------|
| Langage | YAML natif | Rego (langage spécifique) |
| Courbe d'apprentissage | Facile | Difficile |
| Intégration K8s | Native | Via CRD |
| Cas d'usage | Politiques K8s | Politiques génériques |
| Mutation | Native | Limitée |
| Génération | Native | Non supportée |

---

## ⚠️ IMPORTANT : Configuration Personnalisée

**Chaque étudiant a son propre environnement isolé sur le cluster.**

### 🔢 Votre numéro d'étudiant

Au début de la session, vous avez reçu un **numéro d'étudiant** de 1 à 17.

**Exemple** : Si vous êtes l'étudiant n°3, votre numéro est `3`.

### 📋 Namespace de travail

Tout au long de cet exercice, vous travaillerez dans **votre namespace** : `cesiX` (où X = votre numéro).

**Exemples** :
- Étudiant n°1 → namespace `cesi1`
- Étudiant n°3 → namespace `cesi3`
- Étudiant n°10 → namespace `cesi10`
- Étudiant n°17 → namespace `cesi17`

**⚠️ Remplacez `cesiX` par votre namespace réel dans toutes les commandes !**

---

## 📋 Prérequis

- Cluster Kubernetes fonctionnel (K3s)
- `kubectl` configuré
- Votre namespace `cesiX` créé (créé automatiquement lors de l'Exercice 12)
- Kyverno déjà installé sur le cluster (fait par l'admin)
- Politiques Kyverno déjà déployées (fait par l'admin via ArgoCD)

---

## 📝 Instructions

### Étape 13.1 : Vérifier l'installation de Kyverno

Kyverno est déjà installé sur le cluster de formation. Vérifiez l'installation :

```bash
# Vérifier que Kyverno est installé
kubectl get pods -n kyverno

# Résultat attendu :
# NAME                       READY   STATUS    RESTARTS   AGE
# kyverno-xxxxx              1/1     Running   0          Xd
```

### Étape 13.2 : Vérifier les politiques déployées

Les politiques Kyverno ont déjà été déployées par l'admin via ArgoCD. Vérifions-les :

```bash
# Lister toutes les ClusterPolicies
kubectl get clusterpolicy

# Résultat attendu :
# NAME                             BACKGROUND   VALIDATE ACTION   READY
# formation-security-policies      true         Enforce           True
# formation-mutation-policies      true         N/A               True
# formation-generate-policies      true         N/A               True

# Voir les détails d'une politique
kubectl describe clusterpolicy formation-security-policies
```

### Étape 13.3 : Vérifier les ressources générées automatiquement

Kyverno a automatiquement créé des ressources dans votre namespace. Vérifions :

```bash
# IMPORTANT : Remplacez cesiX par votre namespace (cesi1, cesi2, etc.)

# 1. Vérifier la ResourceQuota
kubectl get resourcequota -n cesiX

# Résultat attendu :
# NAME              AGE   REQUEST                                              LIMIT
# namespace-quota   Xd    persistentvolumeclaims: 0/5, pods: 2/20, ...         limits.cpu: 2/8, ...

# Voir les détails
kubectl describe resourcequota namespace-quota -n cesiX

# 2. Vérifier la NetworkPolicy
kubectl get networkpolicy -n cesiX

# Résultat attendu :
# NAME                   POD-SELECTOR   AGE
# default-deny-ingress   <none>         Xd

# 3. Vérifier le ConfigMap d'information
kubectl get configmap namespace-info -n cesiX

# Voir le contenu du ConfigMap
kubectl get configmap namespace-info -n cesiX -o yaml
```

✅ **Fantastique !** Kyverno a automatiquement créé 3 ressources dans votre namespace lors de sa création.

---

## 🧪 SECTION 1 : Tests de VALIDATION - Cas qui DOIVENT ÉCHOUER

Les politiques de validation bloquent les déploiements non conformes. Testons-les !

### Test 1.1 : Conteneur privileged (DOIT ÉCHOUER ❌)

```bash
# Tester un conteneur privileged (DANGEREUX)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-privileged
  namespace: cesiX  # ⚠️ REMPLACEZ par votre namespace !
  labels:
    test: "validation-privileged"
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    securityContext:
      privileged: true  # ❌ VIOLATION
EOF
```

**Résultat attendu** :
```
Error from server: admission webhook "validate.kyverno.svc" denied the request:

resource Pod/cesiX/nginx-privileged was blocked due to the following policies

formation-security-policies:
  deny-privileged-containers: ❌ SÉCURITÉ : Les conteneurs privileged sont interdits.
```

✅ **Parfait !** Kyverno bloque le déploiement d'un conteneur privileged.

**Explication** : Un conteneur privileged a un accès complet au host, ce qui représente un risque de sécurité majeur.

---

### Test 1.2 : Pas de limites de ressources (DOIT ÉCHOUER ❌)

```bash
# Tester un déploiement sans limites de ressources
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-no-limits
  namespace: cesiX  # ⚠️ REMPLACEZ par votre namespace !
  labels:
    test: "validation-no-limits"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-no-limits
  template:
    metadata:
      labels:
        app: nginx-no-limits
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        # ❌ VIOLATION : pas de resources.limits ni requests
EOF
```

**Résultat attendu** :
```
Error from server: admission webhook "validate.kyverno.svc" denied the request:

resource Deployment/cesiX/nginx-no-limits was blocked due to the following policies

formation-security-policies:
  require-resources-limits: ❌ RESSOURCES : Les conteneurs doivent avoir des limites...
```

✅ **Excellent !** Kyverno force la définition de limites de ressources.

**Explication** : Sans limites, un conteneur peut consommer toutes les ressources du nœud et impacter les autres applications.

---

### Test 1.3 : Exécution en tant que root (DOIT ÉCHOUER ❌)

```bash
# Tester un conteneur qui s'exécute en tant que root
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-as-root
  namespace: cesiX  # ⚠️ REMPLACEZ par votre namespace !
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-as-root
  template:
    metadata:
      labels:
        app: nginx-as-root
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "250m"
            memory: "256Mi"
        securityContext:
          runAsUser: 0  # ❌ VIOLATION : root user (UID 0)
EOF
```

**Résultat attendu** : Erreur de Kyverno

✅ **Bloqué !** Kyverno empêche l'exécution en tant que root.

**Explication** : Le principe du moindre privilège exige que les conteneurs ne s'exécutent pas en tant que root.

---

### Test 1.4 : Pas de readOnlyRootFilesystem (DOIT ÉCHOUER ❌)

```bash
# Tester sans readOnlyRootFilesystem
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-writable-root
  namespace: cesiX  # ⚠️ REMPLACEZ par votre namespace !
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-writable-root
  template:
    metadata:
      labels:
        app: nginx-writable-root
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "250m"
            memory: "256Mi"
        securityContext:
          runAsUser: 101
          # ❌ VIOLATION : pas de readOnlyRootFilesystem: true
EOF
```

**Résultat attendu** : Erreur de Kyverno

✅ **Rejeté !** Le système de fichiers root doit être en lecture seule.

**Explication** : Un rootfs en lecture seule empêche la modification de binaires et l'écriture de malwares.

---

### Test 1.5 : Trop de replicas (DOIT ÉCHOUER ❌)

```bash
# Tester avec plus de 5 replicas
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-too-many-replicas
  namespace: cesiX  # ⚠️ REMPLACEZ par votre namespace !
spec:
  replicas: 10  # ❌ VIOLATION : max 5 replicas
  selector:
    matchLabels:
      app: nginx-replicas
  template:
    metadata:
      labels:
        app: nginx-replicas
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        resources:
          limits:
            cpu: "100m"
            memory: "128Mi"
          requests:
            cpu: "50m"
            memory: "64Mi"
        securityContext:
          runAsUser: 101
          readOnlyRootFilesystem: true
        volumeMounts:
        - name: cache
          mountPath: /var/cache/nginx
      volumes:
      - name: cache
        emptyDir: {}
EOF
```

**Résultat attendu** : Erreur de Kyverno

✅ **Limité !** Maximum 5 replicas pour partager les ressources du cluster.

**Explication** : Le cluster est partagé entre tous les étudiants, donc on limite le nombre de replicas.

---

## 🧪 SECTION 2 : Tests de VALIDATION - Cas qui DOIT RÉUSSIR

### Test 2.1 : Déploiement nginx CONFORME (DOIT RÉUSSIR ✅)

Maintenant, testons un déploiement nginx totalement conforme :

```bash
# Déployer nginx de manière sécurisée et conforme
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-secure
  namespace: cesiX  # ⚠️ REMPLACEZ par votre namespace !
  labels:
    app: nginx-secure
    test: "validation-success"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-secure
  template:
    metadata:
      labels:
        app: nginx-secure
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        fsGroup: 101
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 8080
          name: http
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "250m"
            memory: "256Mi"
        securityContext:
          allowPrivilegeEscalation: false
          runAsNonRoot: true
          runAsUser: 101
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: cache
          mountPath: /var/cache/nginx
        - name: run
          mountPath: /var/run
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: cache
        emptyDir: {}
      - name: run
        emptyDir: {}
      - name: tmp
        emptyDir: {}
EOF
```

**Résultat attendu** :
```
deployment.apps/nginx-secure created
```

✅ **Succès !** Le déploiement est accepté car il respecte toutes les politiques.

**Vérifiez le déploiement** :
```bash
# Voir les pods (REMPLACEZ cesiX)
kubectl get pods -n cesiX -l app=nginx-secure

# Voir les détails du déploiement
kubectl describe deployment nginx-secure -n cesiX

# Tester nginx
kubectl exec -n cesiX deployment/nginx-secure -- curl -s localhost:8080
```

---

## 🧪 SECTION 3 : Tests de MUTATION

Les politiques de mutation modifient automatiquement les ressources pour les rendre conformes.

### Test 3.1 : Ajout automatique de labels

```bash
# Déployer nginx SANS les labels managed-by et team
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-mutation-labels
  namespace: cesiX  # ⚠️ REMPLACEZ par votre namespace !
  labels:
    app: nginx-mutation
    test: "mutation-labels"
    # On ne met PAS les labels managed-by, team, environment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-mutation
  template:
    metadata:
      labels:
        app: nginx-mutation
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "250m"
            memory: "256Mi"
        securityContext:
          runAsUser: 101
          readOnlyRootFilesystem: true
        volumeMounts:
        - name: cache
          mountPath: /var/cache/nginx
        - name: run
          mountPath: /var/run
      volumes:
      - name: cache
        emptyDir: {}
      - name: run
        emptyDir: {}
EOF
```

**Vérifier que Kyverno a ajouté les labels automatiquement** :
```bash
# Voir TOUS les labels du déploiement (REMPLACEZ cesiX)
kubectl get deployment nginx-mutation-labels -n cesiX -o jsonpath='{.metadata.labels}' | jq

# Résultat attendu :
# {
#   "app": "nginx-mutation",
#   "managed-by": "kyverno",      ← ✅ Ajouté par Kyverno !
#   "team": "formation",          ← ✅ Ajouté par Kyverno !
#   "environment": "training",    ← ✅ Ajouté par Kyverno !
#   "test": "mutation-labels"
# }
```

✅ **Magie !** Kyverno a automatiquement ajouté 3 labels : `managed-by`, `team`, et `environment`.

---

### Test 3.2 : Renforcement automatique du securityContext

```bash
# Déployer nginx SANS securityContext au niveau pod
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-mutation-securitycontext
  namespace: cesiX  # ⚠️ REMPLACEZ par votre namespace !
  labels:
    test: "mutation-securitycontext"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-mutation-sec
  template:
    metadata:
      labels:
        app: nginx-mutation-sec
    spec:
      # Pas de securityContext au niveau pod !
      containers:
      - name: nginx
        image: nginx:1.25
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "250m"
            memory: "256Mi"
        securityContext:
          runAsUser: 101
          readOnlyRootFilesystem: true
        volumeMounts:
        - name: cache
          mountPath: /var/cache/nginx
        - name: run
          mountPath: /var/run
      volumes:
      - name: cache
        emptyDir: {}
      - name: run
        emptyDir: {}
EOF
```

**Vérifier les ajouts automatiques** :
```bash
# Vérifier le securityContext au niveau POD (REMPLACEZ cesiX)
kubectl get deployment nginx-mutation-securitycontext -n cesiX \
  -o jsonpath='{.spec.template.spec.securityContext}' | jq

# Résultat attendu :
# {
#   "runAsNonRoot": true,           ← ✅ Ajouté par Kyverno !
#   "seccompProfile": {
#     "type": "RuntimeDefault"       ← ✅ Ajouté par Kyverno !
#   }
# }

# Vérifier le securityContext au niveau CONTENEUR
kubectl get deployment nginx-mutation-securitycontext -n cesiX \
  -o jsonpath='{.spec.template.spec.containers[0].securityContext}' | jq

# Résultat attendu :
# {
#   "allowPrivilegeEscalation": false,  ← ✅ Ajouté par Kyverno !
#   "capabilities": {
#     "drop": ["ALL"]                    ← ✅ Ajouté par Kyverno !
#   },
#   "readOnlyRootFilesystem": true,
#   "runAsUser": 101
# }
```

✅ **Incroyable !** Kyverno a automatiquement renforcé la sécurité en ajoutant :
- `runAsNonRoot: true`
- `seccompProfile.type: RuntimeDefault`
- `allowPrivilegeEscalation: false`
- `capabilities.drop: [ALL]`

---

## 🧪 SECTION 4 : Adapter votre application demo-java

Maintenant que vous comprenez les politiques, adaptons votre application `cesiX-demo-java` pour être conforme.

### Étape 4.1 : Modifier le déploiement demo-java

Modifiez `k8s/appli/appli.yaml` pour respecter les politiques Kyverno :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ema-demo-java
  namespace: cesiX  # ⚠️ Votre namespace !
  labels:
    app: ema-demo-java
    version: "1.0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ema-demo-java
  template:
    metadata:
      labels:
        app: ema-demo-java
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: ema-demo-java
        image: emanzat/ema-demo-java:latest  # Sera remplacé par GitHub Actions
        ports:
        - containerPort: 8080
          name: http
        resources:
          limits:
            cpu: "1000m"
            memory: "1Gi"
          requests:
            cpu: "500m"
            memory: "512Mi"
        securityContext:
          allowPrivilegeEscalation: false
          runAsNonRoot: true
          runAsUser: 1000
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: logs
          mountPath: /app/logs
      volumes:
      - name: tmp
        emptyDir: {}
      - name: logs
        emptyDir: {}
```

**Points importants** :
- ✅ `runAsNonRoot: true` et `runAsUser: 1000` : N'exécute pas en tant que root
- ✅ `readOnlyRootFilesystem: true` : Système de fichiers root en lecture seule
- ✅ `allowPrivilegeEscalation: false` : Empêche l'escalade de privilèges
- ✅ `capabilities: drop: ALL` : Supprime toutes les capacités Linux
- ✅ `resources.limits` et `requests` : Limites de ressources définies
- ✅ Volumes `emptyDir` pour `/tmp` et `/logs` (car rootFS en lecture seule)

### Étape 4.2 : Tester localement avec dry-run

Avant de commiter, testez que le déploiement est conforme :

```bash
# Dry-run pour vérifier (REMPLACEZ cesiX)
kubectl apply -f k8s/appli/appli.yaml --dry-run=server -n cesiX

# Si aucune erreur → Le déploiement est conforme ✅
```

### Étape 4.3 : Commit et déploiement via ArgoCD

```bash
# Commit les changements
git add k8s/appli/appli.yaml
git commit -m "feat: adapt demo-java deployment to Kyverno policies"
git push origin main

# ArgoCD va automatiquement synchroniser
# Ou forcer la sync :
argocd app sync cesiX-demo-java

# Vérifier que le déploiement est conforme
kubectl get deployment ema-demo-java -n cesiX
kubectl describe deployment ema-demo-java -n cesiX
```

---

## 🧪 SECTION 5 : Nettoyage

Avant de terminer, nettoyons les ressources de test :

```bash
# Supprimer tous les déploiements/pods de test (REMPLACEZ cesiX)
kubectl delete deployment,pod -n cesiX -l test

# Vérifier
kubectl get deployment,pod -n cesiX

# ✅ Seuls nginx-secure et ema-demo-java devraient rester
```

---

## 🤖 BONUS : Script de test automatisé

Un script de test automatisé est disponible pour valider rapidement les politiques Kyverno :

```bash
# Rendre le script exécutable (une seule fois)
chmod +x tp/scripts/test-tp13-kyverno.sh

# Exécuter le script (adapté pour votre namespace)
# Modifier la variable NAMESPACE="cesi6" dans le script d'abord
bash tp/scripts/test-tp13-kyverno.sh
```

**Ce que fait le script** :
- ✅ Vérifie l'installation de Kyverno
- ✅ Vérifie les 3 ClusterPolicies
- ✅ Teste les ressources auto-générées (ResourceQuota, NetworkPolicy, ConfigMap)
- ✅ Teste les validations (doivent échouer)
- ✅ Teste un déploiement conforme (doit réussir)
- ✅ Affiche un résumé coloré des résultats
- ✅ **Propose un nettoyage automatique** des ressources de test

**Avantages** :
- Gain de temps (automatise tous les tests)
- Validation complète en quelques secondes
- Résultats visuels clairs (✅ / ❌)
- Utilise `--dry-run=server` (ne crée rien sur le cluster)
- **Nettoyage interactif** à la fin (supprime les ressources de test)

---

## 📊 Récapitulatif des tests effectués

| # | Test | Type | Résultat attendu | Politique testée | ✅ |
|---|------|------|------------------|------------------|---|
| 1.1 | `nginx-privileged` | Validation | Bloqué ❌ | deny-privileged-containers | |
| 1.2 | `nginx-no-limits` | Validation | Bloqué ❌ | require-resources-limits | |
| 1.3 | `nginx-as-root` | Validation | Bloqué ❌ | deny-root-user | |
| 1.4 | `nginx-writable-root` | Validation | Bloqué ❌ | require-readonly-rootfs | |
| 1.5 | `nginx-too-many-replicas` | Validation | Bloqué ❌ | limit-replicas | |
| 2.1 | `nginx-secure` | Validation | Accepté ✅ | Toutes | |
| 3.1 | `nginx-mutation-labels` | Mutation | Labels ajoutés ✅ | add-required-labels | |
| 3.2 | `nginx-mutation-securitycontext` | Mutation | SecurityContext renforcé ✅ | add-safe-securitycontext | |
| 4 | `ema-demo-java` | Production | Conforme et déployé ✅ | Toutes | |

---

## ✅ Critères de Validation

- [ ] Kyverno est vérifié comme installé
- [ ] Les 3 ClusterPolicies sont vérifiées (security, mutation, generate)
- [ ] Les ressources générées automatiquement sont vérifiées (ResourceQuota, NetworkPolicy, ConfigMap)
- [ ] **Test 1.1-1.5** : 5 tests de validation bloqués comme attendu
- [ ] **Test 2.1** : nginx-secure accepté
- [ ] **Test 3.1** : Labels ajoutés automatiquement par Kyverno
- [ ] **Test 3.2** : SecurityContext renforcé automatiquement
- [ ] **Application demo-java** : Déploiement adapté et conforme
- [ ] Nettoyage des ressources de test effectué

---

## 🔧 Troubleshooting

### Problème 1 : Mon déploiement est bloqué mais je ne comprends pas pourquoi

**Symptôme** :
```
Error from server: admission webhook "validate.kyverno.svc" denied the request
```

**Solution** :
1. Lire attentivement le message d'erreur - il indique quelle règle bloque
2. Utiliser `--dry-run=server` pour tester sans créer :
   ```bash
   kubectl apply -f mon-fichier.yaml --dry-run=server
   ```
3. Vérifier les politiques actives :
   ```bash
   kubectl get clusterpolicy
   kubectl describe clusterpolicy formation-security-policies
   ```

### Problème 2 : Je ne vois pas les labels/securityContext ajoutés par Kyverno

**Symptôme** :
Les mutations ne semblent pas s'appliquer

**Solution** :
1. Vérifier que les politiques de mutation sont actives :
   ```bash
   kubectl get clusterpolicy formation-mutation-policies
   ```
2. Les mutations s'appliquent aux **nouvelles** ressources, pas aux existantes
3. Pour voir les mutations, interroger la ressource après création :
   ```bash
   kubectl get deployment mon-app -n cesi6 -o yaml
   ```

### Problème 3 : La ResourceQuota n'a pas été générée dans mon namespace

**Symptôme** :
```bash
kubectl get resourcequota -n cesi6
# No resources found
```

**Solution** :
1. Vérifier que le namespace matche le pattern `cesi*` :
   ```bash
   kubectl get namespace cesi6 -o yaml | grep name:
   ```
2. Attendre 10-15 secondes (génération asynchrone)
3. Vérifier les logs Kyverno :
   ```bash
   kubectl logs -n kyverno -l app.kubernetes.io/component=background-controller --tail=50
   ```
4. Si toujours rien, vérifier la politique de génération :
   ```bash
   kubectl describe clusterpolicy formation-generate-policies
   ```

### Problème 4 : Erreur "image pull" avec nginx

**Symptôme** :
```
Failed to pull image "nginx:1.25": rpc error
```

**Solution** :
1. Vérifier la connectivité internet du cluster
2. Utiliser une image alternative :
   ```yaml
   image: nginx:1.24  # ou nginx:latest
   ```

### Problème 5 : Mon application demo-java ne démarre pas après adaptation

**Symptôme** :
```
CrashLoopBackOff
```

**Solution** :
1. Vérifier les logs du pod :
   ```bash
   kubectl logs -n cesi6 <pod-name>
   ```
2. Problème courant : Pas de volumes pour `/tmp` ou `/logs`
   ```yaml
   # Ajouter ces volumes si manquants :
   volumeMounts:
   - name: tmp
     mountPath: /tmp
   - name: logs
     mountPath: /app/logs
   volumes:
   - name: tmp
     emptyDir: {}
   - name: logs
     emptyDir: {}
   ```
3. Vérifier que le port 8080 est bien exposé
4. Vérifier que l'utilisateur 1000 existe dans l'image

### Problème 6 : Commande kubectl trop longue, erreur de frappe

**Solution** :
Utiliser des variables shell :
```bash
# Définir votre namespace une fois
export MY_NS=cesi6

# Puis utiliser $MY_NS partout
kubectl get pods -n $MY_NS
kubectl apply -f mon-fichier.yaml -n $MY_NS
```

### Problème 7 : Je veux voir exactement ce que Kyverno a modifié

**Solution** :
Comparer avant/après avec dry-run :
```bash
# Voir ce que vous avez écrit
cat mon-fichier.yaml

# Voir ce que Kyverno va créer (avec mutations)
kubectl apply -f mon-fichier.yaml --dry-run=server -o yaml
```

### Problème 8 : Erreur "jq: command not found"

**Symptôme** :
La commande pour voir les labels en JSON ne fonctionne pas

**Solution** :
Alternative sans jq :
```bash
# Au lieu de :
kubectl get deployment mon-app -n cesi6 -o jsonpath='{.metadata.labels}' | jq

# Utiliser :
kubectl get deployment mon-app -n cesi6 -o jsonpath='{.metadata.labels}' | python3 -m json.tool

# Ou simplement :
kubectl get deployment mon-app -n cesi6 -o yaml | grep -A 5 "labels:"
```

### Commandes utiles pour déboguer

```bash
# Voir tous les événements dans votre namespace
kubectl get events -n cesi6 --sort-by='.lastTimestamp'

# Voir les PolicyReports (violations)
kubectl get policyreport -n cesi6

# Voir le détail d'une violation
kubectl describe policyreport -n cesi6

# Voir les logs Kyverno en temps réel
kubectl logs -n kyverno -l app.kubernetes.io/name=kyverno -f

# Vérifier la configuration complète d'un déploiement
kubectl get deployment mon-app -n cesi6 -o yaml

# Tester une ressource sans la créer
kubectl apply -f mon-fichier.yaml --dry-run=server -v=8
```

---

## 🤔 Questions de Compréhension

### 1. Quelle est la différence entre `validationFailureAction: Enforce` et `Audit` ?

<details>
<summary>Voir la réponse</summary>

**Enforce** :
- Bloque la création/modification de ressources non conformes
- L'admission webhook rejette la requête
- Mode **production** pour appliquer strictement les règles
- Exemple : `kubectl apply` → Erreur si non conforme

**Audit** :
- Autorise la création mais génère un rapport de violation
- Aucun blocage, juste un avertissement
- Mode **apprentissage** pour tester les politiques
- Permet de voir l'impact sans casser les déploiements existants

**Best Practice** :
1. Commencer en mode `Audit`
2. Analyser les violations
3. Corriger les ressources non conformes
4. Passer en mode `Enforce`

Dans cet exercice, les politiques sont en mode **Enforce** pour que vous voyiez directement les blocages.
</details>

---

### 2. Pourquoi `readOnlyRootFilesystem: true` améliore-t-il la sécurité ?

<details>
<summary>Voir la réponse</summary>

Un système de fichiers root en lecture seule empêche :
- **Modification de binaires** : Un attaquant ne peut pas remplacer `/bin/sh`
- **Écriture de malwares** : Impossible d'écrire des scripts malveillants
- **Persistance** : Les modifications ne survivent pas au redémarrage du pod
- **Détection d'intrusion** : Toute tentative d'écriture échoue et peut être détectée

**Conséquence** : Il faut monter des volumes `emptyDir` ou `PersistentVolume` pour les dossiers qui ont besoin d'écriture (`/tmp`, `/var/log`, etc.)

**Exemple** :
```yaml
securityContext:
  readOnlyRootFilesystem: true
volumeMounts:
- name: tmp
  mountPath: /tmp
- name: logs
  mountPath: /app/logs
volumes:
- name: tmp
  emptyDir: {}
- name: logs
  emptyDir: {}
```

C'est une couche de défense en profondeur (Defense in Depth).
</details>

---

### 3. Que signifie `drop: ALL` dans capabilities ?

<details>
<summary>Voir la réponse</summary>

Les **Linux capabilities** sont des permissions fines (plus granulaires que root/non-root).

**Exemples de capabilities** :
- `CAP_NET_BIND_SERVICE` : Bind sur ports < 1024
- `CAP_SYS_ADMIN` : Opérations admin système
- `CAP_NET_RAW` : Créer des sockets raw (ping, etc.)

`drop: ALL` supprime **toutes** les capabilities, rendant le conteneur minimal.

**Impact** :
- ✅ Réduit la surface d'attaque
- ✅ Principe du moindre privilège
- ❌ Peut casser certaines apps qui nécessitent des capabilities spécifiques

**Si besoin, rajouter des capabilities spécifiques** :
```yaml
capabilities:
  drop:
  - ALL
  add:
  - NET_BIND_SERVICE  # Pour bind sur port 80
```

C'est l'approche "deny-all, allow-specific" (liste blanche).
</details>

---

### 4. Pourquoi Kyverno génère-t-il automatiquement une ResourceQuota ?

<details>
<summary>Voir la réponse</summary>

**Objectifs** :
1. **Multi-tenancy** : Isoler les ressources entre les namespaces étudiants
2. **Fairness** : Garantir que chaque étudiant a un quota équitable
3. **Prévention** : Empêcher qu'un étudiant consomme toutes les ressources du cluster
4. **Gouvernance** : Appliquer automatiquement les limites sans intervention manuelle

**Dans notre cas** :
- 17 étudiants (cesi1 à cesi17) partagent le même cluster
- Sans ResourceQuota, un étudiant pourrait déployer 100 replicas et saturer le cluster
- Avec ResourceQuota automatique, chaque étudiant a des limites claires :
  - CPU requests: 4 cores max
  - CPU limits: 8 cores max
  - Memory requests: 8Gi max
  - Memory limits: 16Gi max
  - Pods: 20 max

**Génération automatique avec Kyverno** :
- Dès qu'un namespace `cesiX` est créé → ResourceQuota créée automatiquement
- Pas d'oubli possible
- Cohérence garantie
- Self-healing : Si supprimée manuellement, Kyverno la recrée

C'est un exemple de **Policy as Code** appliquée à la gouvernance.
</details>

---

### 5. Comment les mutations de Kyverno interagissent-elles avec les validations ?

<details>
<summary>Voir la réponse</summary>

**Ordre d'exécution dans le webhook d'admission** :

1. **Mutation** : Kyverno modifie d'abord la ressource
2. **Validation** : Kyverno valide ensuite la ressource (après mutation)

**Exemple pratique** :

Vous déployez :
```yaml
spec:
  containers:
  - name: nginx
    # Pas de securityContext !
```

Kyverno :
1. **Mutation** : Ajoute automatiquement :
   ```yaml
   securityContext:
     allowPrivilegeEscalation: false
     capabilities:
       drop: [ALL]
   ```

2. **Validation** : Vérifie que la ressource (après mutation) est conforme
   - ✅ OK car le securityContext a été ajouté

**Avantage** :
- Les développeurs peuvent écrire des manifests simples
- Kyverno complète automatiquement (mutation)
- Et garantit la conformité (validation)

**Attention** :
- Si une politique de mutation échoue → La validation ne s'exécute pas
- L'ordre des politiques peut avoir un impact

C'est le principe du **Guardrails** : on aide les développeurs à faire le bon choix par défaut.
</details>

---

### 6. Que se passe-t-il si vous supprimez manuellement la ResourceQuota ?

<details>
<summary>Voir la réponse</summary>

**Avec `synchronize: true` dans la politique de génération** :

1. Vous supprimez : `kubectl delete resourcequota namespace-quota -n cesiX`
2. Kyverno détecte la suppression (grâce au controller)
3. Kyverno **recrée automatiquement** la ResourceQuota (self-healing)
4. Délai : ~5-10 secondes

**Test** :
```bash
# Supprimer la quota
kubectl delete resourcequota namespace-quota -n cesiX

# Attendre
sleep 10

# Vérifier qu'elle est revenue
kubectl get resourcequota -n cesiX

# ✅ La ResourceQuota est revenue !
```

**Pourquoi c'est important** :
- Empêche les modifications manuelles non documentées
- Force le passage par Git (GitOps)
- Garantit la conformité continue
- Protège contre les erreurs humaines

**Pour modifier la quota** :
1. L'admin modifie le fichier `k8s/kyverno/admin/cluster-policies.yaml` dans Git
2. ArgoCD synchronise automatiquement
3. Kyverno met à jour toutes les ResourceQuotas existantes

**Note** : En tant qu'étudiant, vous ne modifiez PAS les politiques Kyverno.

C'est l'application du principe **GitOps : Git is the source of truth**.
</details>

---

## 🎯 Architecture Kyverno dans le cluster

```
┌─────────────────────────────────────────────────────────────┐
│                        ADMIN                                 │
│                                                               │
│  1. Déploie les politiques via ArgoCD                        │
│     kubectl apply -f argocd-crds/argocd-kyverno-policies.yaml│
│                                                               │
│  2. ArgoCD sync k8s/kyverno/cluster-policies.yaml            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   Kyverno (namespace: kyverno)               │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  ClusterPolicy: formation-security-policies          │   │
│  │    - deny-privileged-containers                       │   │
│  │    - require-resources-limits                         │   │
│  │    - deny-root-user                                   │   │
│  │    - require-readonly-rootfs                          │   │
│  │    - limit-replicas                                   │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  ClusterPolicy: formation-mutation-policies          │   │
│  │    - add-required-labels                              │   │
│  │    - add-safe-securitycontext                         │   │
│  │    - add-documentation-annotations                    │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  ClusterPolicy: formation-generate-policies          │   │
│  │    - generate-resourcequota                           │   │
│  │    - generate-networkpolicy                           │   │
│  │    - generate-configmap                               │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                     ÉTUDIANTS (cesi1 ... cesi17)             │
│                                                               │
│  kubectl apply -f deployment.yaml                            │
│         │                                                     │
│         ▼                                                     │
│  Kyverno Webhook intercepts                                  │
│         │                                                     │
│         ├─► MUTATE (ajoute labels, securityContext)          │
│         │                                                     │
│         ├─► VALIDATE (vérifie conformité)                    │
│         │     │                                               │
│         │     ├─► ✅ Conforme → ACCEPT                       │
│         │     └─► ❌ Non conforme → REJECT                   │
│         │                                                     │
│         └─► GENERATE (ResourceQuota, NetworkPolicy, ...)     │
└─────────────────────────────────────────────────────────────┘
```

---

## 💡 Points Importants

### Différence entre ClusterPolicy et Policy

| Aspect | ClusterPolicy | Policy (namespace) |
|--------|--------------|---------|
| Scope | Cluster-wide (tous les namespaces) | Un seul namespace |
| Cas d'usage | Standards globaux (admin) | Règles spécifiques équipe |
| Appliqué par | Admin | Équipe/Étudiant |

Dans cet exercice, toutes les politiques sont des **ClusterPolicies** déployées par l'admin.

### Workflow GitOps complet

1. **Admin** : Modifie `k8s/kyverno/admin/cluster-policies.yaml` → Git push
2. **ArgoCD** : Détecte le changement → Sync automatique (pointe sur `k8s/kyverno/admin/`)
3. **Kyverno** : Applique les nouvelles politiques cluster-wide
4. **Étudiants** : Testent avec nginx → Voient les nouvelles règles
5. **Étudiants** : Adaptent leur app demo-java → Conformité

**Séparation des responsabilités** :
- Dossier `k8s/kyverno/admin/` → Déployé par ArgoCD (admin uniquement)
- Dossier `k8s/kyverno/student-examples/` → Exemples pour apprendre (pas déployés)

### Ordre des opérations lors d'un `kubectl apply`

```
kubectl apply
    ↓
API Server
    ↓
Kyverno Admission Webhook
    ↓
1. MUTATE (modifications)
    ↓
2. VALIDATE (vérifications)
    ↓
3. GENERATE (ressources associées)
    ↓
✅ ACCEPT ou ❌ REJECT
    ↓
Ressource créée dans etcd
```

---

## 📚 Ressources

### Documentation officielle
- [Kyverno Documentation](https://kyverno.io/docs/)
- [Kyverno Policies Library](https://kyverno.io/policies/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Linux Capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html)

### Fichiers du projet
- **Politiques déployées** : `k8s/kyverno/admin/cluster-policies.yaml` (voir le fichier pour comprendre la syntaxe)
- **Configuration ArgoCD** : `k8s/argocd-crds/argocd-kyverno-policies.yaml` (pour l'admin)
- **Script de test automatisé** : `tp/scripts/test-tp13-kyverno.sh` (pour tester rapidement)

### 💡 Pour aller plus loin
Si vous voulez comprendre comment les politiques sont écrites, ouvrez le fichier `k8s/kyverno/admin/cluster-policies.yaml` :

```bash
# Voir les politiques déployées
cat k8s/kyverno/admin/cluster-policies.yaml | less

# Rechercher une règle spécifique
grep -A 20 "deny-privileged-containers" k8s/kyverno/admin/cluster-policies.yaml
```

---

## 🎉 Félicitations !

Vous avez testé avec succès les politiques de sécurité Kyverno ! Vous comprenez maintenant :

- ✅ Comment Kyverno valide, mute et génère des ressources
- ✅ Pourquoi les bonnes pratiques de sécurité sont importantes
- ✅ Comment adapter votre application pour être conforme
- ✅ Le principe de Policy as Code et GitOps

Votre cluster Kubernetes est maintenant :
- 🛡️ **Sécurisé** contre les configurations dangereuses
- 🤖 **Automatisé** avec des mutations et générations
- 📋 **Gouverné** avec des quotas et des limites
- ✅ **Conforme** aux bonnes pratiques

[🏠 Retour au sommaire](README.md)

---

