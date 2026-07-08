# Plan de remédiation sécurité

## Contexte
- Repo audité en lecture seule : serveur Python MCP `src/`, cache SQLite, client YNAB, site Astro `website/`, workflow PyPI, dépendances Python/npm.
- Le serveur MCP fonctionne en `stdio` et utilise un `YNAB_API_KEY` long-vivant avec droits YNAB complets.
- Aucune source ou documentation non-plan n'a été modifiée pendant l'audit.

## Constats prioritaires
1. `High` : outils MCP d'écriture/destruction exposés par défaut (`create_*`, `update_*`, `delete_*`, `import_transactions`) sans garde serveur contre prompt injection ou appels accidentels.
2. `High` : cache SQLite financier en clair, persistant, sans permissions explicites ni nettoyage de rétention.
3. `High` : `npm audit --package-lock-only --audit-level=high` signale 10 vulnérabilités, dont Astro/Vite/esbuild/defu/devalue/picomatch en sévérité haute.
4. `High` : workflow de publication PyPI utilise des actions GitHub à tags mutables alors que le job a `id-token: write`.
5. `Medium` : cache non suffisamment lié à l'identité d'authentification ; risque de données obsolètes après révocation/changement de token.
6. `Medium` : redactions de champs sensibles contournables via `exclude_fields=[]` et incomplètes pour les modèles imbriqués.
7. `Medium` : paramètres d'URL YNAB interpolés dans les chemins sans validation/encodage strict.
8. `Medium` : site sans CSP/headers de sécurité, et docs qui encouragent stockage plaintext du token + installateurs shell distants.

## Décisions de remédiation
- Prioriser la sécurité de l'utilisateur final : appliquer un mode lecture seule par défaut et exiger une activation explicite des écritures.
- Préserver l'API existante autant que possible : garder les noms d'outils, mais bloquer côté serveur quand les écritures sont désactivées.
- Ne pas supprimer le cache : le durcir d'abord via permissions, scoping, rétention et purge ; rendre l'opt-out possible si simple.
- Traiter le site comme statique public : corriger les dépendances et ajouter headers de défense en profondeur.
- Corriger CI/publish sans introduire de secrets statiques.

## Plan d'implémentation

### 1. Mettre les écritures MCP derrière un garde explicite
- Ajouter dans `src/config.py` une option booléenne, par exemple `enable_writes: bool = False` avec alias env `YNAB_MCP_ENABLE_WRITES` si nécessaire.
- Ajouter dans `src/server/_shared.py` un helper/decorator `require_writes_enabled` qui retourne une erreur JSON claire quand `settings.enable_writes` est faux.
- Appliquer ce garde à tous les outils mutateurs :
  - `src/server/accounts.py`: `create_account`.
  - `src/server/transactions.py`: `create_transaction`, `create_transactions`, `update_transaction`, `update_transactions`, `delete_transaction`, `import_transactions`.
  - `src/server/categories.py`: `create_category`, `update_category`, `create_category_group`, `update_category_group`, `update_category_for_month`.
  - `src/server/payees.py`: `update_payee`.
  - `src/server/scheduled.py`: `create_scheduled_transaction`, `update_scheduled_transaction`, `delete_scheduled_transaction`.
- Documenter dans les réponses d'erreur que l'utilisateur doit activer explicitement les écritures dans son environnement local.
- Ajouter tests serveur couvrant au minimum un outil d'écriture bloqué par défaut et autorisé quand `enable_writes=True`.

### 2. Durcir le cache SQLite et l'isolation des données
- Dans `src/config.py` / `src/db/engine.py`, créer le dossier cache avec permissions `0700` sur POSIX et appliquer `0600` au fichier DB après création quand possible.
- Prendre en compte les fichiers SQLite WAL/SHM (`cache.db-wal`, `cache.db-shm`) dans la stratégie de permissions.
- Ajouter un namespace de cache dérivé du token, par exemple HMAC/SHA-256 non réversible de `YNAB_API_KEY`, et l'inclure dans les clés/tables de cache :
  - `ResponseCache.cache_key` pour les clés globales `user`, `plans`, `plan:*`.
  - `CachedEntity` / `ServerKnowledge`, idéalement via une colonne `auth_scope` incluse dans les contraintes uniques.
- Ajouter une migration destructive acceptable pour les tables de cache si le schéma change, puisque les données se reconstruisent automatiquement.
- Modifier le fallback delta pour ne jamais servir le cache après erreurs `401`/`403`; fallback uniquement pour réseau, `429`, `5xx`.
- Ajouter une fonction de purge/retention interne ou un outil MCP `clear_cache` seulement si explicitement voulu ; sinon documenter une commande de suppression locale dans la doc de sécurité.

### 3. Valider et encoder les entrées envoyées à l'API YNAB
- Centraliser dans `src/ynab_client.py` des helpers :
  - validation d'ID YNAB (`^[A-Za-z0-9_-]+$` ou format réel confirmé par tests/fixtures) ;
  - validation de dates `YYYY-MM-DD` et mois `current` ou premier jour du mois ;
  - validation d'énums (`type`, `cleared`, `flag_color`, account types).
- Encoder chaque segment de chemin avec `urllib.parse.quote(value, safe="")` au lieu d'interpoler directement les strings.
- Continuer à passer les query params via `params=` uniquement.
- Ajouter limites de taille : nombre max d'éléments dans bulk create/update, longueur max des `memo`, `name`, `query`, `import_id`, nombre max de subtransactions.
- Retourner des erreurs JSON stables pour validation utilisateur au lieu de laisser `KeyError`/`ValidationError` remonter.

