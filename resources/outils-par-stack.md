# Outils et pièges par technologie

Choisis la méthode d'installation selon le système de l'utilisateur (Windows : `winget`, `scoop`, `pip`, Docker ou WSL ; macOS : `brew` ; Linux : gestionnaire de paquets, `pip`, Docker). **Demande l'accord avant toute installation.** Si un outil ne s'installe pas, passe au suivant et continue la revue manuelle.

Toutes les sorties vont dans `audit/raw/`.

---

## 1. Outils universels (toutes stacks)

| Objectif | Outil | Commande type |
|---|---|---|
| Secrets dans le code et l'historique git | **gitleaks** | `gitleaks git --redact -v .` (versions anciennes : `gitleaks detect --redact -v`) |
| Secrets dans les fichiers non commités | gitleaks | `gitleaks dir --redact -v .` |
| Dépendances, secrets, configuration | **Trivy** | `trivy fs --scanners vuln,secret,misconfig .` |
| Dépendances (base OSV) | **OSV-Scanner** | `osv-scanner scan source -r .` |
| Analyse statique multi-langages | **Semgrep** | `semgrep scan --config p/owasp-top-ten --config p/secrets --metrics=off .` |
| Scan dynamique passif (app locale) | **OWASP ZAP baseline** | voir ci-dessous |
| En-têtes HTTP | curl | `curl -sI http://localhost:<port>` |

Équivalents Docker (si l'outil ne s'installe pas) :
```
docker run --rm -v "${PWD}:/repo" zricethezav/gitleaks:latest git /repo --redact -v
docker run --rm -v "${PWD}:/src" semgrep/semgrep semgrep scan --config p/owasp-top-ten --metrics=off /src
docker run --rm -v "${PWD}:/repo" aquasec/trivy fs --scanners vuln,secret,misconfig /repo
```

**OWASP ZAP baseline** (passif, ne modifie rien, **localhost uniquement**) :
```
docker run --rm -t -v "${PWD}/audit/raw:/zap/wrk:rw" ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py -t http://host.docker.internal:<port> -r zap-baseline.html
```
Sous Linux sans Docker Desktop, ajouter `--add-host=host.docker.internal:host-gateway`.

---

## 2. Par langage

| Langage | Dépendances | Analyse statique |
|---|---|---|
| JavaScript / TypeScript | `npm audit --omit=dev` · `pnpm audit --prod` · `yarn npm audit` (Yarn 2+) ou `yarn audit` (Yarn 1) | Semgrep, `eslint-plugin-security` |
| Python | `pip-audit` | `bandit -r . -x ./venv,./.venv,./tests` |
| PHP | `composer audit` | Semgrep (règles PHP) |
| Java / Kotlin | OWASP Dependency-Check (`mvn org.owasp:dependency-check-maven:check` ou plugin Gradle) | SpotBugs + FindSecBugs, Semgrep |
| Go | `govulncheck ./...` | `gosec ./...` |
| Ruby | `bundle-audit check --update` | `brakeman` (Rails) |
| C# / .NET | `dotnet list package --vulnerable --include-transitive` | Semgrep, analyseurs Roslyn de sécurité |
| Dart / Flutter | OSV-Scanner (lit `pubspec.lock`) | Semgrep |

---

## 3. Pièges propres aux frameworks (à vérifier en phase 4)

### Next.js / React
- **Les server actions et les route handlers sont des endpoints publics** : chacun doit vérifier la session et les droits lui-même.
- Le middleware ne doit **jamais** être le seul contrôle d'accès (exemple réel : CVE-2025-29927, contournement du middleware par un en-tête, corrigé en 15.2.3, 14.2.25, 13.5.9 et 12.3.5). Revérifier dans la page, l'action ou la route.
- Garder React et Next.js à jour : des failles critiques ont touché les React Server Components (ex. CVE-2025-55182, décembre 2025). Lancer l'audit des dépendances.
- Toute variable `NEXT_PUBLIC_*` est **visible par tout le monde** : jamais de secret dedans.
- `dangerouslySetInnerHTML` seulement avec DOMPurify.
- En-têtes de sécurité via `headers()` dans `next.config.*` ou le middleware.

