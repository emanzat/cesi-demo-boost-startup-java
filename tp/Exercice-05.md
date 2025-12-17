# Exercice 5 : Ajouter l'Analyse des Dépendances (SCA)

[⬅️ Exercice précédent](Exercice-04.md) | [🏠 Sommaire](README.md) | [Exercice suivant ➡️](Exercice-06.md)

---

## 🎯 Objectif

Identifier les vulnérabilités dans les dépendances Maven (bibliothèques tierces) avec OWASP Dependency-Check et accélérer le téléchargement de la base NVD avec une API Key gratuite.

## ⏱️ Durée Estimée

45 minutes

---

## 📝 Instructions

### Étape 5.1 : Obtenir une NVD API Key (gratuite)

OWASP Dependency-Check télécharge la base de données NVD (National Vulnerability Database) qui contient toutes les CVE connues. Une API Key permet d'accélérer ce téléchargement de **2-3x plus rapide**.

#### a) Demander la clé

1. Allez sur https://nvd.nist.gov/developers/request-an-api-key
2. Remplissez le formulaire avec votre email
3. Soumettez la demande

#### b) Confirmer votre email

1. Vous recevrez un email de confirmation de NVD
2. L'email contient un **UUID** (identifiant unique)
3. Cliquez sur le lien dans l'email OU allez sur https://nvd.nist.gov/developers/confirm-api-key
4. Entrez l'**UUID** reçu par email
5. Cliquez sur "Confirm"

#### c) Récupérer votre API Key

1. Après confirmation, vous recevrez un second email contenant votre **API Key**
2. Cette clé ressemble à : `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` (format UUID)
3. Conservez cette clé en sécurité

#### d) Ajouter le secret GitHub

1. Allez dans votre repo → `Settings` → `Secrets and variables` → `Actions`
2. Cliquez sur `New repository secret`
3. **Nom** : `NVD_API_KEY`
4. **Valeur** : Votre clé API NVD (format UUID complet)
5. Cliquez sur `Add secret`

---

### Étape 5.2 : Créer le workflow SCA (sans suppressions)

Créez `.github/workflows/sca-dependency-scan.yml` :

```yaml
on:
  workflow_call:
    secrets:
      NVD_API_KEY:
        required: false
        description: 'NVD API Key for faster dependency database download (optional)'
permissions:
  security-events: write
  contents: read

jobs:
  sca-dependency-scan:
    name: SCA - OWASP Dependency Check
    runs-on: ubuntu-latest

    steps:
      - name: 📥 Checkout code
        uses: actions/checkout@v4

      - name: ☕ Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: '25'
          distribution: 'liberica'
          cache: 'maven'

      - name: 🔍 Check NVD API Key availability
        env:
          NVD_API_KEY: ${{ secrets.NVD_API_KEY }}
        run: |
          if [ -n "${NVD_API_KEY}" ]; then
            echo "✅ NVD API Key is configured (length: ${#NVD_API_KEY} chars)"
            echo "🔑 First 8 chars: ${NVD_API_KEY:0:8}..."
          else
            echo "⚠️ NVD API Key not configured - download will be slower"
            echo "💡 Configure it in: Settings → Secrets → Actions → NVD_API_KEY"
            echo "💡 Get your key at: https://nvd.nist.gov/developers/request-an-api-key"
          fi

      - name: 📦 Run OWASP Dependency Check
        env:
          NVD_API_KEY: ${{ secrets.NVD_API_KEY }}
        run: |
          if [ -n "${NVD_API_KEY}" ]; then
            echo "🚀 Running with NVD API Key for faster download"
            mvn org.owasp:dependency-check-maven:check \
              -DfailBuildOnCVSS=7 \
              -DnvdApiKey=${NVD_API_KEY}
          else
            echo "🐌 Running without NVD API Key (slower download)"
            mvn org.owasp:dependency-check-maven:check \
              -DfailBuildOnCVSS=7
          fi

      - name: 📤 Upload Dependency Check SARIF
        uses: github/codeql-action/upload-sarif@v4
        if: always() && hashFiles('target/dependency-check-report.sarif') != ''
        with:
          sarif_file: target/dependency-check-report.sarif
          category: owasp-dependency-check

      - name: 🔍 Run Trivy SCA (filesystem scan)
        uses: aquasecurity/trivy-action@0.27.0
        with:
          scan-type: 'fs'
          format: 'json'
          output: 'trivy-deps-report.json'
          severity: 'CRITICAL,HIGH,MEDIUM'
          ignore-unfixed: true

      - name: 📤 Upload Trivy SCA report
        uses: actions/upload-artifact@v4
        with:
          name: trivy-deps-report
          path: trivy-deps-report.json
          retention-days: 7
```

