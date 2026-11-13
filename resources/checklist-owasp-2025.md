# Checklist de revue : OWASP Top 10:2025

Pour chaque catégorie : **Signaux** (ce qu'il faut chercher dans le code), **Vérifier** (comment confirmer sans rien casser), **Corriger** (le bon correctif).

---

## A01:2025 Contrôle d'accès défaillant (Broken Access Control)

Catégorie n°1. Inclut désormais le SSRF.

**Signaux**
- Route ou action qui reçoit un ID (`/api/orders/:id`, `?userId=`, `params.id`) et charge la ressource **sans filtrer par l'utilisateur connecté** (`findById(id)` au lieu de `findOne({ id, ownerId: user.id })`).
- Vérification de rôle faite seulement côté client (bouton masqué, route front protégée) mais pas côté serveur.
- `userId`, `role`, `isAdmin`, `ownerId` lus depuis le corps de la requête ou un champ caché au lieu de la session.
- Middleware d'authentification utilisé comme **seul** contrôle (il peut être contourné, voir Next.js dans `outils-par-stack.md`).
- Base de données accessible depuis le client (Supabase, Firebase) sans règles d'accès par ligne ou par document.
- CORS `Access-Control-Allow-Origin: *` combiné à des cookies ou à des données privées.
- **SSRF** : le serveur appelle une URL fournie par l'utilisateur (aperçu de lien, import par URL, webhook configurable, téléchargement d'image distante).
- Chemins de fichiers construits avec une entrée utilisateur (`path.join(dir, req.query.file)`), ce qui permet de remonter l'arborescence.

**Vérifier**
- Avec deux comptes de test A et B : A demande, modifie, supprime une ressource de B. Attendu : 403 ou 404.
- Compte non admin sur chaque route admin. Attendu : 403.
- SSRF : test unitaire de la fonction de validation d'URL, qui doit refuser `localhost`, `127.0.0.1`, `0.0.0.0`, `169.254.169.254`, `10.x`, `172.16-31.x`, `192.168.x`, `[::1]` et les schémas autres que `https`.

**Corriger**
- Refuser par défaut. Toujours dériver l'identité de la **session côté serveur**, jamais du client.
- Filtrer chaque requête par propriétaire (`WHERE id = ? AND owner_id = ?`), ou utiliser une couche de politiques centralisée.
- Activer les politiques d'accès de la base (RLS Supabase, règles Firebase) sur **toutes** les tables et collections.
- SSRF : liste blanche de domaines, résolution DNS puis blocage des IP privées, pas de suivi automatique des redirections.
- Fichiers : n'accepter qu'un nom (pas un chemin), le normaliser et vérifier qu'il reste dans le dossier autorisé.

---

## A02:2025 Mauvaise configuration de sécurité (Security Misconfiguration)

**Signaux**
- Mode debug actif hors développement (`DEBUG=True`, `APP_DEBUG=true`, `NODE_ENV` absent en production).
- Pages ou routes de démo, de test, d'administration par défaut laissées accessibles (`/admin`, `/phpinfo.php`, `/swagger` public, `/.env` servi).
- Identifiants par défaut, comptes de démo.
- En-têtes de sécurité absents.
- Cookies de session sans `HttpOnly`, `Secure`, `SameSite`.
- Listage de répertoires activé, en-tête `X-Powered-By` ou version du serveur exposés.
- Bucket ou stockage de fichiers public par erreur.
- Conteneur qui tourne en root, ports internes exposés.

**Vérifier**
- `curl -sI http://localhost:<port>` et comparer aux en-têtes attendus :
  - `Content-Security-Policy` (au minimum `default-src 'self'`, `frame-ancestors 'none'` ou `'self'`)
  - `Strict-Transport-Security` (en production HTTPS)
  - `X-Content-Type-Options: nosniff`
  - `Referrer-Policy: strict-origin-when-cross-origin`
  - `Permissions-Policy` (désactiver caméra, micro, géolocalisation si inutiles)
- Inspecter les attributs du cookie de session.
- Tenter d'accéder à `/.env`, `/.git/config`, `/swagger`, `/admin` sur l'instance locale. Attendu : 404 ou accès refusé.

**Corriger**
- Configuration de production séparée, debug désactivé, erreurs génériques.
- Ajouter les en-têtes (middleware `helmet` pour Express, `headers()` dans la config Next.js, middleware Laravel ou Django).
- Cookies : `HttpOnly; Secure; SameSite=Lax` (ou `Strict`).
- Supprimer routes et comptes de démo, masquer les versions.

