# AGENTS.md

## Présentation
Polyfin est un serveur auto-hébergé qui parle l'API Jellyfin, en servant du contenu provenant d'addons Stremio/Eclipse et d'IPTV. Backend Go + application d'admin React (`/admin/`) + jellyfin-web embarqué (`/web/`).

Ce dépôt est un fork (`origin` = kesurof/polyfin). Le chemin de module Go reste `github.com/moodiness/polyfin` ; ne réécris pas les imports.

## Fork de travail
- `origin` : votre fork. `upstream` : `moodiness/polyfin`, en lecture seule (push désactivé).
- Synchroniser : `git fetch upstream`, puis `git switch main && git merge upstream/main` (ou rebase), puis `git push origin main`.
- `AGENTS.md` est propre au fork : il vit sur `main` et ne doit jamais partir dans une PR upstream.
- Développer depuis `main` pour que l'agent lise `AGENTS.md` : `git switch -c feat/... main`.
- Pour une PR upstream, créer une branche propre depuis `upstream/main` et n'y cherry-picker que les commits de code :
  ```sh
  git switch -c pr/<nom> upstream/main
  git cherry-pick <sha-code>...
  git push origin pr/<nom>
  ```
- Un hook local `.git/hooks/pre-push` refuse de pousser toute branche autre que `main` dont le diff avec `upstream/main` contient `AGENTS.md`.
- Ne jamais pousser sur `upstream`.

## Outillage
- Go 1.27.1 (`go.mod`), Node 24, PostgreSQL 18, FFmpeg/ffprobe 9.
- Lint : uniquement `gofmt` + `go vet` pour Go, Prettier pour `web/`. Pas de golangci-lint.

## Commandes (l'ordre compte)
- `make web` d'abord : construit `web/dist`. Les builds de production l'embarquent (`//go:build production`) ; les autres le lisent sur disque à l'exécution.
- `make dev` : `make web` puis `go run ./cmd/polyfin serve` sur `:8096`. Nécessite `POLYFIN_DATABASE_URL`.
  - Rechargement à chaud de l'admin : `npm --prefix web run dev` (proxy `/admin/api`, `/healthz`, `/readyz` vers `:8096`).
- `make lint` : vérification Prettier + `gofmt -l cmd internal web/*.go` + `go vet ./...`.
- `make test` : `go test -race ./...`.
- `make test-jellyfin` / `make test-others` : la séparation utilisée par la CI ; `internal/jellyfin` est de loin le paquet le plus lent.
- `make bounds` : test de performance IPTV, volontairement lancé sans `-race`.
- `make check` : `web lint test bounds` — la validation locale complète.
- Un seul test : `go test -race ./internal/<pkg> -run 'TestName'`.
- Typecheck web : `npm --prefix web run typecheck`.
- Site de docs : `npm --prefix docs ci && npm --prefix docs run build` (échoue sur les liens internes cassés).
- Changements Compose/Docker : `make compose-check` ; contrôles d'image : `make image-check`, `make image-health`.

## Tests ignorés silencieusement
La couverture d'intégration ne s'exécute que si ces variables sont définies :
- `POLYFIN_TEST_DATABASE_URL` : PostgreSQL jetable dont le rôle peut créer des bases.
- `POLYFIN_TEST_FFMPEG` : ffmpeg avec `ffprobe` à côté (tests de remux).
- Aussi utilisées par la CI : `POLYFIN_TEST_PG_DUMP`, `POLYFIN_TEST_PG_RESTORE`, `POLYFIN_TEST_TESSERACT`, `POLYFIN_TEST_TORRENT_INTERFACE`, `POLYFIN_TEST_SMB`.
Un run vert sans elles prouve bien moins qu'il n'y paraît. Précise quelles variables tu as définies.

## Migrations
- `internal/database/migrations/NNNN_snake_case.sql`, embarquées et appliquées dans l'ordre, chacune dans sa propre transaction.
- Ne modifie jamais une migration déjà appliquée : les sommes de contrôle sont stockées et Polyfin refuse de démarrer en cas d'écart.
- Les fichiers doivent être en LF uniquement (test CI `TestMigrationsEndTheirLinesWithLF` ; `.gitattributes` utilise `* -text`).

## i18n
- Exactement six langues, dans cet ordre : `en, fr, de, es, it, pt-BR` (`internal/i18n.Languages`) ; `i18n.Text` est un `[6]string`.
- Chaque texte visible par l'utilisateur doit exister dans les six : littéraux Go `i18n.Text`/map indexées par langue, et `web/src/i18n/<lang>/<section>.ts`. Sinon `i18n.TestTexts` échoue.

## Organisation
- `cmd/polyfin` : point d'entrée du serveur (`serve`, `healthcheck`, `version`) ; le câblage des dépendances vit dans `main.go` et fait foi.
- `cmd/polyfin-desktop` : application de barre système.
- `internal/server` : mux de premier niveau (`/admin/`, `/web/`, `/api/v2/` Silo, `/dlna/`, `/share/`, `/metrics`, `/healthz`, `/readyz`, API Jellyfin sur `/`).
- `internal/jellyfin` : API Jellyfin ; `internal/admin` : API d'admin.
- `internal/<feature>` : un paquet par domaine (library, iptv, playback, localfiles, torrent, subtitles, ...).
- `web/` : application d'admin React 19 + Vite + TypeScript ; `docs/` : documentation VitePress, à garder synchronisée avec le comportement.
- `third_party/go-smb2` : vendorisé via une directive `replace`.

## Conventions
- Les titres de PR suivent Conventional Commits ; les PR sont squash-mergées.
- Les commentaires expliquent le *pourquoi*, pas le *quoi* ; respecte le style existant.
- Ne commite jamais `.env`, des identifiants, des URLs de manifest d'addon ou des liens vers du contenu.