### 4. Revoir la redaction et les champs sensibles
- Décider que les champs sensibles ne sont pas entièrement contrôlables par le client MCP par défaut.
- Remplacer ou compléter `exclude_fields` par une politique allowlist/safe profile :
  - défaut : champs nécessaires à l'usage courant seulement ;
  - mode avancé explicite via config, pas via simple argument d'outil, pour exposer `import_id`, payee original, transfer ids, géolocalisation, etc.
- Appliquer les excludes récursivement à tous les modèles imbriqués dans `PlanDetail`, pas seulement `CategoryGroup` et `MonthDetail`.
- Reconsidérer `PayeeLocation.latitude` / `longitude` comme sensibles ; les exclure par défaut ou déplacer derrière mode avancé.
- Ajouter tests de serialization pour `get_plan`/modèles imbriqués et `exclude_fields=[]`.

### 5. Corriger les dépendances et la supply chain
- Dans `website/`, lancer `npm audit fix` ou mettre à jour explicitement Astro/Vite et régénérer `package-lock.json`.
- Valider que `npm audit --package-lock-only --audit-level=high` ne signale plus de vulnérabilités hautes.
- Mettre à jour les dépendances Python transitives vulnérables via `uv lock` / upgrade `mcp` si disponible ; sinon ajouter contraintes compatibles pour `pyjwt >=2.13.0` et Starlette corrigé si le graphe MCP le permet.
- Remplacer `mcp[cli]` par `mcp` dans `pyproject.toml` si les fonctionnalités CLI ne sont pas nécessaires au runtime package.
- Ajouter bornes hautes raisonnables ou contraintes compatibles aux dépendances runtime et au build backend `hatchling`.
- Ajouter une configuration Dependabot/Renovate couvrant `uv`, npm et GitHub Actions.

### 6. Sécuriser le workflow de publication
- Dans `.github/workflows/publish.yml`, épingler les actions à des SHAs complets : `actions/checkout`, `astral-sh/setup-uv`, `pypa/gh-action-pypi-publish`.
- Ajouter permissions minimales : top-level `permissions: contents: read`, `id-token: write` uniquement dans le job publish.
- Utiliser `uv sync --locked` ou équivalent pour les tests et `uv build --locked`/mode verrouillé si supporté.
- Éviter `ubuntu-latest` si la reproductibilité est prioritaire ; préférer `ubuntu-24.04`.
- Option plus robuste : construire une seule fois, tester/installer l'artefact construit, puis publier exactement cet artefact.

### 7. Ajouter headers de sécurité au site et ajuster les docs sensibles
- Ajouter une configuration Vercel ou hosting headers pour :
  - `Content-Security-Policy` adaptée au site statique, Vercel Analytics et Google Fonts ou fonts self-hosted ;
  - `Strict-Transport-Security` ;
  - `X-Content-Type-Options: nosniff` ;
  - `Referrer-Policy` ;
  - `Permissions-Policy` ;
  - `frame-ancestors 'none'`.
- Traiter les scripts inline Astro (`Layout.astro`, `CopyButton.astro`, `setup.astro`) avant CSP stricte : nonce/hash ou déplacement vers assets externes.
- Mettre à jour les snippets README/site pour avertir que le token YNAB est stocké en clair dans les configs MCP et peut apparaître dans l'historique shell.
- Recommander permissions restrictives des fichiers config, rotation/révocation du token, et wrapper local lisant `YNAB_API_KEY` depuis un environnement protégé.
- Remplacer ou compléter `curl | sh` / `irm | iex` par des instructions vérifiables : téléchargement, inspection, checksum/signature si disponible, lien docs officiel.

## Validation attendue
- Python : `uv run pytest` après les changements serveur/cache.
- Python lock : `uv lock --check` ou équivalent ; si `uv` indisponible localement, le signaler.
- Frontend build : `cd website && npm run build`.
- Frontend audit : `cd website && npm audit --package-lock-only --audit-level=high` doit passer sans vulnérabilité haute.
- Security scans si outils disponibles : `uv run pip-audit`, `osv-scanner --lockfile uv.lock --lockfile website/package-lock.json`.
- Tests ciblés à ajouter : écriture bloquée par défaut, fallback cache interdit sur `401/403`, permissions cache, validation de path params, redaction récursive.

## Notes d'audit déjà vérifiées
- `git status --short` était propre après audit.
- Aucun fichier `.env*`, clé privée, DB/cache ou certificat n'a été trouvé dans le repo via glob ciblé.
- Le grep secret n'a trouvé que des faux positifs npm lockfile.
- `uv` n'était pas installé dans l'environnement d'audit ; `python3` existe mais `pip_audit` et `osv-scanner` ne sont pas installés.
- `npm audit --package-lock-only --audit-level=high` a été exécuté dans `website/` et a échoué avec vulnérabilités hautes.

## Critères d'acceptation
- Les outils mutateurs ne peuvent pas modifier YNAB tant qu'une option explicite locale n'est pas activée.
- Le cache local ne mélange pas les données entre tokens/utilisateurs et ne sert pas de données après révocation/auth failure.
- Les champs sensibles restent masqués par défaut, y compris dans les réponses imbriquées.
- Les IDs et dates malformés sont refusés avant appel HTTP.
- Les audits npm/Python disponibles ne rapportent pas de vulnérabilités hautes connues.
- Le workflow PyPI n'utilise plus de tags d'actions mutables avec `id-token: write`.
- Le site public a des headers de sécurité et la documentation ne banalise plus le stockage plaintext du token.

## Hors périmètre pour cette passe
- Implémentation d'un chiffrement complet multi-plateforme du cache si elle nécessite dépendances natives ou UX de gestion de clé complexe.
- Refonte complète des noms/outils MCP.
- Création de secrets manager intégré ; privilégier documentation/wrapper local.