---

## A03:2025 Défaillances de la chaîne d'approvisionnement logicielle (Software Supply Chain Failures)

Nouvelle catégorie. Couvre les dépendances, le build et la distribution.

**Signaux**
- Dépendances avec vulnérabilités connues (sortie des outils de la phase 3).
- Pas de fichier de verrouillage (`package-lock.json`, `pnpm-lock.yaml`, `poetry.lock`, `composer.lock`) ou fichier non commité.
- Versions non figées (`"*"`, `"latest"`).
- Paquets inconnus, peu téléchargés, ou dont le nom ressemble à un paquet populaire (typosquatting). **Attention aux paquets suggérés par une IA qui n'existent pas** : un attaquant peut publier ce nom.
- Scripts `postinstall` exécutés par des dépendances.
- Scripts chargés depuis un CDN sans attribut `integrity` (SRI).
- CI/CD : secrets en clair dans les workflows, actions tierces non épinglées par leur hash de commit.

**Vérifier**
- Outils d'audit de dépendances (phase 3).
- Pour chaque dépendance directe inconnue : existe-t-elle vraiment sur le registre officiel, est-elle maintenue, combien de téléchargements ?

**Corriger**
- Mettre à jour les dépendances vulnérables (mineures et correctifs directement, majeures après accord).
- Commiter le fichier de verrouillage, installer en CI avec `npm ci` (ou l'équivalent).
- Supprimer les dépendances inutilisées.
- Ajouter `integrity` et `crossorigin` aux scripts tiers, épingler les actions CI par hash.
- Activer Dependabot ou Renovate.

---

## A04:2025 Défaillances cryptographiques (Cryptographic Failures)

**Signaux**
- Mots de passe stockés en clair, en MD5, en SHA-1 ou en SHA-256 simple (sans algorithme lent et sans sel).
- Secrets, clés d'API, chaînes de connexion **écrits dans le code** ou commités.
- Clé secrète exposée au client (`NEXT_PUBLIC_*SECRET*`, clé `service_role` Supabase dans le front, clé Stripe secrète côté client).
- Jetons générés avec `Math.random()`, `rand()`, `random.random()`.
- Secret JWT faible ou par défaut (`secret`, `changeme`), algorithme `none` accepté.
- Données sensibles transmises en HTTP, ou stockées sans chiffrement quand c'est nécessaire.
- Données sensibles dans `localStorage` (jetons de session).

**Vérifier**
- Recherche dans le code : `md5(`, `sha1(`, `createHash('sha`, `Math.random`, `password` à proximité d'un stockage.
- Lire le JavaScript livré au navigateur (onglet Sources) à la recherche de clés.

**Corriger**
- Mots de passe : **Argon2id** (recommandé) ou **bcrypt** (coût ≥ 12) via une bibliothèque reconnue.
- Jetons : générateur cryptographique (`crypto.randomBytes`, `secrets.token_urlsafe`, `random_bytes`).
- Secrets dans des variables d'environnement, `.env` dans `.gitignore`, `.env.example` sans valeurs.
- **Toute clé ayant été commitée est compromise : la régénérer chez le fournisseur**, même après suppression du fichier (elle reste dans l'historique git).
- JWT : secret long et aléatoire, algorithme fixé côté vérification, expiration courte.
- HTTPS partout en production.

---

## A05:2025 Injection

Couvre SQL, NoSQL, commandes système, LDAP, templates, et le XSS (injection dans le HTML).

**Signaux**
- Requête construite par concaténation ou interpolation : `"SELECT ... WHERE id = " + id`, `` `... ${req.body.x}` ``, `f"SELECT ... {x}"`, `"... '$var'"`.
- Méthodes « raw » des ORM avec interpolation (`$queryRawUnsafe`, `whereRaw`, `.raw()`, `extra()`).
- Objet de requête NoSQL construit directement depuis le corps de la requête (`find(req.body)`), qui permet d'injecter des opérateurs.
- Commandes système avec entrée utilisateur : `exec`, `system`, `shell_exec`, `os.system`, `subprocess(..., shell=True)`, `child_process.exec`.
- `eval`, `new Function`, `unserialize`, `pickle.loads` sur une donnée externe.
- **XSS** : `innerHTML`, `outerHTML`, `document.write`, `dangerouslySetInnerHTML`, `v-html`, `{!! !!}` (Blade), `|safe` (Django/Jinja), `html_safe`, Markdown rendu sans nettoyage.
- Liens `href` construits depuis une entrée sans vérifier le schéma (risque `javascript:`).

**Vérifier**
- Revue de code des endroits listés ci-dessus : l'entrée utilisateur atteint-elle l'interpréteur sans paramétrage ou échappement ?
- Test inoffensif : un nom contenant une apostrophe (`O'Brien`) doit s'enregistrer sans erreur ; un commentaire contenant `<b>test</b>` doit s'afficher **littéralement**, sans mise en gras.

**Corriger**
- Requêtes **paramétrées** ou ORM, toujours. Jamais de concaténation.
- Valider le type et la forme de chaque entrée côté serveur (Zod, Joi, Pydantic, règles de validation Laravel, etc.).
- Éviter le shell : appeler le programme avec une liste d'arguments, sans `shell=True`, et une liste blanche de valeurs.
- XSS : laisser le framework échapper (React, Vue, Blade `{{ }}`, Django échappent par défaut) ; si du HTML riche est nécessaire, le nettoyer avec **DOMPurify** ou un équivalent ; ajouter une CSP.

---

## A06:2025 Conception non sécurisée (Insecure Design)

Failles de logique métier : le code fait exactement ce qui a été conçu, mais la conception est mauvaise.

**Signaux**
- Prix, remise, total, quantité ou rôle **envoyés par le client** et acceptés par le serveur.
- Étapes d'un processus que l'on peut sauter (payer → confirmer : peut-on appeler « confirmer » directement ?).
- Quantités négatives ou nulles acceptées, coupons réutilisables à l'infini.
- Réinitialisation de mot de passe par question secrète ou lien sans expiration.
- Pas de limite sur les actions coûteuses (envoi d'emails ou de SMS, appels à une API d'IA payante, création de comptes).
- Situation de concurrence (race condition) : deux requêtes simultanées qui utilisent deux fois le même solde ou le même coupon.

**Vérifier**
- Relire les 5 scénarios d'abus de la phase 2 et vérifier chacun dans le code.
- Test avec des valeurs limites inoffensives (quantité 0, -1) sur l'instance locale.

**Corriger**
- Le serveur recalcule tout ce qui a de la valeur (prix depuis la base, rôle depuis la session).
- Machine à états côté serveur pour les processus en plusieurs étapes.
- Transactions et verrous en base pour les opérations sur un solde ou un stock.
- Quotas et limites de débit sur les actions coûteuses.

---

## A07:2025 Défaillances d'authentification (Authentication Failures)

**Signaux**
- Aucune limite de tentatives de connexion.
- Messages différents pour « utilisateur inconnu » et « mot de passe incorrect » (énumération des comptes).
- Politique de mot de passe trop faible (moins de 8 caractères) ou mots de passe compromis acceptés.
- Session non régénérée après connexion, non invalidée après déconnexion ou changement de mot de passe.
- Jeton de réinitialisation prévisible, réutilisable ou sans expiration.
- JWT sans expiration, stocké dans `localStorage`.
- Pas de double authentification (MFA) possible pour les comptes administrateurs.

**Vérifier**
- 10 échecs de connexion au maximum : y a-t-il un blocage, un ralentissement, un captcha ?
- Comparer les messages d'erreur pour un email inexistant et un mauvais mot de passe.
- Se déconnecter puis réutiliser l'ancien cookie ou jeton. Attendu : refus.

**Corriger**
- Utiliser l'authentification du framework ou d'un fournisseur reconnu plutôt qu'un système maison.
- Limite de débit sur connexion, inscription et réinitialisation.
- Message unique : « Email ou mot de passe incorrect ».
- Mots de passe de 8 caractères minimum (12 recommandés), refus des mots de passe compromis (vérification Have I Been Pwned par k-anonymat).
- Régénérer la session à la connexion, l'invalider à la déconnexion, expiration raisonnable.
- MFA au moins pour les administrateurs.

---

## A08:2025 Défaillances d'intégrité des logiciels et des données (Software or Data Integrity Failures)

**Signaux**
- Désérialisation de données non fiables (`unserialize`, `pickle`, `yaml.load` non sécurisé, `ObjectInputStream`).
- Webhooks (paiement, etc.) traités **sans vérifier la signature**.
- Mise à jour automatique ou plugin téléchargé sans vérification d'intégrité.
- Données critiques stockées côté client (cookie non signé contenant le rôle, panier avec prix).
- Mass assignment : `Model.create(req.body)`, `update(**request.data)`, `$fillable` trop large, qui permet de modifier `role` ou `isAdmin`.

**Vérifier**
- Lire chaque gestionnaire de webhook : la signature est-elle vérifiée avant tout traitement ?
- Lire chaque création ou mise à jour de modèle : quels champs sont acceptés ?

**Corriger**
- Vérifier les signatures des webhooks avec la méthode officielle du fournisseur (ex. `stripe.webhooks.constructEvent`).
- Liste blanche explicite des champs modifiables (DTO, schéma de validation, `$fillable` précis, `fields` explicites).
- Formats sûrs (JSON) au lieu de la sérialisation native, `yaml.safe_load`.
- Cookies signés ou données gardées côté serveur.

---

## A09:2025 Défaillances de journalisation et d'alerte (Security Logging and Alerting Failures)

**Signaux**
- Aucune trace des connexions, échecs de connexion, changements de mot de passe ou de rôle, actions d'administration.
- **Données sensibles dans les journaux** : mots de passe, jetons, numéros de carte, corps de requête complets.
- Journaux qui acceptent des retours à la ligne venant de l'utilisateur (falsification de lignes de journal).
- Aucune alerte en cas d'activité anormale.

**Vérifier**
- Rechercher les appels de journalisation (`console.log`, `logger.`, `print(`, `Log::`) et leur contenu.
- Déclencher une connexion échouée en local : est-elle tracée, sans le mot de passe ?

**Corriger**
- Journaliser les événements de sécurité (qui, quoi, quand, depuis quelle IP, résultat) dans un format structuré.
- Ne jamais journaliser de secrets ni de données personnelles inutiles.
- Encoder ou filtrer les entrées utilisateur dans les journaux.
- Mettre en place une alerte simple (email, outil de suivi comme Sentry) sur les pics d'échecs ou les actions admin.

---

## A10:2025 Mauvaise gestion des cas exceptionnels (Mishandling of Exceptional Conditions)

Nouvelle catégorie : comportement de l'application quand quelque chose se passe mal.

**Signaux**
- **Fail-open** : en cas d'erreur, le contrôle laisse passer (`try { verifier() } catch { /* on continue */ }`, autorisation qui renvoie `true` par défaut).
- `catch` vides, erreurs avalées sans traitement.
- Traces de pile, requêtes SQL ou chemins internes renvoyés au client.
- Processus en plusieurs étapes sans annulation (rollback) en cas d'échec au milieu (débit fait, commande non créée).
- Absence de délai maximum (timeout) sur les appels externes, absence de limite de taille sur les requêtes et les fichiers envoyés.
- Promesses non gérées, exceptions non capturées qui font tomber le serveur.

**Vérifier**
- Rechercher `catch` vides, `except: pass`, `catch (e) {}`, `@` (suppression d'erreurs PHP).
- Requête mal formée en local : la réponse ne doit révéler aucun détail interne.

**Corriger**
- **Fail-closed** : en cas de doute ou d'erreur, refuser.
- Gestionnaire d'erreurs global qui renvoie un message générique au client et journalise le détail côté serveur.
- Transactions de base de données pour les opérations en plusieurs étapes.
- Timeouts, limites de taille du corps de requête et des fichiers, validation des types.

---

## Section spéciale : fonctionnalités IA de l'application (OWASP Top 10 pour les applications LLM, 2025)

À appliquer seulement si l'application envoie des données à un modèle de langage.

**Signaux**
- Clé d'API du fournisseur d'IA utilisée côté client.
- Entrée utilisateur insérée directement dans le prompt système, sans séparation.
- Réponse du modèle rendue en HTML sans nettoyage, ou exécutée (SQL, code, commandes).
- Le modèle a accès à des outils ou à des données au-delà de ce dont l'utilisateur a le droit (fuite entre utilisateurs via la recherche documentaire RAG).
- Aucun quota par utilisateur sur les appels au modèle (coût illimité).
- Secrets ou données personnelles placés dans le prompt système.

**Corriger**
- Appels au modèle uniquement côté serveur.
- Traiter la sortie du modèle comme une **entrée non fiable** : validation, échappement, jamais d'exécution directe.
- Les outils et documents accessibles au modèle sont filtrés selon les droits de l'utilisateur connecté, côté serveur.
- Quotas et limites de débit par utilisateur, plafond de dépense chez le fournisseur.
- Pas de secrets dans les prompts.