**💡 Note** : Le workflow vérifie automatiquement si la clé NVD_API_KEY est configurée et l'utilise pour accélérer le téléchargement.

**⚠️ Important** : Pour l'instant, nous n'utilisons **PAS** le fichier de suppressions (`-DsuppressionFiles`). Vous allez voir pourquoi dans les prochaines étapes.

---

### Étape 5.3 : Ajouter au pipeline principal

Modifiez `main-pipeline.yml` :

```yaml
  secret-scanning:
    needs: build-and-test
    uses: ./.github/workflows/secret-scanning.yml

  # ═══════════════════════════════════════════════
  # ÉTAPE 4 : ANALYSE DES DÉPENDANCES (SCA)
  # ═══════════════════════════════════════════════
  sca-dependency-scan:
    needs: build-and-test  # Également en parallèle
    uses: ./.github/workflows/sca-dependency-scan.yml
    secrets: inherit  # ⚠️ IMPORTANT pour passer NVD_API_KEY
```

---

### Étape 5.4 : Premier test (sans suppressions)

```bash
git add .
git commit -m "feat: add SCA dependency scanning"
git push origin main
```

**🔍 Observer les résultats :**

1. Allez dans `Actions` → Cliquez sur votre workflow
2. Attendez la fin du job `sca-dependency-scan`
3. Consultez les logs du step "📦 Run OWASP Dependency Check"
4. Allez dans `Security` → `Code scanning` pour voir les alertes OWASP

**📊 Que voyez-vous ?**

Le scan OWASP Dependency-Check va probablement détecter des vulnérabilités dans vos dépendances Maven. Vous verrez :

- Des **CVE** (Common Vulnerabilities and Exposures) détectées
- Leur score **CVSS** (0-10)
- Les dépendances affectées
- Le job peut **échouer** si des vulnérabilités CVSS >= 7 sont trouvées
- Des alertes dans GitHub Security

**🎯 Exemple de sortie :**

```
[WARNING]
One or more dependencies were identified with known vulnerabilities in demo-boost-startup-java:
  - CVE-2024-XXXXX (CVSS: 7.5) - spring-boot-starter-web:3.x.x
  - CVE-2023-YYYYY (CVSS: 8.1) - jackson-databind:2.x.x
```

**💡 Problème constaté :**

Parmi ces vulnérabilités, certaines peuvent être :
- **Légitimes** : Vraies vulnérabilités à corriger en mettant à jour les dépendances
- **Faux positifs** : Vulnérabilités qui ne s'appliquent pas à votre usage
- **Non corrigées** : Pas encore de version fixée disponible

**Comment gérer les faux positifs et les cas non applicables ?** → C'est l'objectif de l'étape suivante !

---

### Étape 5.5 : Créer le fichier de suppressions

Maintenant que vous avez vu les résultats bruts du scan, nous allons créer un fichier de suppressions pour gérer les **faux positifs**.

Créez `.owasp-suppressions.xml` à la racine du projet :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<suppressions xmlns="https://jeremylong.github.io/DependencyCheck/dependency-suppression.1.3.xsd">
    <!-- Exemple : Supprimer un faux positif -->
    <!--
    <suppress>
        <notes>False positive for Spring Boot Actuator</notes>
        <packageUrl regex="true">^pkg:maven/org\.springframework\.boot/spring\-boot\-actuator.*$</packageUrl>
        <cve>CVE-2023-XXXXX</cve>
    </suppress>
    -->

    <!-- Exemple : Supprimer une CVE non applicable -->
    <!--
    <suppress>
        <notes>Cette CVE concerne une fonctionnalité que nous n'utilisons pas</notes>
        <packageUrl regex="true">^pkg:maven/com\.fasterxml\.jackson\.core/jackson-databind.*$</packageUrl>
        <cve>CVE-2024-YYYYY</cve>
    </suppress>
    -->
