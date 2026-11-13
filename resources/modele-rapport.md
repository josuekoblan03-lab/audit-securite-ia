# Modèle : SECURITY_REPORT.md

Remplis ce modèle à la phase 8. Écris pour deux publics : le résumé pour un non-développeur, le détail pour un développeur.

```markdown
# Rapport d'audit de sécurité : <Nom de l'application>

- Date : AAAA-MM-JJ
- Version auditée : <commit git ou version>
- Mode : complet | rapide
- Auditeur : agent IA (Antigravity), sous la supervision de <nom>
- Périmètre : code du dépôt + instance locale (http://localhost:<port>)

## 1. Résumé exécutif
<5 lignes : niveau de risque global avant/après, nombre de failles par sévérité, failles les plus graves en langage simple, ce qui reste à faire.>

| Sévérité | Trouvées | Corrigées | Restantes |
|---|---|---|---|
| Critique | | | |
| Haute | | | |
| Moyenne | | | |
| Basse | | | |
| Info | | | |

## 2. Cartographie
<Stack, points d'entrée, authentification, données sensibles : résumé de audit/01-cartographie.md.>

## 3. Principaux scénarios de menace
<Les 5 scénarios d'abus de la phase 2 et leur statut (vérifié, faille trouvée, non applicable).>

## 4. Failles

### SEC-001 | Critique | A01 Contrôle d'accès défaillant
- **Où** : `src/app/api/orders/[id]/route.ts:12`
- **Problème** : ...
- **Scénario** : ...
- **Preuve** : ...
- **Correctif appliqué** : ... (commit `abc1234`)
- **Nouvelle vérification** : test `orders.access.test.ts` vert ; l'utilisateur A reçoit 404 sur la commande de B.
- **Statut** : Corrigée

(une section par faille)

## 5. Outils utilisés et résultats
| Outil | Version | Résultat | Fichier brut |
|---|---|---|---|

## 6. Limites de l'audit
<Ce qui n'a pas pu être vérifié et pourquoi (outil non installable, fonctionnalité non lançable en local, service tiers hors périmètre). Rappel : cet audit par IA ne remplace pas un audit professionnel.>

## 7. Actions qui restent à l'utilisateur
<Ex. régénérer la clé X exposée dans l'historique git chez le fournisseur ; activer la MFA du compte d'hébergement ; mettre à jour la dépendance Y en version majeure.>

## 8. Checklist avant mise en production
- [ ] Aucun secret dans le code ni dans l'historique git (gitleaks propre), clés exposées régénérées
- [ ] `.env` dans `.gitignore`, `.env.example` à jour sans valeurs
- [ ] Aucune dépendance avec vulnérabilité Critique ou Haute
- [ ] Mode debug désactivé, erreurs génériques côté client
- [ ] Contrôle d'accès vérifié côté serveur sur chaque route et action (test A contre B)
- [ ] Politiques d'accès de la base actives (RLS, règles Firebase) si la base est accessible depuis le client
- [ ] Mots de passe hachés avec Argon2id ou bcrypt
- [ ] Limite de débit sur connexion, inscription et réinitialisation
- [ ] Cookies de session `HttpOnly`, `Secure`, `SameSite`
- [ ] En-têtes de sécurité présents (CSP, HSTS, nosniff, Referrer-Policy)
- [ ] HTTPS partout
- [ ] Validation des entrées côté serveur sur tous les endpoints
- [ ] Webhooks : signature vérifiée
- [ ] Événements de sécurité journalisés, sans données sensibles
- [ ] Sauvegarde de la base et procédure de restauration testée

## 9. Recommandations
1. ...
2. ...
3. ...
```
