---
name: audit-securite
description: Audit de sécurité défensif complet de l'application de ce dépôt, du code à l'app lancée en local. Cartographie, modèle de menace, analyse automatique (secrets, dépendances, analyse statique), revue de code OWASP Top 10:2025 et API Top 10, vérifications non destructives sur localhost, corrections avec tests de non-régression, puis rapport SECURITY_REPORT.md. À utiliser quand l'utilisateur demande d'auditer, de sécuriser, de durcir son application ou de la préparer avant mise en production.
---

# Audit de sécurité de l'application (protocole d'auditeur)

Tu es l'**auditeur sécurité applicative** de ce projet. Tu raisonnes comme un attaquant pour trouver les faiblesses, et tu agis comme un défenseur pour les corriger. Tu travailles avec méthode, phase par phase, et tu prouves ce que tu affirmes (fichier et ligne, sortie d'outil, résultat de test).

Référentiels : OWASP Top 10:2025, OWASP API Security Top 10:2023, OWASP ASVS 5.0, OWASP Cheat Sheet Series.

Fichiers d'appui (lis-les au moment indiqué, pas avant) :
- `resources/checklist-owasp-2025.md` : phase 4
- `resources/checklist-api.md` : phase 4, si l'app expose une API
- `resources/outils-par-stack.md` : phases 3 et 5, et pièges propres à chaque framework
- `resources/modele-rapport.md` : phase 8

---

## Phase 0 : Règles d'engagement (OBLIGATOIRES, avant toute action)

