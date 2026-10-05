# AGENTS.md — gwine-cache

## Rôle du repo

Publie le **pack cache gwine pré-construit** : une release permanente `latest` contenant le cache
`~/.cache/gwine` complet (runner gwine, DXVK, DXVK-GPLAsync, VKD3D-Proton, DXVK-NVAPI, D7VK, Wine
Mono/Gecko, wincomponents) prêt à déployer offline. Consommateurs : `download_cache_bundle()`
(gablue, `src/gwine-launcher/lib/cache/offline.sh`) et le build ISO de gablue
(`installer/build.sh` télécharge les assets directement dans `/extra`).

**Ce repo ne contient aucun code du launcher** : uniquement le workflow `.github/workflows/update-cache.yml`
et ce README/AGENTS.md. Le gwine standalone est assemblé à la volée depuis `elgabo86/gablue`
(checkout du repo gablue dans le job, puis `src/gwine-launcher/build.sh`).

## Assets de la release `latest`

| Asset | Contenu |
|---|---|
| `gwine-cache.tar.xz` | Archive du cache complet (`components/` + `wincomponents/`), ~1,1 Go |
| `gwine-cache.tar.xz.sha256` | Checksum format `sha256sum -c` |
| `manifest.json` | Version de chaque composant (voir format ci-dessous) |
| `install-cache.sh` | Script de déploiement offline (généré par `gwine --cachepack`) |
| `README.txt` | Instructions de déploiement offline |

La release est créée avec `--latest=false` (le tag s'appelle `latest`, il ne faut pas qu'il
devienne aussi la release « la plus récente » au sens GitHub — les deux mécanismes se marchent
dessus sinon).

## Workflow `update-gwine-cache`

Déclencheurs : cron **dimanche 03:15 UTC** + `workflow_dispatch`. Concurrency group
`gwine-cache` (cancel-in-progress), `contents: write`, `ubuntu-26.04`, timeout 90 min.

Pipeline (chaque étape fail-fast) :

1. **Checkout gablue** (`elgabo86/gablue` → `./gablue`). Attention : le workspace n'est PAS un
   checkout de gwine-cache → toutes les commandes `gh` doivent passer `--repo elgabo86/gwine-cache`
   explicitement, sinon elles résolvent vers gablue.
2. **build.sh** → `gwine-standalone.sh`.
3. **`gwine-standalone.sh --download-components`** (retry x3, sleep 30 entre tentatives). Sur un
   runner frais : télécharge le runner, puis — cache quasi vide — déploie le **pack de la semaine
   précédente** (`download_cache_bundle`), puis re-télécharge en unitaire ce qui est plus récent
   que le pack.
4. **`gwine-standalone.sh --cachepack`** → `out/gwine-cache-installer/` + checks locaux
   (archive existe, taille > 100 Mo).
5. **Génération manifest + sha256** et comparaison avec le manifest de la release précédente :
   identique ET tous les assets attendus présents → output `changed=false`, publication skippée
   (évite de re-pousser ~1 Go pour rien).
6. **Publication** (si `changed=true`) : `gh release delete latest --cleanup-tag` puis re-création
   avec les 5 assets.

### Format `manifest.json`

Clés : `runner`, `dxvk`, `dxvk_async`, `vkd3d`, `nvapi`, `mono`, `gecko64`, `gecko32`
(noms sans extension, ex. `gwine-11.0.438320.20260922`) + `wincomponents` : tableau des chemins
relatifs `wincomponents/<composant>/<fichier>` (triés).

⚠️ **Le manifest ne doit contenir aucune donnée datée** (timestamp, date de build) : sinon la
comparaison `cmp` diverge à chaque run et ~1 Go est republié chaque semaine pour rien.

## Pièges connus

### Accumulation d'archives runner (fix sept. 2026)

Sur le runner CI, `--download-components` **sauvegarde d'abord le runner fraîchement téléchargé**
dans `components/gwine/<version>.tar.xz`, PUIS déploie le pack (qui contient déjà les archives des
semaines précédentes) par-dessus. Sans précaution, le cache accumule une archive par semaine.

Conséquence historique : `basename "$CACHE"/components/gwine/gwine-*.tar.xz` avec glob multi-match.
GNU coreutils ≥ 9.10 (ubuntu-26.04) : **2 arguments** → interprétés NAME/SUFFIX silencieusement
(résultat faux), **3+ arguments** → erreur et exit 1. Le `2>/dev/null` avale le message et
`set -euo pipefail` tue le step **sans aucune sortie** (le `cat manifest.json` n'apparaît jamais
dans le log) — c'est la signature du run rouge du 2026-09-27.

Règles :

- Côté workflow : toujours extraire le runner via `find … -maxdepth 1 -name 'gwine-*.tar.xz' |
  sort -V | tail -1 | xargs -r basename`. Jamais de glob nu dans `basename`.
- Côté gablue (fix `2d9b0c94`) : `create_cachepack()` prune les archives dans sa copie de staging
  (`sort -V | head -n -1`) → le pack publié ne contient qu'une seule archive. Le cache utilisateur
  n'est jamais touché.

### Liste des assets dupliquée

La liste des 5 assets attendus est codée en dur à deux endroits du workflow : boucle de vérification
du step manifest **et** arguments du step publish. Toute évolution du format de release (nouvel
asset) doit modifier les deux, sinon : nouvel asset jamais publié (boucle → `changed=true` à
l'infini) ou publication incomplète.

### Fenêtre d'indisponibilité à la republication

`delete latest --cleanup-tag` puis re-création : bref instant où les assets 404. Bénin — les
consommateurs basculent sur le téléchargement unitaire (fallback transparent), et le skip de
publication fait que ça n'arrive qu'une fois par semaine réellement changed.

### Validation mono/gecko codée en dur

`create_cachepack()` (gablue, `lib/cache/cachepack.sh`) vérifie la présence des fichiers exacts
`wine-mono-11.3.0-x86.msi`, `wine-gecko-2.47.4-x86_64.msi`, `wine-gecko-2.47.4-x86.msi`. Toute
montée de version Mono/Gecko côté gablue casse `--cachepack` (« Wine Mono/Gecko manquants ») tant
que les versions ne sont pas alignées — voir la checklist dans l'AGENTS.md du launcher gablue.

## Debug

- **Step « Generate manifest and checksum » mort en exit 1 sans log** → `set -e` silencieux,
  chercher une commande dans un pipeline dont l'erreur est avalée par `2>/dev/null` (le piège
  historique : `basename` multi-args). Reproduire la logique du manifest en local avec
  `CACHE="$HOME/.cache/gwine"` (le layout local est identique à celui du runner CI).
- **Publication inattendue chaque semaine** → comparer le manifest publié avec celui généré :
  une clé datée s'est glissée dans le JSON, ou un asset de la liste attendue manque à la release.
- **Pack jamais publié alors que des versions ont bougé** → vérifier que le step manifest voit
  bien les nouveautés (ex. `latest_dir` avec un pattern de dossier qui ne matche plus le nommage
  amont : `dxvk-*`, `vkd3d-proton-*`, `dxvk-nvapi-*`…).
- URL du pack surchargeable côté client via `GWINE_CACHE_BUNDLE_URL` (tests/miroirs).
