# Deployment

Bridge Buddies is a static Unity WebGL build. There is no bundler, no package
manager and no server code: `scripts/prepare-site.mjs` reshapes the export and
GitHub Actions publishes the result.

## Where it lives

| Piece | Value |
| --- | --- |
| Repository | `jd4rider/bridge-buddies-web` (public) |
| Play URL | <https://jd4rider.github.io/bridge-buddies-web/play/> |
| Cover page | <https://jd4rider.github.io/bridge-buddies-web/> |
| Host | GitHub Pages, Actions source (`pages.yml`), no custom domain |
| Catalogue id | `bridge-buddies` in `littlejoyarcade/src/data/games.ts` and `public.games` |
| Portal entry | <https://littlejoyarcade.site/#/game/bridge-buddies> |

The portal embeds the `play/` URL in an iframe, so the game has to keep working
inside a nested browsing context: no custom domain, no service worker, no
`X-Frame-Options`.

## Why no custom domain

GitHub gives a Pages site exactly one canonical hostname. A `public/CNAME` or a
Pages custom domain would make `jd4rider.github.io/bridge-buddies-web/` answer
with a 301 to a hostname whose certificate may not exist yet, and the arcade's
embedded player would show a black screen. Game repos therefore stay on
`github.io`; the portal links that URL directly. See `../littlejoyarcade/DEPLOYMENT.md`.

`bridgebuddies.littlejoyarcade.site` is still pointed at the old Babylon Pages
site and is not used by the catalogue entry.

## Deploy

```bash
git push origin main
gh run list --repo jd4rider/bridge-buddies-web --limit 3
```

Pages must have its source set to **GitHub Actions** once:

```bash
gh api -X POST repos/jd4rider/bridge-buddies-web/pages -f build_type=workflow
```

## Verify a deploy

```bash
curl -sI https://jd4rider.github.io/bridge-buddies-web/play/               # 200 text/html
curl -sI https://jd4rider.github.io/bridge-buddies-web/play/Build/WebGL.wasm
curl -s -r 0-3 https://jd4rider.github.io/bridge-buddies-web/play/Build/WebGL.wasm | xxd -p
# expect 0061736d (\0asm) - a Brotli or gzip payload would start with something else
```

A 200 is not proof the game runs: the payload is 61 MB, so also open the URL in a
browser and wait for the progress bar to disappear.

## Publishing to the catalogue

The portal reads its catalogue from the `public.games` table and falls back to
`src/data/games.ts`, so both have to change:

```bash
cd ../littlejoyarcade
# 1. update the bridge-buddies entry in src/data/games.ts
# 2. add supabase/migrations/00NN_bridge_buddies_unity.sql
# 3. write the row the live site actually renders
KEY=$(npx supabase projects api-keys --project-ref fqovramltegcfiyebyrt \
  | awk -F'|' '/service_role/{gsub(/[[:space:]]/,"",$2); print $2; exit}')
SUPABASE_SERVICE_ROLE_KEY="$KEY" node scripts/publish-game-rest.mjs \
  ../bridge-buddies-web/arcade-manifest.json
# 4. push, which rebuilds the portal itself
git add -A && git commit -m "Point Bridge Buddies at the Unity build" && git push
```

## Payload notes

- 61 MB total, 47.8 MB of it `WebGL.wasm`. GitHub Pages allows 1 GB per site and
  100 MB per file, so the build fits, but the first load is slow on mobile.
- Pages gzips the wasm in transit; the payload is stored uncompressed because a
  Brotli export would need `Content-Encoding: br`, which Pages cannot send.
- To cut the download, re-export with **Publishing Settings → Compression Format
  → Brotli** and the script will expand it at build time.

## Rollback

Pages keeps the last deploy only. To roll back, revert the commit that changed
`build/WebGL/` and push:

```bash
git revert <sha> && git push
```