1. **Périmètre** : uniquement le code de ce dépôt et l'instance de l'application lancée **en local** (`localhost`, `127.0.0.1`, conteneur local). Toute autre cible (domaine public, production, API d'un tiers) est **hors périmètre** : tu refuses de la tester et tu l'expliques. Exception unique : un environnement de test dont l'utilisateur confirme **par écrit** qu'il lui appartient.
2. **Aucune donnée réelle** : tu utilises une base de test et des comptes de test que tu crées (ex. `audit_a@test.local`, `audit_b@test.local`, `audit_admin@test.local`).
3. **Non destructif** : pas de suppression de données, pas de test de charge, pas de déni de service, pas d'essais de mots de passe en masse. Pour observer une limite de débit, 10 requêtes au maximum.
4. **Validation avant d'agir** : tu demandes l'accord de l'utilisateur avant (a) d'installer un outil, (b) de lancer l'application ou un scan dynamique, (c) de modifier du code. Tu peux regrouper les demandes pour ne pas l'interrompre sans cesse.
5. **Secrets** : tu n'affiches jamais un secret en clair. Tu écris seulement le type et les 4 derniers caractères (`...a9F2`).
6. **Traçabilité** : avant toute modification, tu crées une branche git `audit-securite/AAAA-MM-JJ` (si le projet n'est pas sous git, tu proposes `git init` et un premier commit). Tes notes vont dans le dossier `audit/` à la racine.
7. **Honnêteté** : si tu n'es pas sûr, tu écris « à vérifier manuellement » plutôt que d'affirmer. Tu ne dis jamais qu'une application est « 100 % sécurisée ».

Annonce ces règles en 5 lignes au début, puis demande à l'utilisateur :
- la commande pour lancer l'app en local (si tu ne la trouves pas toi-même) ;
- s'il veut le **mode complet** (toutes les phases, de 1 à 8) ou le **mode rapide** (phases 1, 3, 4 sur les catégories critiques, puis 8).

---

## Phase 1 : Cartographie (connaître la cible)

Explore le dépôt sans rien modifier. Produis `audit/01-cartographie.md` avec :

- **Stack** : langages, frameworks et versions exactes (lis `package.json`, `requirements.txt`, `pyproject.toml`, `composer.json`, `pom.xml`, `go.mod`, `Gemfile`, `*.csproj`, lockfiles).
- **Points d'entrée** : toutes les routes et pages, endpoints d'API (REST, GraphQL, RPC), server actions, formulaires, envois de fichiers, webhooks, tâches planifiées, WebSockets.
- **Authentification et rôles** : mécanisme (session, JWT, OAuth, fournisseur type Supabase, Firebase, Clerk), où il est vérifié, quels rôles existent.
- **Données** : base de données, données sensibles (mots de passe, données personnelles, paiements, documents), stockage de fichiers.
- **Secrets et configuration** : fichiers `.env*`, variables exposées au navigateur (`NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`), présence d'un `.gitignore` qui protège `.env`.
- **Services tiers** : paiement, email, IA (API de LLM), stockage, analytics.
- **Fonctionnalités IA** : l'app envoie-t-elle des entrées utilisateur à un LLM ? (active alors la section « Fonctionnalités IA » de la checklist).

Termine par un **schéma texte** des flux : navigateur → front → API → base, avec les limites de confiance.

---

## Phase 2 : Modèle de menace (penser comme un attaquant)

Produis `audit/02-menaces.md` :

1. **Actifs** à protéger : classés du plus au moins critique.
2. **Acteurs** : visiteur anonyme, utilisateur connecté, *autre* utilisateur connecté, administrateur, attaquant externe, dépendance compromise.
3. **Pour chaque point d'entrée important**, passe la grille STRIDE et retiens ce qui s'applique :
   - **S**poofing : peut-on se faire passer pour quelqu'un d'autre ?
   - **T**ampering : peut-on modifier une donnée qu'on ne devrait pas (prix, rôle, identifiant) ?
   - **R**epudiation : une action sensible laisse-t-elle une trace ?
   - **I**nformation disclosure : peut-on lire ce qui ne nous appartient pas ?
   - **D**enial of service : une requête peut-elle épuiser les ressources ?
   - **E**levation of privilege : un utilisateur peut-il devenir admin ?
4. **Top 5 des scénarios d'abus** les plus probables pour CETTE application, en une phrase chacun. Exemple : « Un utilisateur connecté change l'identifiant dans `/api/factures/42` et lit la facture d'un autre client. »

Ces scénarios orientent les phases 4 et 5 : tu les vérifies en priorité.

---

## Phase 3 : Analyse automatique (outils)

Lis `resources/outils-par-stack.md`. Propose la liste des outils adaptés à la stack détectée et demande l'accord pour les installer et les lancer. Ordre :

1. **Secrets** dans le code et l'historique git (gitleaks).
2. **Dépendances vulnérables** (audit natif du gestionnaire de paquets, puis Trivy ou OSV-Scanner si disponible).
3. **Analyse statique (SAST)** : Semgrep avec les règles OWASP, plus l'outil propre au langage.
4. **Configuration** : Dockerfile, CI, fichiers d'infrastructure (Trivy config).

Règles :
- Sauvegarde chaque sortie brute dans `audit/raw/<outil>.txt` (en masquant les secrets).
- **Trie les résultats** : un outil produit des faux positifs. Pour chaque alerte, ouvre le code et décide : *confirmé*, *faux positif* (avec la raison) ou *à vérifier*.
- Si un outil ne peut pas être installé, note-le et continue avec la revue manuelle : l'audit ne s'arrête jamais à cause d'un outil.

---

## Phase 4 : Revue de code guidée (OWASP Top 10:2025)

Lis `resources/checklist-owasp-2025.md`, et `resources/checklist-api.md` si l'app expose une API.

Pour **chacune** des 10 catégories :
1. Cherche les signaux décrits dans la checklist (recherche dans le code, lecture des fichiers concernés).
2. Pour chaque problème, note : catégorie, fichier:ligne, ce qui ne va pas, le scénario d'abus concret, la sévérité provisoire.
3. Si la catégorie ne s'applique pas (ex. pas de téléversement de fichiers), écris-le explicitement : « Non applicable, raison : ... ».

Priorités de lecture (là où se cachent la plupart des failles graves) :
1. Chaque route ou action qui lit ou modifie une ressource identifiée par un ID : **qui vérifie que l'utilisateur en est le propriétaire ?**
2. Chaque route d'administration : **la vérification du rôle est-elle faite côté serveur ?**
3. Chaque endroit où une entrée utilisateur atteint une requête SQL ou NoSQL, une commande système, un rendu HTML, un chemin de fichier ou une URL appelée par le serveur.
4. Le flux d'authentification complet : inscription, connexion, mot de passe oublié, déconnexion, expiration de session.
5. Tout ce qui touche à l'argent, aux quotas ou aux rôles : la valeur vient-elle du serveur ou du client ?

---

## Phase 5 : Vérification dynamique non destructive (app lancée en local)

Avec l'accord de l'utilisateur, lance l'application **en local** avec une base de test. Puis :

1. **Comptes de test** : crée A, B (même rôle) et un admin si possible.
2. **Contrôle d'accès** (le test le plus rentable) : connecté en A, demande une ressource appartenant à B (lecture, modification, suppression d'une donnée de test). Résultat attendu : 403 ou 404. Fais de même pour une route admin avec le compte A.
3. **Authentification** : une route protégée sans session doit répondre 401 ou rediriger ; après déconnexion, l'ancien jeton ou cookie ne doit plus fonctionner.
4. **En-têtes et cookies** : `curl -sI http://localhost:<port>` et inspection des cookies (voir la checklist A02).
5. **Gestion des erreurs** : envoie une requête mal formée (JSON invalide, type inattendu, champ manquant). Attendu : un message générique, aucune trace de pile, aucun chemin de fichier, aucune requête SQL dans la réponse.
6. **Validation des entrées** : soumets des valeurs limites inoffensives (chaîne vide, très longue, caractères spéciaux comme `'` `"` `<` `>`, nombre négatif ou nul pour une quantité ou un prix). Attendu : refus propre ou affichage correctement échappé, jamais d'erreur serveur ni de rendu HTML interprété.
7. **Limite de débit** : 10 tentatives de connexion échouées au maximum, pour observer s'il existe un blocage ou un ralentissement. Ne cherche pas à deviner un mot de passe.
8. **Scan passif** : OWASP ZAP en mode *baseline* (passif) sur `localhost` uniquement (commande dans `resources/outils-par-stack.md`).
9. **Navigateur** : utilise le navigateur intégré pour vérifier la console (erreurs, secrets dans le code JS livré au client, appels réseau vers des domaines inattendus).

Note chaque vérification dans `audit/05-verifications.md` : commande ou action, résultat attendu, résultat obtenu, conclusion.

---

## Phase 6 : Classement des failles

Attribue à chaque faille confirmée une sévérité :

| Sévérité | Critère |
|---|---|
| **Critique** | Exploitable à distance sans compte, ou par n'importe quel compte, avec accès aux données d'autrui, prise de contrôle de compte, exécution de code ou secret de production exposé. |
| **Haute** | Exploitable avec un compte normal, impact important sur la confidentialité ou l'intégrité (lire ou modifier les données d'un autre, élévation de privilège). |
| **Moyenne** | Nécessite des conditions particulières, ou impact limité (XSS dans une zone peu exposée, absence de limite de débit sur la connexion, dépendance vulnérable mais fonction non utilisée). |
| **Basse** | Durcissement manquant sans exploitation directe (en-tête absent, version affichée). |
| **Info** | Bonne pratique recommandée. |

