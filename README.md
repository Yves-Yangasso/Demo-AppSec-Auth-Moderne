# Demo-Securite-App-Moderne

Démo pédagogique de sécurisation d'applications modernes avec **Keycloak 25** (OpenID Connect / OAuth2) :
une application web Spring Boot, une API REST protégée par JWT, un client Angular et un client JavaScript « vanilla ».

---

## 1. Architecture et ports

```
                         ┌──────────────────────────────┐
                         │  Keycloak  :8080             │
                         │  realm « bzhcamp »           │
                         │  (Postgres + Mailpit SMTP)   │
                         └──────────────┬───────────────┘
            OIDC login / JWT            │
   ┌──────────────────┬─────────────────┼───────────────────┐
   ▼                  ▼                 ▼                   │
app-springboot    keycloak-angular   myapp (nginx)          │ JWKS
  :8090             :4200              :8081                │
   │  Bearer token                                          │
   └──────────────────────────► service-springboot-rest :8091
```

| Composant                  | Port  | Rôle                                                 |
|----------------------------|-------|------------------------------------------------------|
| Keycloak                   | 8080  | Serveur d'identité (console admin : `/admin`)        |
| Keycloak health / metrics  | 9000  | `/health/ready` (interne au conteneur)               |
| PostgreSQL                 | —     | Base de Keycloak (réseau Docker interne)             |
| Mailpit (SMTP / UI web)    | 1025 / 8025 | Capture des e-mails Keycloak (http://localhost:8025) |
| nginx → `myapp`            | 8081  | Client JS public                                     |
| `app-springboot`           | 8090  | Application web (client OAuth2 confidentiel)         |
| `service-springboot-rest`  | 8091  | API REST (resource server JWT)                       |
| `keycloak-angular`         | 4200  | SPA Angular (client public)                          |

---

## 2. Référentiel de configuration (clientID, realm, comptes…)

### Realm

| Paramètre        | Valeur                                        |
|------------------|-----------------------------------------------|
| Realm            | `bzhcamp`                                     |
| Issuer URI       | `http://localhost:8080/realms/bzhcamp`        |
| JWKS URI         | `http://localhost:8080/realms/bzhcamp/protocol/openid-connect/certs` |
| Découverte OIDC  | `http://localhost:8080/realms/bzhcamp/.well-known/openid-configuration` |
| Langues          | `fr` (défaut), `en`                           |
| Thème de login   | `bzhcamp` (`demo/themes/bzhcamp`)             |
| Rôles realm      | `user`, `admin`                               |

Le realm est importé automatiquement au démarrage depuis `demo/keycloak/import/realm-export.json`.

### Clients

| Client ID          | Type                       | Flux                                        | Redirect URIs              | Web origins             | Défini dans l'export ? |
|--------------------|----------------------------|---------------------------------------------|----------------------------|-------------------------|------------------------|
| `app-springboot`   | **Confidentiel** (secret)  | Authorization Code + Direct Access Grants   | `http://localhost:8090/*`  | `http://localhost:8090` | ✅ oui                 |
| `keycloak-angular` | **Public**                 | Authorization Code                          | `http://localhost:4200/*`  | `http://localhost:4200` | ✅ oui                 |
| `myapp`            | **Public**                 | Authorization Code                          | `http://localhost:8081/*`  | `http://localhost:8081` | ✅ oui                 |

`service-springboot-rest` n'est pas un client Keycloak : c'est un *resource server* qui se contente de valider les JWT (signature, issuer, expiration, et audience en option).

### Comptes

| Usage                     | Identifiant | Mot de passe | Où                          |
|---------------------------|-------------|--------------|-----------------------------|
| Admin Keycloak (master)   | `admin`     | `admin`      | http://localhost:8080/admin |
| Utilisateur de test       | `testuser`  | `password`   | realm `bzhcamp`, rôle `user` |
| Base Postgres             | `keycloak`  | `password`   | base `keycloak`             |

> ⚠️ Identifiants de démo uniquement. Ne jamais réutiliser ces valeurs hors poste local.

### Variables d'environnement

| Variable                 | Utilisée par              | Obligatoire | Description |
|--------------------------|---------------------------|-------------|-------------|
| `KEYCLOAK_CLIENT_SECRET` | `app-springboot`          | **Oui**     | Secret du client `app-springboot` (généré par Keycloak, voir §4.2) |
| `SPRING_PROFILES_ACTIVE` | `app-springboot`          | Non         | `dev`, `uat`, `preprod` (défaut : aucun → `application.yml`) |
| `KEYCLOAK_ISSUER_URI`    | `app-springboot` (uat/preprod) | Non    | Surcharge l'issuer (ex. `http://keycloak-uat:8080/realms/bzhcamp`) |
| `KEYCLOAK_AUDIENCE`      | `service-springboot-rest` | Non         | Active la validation du claim `aud` (vide = désactivée) |

---

## 3. Prérequis

- Docker + Docker Compose
- Java 17 et Maven 3.9 (pour lancer les applis Spring Boot en local)
- Node.js + npm (pour l'application Angular)
- Ports libres : 8080, 8081, 8090, 8091, 4200, 1025, 8025

---

## 4. Étapes de configuration

### 4.1 Démarrer Keycloak (+ Postgres, Mailpit, nginx)

```bash
cd demo
docker compose up -d --build
docker compose ps          # attendre que keycloak soit "healthy"
```

Images et conteneurs créés par `demo/docker-compose.yml` :

| Service    | Image                    | Origine                                   | Conteneur          | Ports exposés |
|------------|--------------------------|-------------------------------------------|--------------------|---------------|
| `keycloak` | `demo-keycloak:latest`   | **Construite** depuis `demo/Dockerfile` (Keycloak 25.0.0 + realm intégré) | `demo-keycloak-1`  | 8080 |
| `postgres` | `postgres:16.1`          | Docker Hub                                | `demo-postgres-1`  | — (interne) |
| `smtp`     | `axllent/mailpit:latest` | Docker Hub                                | `demo-smtp-1`      | 1025, 8025 |
| `nginx`    | `nginx:1.25.3-alpine`    | Docker Hub (sert `demo/myapp` en lecture seule) | `demo-nginx-1` | 8081 |

Volume : `demo_pgdata` (données Postgres, donc tout le realm Keycloak).

> `realm-export.json` est **copié dans l'image** `demo-keycloak` au build. Après l'avoir modifié :
> `docker compose up -d --build` (reconstruit l'image), et `docker compose down -v` avant si le realm existe déjà en base.

Vérification :

```bash
docker compose ps        # les 4 conteneurs "Up", keycloak et postgres "healthy"
docker images demo-keycloak
curl -s http://localhost:8080/realms/bzhcamp/.well-known/openid-configuration | head -c 200
```

Puis se connecter à la console : http://localhost:8080/admin (`admin` / `admin`) et sélectionner le realm **bzhcamp**.

> L'import (`--import-realm`) ne s'exécute que si le realm n'existe pas encore. Pour repartir de zéro :
> `docker compose down -v` (supprime le volume Postgres).

### 4.2 Récupérer le secret du client `app-springboot`

L'export ne contient pas de secret : Keycloak en génère un à l'import.

1. Console admin → realm **bzhcamp** → **Clients** → `app-springboot`
2. Onglet **Credentials** → copier **Client secret** (ou *Regenerate*)
3. L'exporter dans le terminal qui lancera l'application :

```bash
export KEYCLOAK_CLIENT_SECRET=<secret_copié>
```

### 4.3 Vérifier le client `myapp` (client JS servi par nginx)

Le client `myapp` (référencé par `demo/myapp/keycloak.json`) est importé automatiquement.
Si le realm existait déjà avant son ajout à l'export, l'import ne le crée pas : faire `docker compose down -v` puis relancer, ou le créer à la main :

1. **Clients** → **Create client**
   - Client type : `OpenID Connect`
   - Client ID : `myapp`
2. **Capability config**
   - Client authentication : **Off** (client public)
   - Standard flow : **On** — Direct access grants : **Off**
3. **Login settings**
   - Root URL : `http://localhost:8081`
   - Valid redirect URIs : `http://localhost:8081/*`
   - Web origins : `http://localhost:8081`
4. **Save**

### 4.4 (Optionnel) Activer la validation d'audience sur l'API

1. **Clients** → client émetteur du jeton (ex. `app-springboot`) → **Client scopes** → `<client>-dedicated` → **Add mapper** → **By configuration** → **Audience**
   - Included Custom Audience : `service-planets` — Add to access token : **On**
2. Lancer le service avec `export KEYCLOAK_AUDIENCE=service-planets`.

### 4.5 (Optionnel) Donner le rôle `admin` à un utilisateur

**Users** → `testuser` → **Role mapping** → **Assign role** → `admin`.
Les rôles realm arrivent dans le claim `realm_access.roles` du JWT.

---

## 5. Lancer les applications

### API REST — `service-springboot-rest` (port 8091)

```bash
cd demo/service-springboot-rest
mvn spring-boot:run
```

| Endpoint     | Accès                                                  |
|--------------|--------------------------------------------------------|
| `GET /public`  | Public                                               |
| `GET /planets` | JWT requis + rôle realm `user` (autorité `ROLE_realm_user`) |

CORS autorisé pour `localhost:8090`, `localhost:4200`, `localhost:8081`.

### Application web — `app-springboot` (port 8090)

```bash
cd demo/app-springboot
export KEYCLOAK_CLIENT_SECRET=<secret>   # voir §4.2
mvn spring-boot:run                      # ou -Dspring-boot.run.profiles=dev
```

Ouvrir http://localhost:8090 → **/planets** redirige vers Keycloak → se connecter avec `testuser` / `password`.
L'application appelle ensuite l'API `http://localhost:8091/planets` en propageant le jeton (Bearer).

### SPA Angular — `keycloak-angular` (port 4200)

```bash
cd demo/keycloak-angular
npm install
npm run serve
```

Configuration (`src/app/app.module.ts`) : realm `bzhcamp`, url `http://localhost:8080`, clientId `keycloak-angular`, `onLoad: 'check-sso'`.

### Client JS — `myapp` (port 8081)

Déjà servi par le conteneur nginx. Ouvrir http://localhost:8081 → **Se connecter**.
La configuration est lue depuis `demo/myapp/keycloak.json`.

---

## 6. Tester l'API en ligne de commande

Le client `app-springboot` a les *Direct Access Grants* activés, ce qui permet d'obtenir un jeton directement (usage démo uniquement) :

```bash
TOKEN=$(curl -s -X POST http://localhost:8080/realms/bzhcamp/protocol/openid-connect/token \
  -d grant_type=password \
  -d client_id=app-springboot \
  -d client_secret=$KEYCLOAK_CLIENT_SECRET \
  -d username=testuser -d password=password | jq -r .access_token)

curl -s http://localhost:8091/public
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8091/planets
```

Décoder le jeton : https://jwt.io (vérifier `iss`, `exp`, `aud`, `realm_access.roles`).

---

## 7. Guide d'utilisation de Keycloak

Console d'administration : http://localhost:8080/admin (`admin` / `admin`).
Toujours vérifier en haut à gauche que le realm sélectionné est **bzhcamp** (et non `master`).

### Vocabulaire

| Terme         | Signification |
|---------------|---------------|
| **Realm**     | Espace isolé (utilisateurs, clients, rôles). `master` sert uniquement à administrer Keycloak. |
| **Client**    | Application qui délègue l'authentification à Keycloak (ici : `app-springboot`, `keycloak-angular`, `myapp`). |
| **Client public** | Application qui ne peut pas garder de secret (SPA, JS navigateur). |
| **Client confidentiel** | Application côté serveur qui s'authentifie avec un secret. |
| **Rôle realm** | Rôle global (`user`, `admin`) → claim `realm_access.roles` du jeton. |
| **Rôle client** | Rôle propre à un client → claim `resource_access.<clientId>.roles`. |
| **Client scope / mapper** | Règle qui ajoute des informations (claims) dans les jetons. |

### Gérer les utilisateurs

- **Créer** : *Users* → *Add user* → renseigner `Username`, e-mail, prénom, nom → *Create*.
- **Mot de passe** : onglet *Credentials* → *Set password* (décocher *Temporary* pour éviter le changement forcé à la première connexion).
- **Rôles** : onglet *Role mapping* → *Assign role* → filtrer par *realm roles* → `user` et/ou `admin`.
- **Désactiver** : bouton *Enabled* sur la fiche de l'utilisateur.

### Gérer les rôles

- *Realm roles* → *Create role* (ex. `manager`), puis l'assigner aux utilisateurs.
- Côté API (`service-springboot-rest`), un rôle realm `xxx` devient l'autorité Spring `ROLE_realm_xxx`.
- Côté `app-springboot`, il devient `ROLE_xxx` → utilisable avec `@PreAuthorize("hasRole('xxx')")`.

### Gérer les clients

- *Clients* → choisir le client :
  - *Settings* : redirect URIs, web origins, flux autorisés, authentification client (public / confidentiel).
  - *Credentials* (clients confidentiels) : voir ou régénérer le secret. **Régénérer le secret oblige à mettre à jour `KEYCLOAK_CLIENT_SECRET`.**
  - *Client scopes* → `<client>-dedicated` : ajouter des mappers (audience, attributs…).
- Règle de sécurité : redirect URIs les plus précises possibles, jamais `*` seul.

### Sessions et déconnexion

- *Sessions* : liste des sessions actives ; *Sign out* pour forcer la déconnexion d'un utilisateur.
- *Users* → utilisateur → *Sessions* : sessions d'un utilisateur précis.

### Événements (audit)

- *Realm settings* → *Events* → *User events settings* → activer *Save events* (connexions, échecs, déconnexions).
- Consulter ensuite dans le menu *Events*. Les événements sont aussi visibles dans les logs du conteneur :
  `docker compose logs -f keycloak` (niveau `org.keycloak.events:debug` déjà activé).

### E-mails (mot de passe oublié, vérification)

1. *Realm settings* → *Email* : Host `smtp`, Port `1025`, adresse *From* quelconque (ex. `noreply@bzhcamp.local`), pas de SSL/TLS.
2. *Realm settings* → *Login* : activer *Forgot password* et/ou *Verify email*.
3. Les e-mails envoyés sont visibles dans Mailpit : http://localhost:8025.

### Sécurité du login

- *Authentication* → *Policies* → *Password policy* : longueur minimale, chiffres, caractères spéciaux…
- *Realm settings* → *Security defenses* → *Brute force detection* : verrouillage après N échecs.
- *Authentication* → *Required actions* : *Configure OTP* (MFA) ; les passkeys sont activées dans l'image (`KC_FEATURES=passkeys`).

### Thème de connexion

- *Realm settings* → *Themes* → *Login theme* : `bzhcamp`.
- Le thème est monté en volume (`demo/themes/bzhcamp`) avec le cache désactivé : une modification CSS / messages est visible en rechargeant la page.

### Inspecter un jeton

- *Clients* → client → *Client scopes* → *Evaluate* → choisir un utilisateur → *Generated access token* : montre le contenu exact du jeton sans se connecter.
- Ou bien décoder un jeton réel sur https://jwt.io.

### Sauvegarder la configuration

- *Realm settings* → menu *Action* (en haut à droite) → *Partial export* (clients, rôles, groupes ; les secrets sont masqués et les utilisateurs non inclus).
- Pour que la modification soit appliquée aux prochains démarrages, reporter les changements dans `demo/keycloak/import/realm-export.json`, puis `docker compose down -v && docker compose up -d --build`.

---

## 8. Environnements UAT / PREPROD (CI Jenkins)

Le `Jenkinsfile` injecte la configuration via des credentials Jenkins :

| Credential Jenkins               | Variable injectée        |
|----------------------------------|--------------------------|
| `keycloak-url-uat` / `keycloak-url-preprod`                       | `KEYCLOAK_ISSUER_URI = <url>/realms/bzhcamp` |
| `keycloak-client-secret-uat` / `keycloak-client-secret-preprod`   | `KEYCLOAK_CLIENT_SECRET` |
| `dockerhub-credentials`          | push des images          |
| `nvd-api-key`                    | OWASP Dependency Check   |

Procédure complète : [`ci/runbook-ci.md`](ci/runbook-ci.md).

---

## 9. Problèmes fréquents

| Symptôme | Cause probable / solution |
|----------|---------------------------|
| `Invalid parameter: redirect_uri` | Redirect URI du client ne correspond pas au port utilisé (§2) |
| `unauthorized_client` / `invalid_client_credentials` | `KEYCLOAK_CLIENT_SECRET` absent ou périmé (§4.2) |
| `Client not found` sur :8081 | Realm importé avant l'ajout de `myapp` → `docker compose down -v` (§4.3) |
| 401 sur `/planets` (API) | Jeton expiré ou `iss` ≠ `http://localhost:8080/realms/bzhcamp` |
| 403 sur `/planets` (API) | Utilisateur sans rôle realm `user` |
| 401 alors que le jeton est valide | `KEYCLOAK_AUDIENCE` défini sans *Audience mapper* (§4.4) |
| Modifs de l'export non prises en compte | Realm déjà en base → `docker compose down -v` puis relancer |

---

## Documentation complémentaire

- [`demo/app-springboot/README.md`](demo/app-springboot/README.md) — détails de l'app web Spring Boot
- [`demo/keycloak-angular/README.md`](demo/keycloak-angular/README.md) — exemple Angular
- [`ci/runbook-ci.md`](ci/runbook-ci.md) — pipeline DevSecOps
- [`ci/README.md`](ci/README.md) — tests Jenkins et déploiement UAT / PREPROD