</suppressions>
```

**💡 Comment identifier une suppression nécessaire ?**

1. Consultez le rapport dans `Security` → `Code scanning`
2. Pour chaque CVE, évaluez :
   - **Est-ce un vrai risque ?** → Mettez à jour la dépendance dans `pom.xml`
   - **Est-ce un faux positif ?** → Ajoutez une suppression
   - **La vulnérabilité ne s'applique pas à votre code ?** → Ajoutez une suppression avec justification

**⚠️ Règle d'or** : Toujours documenter **pourquoi** vous supprimez une alerte dans la balise `<notes>` !

---

### Étape 5.6 : Mettre à jour le workflow avec le fichier de suppressions

Modifiez `.github/workflows/sca-dependency-scan.yml` pour utiliser le fichier de suppressions :

```yaml
      - name: 📦 Run OWASP Dependency Check
        env:
          NVD_API_KEY: ${{ secrets.NVD_API_KEY }}
        run: |
          if [ -n "${NVD_API_KEY}" ]; then
            echo "🚀 Running with NVD API Key for faster download"
            mvn org.owasp:dependency-check-maven:check \
              -DfailBuildOnCVSS=7 \
              -DsuppressionFiles=.owasp-suppressions.xml \
              -DnvdApiKey=${NVD_API_KEY}
          else
            echo "🐌 Running without NVD API Key (slower download)"
            mvn org.owasp:dependency-check-maven:check \
              -DfailBuildOnCVSS=7 \
              -DsuppressionFiles=.owasp-suppressions.xml
          fi
```

**Changement** : Ajout de `-DsuppressionFiles=.owasp-suppressions.xml` dans les deux branches.

---

### Étape 5.7 : Tester avec les suppressions

```bash
git add .
git commit -m "feat: add OWASP suppressions file"
git push origin main
```

**🔍 Observer la différence :**

1. Allez dans `Actions` → Nouveau workflow exécuté
2. Comparez les résultats avec le premier run
3. Les CVE supprimées ne devraient plus apparaître
4. Le job devrait passer si toutes les vulnérabilités >= 7 sont supprimées ou corrigées

---

## ✅ Critères de Validation

**Premier test (sans suppressions) :**
- [ ] Le scan OWASP Dependency-Check s'exécute
- [ ] Le scan Trivy SCA (filesystem) s'exécute
- [ ] Des vulnérabilités sont détectées et affichées
- [ ] Le rapport SARIF OWASP est uploadé vers GitHub Security
- [ ] Le rapport JSON Trivy est disponible dans les Artifacts
- [ ] Les résultats apparaissent dans Security → Code scanning
- [ ] Le job peut échouer si CVSS >= 7 (c'est normal !)

**Second test (avec suppressions) :**
- [ ] Le fichier `.owasp-suppressions.xml` existe
- [ ] Le workflow utilise `-DsuppressionFiles=.owasp-suppressions.xml`
- [ ] Les CVE supprimées ne remontent plus dans les alertes
- [ ] Le job passe si toutes les vulnérabilités critiques sont gérées
- [ ] S'exécute en parallèle avec SAST et Secret Scanning

---

## 🤔 Questions de Compréhension

1. **Qu'est-ce qu'un score CVSS ?**
   <details>
   <summary>Voir la réponse</summary>

   CVSS (Common Vulnerability Scoring System) est un score de 0 à 10 qui évalue la gravité d'une vulnérabilité :
   - **0.0** : Aucune vulnérabilité
   - **0.1-3.9** : LOW
   - **4.0-6.9** : MEDIUM
   - **7.0-8.9** : HIGH
   - **9.0-10.0** : CRITICAL

   Le score prend en compte : complexité d'exploitation, impact, portée, etc.
   </details>

2. **Pourquoi choisir un seuil de 7 ?**
   <details>
   <summary>Voir la réponse</summary>

   - Un seuil de 7 bloque les vulnérabilités HIGH et CRITICAL
   - C'est un bon équilibre entre sécurité et pragmatisme
   - Les vulnérabilités MEDIUM (< 7) peuvent être traitées plus tard
   - Évite de bloquer le pipeline pour des vulnérabilités mineures
   - Ajustable selon la politique de sécurité de l'entreprise
   </details>

3. **Comment mettre à jour une dépendance vulnérable ?**
   <details>
   <summary>Voir la réponse</summary>

   1. Identifier la dépendance dans le rapport SARIF ou JSON
   2. Dans `pom.xml`, mettre à jour la version :
      ```xml
      <dependency>
        <groupId>com.example</groupId>
        <artifactId>vulnerable-lib</artifactId>
        <version>2.0.0</version> <!-- Version corrigée -->
      </dependency>
      ```
   3. Tester localement : `mvn clean test`
   4. Commit et push
   5. Si pas de version corrigée : ajouter suppression dans `.owasp-suppressions.xml` (temporaire)
   </details>

4. **Qu'est-ce que la base de données NVD ?**
   <details>
   <summary>Voir la réponse</summary>

   NVD (National Vulnerability Database) est la base de données officielle des vulnérabilités :
   - Maintenue par le NIST (US)
   - Contient toutes les CVE (Common Vulnerabilities and Exposures)
   - Mise à jour quotidiennement
   - OWASP Dependency-Check l'utilise pour détecter les vulnérabilités
   </details>

5. **Pourquoi utiliser deux outils SCA (OWASP + Trivy) ?**
   <details>
   <summary>Voir la réponse</summary>

   - **Couverture complémentaire** : Chaque outil a sa propre base de vulnérabilités
   - **OWASP Dependency-Check** : Spécialisé pour Maven/Java, NVD database
   - **Trivy** : Base de données plus large, détection plus rapide
   - **Redondance** : Réduit les faux négatifs (vulnérabilités manquées)
   - **Formats différents** : SARIF pour OWASP, JSON pour Trivy
   </details>

6. **Pourquoi créer d'abord le workflow SANS le fichier de suppressions ?**
   <details>
   <summary>Voir la réponse</summary>

   C'est une approche pédagogique **"fail-first"** :
   - **Voir le problème en premier** : Vous observez les vraies vulnérabilités détectées
   - **Comprendre l'impact** : Vous voyez pourquoi certains builds échouent
   - **Apprécier la solution** : Le fichier de suppressions devient utile une fois le problème identifié
   - **Meilleure pratique** : En production, commencez toujours par analyser TOUTES les vulnérabilités avant de supprimer quoi que ce soit
   - **Documentation** : Vous documentez pourquoi chaque suppression est nécessaire
   </details>

---

## 🎯 Architecture Actuelle

```
build-and-test
    ├── code-quality-sast      (parallèle)
    ├── secret-scanning        (parallèle)
    └── sca-dependency-scan    (parallèle)