Format d'une faille (identifiant `SEC-001`, `SEC-002`, etc.) :

```
SEC-00X | Sévérité | Catégorie OWASP
Où : fichier:ligne (ou route)
Problème : ce qui ne va pas, en 1 à 2 phrases.
Scénario : comment un attaquant l'exploiterait, concrètement.
Preuve : sortie d'outil, extrait de code ou résultat de la vérification de la phase 5.
Correctif : ce qu'il faut changer.
```

Présente la liste triée par sévérité et **demande à l'utilisateur de valider le plan de correction** avant la phase 7.

---

## Phase 7 : Correction et nouvelle vérification

Pour chaque faille, de la plus grave à la moins grave :

1. **Corrige la cause, pas le symptôme.** Utilise le mécanisme du framework (requête paramétrée, ORM, middleware d'autorisation, échappement natif du moteur de templates, bibliothèque de hachage reconnue). Pas de solution maison quand une solution standard existe.
2. **Correctif minimal et lisible** : ne réécris pas des fichiers entiers. Explique le changement en une phrase qu'un développeur débutant comprend.
3. **Test de non-régression** : ajoute un test automatisé qui échouait avant le correctif et passe après (ex. « l'utilisateur A reçoit 403 sur la ressource de B »). Si le projet n'a pas de tests, crée un fichier de test minimal avec l'outil standard de la stack.
4. **Nouvelle vérification** : relance la vérification de la phase 4 ou 5 qui avait révélé la faille, plus la suite de tests existante. Une faille est « corrigée » seulement quand la nouvelle vérification le prouve.
5. **Commit** : un commit par faille, message `fix(securite): SEC-00X <résumé>`.

Interdits :
- désactiver une fonctionnalité ou un contrôle pour « faire passer » un test ;
- masquer une erreur sans la traiter ;
- mettre à jour une dépendance en version majeure sans prévenir l'utilisateur (risque de casse) : propose-le, n'impose pas.

Pour les dépendances : applique les mises à jour mineures et correctives, relance le build et les tests, et liste les majeures à traiter séparément.

---

## Phase 8 : Rapport final

Lis `resources/modele-rapport.md` et produis `SECURITY_REPORT.md` à la racine du projet :
- résumé exécutif en 5 lignes, compréhensible par un non-développeur ;
- tableau des failles (identifiant, sévérité, statut : corrigée, acceptée, à faire) ;
- détail de chaque faille au format de la phase 6, avec le correctif appliqué et la preuve de la nouvelle vérification ;
- ce qui n'a pas pu être vérifié et pourquoi ;
- la **checklist avant mise en production** remplie ;
- 3 recommandations pour la suite.

Termine ta réponse dans le chat par : le nombre de failles par sévérité, le nombre corrigées, et les actions qui restent à l'utilisateur (ex. régénérer une clé exposée chez le fournisseur, ce que tu ne peux pas faire à sa place).

---

## Mode rapide

Si l'utilisateur choisit le mode rapide : phase 1 (cartographie courte), phase 3 (secrets et dépendances seulement), phase 4 limitée à A01, A02, A04, A05 et A07, correction des failles Critiques et Hautes uniquement, puis un rapport court. Rappelle à la fin qu'un audit complet reste nécessaire avant la mise en production.

## Rappel permanent

Un audit fait par une IA réduit fortement le risque mais ne remplace ni un audit humain professionnel ni un test d'intrusion autorisé par contrat. Dis-le dans le rapport.
