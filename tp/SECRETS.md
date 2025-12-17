# Guide de Configuration des Secrets GitHub

Ce document liste tous les Secrets GitHub requis pour que le pipeline CI/CD fonctionne correctement.

---

## 🔀 Étape 0 : Fork du Projet

**Avant toute configuration, vous devez fork le projet :**

1. Allez sur https://github.com/emanzat/demo-boost-startup-java
2. Cliquez sur le bouton **"Fork"** en haut à droite
3. Créez le fork dans votre compte GitHub personnel ou organisation
4. Clonez votre fork sur votre machine locale :
   ```bash
   git clone https://github.com/VOTRE_USERNAME/demo-boost-startup-java.git
   cd demo-boost-startup-java
   ```

**⚠️ Important** : Toutes les configurations suivantes doivent être effectuées dans **votre fork**, pas dans le dépôt original.

---

## 🔐 Secrets Requis

Configurez ces secrets dans **votre dépôt forké** :
`Settings` → `Secrets and variables` → `Actions` → `New repository secret`

### Authentification Docker Hub

| Nom du Secret | Description | Exemple |
|---------------|-------------|---------|
| `DOCKERHUB_USERNAME` | Votre nom d'utilisateur Docker Hub | `monusername` |
| `DOCKERHUB_TOKEN` | Token d'accès Docker Hub (PAS le mot de passe) | Générer sur https://hub.docker.com/settings/security |

**Comment créer un token Docker Hub :**
1. Allez sur https://hub.docker.com/settings/security
2. Cliquez sur "New Access Token"
3. Nommez-le "GitHub Actions CI/CD"
4. Copiez le token et enregistrez-le comme secret `DOCKERHUB_TOKEN`

---

### Configuration du Déploiement SSH

| Nom du Secret | Description | Exemple | Valeur |
|---------------|-------------|--------|--------|
| `DEPLOY_SERVER` | Adresse IP ou nom d'hôte du serveur de déploiement | `193.70.40.85` ou `193.70.42.147` (selon votre groupe) | - |
| `DEPLOY_SSH_USER` | Nom d'utilisateur SSH pour le serveur de déploiement | `ubuntu` | - |
| `DEPLOY_SSH_PRIVATE_KEY` | Clé privée SSH pour l'authentification | Contenu complet de la clé privée (voir ci-dessous) | - |
| `DEPLOY_SSH_PORT` | Port SSH (optionnel, par défaut 22) | `22` | - |
| `DEPLOY_APPLI_PORT` | Port de l'application personnalisé selon votre numéro d'étudiant | `8081` à `8097` | - |
| `DEPLOY_APPLI_NAME` | Nom de l'application personnalisé selon votre numéro d'étudiant | `cesi1-demo-boost-startup` à `cesi17-demo-boost-startup` | - |
| `MONGODB_COLLECTION_NAME` | Nom de la collection MongoDB personnalisé selon votre numéro d'étudiant | `personscesi1` à `personscesi17` | - |

---

### ⚠️ Configuration Personnalisée : Secrets par Étudiant

**Chaque étudiant doit personnaliser 4 secrets GitHub pour éviter les conflits :**

Au début de la session, vous avez reçu un **numéro d'étudiant** de 1 à 17 (le même que pour ArgoCD).

**Serveurs de déploiement par groupe** :
- **Groupe 1** (étudiants 1 à 8) : `DEPLOY_SERVER` = **`193.70.40.85`**
- **Groupe 2** (étudiants 9 à 17) : `DEPLOY_SERVER` = **`193.70.42.147`**

**Formats à respecter** :
- **Serveur de déploiement** : `193.70.40.85` (groupe 1) ou `193.70.42.147` (groupe 2)
- **Port de déploiement** : `808X` où X = votre numéro (étudiant 1 = 8081, étudiant 2 = 8082, etc.)
- **Nom de l'application** : `cesiX-demo-boost-startup` où X = votre numéro
- **Collection MongoDB** : `personscesiX` où X = votre numéro

#### Exemples par étudiant

