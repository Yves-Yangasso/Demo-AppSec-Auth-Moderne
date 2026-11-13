# CI/CD Jenkins — Tests et déploiement

Ce guide décrit **comment lancer le pipeline Jenkins, lire le résultat de chaque test et déployer en UAT puis en PREPROD**.

> L'installation de Jenkins et SonarQube (plugins, credentials, webhook, Quality Gate) est décrite dans [`runbook-ci.md`](runbook-ci.md). Faites-la une fois avant de suivre ce guide.

---

## 1. Vue d'ensemble du pipeline

```
Checkout
 └─ 1. DEV : build + tests unitaires        (parallèle : app / service)
     └─ 2. OWASP Dependency Check            (CVE des dépendances Maven)
         └─ 3. SonarQube Analysis
             └─ 4. Quality Gate              ← bloquant
                 └─ 5. Trivy Filesystem Scan
                     └─ 6. Docker Build      (parallèle : app / service)
                         └─ 7. Trivy Image Scan
                             └─ 8. Docker Push (Docker Hub)
                                 └─ 9.  UAT     : approbation → déploiement K8s (poc-uat)     :30091
                                     └─ 10. PREPROD : approbation → déploiement K8s (poc-preprod) :30092
```

| Outil      | URL                    | Compte             |
|------------|------------------------|--------------------|
| Jenkins    | http://localhost:9090  | `admin` / `admin123` |
| SonarQube  | http://localhost:9000  | `admin` / `admin123` (après changement au 1er login) |
| App UAT    | http://localhost:30091 | `testuser` / `password` |
| App PREPROD| http://localhost:30092 | `testuser` / `password` |

---

## 2. Avant de lancer : vérifier en local

Exécuter localement les mêmes commandes que l'étape DEV évite un aller-retour Jenkins :

```bash
# Tests unitaires du service REST (identique au stage « Service - Verify »)
cd demo/service-springboot-rest
mvn clean verify

# Compilation de l'application web (identique au stage « App - Package »)
cd ../app-springboot
mvn clean package -DskipTests
```

Puis pousser sur la branche suivie par le job (`master`) :

```bash
git push origin master
```

---

## 3. Lancer le pipeline

1. Démarrer les outils CI :
   ```bash
   docker compose -f ci/docker-compose-ci-tools.yml up -d
   docker compose -f ci/docker-compose-ci-tools.yml ps   # jenkins, sonarqube, sonar-db → healthy
   ```
2. Ouvrir Jenkins → job **`poc-keycloak-pipeline`** → **Build Now**.
   (Si le webhook GitHub est configuré, le build part tout seul à chaque push.)
3. Cliquer sur le numéro du build → **Console Output** pour suivre en direct, ou **Pipeline Stage View** pour la vue par étape.

Le tag d'image produit est `v<numéro de build>` (ex. build #12 → `v12`).

---

## 4. Les tests, étape par étape

### Étape 1 — DEV : build et tests unitaires

| Module                    | Commande            | Tests exécutés |
|---------------------------|---------------------|----------------|
| `service-springboot-rest` | `mvn clean verify`  | 10 tests JUnit (voir ci-dessous) |
| `app-springboot`          | `mvn clean package -DskipTests` | aucun (compilation seule) |

Tests du service REST :

| Classe de test                    | Ce qui est vérifié |
|-----------------------------------|--------------------|
| `KeycloakJwtRolesConverterTest`   | Extraction des rôles realm et client depuis un JWT Keycloak, cas sans rôle |
| `PlanetsServiceControllerTest`    | `GET /planets` → 200 si authentifié, 401 sinon ; endpoint public → 200 sans jeton |
| `PlanetsServiceTest`              | La liste des planètes est non vide, contient les valeurs attendues, est stable |

**Où voir le résultat :** build → **Test Result** (publié par le plugin JUnit depuis `target/surefire-reports/*.xml`).
**Échec :** un seul test rouge fait échouer le build.

### Étape 2 — OWASP Dependency Check

Analyse les dépendances Maven des deux modules contre la base NVD (credential `nvd-api-key`).

**Où voir le résultat :** build → **Dependency-Check Results** (menu de gauche).

| Seuil                                    | Effet sur le build |
|------------------------------------------|--------------------|
| ≥ 1 vulnérabilité **critique**           | `FAILURE`          |
| ≥ 5 vulnérabilités **hautes**            | `UNSTABLE` (jaune) |

