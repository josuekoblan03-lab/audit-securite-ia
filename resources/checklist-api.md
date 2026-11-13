# Checklist API : OWASP API Security Top 10:2023

À utiliser si l'application expose une API (REST, GraphQL, routes `/api`, server actions, fonctions serverless).

Commence par **inventorier tous les endpoints** : méthode, chemin, authentification requise, rôle requis, ressource manipulée. Mets ce tableau dans `audit/01-cartographie.md`.

| # | Risque | Question à se poser pour chaque endpoint | Bon correctif |
|---|---|---|---|
| API1 | **Autorisation au niveau objet défaillante (BOLA)** | L'endpoint reçoit un ID : vérifie-t-il que l'objet appartient à l'utilisateur connecté ? | Filtre par propriétaire dans chaque requête, ou politique centralisée. Test A contre B (phase 5). |
| API2 | **Authentification défaillante** | Endpoints sans authentification qui devraient en avoir ? Jetons sans expiration, non vérifiés, acceptés sans signature ? | Middleware d'authentification sur toutes les routes privées, vérification complète du jeton (signature, expiration, audience). |
| API3 | **Autorisation au niveau propriété défaillante** | La réponse renvoie-t-elle des champs inutiles (hash de mot de passe, email d'autres utilisateurs, champs internes) ? Le client peut-il modifier des champs protégés (`role`, `balance`, `verified`) ? | Schéma de sortie explicite (DTO, `select` précis) et liste blanche des champs modifiables. |
| API4 | **Consommation de ressources illimitée** | Pagination sans maximum ? Taille de requête ou de fichier illimitée ? Pas de limite de débit ? Requêtes GraphQL imbriquées sans limite de profondeur ? | `limit` plafonné, taille maximale du corps, limite de débit par IP et par utilisateur, profondeur et complexité GraphQL plafonnées. |
| API5 | **Autorisation au niveau fonction défaillante** | Un utilisateur normal peut-il appeler une fonction d'admin (changer de méthode HTTP, deviner `/api/admin/...`) ? | Vérification du rôle côté serveur sur chaque fonction privilégiée, refus par défaut. |
| API6 | **Accès illimité aux flux métier sensibles** | Un script peut-il automatiser en masse un flux sensible (création de comptes, réservation, achat, envoi de messages) ? | Limites de débit métier, captcha sur les flux exposés, détection d'automatisation. |
| API7 | **SSRF** | L'API appelle-t-elle une URL fournie par l'utilisateur ? | Liste blanche, blocage des IP internes et des métadonnées cloud (voir A01). |
| API8 | **Mauvaise configuration** | CORS trop ouvert ? Messages d'erreur détaillés ? Méthodes HTTP inutiles activées ? Documentation Swagger publique en production ? | CORS limité aux origines connues, erreurs génériques, documentation protégée ou retirée en production. |
| API9 | **Mauvaise gestion de l'inventaire** | D'anciennes versions (`/api/v1`) ou des endpoints de test (`/api/debug`, `/api/seed`) restent-ils actifs ? | Supprimer ou protéger les anciens endpoints et ceux de test, tenir l'inventaire à jour. |
| API10 | **Consommation non sûre d'API tierces** | Les réponses d'API externes sont-elles validées avant usage ? Appels en HTTPS ? Timeouts ? | Valider le schéma des réponses tierces, HTTPS, timeouts, ne pas suivre aveuglément les redirections. |

## Spécifique GraphQL
- Introspection désactivée en production.
- Limite de profondeur et de complexité des requêtes.
- Autorisation vérifiée dans **chaque resolver**, pas seulement à l'entrée.

## Spécifique server actions et fonctions serverless
- Chaque server action (Next.js) ou fonction exposée est un **endpoint public**. Elle doit elle-même vérifier la session et les droits, même si la page qui l'appelle est protégée.
- Valider les arguments de chaque action avec un schéma (Zod ou équivalent).