| N° Étudiant | DEPLOY_SERVER | DEPLOY_APPLI_PORT | DEPLOY_APPLI_NAME | MONGODB_COLLECTION_NAME |
|-------------|---------------|-------------------|-------------------|-------------------------|
| 1  | `193.70.40.85` | `8081` | `cesi1-demo-boost-startup` | `personscesi1` |
| 2  | `193.70.40.85` | `8082` | `cesi2-demo-boost-startup` | `personscesi2` |
| 3  | `193.70.40.85` | `8083` | `cesi3-demo-boost-startup` | `personscesi3` |
| 4  | `193.70.40.85` | `8084` | `cesi4-demo-boost-startup` | `personscesi4` |
| 5  | `193.70.40.85` | `8085` | `cesi5-demo-boost-startup` | `personscesi5` |
| 6  | `193.70.40.85` | `8086` | `cesi6-demo-boost-startup` | `personscesi6` |
| 7  | `193.70.40.85` | `8087` | `cesi7-demo-boost-startup` | `personscesi7` |
| 8  | `193.70.40.85` | `8088` | `cesi8-demo-boost-startup` | `personscesi8` |
| 9  | `193.70.42.147` | `8089` | `cesi9-demo-boost-startup` | `personscesi9` |
| 10 | `193.70.42.147` | `8090` | `cesi10-demo-boost-startup` | `personscesi10` |
| 11 | `193.70.42.147` | `8091` | `cesi11-demo-boost-startup` | `personscesi11` |
| 12 | `193.70.42.147` | `8092` | `cesi12-demo-boost-startup` | `personscesi12` |
| 13 | `193.70.42.147` | `8093` | `cesi13-demo-boost-startup` | `personscesi13` |
| 14 | `193.70.42.147` | `8094` | `cesi14-demo-boost-startup` | `personscesi14` |
| 15 | `193.70.42.147` | `8095` | `cesi15-demo-boost-startup` | `personscesi15` |
| 16 | `193.70.42.147` | `8096` | `cesi16-demo-boost-startup` | `personscesi16` |
| 17 | `193.70.42.147` | `8097` | `cesi17-demo-boost-startup` | `personscesi17` |

**Exemple** : Si vous êtes l'étudiant n°3 (groupe 1), configurez les secrets comme ceci :
- **Secret** : `DEPLOY_SERVER` → **Valeur** : `193.70.40.85`
- **Secret** : `DEPLOY_APPLI_PORT` → **Valeur** : `8083`
- **Secret** : `DEPLOY_APPLI_NAME` → **Valeur** : `cesi3-demo-boost-startup`
- **Secret** : `MONGODB_COLLECTION_NAME` → **Valeur** : `personscesi3`

**Exemple** : Si vous êtes l'étudiant n°12 (groupe 2), configurez les secrets comme ceci :
- **Secret** : `DEPLOY_SERVER` → **Valeur** : `193.70.42.147`
- **Secret** : `DEPLOY_APPLI_PORT` → **Valeur** : `8092`
- **Secret** : `DEPLOY_APPLI_NAME` → **Valeur** : `cesi12-demo-boost-startup`
- **Secret** : `MONGODB_COLLECTION_NAME` → **Valeur** : `personscesi12`

**💡 Notes importantes** :
- **Serveur assigné par groupe** : Groupe 1 (étudiants 1-8) → 193.70.40.85 | Groupe 2 (étudiants 9-17) → 193.70.42.147
- **Port unique** : Chaque étudiant utilise un port différent (8081 à 8097) pour éviter les conflits sur le serveur de déploiement
- **Nom d'application unique** : Le nom du conteneur Docker doit être unique pour éviter les conflits
- **Collection MongoDB isolée** : Chaque étudiant a sa propre collection dans la base de données MongoDB partagée pour isoler les données

---

**Comment générer une paire de clés SSH :**

```bash
# Sur votre machine locale, générez une nouvelle paire de clés SSH
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/deploy_key

# Copiez la clé PUBLIQUE sur votre serveur de déploiement
# Groupe 1 (étudiants 1-8) :
ssh-copy-id -i ~/.ssh/deploy_key.pub ubuntu@193.70.40.85

# Groupe 2 (étudiants 9-17) :
ssh-copy-id -i ~/.ssh/deploy_key.pub ubuntu@193.70.42.147

# Ou ajoutez-la manuellement aux authorized_keys du serveur :
# Groupe 1: cat ~/.ssh/deploy_key.pub | ssh ubuntu@193.70.40.85 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
# Groupe 2: cat ~/.ssh/deploy_key.pub | ssh ubuntu@193.70.42.147 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# Affichez la clé PRIVÉE pour la copier dans GitHub Secrets
cat ~/.ssh/deploy_key
```

**Format pour `DEPLOY_SSH_PRIVATE_KEY` :**
```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
...
(contenu complet de la clé privée)
...
-----END OPENSSH PRIVATE KEY-----
```