```

**3 scans de sécurité en parallèle !** ⚡ Le pipeline est de plus en plus complet.

---

## 💡 Points Importants

### SCA vs SAST

| Aspect | SAST | SCA |
|--------|------|-----|
| Cible | Votre code source | Vos dépendances |
| Détecte | Bugs de sécurité dans votre code | Vulnérabilités connues dans les libs |
| Base | Analyse du code | Base de données CVE |
| Exemple | Injection SQL dans votre code | Log4Shell dans log4j |

### Approche Fail-First (Pédagogique)

Dans cet exercice, vous avez suivi une approche **fail-first** :

1. **Premier run** : Sans suppressions → Voir toutes les vulnérabilités
2. **Analyse** : Identifier les vraies vulnérabilités vs faux positifs
3. **Action** : Mettre à jour les dépendances OU ajouter des suppressions documentées
4. **Second run** : Avec suppressions → Build propre

**Pourquoi cette approche ?**
- Vous comprenez **pourquoi** le fichier de suppressions est utile
- Vous ne masquez pas aveuglément les problèmes
- Vous documentez vos décisions de sécurité

### Gestion des Faux Positifs

Le fichier de suppressions permet d'ignorer des vulnérabilités qui ne vous affectent pas :

```xml
<suppress>
  <notes>On n'utilise pas cette fonctionnalité vulnérable</notes>
  <packageUrl regex="true">^pkg:maven/org\.springframework\.boot/spring\-boot\-actuator.*$</packageUrl>
  <cve>CVE-2023-12345</cve>
</suppress>
```

**⚠️ Attention** :
- Toujours documenter **pourquoi** vous supprimez une alerte dans `<notes>`
- Ne supprimez que les faux positifs ou CVE non applicables
- Pour les vraies vulnérabilités : **mettez à jour la dépendance** dans `pom.xml`

### Workflow de Gestion des Vulnérabilités

```
CVE détectée
    │
    ├─→ Vraie vulnérabilité applicable ?
    │       └─→ OUI → Mettre à jour la dépendance dans pom.xml
    │
    └─→ Faux positif ou non applicable ?
            └─→ OUI → Ajouter suppression documentée dans .owasp-suppressions.xml
```

---

## 📚 Ressources

- [OWASP Dependency-Check](https://owasp.org/www-project-dependency-check/)
- [NVD Database](https://nvd.nist.gov/)
- [CVSS Calculator](https://www.first.org/cvss/calculator/3.1)
- [Maven Dependency Tree](https://maven.apache.org/plugins/maven-dependency-plugin/tree-mojo.html)

---

## 🎉 Félicitations !

Votre pipeline détecte maintenant les vulnérabilités dans vos dépendances !

[Exercice suivant : Sécurité IaC ➡️](Exercice-06.md)