Les faux positifs de l'app sont ignorés via [`demo/app-springboot/owasp-suppressions.xml`](../demo/app-springboot/owasp-suppressions.xml).
Sans clé NVD, le premier scan peut durer 30 à 60 minutes.

### Étapes 3 et 4 — SonarQube et Quality Gate

Analyse du code des deux modules (projets `poc-keycloak-app` et `poc-keycloak-service`), puis attente du verdict SonarQube via le webhook (max 10 min).

**Où voir le résultat :** http://localhost:9000/projects → onglets *Issues*, *Security Hotspots*, *Coverage*.
**Échec :** Quality Gate `FAILED` → le pipeline **s'arrête** (`abortPipeline: true`). Rien n'est construit ni déployé.

### Étape 5 — Trivy Filesystem Scan

Scan des dossiers `demo/app-springboot` et `demo/service-springboot-rest` (dépendances et secrets en clair), sévérités `HIGH` et `CRITICAL`.

**Où voir le résultat :** Console Output, ou artefacts du build `trivy-fs-app.txt` et `trivy-fs-service.txt`.
**Échec :** non bloquant (`--exit-code 0`), le rapport est à lire manuellement.

### Étapes 6 et 7 — Docker Build et Trivy Image Scan

Construction des images `moustaphisene/poc-keycloak-app` et `moustaphisene/poc-keycloak-service` (tags `vN` et `latest`), puis scan des images avant publication.

**Où voir le résultat :** artefacts `trivy-image-app.txt` et `trivy-image-service.txt`.
**Échec :** non bloquant (`--exit-code 0`).

### Étape 8 — Docker Push

Publication des 4 tags sur Docker Hub (credential `dockerhub-credentials`).
Vérifier : `https://hub.docker.com/r/moustaphisene/poc-keycloak-app/tags`.

### Résumé des critères bloquants

| Étape               | Bloque le pipeline ?               |
|---------------------|------------------------------------|
| Tests unitaires     | Oui                                |
| OWASP               | Oui si ≥ 1 critique, sinon jaune   |
| Quality Gate        | Oui                                |
| Trivy FS / Image    | Non (rapport seulement)            |

---

## 5. Déploiement (Kubernetes)

Jenkins déploie dans le **cluster Kubernetes de Docker Desktop** avec `kubectl apply -k` (Kustomize). Les manifests sont dans [`k8s/`](../k8s) :

```
k8s/
├── base/                 Deployment + Service pour l'app (8090) et le service REST (8091)
└── overlays/
    ├── uat/              namespace poc-uat,     suffixe -uat,     NodePort 30091
    └── preprod/          namespace poc-preprod, suffixe -preprod, NodePort 30092
```

Les deux images (`poc-keycloak-app` et `poc-keycloak-service`) sont déployées. L'app appelle le service REST par son nom DNS interne (`poc-keycloak-service-uat:8091`). Keycloak reste **hors cluster** (`demo/docker-compose.yml`) et les pods le joignent via `host.docker.internal:8080`.

Prérequis (une fois) : Kubernetes activé dans Docker Desktop et credential `kubeconfig-docker-desktop` créé (voir [`runbook-ci.md`](runbook-ci.md) §2.4, credential 7).

### 5.1 UAT

1. Le pipeline se met en pause sur **UAT - Approval** (délai max : 1 h).
2. Dans Jenkins, survoler l'étape en pause (ou ouvrir la Console Output) → **Approuver**. Un commentaire est facultatif.
   Seuls les utilisateurs `admin` ou `release-manager` peuvent approuver.
3. Jenkins exécute alors :
   - création du namespace `poc-uat` s'il n'existe pas
   - création/mise à jour du Secret `keycloak-config` à partir des credentials Jenkins :

     | Clé                      | Valeur |
     |--------------------------|--------|
     | `KEYCLOAK_ISSUER_URI`    | `<keycloak-url-uat>/realms/bzhcamp` |
     | `KEYCLOAK_CLIENT_SECRET` | credential `keycloak-client-secret-uat` |

   - remplacement du tag d'image par `vN` dans `k8s/overlays/uat/kustomization.yaml`, puis `kubectl apply -k k8s/overlays/uat`
   - attente du `rollout status` des deux Deployments (3 min max). Si un pod ne démarre pas, le build échoue.

Sans approbation dans le délai, le build s'arrête en `ABORTED` : les images sont publiées mais rien n'est déployé.