### Vite / React / Vue / Angular (SPA)
- Les variables `VITE_*`, `REACT_APP_*` sont intégrées au JavaScript livré : **publiques**.
- Tout contrôle d'accès fait dans le front est contournable : il doit exister côté API.
- Ne pas stocker de jeton de session dans `localStorage` si un cookie `HttpOnly` est possible.

### Supabase
- **RLS activée sur toutes les tables** du schéma `public` (sinon la clé `anon`, publique, donne accès à tout).
- Des politiques précises par opération (`select`, `insert`, `update`, `delete`) basées sur `auth.uid()`. Une table avec RLS activée mais une politique `using (true)` reste ouverte.
- La clé `service_role` ne doit **jamais** quitter le serveur.
- Politiques sur les buckets de **Storage**.
- Fonctions SQL `security definer` exposées en RPC : vérifier qu'elles contrôlent l'appelant.
- Utiliser le **Security Advisor** du tableau de bord Supabase.

### Firebase
- Règles Firestore, Realtime Database et Storage : jamais `allow read, write: if true;`, ni les règles du « mode test » avec une date d'expiration.
- Les règles vérifient `request.auth.uid` et la propriété du document.
- La configuration web Firebase (apiKey) est publique par conception : la sécurité repose sur les règles et App Check.

### Node.js / Express / Fastify
- `helmet` pour les en-têtes, `app.disable('x-powered-by')`.
- `cors` avec une liste d'origines explicite.
- `express-rate-limit` (ou `@fastify/rate-limit`) sur l'authentification.
- `express.json({ limit: '100kb' })` : taille de corps limitée.
- Gestionnaire d'erreurs global qui ne renvoie pas `err.stack`.

### Laravel
- `.env` : `APP_DEBUG=false`, `APP_ENV=production`, `APP_KEY` défini.
- Modèles : `$fillable` précis (éviter `$guarded = []`).
- Blade : `{{ }}` échappe, `{!! !!}` n'échappe pas.
- `DB::raw`, `whereRaw` : toujours avec des liaisons de paramètres.
- Policies et Gates pour l'autorisation, middleware `auth` sur les routes.
- `php artisan route:list` pour l'inventaire des routes.

### Django
- `python manage.py check --deploy` : liste les réglages de production manquants.
- `DEBUG = False`, `SECRET_KEY` dans l'environnement, `ALLOWED_HOSTS` précis.
- Éviter `|safe`, `mark_safe`, `raw()` et `extra()` avec une entrée utilisateur.
- `@login_required` et contrôle de propriété dans chaque vue.

### Flask / FastAPI
- Flask : `debug=False`, `SECRET_KEY` fort, Jinja2 autoescape, `flask-talisman`, `flask-limiter`.
- FastAPI : dépendances `Depends()` pour l'authentification sur chaque route privée, modèles Pydantic de sortie (`response_model`) pour ne pas exposer de champs internes.

### PHP natif
- PDO avec `prepare()` et paramètres liés.
- `password_hash()` et `password_verify()`.
- `htmlspecialchars($x, ENT_QUOTES, 'UTF-8')` à l'affichage.
- `session_regenerate_id(true)` à la connexion, cookies `httponly`, `secure` et `samesite`.
- `display_errors = Off` en production.

### Spring Boot
- Spring Security configuré avec refus par défaut, `@PreAuthorize` sur les méthodes sensibles.
- Endpoints Actuator non exposés publiquement.
- Requêtes JPA paramétrées, pas de concaténation dans `@Query` natif.

### Docker et déploiement
- Image qui ne tourne pas en root (`USER` dans le Dockerfile).
- Aucun secret dans l'image ni dans le Dockerfile (utiliser les variables d'environnement de la plateforme).
- `.dockerignore` qui exclut `.env` et `.git`.
- Ports de base de données non exposés publiquement.
