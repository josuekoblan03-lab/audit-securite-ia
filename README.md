# Audit de sécurité assisté par l'IA

Une **skill pour Antigravity** qui transforme l'agent en auditeur sécurité de **votre** application web : il cartographie le projet, cherche les failles (OWASP Top 10:2025, API Security Top 10), vérifie sans rien casser sur votre instance locale, corrige avec un test de non-régression, puis rédige `SECURITY_REPORT.md`.

Formation « Bien utiliser l'IA pour mieux coder ses applications web », module Sécurité, animé par Christ Yvan (Abidjan).

📘 Guide complet : [docs/Guide-Audit-Securite-IA.pdf](docs/Guide-Audit-Securite-IA.pdf)

---

## Installer en 1 message

Dans Antigravity, panneau de l'agent, collez :

```
Installe la skill d'audit de sécurité depuis
https://github.com/josuekoblan03-lab/audit-securite-ia
en suivant la section « Instructions pour l'agent » de son README.
```

Puis ouvrez une **nouvelle conversation** : la skill `audit-securite` apparaît dans **…** puis **Customizations**, onglet des skills. Elle est disponible dans tous vos projets. Pour la mettre à jour, renvoyez le même message.

## Utiliser

Ouvrez votre projet dans Antigravity et tapez :

```
/audit-securite
```

L'agent annonce ses règles d'engagement, vous demande comment lancer l'application en local et le mode souhaité :

| Mode | Contenu | Quand |
|---|---|---|
| **Rapide** | Cartographie, secrets et dépendances, 5 catégories critiques, failles Critiques et Hautes | Pendant le développement |
| **Complet** | Les 9 phases, toutes les catégories, toutes les sévérités | Avant chaque mise en production |

Lisez et validez ses demandes (installation d'outils, lancement de l'app, corrections). À la fin, lisez `SECURITY_REPORT.md` et relisez le diff git avant de fusionner.

## Règle d'or

On n'audite que **ce qui nous appartient** : votre code, en local ou sur votre environnement de test, avec des données de test. Jamais la production d'un tiers. Un audit par IA réduit fortement le risque mais ne remplace pas un audit professionnel.

## Contenu

```
SKILL.md                          protocole d'audit (9 phases)
resources/checklist-owasp-2025.md revue OWASP Top 10:2025 + fonctionnalités IA
resources/checklist-api.md        OWASP API Security Top 10:2023
resources/outils-par-stack.md     outils et pièges par technologie
resources/modele-rapport.md       modèle de SECURITY_REPORT.md
docs/Guide-Audit-Securite-IA.pdf  guide du participant
```

---

## Instructions pour l'agent

Tu es un agent IA à qui l'utilisateur demande d'installer cette skill. Fais exactement ceci :

1. **Destination** (installation globale, pour tous les projets de l'utilisateur) :
   - Windows : `%USERPROFILE%\.gemini\config\skills\audit-securite`
   - macOS / Linux : `~/.gemini/config/skills/audit-securite`
2. **Télécharge et copie uniquement `SKILL.md` et le dossier `resources/`** (pas `docs/`, pas `.git`), en remplaçant une version existante.

   Windows (PowerShell) :
   ```powershell
   $tmp  = Join-Path $env:TEMP "audit-securite-ia"
   $dest = Join-Path $env:USERPROFILE ".gemini\config\skills\audit-securite"
   if (Test-Path $tmp) { Remove-Item $tmp -Recurse -Force }
   git clone --depth 1 https://github.com/josuekoblan03-lab/audit-securite-ia.git $tmp
   if (Test-Path $dest) { Remove-Item $dest -Recurse -Force }
   New-Item -ItemType Directory -Force $dest | Out-Null
   Copy-Item (Join-Path $tmp "SKILL.md") $dest
   Copy-Item (Join-Path $tmp "resources") $dest -Recurse
   Remove-Item $tmp -Recurse -Force
   ```

   macOS / Linux :
   ```bash
   tmp="$(mktemp -d)"; dest="$HOME/.gemini/config/skills/audit-securite"
   git clone --depth 1 https://github.com/josuekoblan03-lab/audit-securite-ia.git "$tmp/repo"
   rm -rf "$dest" && mkdir -p "$dest"
   cp "$tmp/repo/SKILL.md" "$dest/" && cp -R "$tmp/repo/resources" "$dest/"
   rm -rf "$tmp"
   ```

   **Si git n'est pas installé**, télécharge l'archive `https://github.com/josuekoblan03-lab/audit-securite-ia/archive/refs/heads/main.zip`, extrais-la dans un dossier temporaire et copie les mêmes éléments (`SKILL.md`, `resources/`) depuis le sous-dossier `audit-securite-ia-main/`.
3. **Si l'écriture dans le dossier global est refusée**, installe à la place dans le projet ouvert : `<racine-du-projet>/.agents/skills/audit-securite/` (mêmes éléments), et préviens l'utilisateur que la skill ne sera disponible que dans ce projet.
4. **Vérifie** que `<destination>/SKILL.md` existe et que `<destination>/resources/` contient 4 fichiers `.md`. Affiche la liste.
5. **Conclus** en disant à l'utilisateur : « Skill installée. Ouvre une nouvelle conversation, puis tape `/audit-securite` dans le projet à auditer. » Ne lance pas l'audit toi-même.
