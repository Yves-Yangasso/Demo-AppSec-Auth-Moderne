# Déploiement Kubernetes avec Jenkins et SonarQube

Ce guide décrit **toutes les étapes pour déployer l'application sur Kubernetes (Docker Desktop) via Jenkins**, avec analyse SonarQube bloquante. Suivre les étapes dans l'ordre : la partie 1 se fait une seule fois, la partie 2 à chaque déploiement.

> Aucun secret ne figure dans ce document. Les valeurs sensibles sont stockées uniquement dans les credentials Jenkins.

---

## 0. Vue d'ensemble

```
GitHub (master)
   │  Jenkinsfile
   ▼
Jenkins (Docker, :9090)
   ├─ Build + tests unitaires (Maven)
   ├─ OWASP Dependency Check (NVD)
   ├─ SonarQube Analysis ──► SonarQube (:9000)
   ├─ Quality Gate  ◄────── webhook        ← bloquant
   ├─ Trivy (code + images)
   ├─ Docker Build + Push ──► Docker Hub (bao236/poc-keycloak-*)
   ├─ UAT     : approbation → kubectl apply -k k8s/overlays/uat      → :30091
   └─ PREPROD : approbation → kubectl apply -k k8s/overlays/preprod  → :30092

Kubernetes (Docker Desktop)                     Docker Compose (demo/)
  namespace poc-uat / poc-preprod                 Keycloak :8080 + Postgres
   ├─ poc-keycloak-app-<env>      (NodePort)  ──►  joint via host.docker.internal:8080
   └─ poc-keycloak-service-<env>  (ClusterIP)
```

| Outil          | URL                     | Compte |
|----------------|-------------------------|--------|
| Jenkins        | http://localhost:9090   | `admin` / `admin123` |
| SonarQube      | http://localhost:9000   | `admin` / mot de passe choisi au 1er login |
| Keycloak       | http://localhost:8080   | `admin` / `admin` |
| App UAT        | http://localhost:30091  | `testuser` / `password` |
| App PREPROD    | http://localhost:30092  | `testuser` / `password` |

### Fichiers concernés

| Fichier | Rôle |
|---------|------|
| [`Jenkinsfile`](../Jenkinsfile) | Pipeline complet (stages UAT / PREPROD en Kubernetes) |
| [`k8s/base/app.yaml`](../k8s/base/app.yaml) | Deployment + Service de l'app web (port 8090) |
| [`k8s/base/service.yaml`](../k8s/base/service.yaml) | Deployment + Service du service REST (port 8091) |
| [`k8s/overlays/uat/kustomization.yaml`](../k8s/overlays/uat/kustomization.yaml) | Namespace `poc-uat`, suffixe `-uat`, NodePort 30091 |
| [`k8s/overlays/preprod/kustomization.yaml`](../k8s/overlays/preprod/kustomization.yaml) | Namespace `poc-preprod`, suffixe `-preprod`, NodePort 30092 |
| [`ci/jenkins/Dockerfile`](jenkins/Dockerfile) | Image Jenkins : Maven, Trivy, Docker CLI, **kubectl v1.34.1** |
| [`ci/docker-compose-ci-tools.yml`](docker-compose-ci-tools.yml) | Jenkins + SonarQube + Postgres |
| [`demo/docker-compose.yml`](../demo/docker-compose.yml) | Keycloak + Postgres + Mailpit |

---

# PARTIE 1 — Installation (une seule fois)

> Toutes les commandes se lancent depuis la **racine du projet** :
> ```bash
> cd /chemin/vers/Demo-AppSec-Auth-Moderne
> ```

## Étape 1 — Activer Kubernetes dans Docker Desktop

Docker Desktop → **Settings → Kubernetes → Enable Kubernetes** → *Apply & restart*.

Vérifier :
```bash
kubectl config use-context docker-desktop
kubectl get nodes
# NAME             STATUS   ROLES           VERSION
# docker-desktop   Ready    control-plane   v1.34.1
```

> La version de kubectl dans l'image Jenkins (`KUBECTL_VERSION` dans `ci/jenkins/Dockerfile`) doit rester à ±1 version mineure de celle du cluster.