---

## ✅ Tester la Connexion SSH

**Avant de configurer le secret dans GitHub, testez que la clé SSH fonctionne correctement.**

### Étape 1 : Tester avec la clé privée générée

```bash
# Tester la connexion SSH avec la clé privée
# Groupe 1 (étudiants 1-8) :
ssh -i ~/.ssh/deploy_key ubuntu@193.70.40.85

# Groupe 2 (étudiants 9-17) :
ssh -i ~/.ssh/deploy_key ubuntu@193.70.42.147

# Ou avec un port personnalisé si nécessaire
ssh -i ~/.ssh/deploy_key -p 2222 ubuntu@VOTRE_SERVEUR
```

**Résultat attendu :**
```
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-91-generic x86_64)
...
ubuntu@server:~$
```


---

## Bonnes Pratiques de Sécurité

1. **Ne jamais commiter de secrets dans git**
   - Utilisez `.gitignore` pour exclure les fichiers sensibles
   - Utilisez GitHub Secrets pour toutes les informations d'identification

2. **Utilisez le principe du moindre privilège**
   - Créez un utilisateur dédié au déploiement avec des permissions minimales
   - Utilisez des clés SSH plutôt que des mots de passe
   - Effectuez une rotation régulière des secrets

3. **Surveillez les accès**
   - Consultez régulièrement les logs GitHub Actions
   - Configurez des alertes pour les déploiements échoués
   - Surveillez les logs d'accès du serveur

4. **Sécurité réseau**
   - Utilisez des règles de pare-feu pour restreindre les accès
   - Envisagez d'utiliser un VPN ou un tunnel SSH
   - Maintenez le serveur et Docker à jour

---

## 📋 Liste de Vérification

Avant d'exécuter le pipeline, vérifiez :

### Secrets GitHub
- [ ] `DOCKERHUB_USERNAME` est défini
- [ ] `DOCKERHUB_TOKEN` est défini et valide
- [ ] `DEPLOY_SERVER` est défini selon votre groupe : `193.70.40.85` (groupe 1) ou `193.70.42.147` (groupe 2)
- [ ] `DEPLOY_SSH_USER` est défini
- [ ] `DEPLOY_SSH_PRIVATE_KEY` est défini avec la clé privée complète
- [ ] `DEPLOY_APPLI_PORT` est personnalisé : `808X` selon votre numéro (8081 pour étudiant 1, 8097 pour étudiant 17)
- [ ] `DEPLOY_APPLI_NAME` est personnalisé : `cesiX-demo-boost-startup` selon votre numéro
- [ ] `MONGODB_COLLECTION_NAME` est personnalisé : `personscesiX` selon votre numéro

### Configuration Personnalisée
- [ ] **Numéro d'étudiant identifié** : Vous connaissez votre numéro (1 à 17)
- [ ] **Serveur assigné** : Groupe 1 (1-8) → 193.70.40.85 | Groupe 2 (9-17) → 193.70.42.147
- [ ] **Port unique** : Configuré avec `808X` correspondant à votre numéro
- [ ] **Nom d'application unique** : Configuré avec `cesiX-demo-boost-startup`
- [ ] **Collection MongoDB isolée** : Configurée avec `personscesiX`
- [ ] **Namespace Kubernetes** : Configuré avec `cesiX` dans le TP ArgoCD (Exercice 12)
- [ ] **Cohérence** : Le même numéro X est utilisé partout (serveur, port, nom appli, collection, namespace, domaine)

### Tests de Connexion SSH
- [ ] **Test local groupe 1** : `ssh -i ~/.ssh/deploy_key ubuntu@193.70.40.85` fonctionne (étudiants 1-8)
- [ ] **Test local groupe 2** : `ssh -i ~/.ssh/deploy_key ubuntu@193.70.42.147` fonctionne (étudiants 9-17)
- [ ] **Permissions clés** : `chmod 600 ~/.ssh/deploy_key` appliqué
- [ ] **Clé publique** : Présente dans `~/.ssh/authorized_keys` sur le serveur

### Vérification Serveur
- [ ] Docker est installé sur le serveur : `ssh ubuntu@VOTRE_SERVEUR "docker --version"`
- [ ] L'utilisateur peut lancer Docker sans sudo : `ssh ubuntu@VOTRE_SERVEUR "docker ps"`
- [ ] Votre port est disponible : `ssh ubuntu@VOTRE_SERVEUR "sudo lsof -i :808X"` (remplacez X par votre numéro)

---