### 5.2 PREPROD

Même principe, après l'UAT :

1. Pause sur **PREPROD - Approval** (délai max : 2 h) → **Approuver**.
2. Déploiement dans le namespace `poc-preprod`, profil `preprod`, NodePort **30092**, credentials `keycloak-url-preprod` et `keycloak-client-secret-preprod`.

### 5.3 Vérifier un déploiement

```bash
kubectl -n poc-uat get pods,svc
kubectl -n poc-uat get deploy -o wide             # image et tag déployés
kubectl -n poc-uat logs -f deploy/poc-keycloak-app-uat
kubectl -n poc-uat logs -f deploy/poc-keycloak-service-uat
```

Puis ouvrir http://localhost:30091 (UAT) ou http://localhost:30092 (PREPROD) et se connecter avec `testuser` / `password`.
Keycloak doit tourner (`docker compose up -d` dans `demo/`) pour que la connexion fonctionne.

### 5.4 Revenir à une version précédente (rollback)

```bash
# Revenir à la révision précédente
kubectl -n poc-uat rollout undo deployment/poc-keycloak-app-uat
kubectl -n poc-uat rollout undo deployment/poc-keycloak-service-uat

# Ou forcer un tag précis (chaque build publie un tag immuable vN)
kubectl -n poc-uat set image deployment/poc-keycloak-app-uat app=moustaphisene/poc-keycloak-app:v11
```

Pour PREPROD, remplacer `uat` par `preprod`.

### 5.5 Arrêter les environnements

```bash
kubectl delete namespace poc-uat poc-preprod
docker compose -f ci/docker-compose-ci-tools.yml stop    # garde les données Jenkins/SonarQube
```

---

## 6. Limites connues

- **Issuer Keycloak** : le navigateur et les pods doivent voir Keycloak à la même URL, sinon le jeton est rejeté (issuer différent). Avec `keycloak-url-* = http://host.docker.internal:8080`, ajouter `127.0.0.1 host.docker.internal` dans `/etc/hosts` du Mac si ce nom ne résout pas.
- **Redirect URIs** : `http://localhost:30091/*` et `http://localhost:30092/*` ont été ajoutés au client `app-springboot` dans `demo/keycloak/import/realm-export.json`. L'import ne s'applique que si le realm n'existe pas encore : sur une base Keycloak existante, les ajouter dans la console admin (Clients → app-springboot) ou recréer le volume `pgdata`.
- **Sondes TCP** : les projets n'incluent pas `spring-boot-starter-actuator`. Les probes vérifient donc seulement que le port HTTP répond.
- **Pas de tests sur l'app web** : `app-springboot` n'a aucun test unitaire, sa couverture SonarQube est donc de 0 %.

---

## 7. Dépannage rapide

| Symptôme                              | Solution |
|---------------------------------------|----------|
| Test rouge à l'étape DEV              | Relancer `mvn clean verify` dans `demo/service-springboot-rest` en local et lire le rapport `target/surefire-reports/` |
| `Quality Gate` bloqué jusqu'au timeout | Webhook SonarQube → `http://jenkins:8080/sonarqube-webhook/` |
| Quality Gate `FAILED`                 | Ouvrir le projet dans SonarQube, corriger les issues ou ajuster le gate (runbook §3.4) |
| `Credentials not found`               | L'ID du credential doit correspondre exactement à celui du `Jenkinsfile` |
| `docker: permission denied`           | Le socket `/var/run/docker.sock` doit être monté dans le conteneur Jenkins |
| `kubectl: connection refused` / TLS   | Kubernetes non activé dans Docker Desktop, ou `server:` du kubeconfig incorrect (runbook §2.4, credential 7) |
| `rollout status` en timeout           | `kubectl -n poc-uat describe pod` puis `logs` : souvent `ImagePullBackOff` (image privée sur Docker Hub) ou Keycloak injoignable |
| Port 30091/30092 déjà utilisé         | `kubectl get svc -A \| grep 3009` |
| Pas de bouton *Approuver*             | Se connecter avec `admin` (ou un compte `release-manager`) |
| Login Keycloak impossible en UAT      | Vérifier `keycloak-url-uat` (`http://host.docker.internal:8080`) et le secret client |

Pour aller plus loin : [`runbook-ci.md`](runbook-ci.md) (installation complète) et le [README principal](../README.md).