## Étape 2 — Déclarer `host.docker.internal` sur le Mac

Le navigateur et les pods doivent joindre Keycloak à **la même URL**, sinon le jeton est rejeté (issuer différent).

```bash
echo "127.0.0.1 host.docker.internal" | sudo tee -a /etc/hosts
ping -c1 host.docker.internal
```

> Dans zsh, ne pas mettre de commentaire `# ...` après une commande : il est passé comme argument.

## Étape 3 — Démarrer Keycloak

```bash
cd demo && docker compose up -d && cd ..
curl -s -o /dev/null -w "%{http_code}\n" \
  http://host.docker.internal:8080/realms/bzhcamp/.well-known/openid-configuration   # attendu : 200
```

### Autoriser les URLs Kubernetes dans Keycloak

Le client `app-springboot` doit accepter les redirections vers les NodePorts. Le fichier `demo/keycloak/import/realm-export.json` les contient déjà, **mais l'import n'a lieu que si le realm n'existe pas**. Sur une base existante :

Console Keycloak → realm **bzhcamp** → **Clients → app-springboot** :

| Champ | Valeurs |
|-------|---------|
| Valid redirect URIs | `http://localhost:8090/*`, `http://localhost:30091/*`, `http://localhost:30092/*` |
| Web origins | `http://localhost:8090`, `http://localhost:30091`, `http://localhost:30092` |

Noter au passage le secret du client : onglet **Credentials → Client secret** (utilisé à l'étape 6).

## Étape 4 — Démarrer Jenkins et SonarQube

```bash
docker compose -f ci/docker-compose-ci-tools.yml up -d --build
```

Vérifier :
```bash
docker ps --format "table {{.Names}}\t{{.Status}}"          # jenkins, sonarqube, sonar-db : Up / healthy
docker exec jenkins kubectl version --client                  # Client Version: v1.34.1
```

> Jenkins s'ouvre sur **http://localhost:9090**. Le port 50000 est réservé aux agents (il affiche `Jenkins-Agent-Protocols…`), et le 8080 est celui de Keycloak.

## Étape 5 — Configurer SonarQube

### 5.1 Connexion
http://localhost:9000 → `admin` / `admin` → choisir un nouveau mot de passe.

### 5.2 Token pour Jenkins
**My Account → Security → Generate Tokens**

| Champ | Valeur |
|-------|--------|
| Name | `jenkins-token` |
| Type | Global Analysis Token |
| Expires | No expiration |

Copier le token (affiché une seule fois).

### 5.3 Webhook vers Jenkins
**Administration → Configuration → Webhooks → Create**

| Champ | Valeur |
|-------|--------|
| Name | `Jenkins` |
| URL | `http://jenkins:8080/sonarqube-webhook/` |

> `jenkins:8080` = nom du conteneur sur le réseau Docker `ci-net`, pas `localhost:9090`. Sans ce webhook, le stage *Quality Gate* attend jusqu'au timeout.

### 5.4 Quality Gate du POC
**Quality Gates → Create** → Name : `POC Gate`
- Condition : **Security Rating** is worse than **A**
- **Set as Default**

> Le gate par défaut *Sonar way* exige 80 % de couverture sur le nouveau code ; `app-springboot` n'a pas de tests, il échouerait.

## Étape 6 — Créer les credentials Jenkins

**Manage Jenkins → Credentials → System → Global credentials → Add Credentials**

| ID (exact) | Kind | Valeur |
|------------|------|--------|
| `dockerhub-credentials` | Username with password | `bao236` + **access token** Docker Hub (Read & Write) |
| `nvd-api-key` | Secret text | clé API NVD |
| `sonar-token` | Secret text | token SonarQube de l'étape 5.2 |
| `keycloak-url-uat` | Secret text | `http://host.docker.internal:8080` |
| `keycloak-url-preprod` | Secret text | `http://host.docker.internal:8080` |
| `keycloak-client-secret-uat` | Secret text | secret du client `app-springboot` (étape 3) |
| `keycloak-client-secret-preprod` | Secret text | même valeur |
| `kubeconfig-docker-desktop` | **Secret file** | fichier généré ci-dessous |

### Obtenir les secrets externes

- **Docker Hub** : hub.docker.com → Account Settings → Personal access tokens → *Read & Write*. Ne jamais utiliser le mot de passe du compte.
- **NVD** : demander une clé sur https://nvd.nist.gov/developers/request-an-api-key. Le mail reçu contient un **UUID d'activation**, ce n'est pas la clé : ouvrir https://nvd.nist.gov/developers/confirm-api-key, saisir e-mail + UUID, et copier la **clé API** affichée. Elle peut mettre quelques minutes à devenir active. Test :
  ```bash
  curl -s -o /dev/null -w "%{http_code}\n" -H "apiKey: <CLE>" \
    "https://services.nvd.nist.gov/rest/json/cves/2.0?resultsPerPage=1"     # 200 = valide, 404 = invalide / pas encore active
  ```

### Générer le kubeconfig pour Jenkins

Le kubeconfig de Docker Desktop pointe sur `127.0.0.1`, injoignable depuis le conteneur Jenkins. On le remplace par `kubernetes.docker.internal` :

```bash
kubectl config view --minify --flatten --context docker-desktop \
  | sed 's#server: https://127.0.0.1:#server: https://kubernetes.docker.internal:#' \
  > /tmp/kubeconfig-jenkins

# Vérifier que Jenkins atteint le cluster
docker cp /tmp/kubeconfig-jenkins jenkins:/tmp/kc
docker exec jenkins kubectl --kubeconfig /tmp/kc get nodes      # docker-desktop   Ready
docker exec jenkins rm /tmp/kc
```

Importer `/tmp/kubeconfig-jenkins` dans le credential `kubeconfig-docker-desktop`, puis **le supprimer** (il contient un certificat admin du cluster) :
```bash
rm /tmp/kubeconfig-jenkins
```

## Étape 7 — Relier Jenkins à SonarQube

**Manage Jenkins → System → SonarQube servers**

| Champ | Valeur |
|-------|--------|
| Environment variables | ☑ |
| Name | `SonarQube` (doit correspondre au `withSonarQubeEnv('SonarQube')` du Jenkinsfile) |
| Server URL | `http://sonarqube:9000` |
| Server authentication token | `sonar-token` |

**Save**. Vérifier la connectivité :
```bash
docker exec jenkins curl -s http://sonarqube:9000/api/system/status    # "status":"UP"
```

## Étape 8 — Créer le job Pipeline

Jenkins → **Nouveau Item** → nom `poc-keycloak-k8s` → **Pipeline** → OK.

Section **Pipeline** :

| Champ | Valeur |
|-------|--------|
| Definition | Pipeline script from SCM |
| SCM | Git |
| Repository URL | `https://github.com/Yves-Yangasso/Demo-AppSec-Auth-Moderne.git` |
| Credentials | aucun (dépôt public) |
| Branch Specifier | `*/master` |
| Script Path | `Jenkinsfile` |

**Sauvegarder.**

> Jenkins lit le Jenkinsfile **sur GitHub** : toute modification locale doit être commitée et poussée (`git push origin master`) avant de lancer un build.

---

# PARTIE 2 — Déployer (à chaque version)

## Étape 9 — Vérifier que tout tourne

```bash
kubectl get nodes                                         # Ready
docker ps --format "{{.Names}}" | grep -E "jenkins|sonarqube|keycloak"
git status -sb                                            # master à jour avec origin
```

## Étape 10 — Lancer le pipeline

http://localhost:9090/job/poc-keycloak-k8s/ → **Lancer un build** → ouvrir le build → **Console Output**.

| Stage | Ce qui est vérifié | Bloquant |
|-------|--------------------|----------|
| DEV - Build & Unit Tests | compilation + tests JUnit du service REST | oui |
| OWASP Dependency Check | CVE des dépendances (1er passage ≈ 10 min, téléchargement NVD) | si CVE critique |
| SonarQube Analysis | envoi de l'analyse | oui |
| Quality Gate | verdict `POC Gate` | **oui** |
| Trivy Filesystem / Image | vulnérabilités HIGH / CRITICAL (rapports archivés) | non |
| Docker Build / Push | images `bao236/poc-keycloak-app:vN` et `bao236/poc-keycloak-service:vN` | oui |

Résultat SonarQube : http://localhost:9000 → projets `poc-keycloak-app` et `poc-keycloak-service`.

## Étape 11 — Approuver et déployer en UAT

Le pipeline se met en pause sur **UAT - Approval** (1 h max). Survoler l'étape → **Approuver** (comptes `admin` ou `release-manager`).

Jenkins exécute alors, avec le kubeconfig du credential :

```bash
kubectl create namespace poc-uat --dry-run=client -o yaml | kubectl apply -f -
kubectl -n poc-uat create secret generic keycloak-config \
    --from-literal=KEYCLOAK_ISSUER_URI="<keycloak-url-uat>/realms/bzhcamp" \
    --from-literal=KEYCLOAK_CLIENT_SECRET="<keycloak-client-secret-uat>" \
    --dry-run=client -o yaml | kubectl apply -f -
sed -i "s/newTag: .*/newTag: vN/" k8s/overlays/uat/kustomization.yaml
kubectl apply -k k8s/overlays/uat
kubectl -n poc-uat rollout status deployment/poc-keycloak-service-uat --timeout=180s
kubectl -n poc-uat rollout status deployment/poc-keycloak-app-uat --timeout=180s
```

Si un pod ne démarre pas en 3 minutes, le build échoue.

### Configuration injectée dans les pods

| Variable | Pod | Source |
|----------|-----|--------|
| `SPRING_PROFILES_ACTIVE` | app | overlay (`uat` / `preprod`) |
| `KEYCLOAK_ISSUER_URI` | app + service | Secret `keycloak-config` |
| `KEYCLOAK_CLIENT_SECRET` | app | Secret `keycloak-config` |
| `SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUERURI` | service | dérivée de `KEYCLOAK_ISSUER_URI` |
| `SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_JWKSETURI` | service | `KEYCLOAK_ISSUER_URI` + `/protocol/openid-connect/certs` |

L'app appelle le service REST via le DNS interne `http://poc-keycloak-service-uat:8091/planets` (défini dans `application-uat.yml`).

### Sécurité des pods

- `allowPrivilegeEscalation: false`, `capabilities: drop ALL`
- `readOnlyRootFilesystem: true` (+ `emptyDir` sur `/tmp`)
- `automountServiceAccountToken: false`
- requests `100m` CPU / `256Mi`, limit `512Mi`
- sondes TCP (pas d'actuator dans les projets)

## Étape 12 — Vérifier l'UAT

```bash
kubectl -n poc-uat get pods,svc
kubectl -n poc-uat get deploy -o wide                     # tag vN déployé
kubectl -n poc-uat logs -f deploy/poc-keycloak-app-uat
kubectl -n poc-uat logs -f deploy/poc-keycloak-service-uat
```

Ouvrir **http://localhost:30091** → se connecter avec `testuser` / `password` → tester la page *Planètes*.

## Étape 13 — PREPROD

Même principe : pause sur **PREPROD - Approval** (2 h max) → **Approuver** → déploiement dans `poc-preprod` → **http://localhost:30092**.

```bash
kubectl -n poc-preprod get pods,svc
```

---

# PARTIE 3 — Exploitation

## Rollback

```bash
# Révision précédente
kubectl -n poc-uat rollout undo deployment/poc-keycloak-app-uat
kubectl -n poc-uat rollout undo deployment/poc-keycloak-service-uat

# Ou une version précise (chaque build publie un tag immuable vN)
kubectl -n poc-uat set image deployment/poc-keycloak-app-uat app=bao236/poc-keycloak-app:v11
kubectl -n poc-uat set image deployment/poc-keycloak-service-uat service=bao236/poc-keycloak-service:v11
```

Pour PREPROD : remplacer `uat` par `preprod`.

## Arrêter / nettoyer

```bash
kubectl delete namespace poc-uat poc-preprod                  # supprime les déploiements
docker compose -f ci/docker-compose-ci-tools.yml stop         # garde les données Jenkins / SonarQube
cd demo && docker compose stop                                # Keycloak
```

## Rotation des secrets

| Secret | Où le changer |
|--------|---------------|
| Token Docker Hub | hub.docker.com → révoquer → nouveau token → credential `dockerhub-credentials` |
| Token SonarQube | SonarQube → My Account → Security → révoquer / régénérer → credential `sonar-token` |
| Secret client Keycloak | Keycloak → Clients → app-springboot → Credentials → *Regenerate* → credentials `keycloak-client-secret-*` |
| Kubeconfig | refaire l'étape 6 (génération) → credential `kubeconfig-docker-desktop` |

Un secret collé dans un chat, un ticket ou un commit doit être considéré comme compromis : le révoquer et le remplacer.

---

# PARTIE 4 — Dépannage

| Symptôme | Cause / solution |
|----------|------------------|
| `zsh: command not found: kubeconfig-docker-desktop` | C'est un ID de credential Jenkins, pas une commande (étape 6) |
| `compose file ... demo/ci/docker-compose-ci-tools.yml: no such file` | Commande lancée depuis `demo/` → revenir à la racine du projet |
| `ping: usage ...` | Commentaire `# ...` passé en argument dans zsh → le retirer |
| Page Jenkins = `Jenkins-Agent-Protocols…` | Mauvais port (50000) → http://localhost:9090 |
| `Credentials not found` / `Could not find credentials entry with ID` | ID différent de celui du Jenkinsfile (tableau étape 6) |
| `No such SonarQube installation: SonarQube` | Étape 7 non faite ou nom différent |
| Quality Gate bloqué jusqu'au timeout | Webhook manquant ou mauvaise URL (`http://jenkins:8080/sonarqube-webhook/`) |
| Quality Gate `FAILED` | Ouvrir le projet dans SonarQube ; vérifier que `POC Gate` est le gate par défaut |
| OWASP très lent / erreur 403-404 NVD | Clé NVD absente, non activée ou invalide (test curl étape 6) |
| `denied: requested access to the resource is denied` au push | Token Docker Hub sans droit *Write*, ou utilisateur ≠ `bao236` |
| `kubectl: connection refused` / erreur TLS | Kubernetes désactivé, ou `server:` du kubeconfig resté sur `127.0.0.1` |
| Warning `version difference ... exceeds the supported minor version skew` | Aligner `KUBECTL_VERSION` du Dockerfile Jenkins sur la version du cluster, rebuild |
| `rollout status` en timeout | `kubectl -n poc-uat describe pod` : `ImagePullBackOff` (image/tag absent sur Docker Hub) ou `CrashLoopBackOff` (voir `logs`) |
| Pod en `CrashLoopBackOff`, erreur issuer / `Unable to resolve the Configuration` | Keycloak arrêté ou injoignable depuis le cluster : vérifier `demo/` et `keycloak-url-*` |
| Login : `Invalid parameter: redirect_uri` | URLs 30091 / 30092 absentes du client `app-springboot` (étape 3) |
| Login OK mais erreur 401 / issuer invalide | Le navigateur n'utilise pas `host.docker.internal` → vérifier `/etc/hosts` (étape 2) |
| Jenkins exécute encore l'ancien pipeline | Modifications non poussées sur GitHub |
| Mot de passe admin SonarQube perdu | Voir ci-dessous |

### Réinitialiser le mot de passe admin SonarQube

Remet le mot de passe à `admin` (changement forcé au prochain login) :
```bash
docker exec sonar-db psql -U sonar -d sonar -c "update users set crypted_password='100000\$t2h8AtNs1AlCHuLobDjHQTn9XppwTIx88UjqUm4s8RsfTuXQHSd/fpFexAnewwPsO6jGFQUv/24DnO55hY6Xew==', salt='k9x9eN127/3e/hf38iNiKwVfaVk=', hash_method='PBKDF2', reset_password=true, user_local=true where login='admin';"
```
La prise en compte peut demander quelques secondes. Les tokens existants (dont `sonar-token`) restent valides.

---

## Voir aussi

- [`ci/README.md`](README.md) — détail de chaque test du pipeline
- [`ci/runbook-ci.md`](runbook-ci.md) — installation Jenkins / SonarQube
- [README principal](../README.md)
