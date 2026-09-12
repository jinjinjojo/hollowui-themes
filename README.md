# hollowui-themes

Canonical theme registry for **HollowUI Studio** and **hollowui.com**.

The Studio app and the marketing site fetch `manifest.json` from this repo at
boot to check for new / updated themes. If the manifest's `version` matches the
client's cached copy, nothing is downloaded — the client uses its embedded
themes as-is. Otherwise it fetches only the theme files whose `hash` has
changed.

Starting with **schema v2** (2026-09-12), individual theme files may also carry
an embedded animated backdrop — SVG markup plus scoped CSS — so a brand-new
theme can be added to the repo and ship a working animation to every client
without any code change to the app or the site.

## Structure

```
manifest.json         # index of all themes (small, cached, ETag-friendly)
themes/<key>.json     # one file per theme with metadata + optional backdrop
```

### `manifest.json`

```json
{
  "version": "2026.09.12",
  "generated_at": "2026-09-12T00:00:00Z",
  "schema": 2,
  "themes": {
    "matrix": {
      "url": "themes/matrix.json",
      "hash": "sha256:...",
      "accent": "#00ff91",
      "name": "Matrix"
    },
    "aether": {
      "url": "themes/aether.json",
      "hash": "sha256:...",
      "accent": "#f5d382",
      "name": "Aether",
      "hasEmbeddedBackdrop": true
    }
  }
}
```

`hasEmbeddedBackdrop: true` tells the client the theme JSON ships its own
`backdrop.svg` + `backdrop.css` and the client should inject them at merge
time. Themes without that flag rely on the built-in vendor bundle
(`hui-themes.js`) for their backdrop factory — same behaviour as v1.

### `themes/<key>.json` — v2 schema

```json
{
  "key": "aether",
  "name": "Aether",
  "description": "Pale gold aurora with wispy vertical mist drifting on velvet dark.",
  "version": "1.0.0",
  "accent": { "r": 245, "g": 211, "b": 130, "hex": "#f5d382" },
  "tags": ["dark", "gold", "aurora", "calm"],
  "author": "hollowui",
  "created": "2026-09-12",
  "updated": "2026-09-12",
  "hasEmbeddedBackdrop": true,
  "backdrop": {
    "svg": "<svg viewBox='0 0 1000 1000' preserveAspectRatio='xMidYMid slice' class='hb-tb-layer hb-tb-aether'>...</svg>",
    "css": ".hb-tb-aether{background:radial-gradient(...)}.hb-tb-aether .wisp{animation:hb-tb-aether-drift 14s ease-in-out infinite}@keyframes hb-tb-aether-drift{0%,100%{transform:translateY(-40px)}50%{transform:translateY(40px)}}"
  }
}
```

Rules for the `backdrop` block:

1. **`backdrop.svg`** — one full SVG string. Must set
   `viewBox="0 0 1000 1000"` and `preserveAspectRatio="xMidYMid slice"` so the
   site engine can slot it into `#site-theme-backdrop` and cover any viewport.
   Root class must include both `hb-tb-layer` (generic hook) and
   `hb-tb-<key>` (theme-scoped hook).
2. **`backdrop.css`** — one string of CSS. Every selector must be scoped under
   `.hb-tb-<key>` so it never leaks into other themes. All keyframes must be
   named `hb-tb-<key>-<label>` for the same reason.
3. Animation is either SMIL (`<animate>`, `<animateMotion>`, `<animateTransform>`
   inside the SVG) or CSS keyframes in the `backdrop.css` string. Both are fine;
   mix as needed.
4. Include a `@media (prefers-reduced-motion:reduce)` rule that disables the
   animation for accessibility.
5. The `hasEmbeddedBackdrop: true` field must be present when `backdrop` is
   present. The manifest index also mirrors the flag so clients can decide
   without fetching the theme JSON.

Themes without a `backdrop` block (the original 22) are still valid — the
site / Studio bundle already contains their factory in `hui-themes.js`.

## Adding a new theme

1. Author `themes/<key>.json` with the fields above. Keep the JSON keys sorted
   alphabetically so diffs stay small.
2. Include a full working `backdrop.svg` (300–800 characters is the sweet spot
   for cache and paint budget; hard cap 8 KB) and a matching `backdrop.css`.
3. From the repo root, compute the SHA-256 of the file:

   ```powershell
   Get-FileHash -Algorithm SHA256 themes/<key>.json
   ```

4. Add the theme to `manifest.json` under `themes.<key>` with `accent`, `name`,
   `url`, `hash`, and (if it ships a backdrop) `hasEmbeddedBackdrop: true`.
5. Bump `manifest.version` to today's date (`YYYY.MM.DD`) and update
   `generated_at`.
6. Commit and push. Clients pick it up on their next boot inside the check
   interval.

## Update protocol (v2)

1. Client boots and reads `.huisite.manifest` from local cache
   (`localStorage` on the site, on-disk cache for Studio).
2. Client GETs
   `https://raw.githubusercontent.com/jinjinjojo/hollowui-themes/main/manifest.json`
   with `If-None-Match` carrying the last-known ETag. GitHub returns
   `304 Not Modified` when nothing changed → zero body payload.
3. On a fresh manifest, the client compares each `themes[key].hash` against
   its cached copy. Any hash mismatch or new key triggers a fetch of the
   matching `themes/<key>.json`.
4. For each theme that comes back with a `backdrop` block:
   * Its `backdrop.css` is injected into `<head>` as
     `<style id="hui-theme-css-<key>">` (replaced in place on re-fetch so no
     dupes ever pile up).
   * Its `backdrop.svg` is stored on `window.SiteTheme.customBackdrops[key]`.
5. When the user selects a theme, the site engine (`site-theme.js`) checks
   `window.SiteTheme.customBackdrops[key]` first; if present, it mounts that
   SVG directly. If absent, it delegates to the built-in factory in
   `hui-themes.js` (v1 path).
6. Accent RGB + name always merges into `window.SiteTheme.accents[key]` and
   the picker (`THEME_ORDER`) is extended so the new theme shows up in the UI
   without a reload.

### Bandwidth & caching

* `manifest.json` — always small (< 8 KB), fetched once per hour per client,
  ETag'd so unchanged fetches are 304 with an empty body.
* `themes/<key>.json` — fetched only when its hash changes or the key is new.
  Cached in `localStorage` under `huisite.themes.data`. The store is capped
  at ~256 KB total; oldest entries are evicted first when a fetch would push
  the store over the cap (see `themes-sync.js`).
* Injected `<style>` and in-memory SVG map — replaced in place on every
  matching sync so a theme update doesn't accumulate duplicate CSS.
* SMIL animations pause automatically when the page is hidden; CSS animations
  respect `prefers-reduced-motion`.

## Themes shipped in v1

| Key         | Name          | Accent        | Backdrop source |
|-------------|---------------|---------------|------------------|
| matrix      | Matrix        | `#00ff91`     | vendor bundle (default) |
| galaxy      | Galaxy        | `#c084fc`     | vendor bundle |
| volcano     | Volcano       | `#ef5735`     | vendor bundle |
| arctic      | Arctic        | `#7dd3fc`     | vendor bundle |
| mythic      | Mythic        | `#facc15`     | vendor bundle |
| neongrid    | Neon Grid     | `#ec4899`     | vendor bundle |
| rainforest  | Rainforest    | `#22c55e`     | vendor bundle |
| sunset      | Sunset        | `#ff8d50`     | vendor bundle |
| cyberpunk   | Cyberpunk     | `#06b6d4`     | vendor bundle |
| ocean       | Ocean         | `#14b8a6`     | vendor bundle |
| zen         | Zen           | `#84cc16`     | vendor bundle |
| fireflies   | Fireflies     | `#fde047`     | vendor bundle |
| aurora      | Aurora        | `#67e8f9`     | vendor bundle |
| sakura      | Sakura        | `#f9a8d4`     | vendor bundle |
| steampunk   | Steampunk     | `#d97706`     | vendor bundle |
| abyss       | Abyss         | `#22d3ee`     | vendor bundle |
| bloodmoon   | Bloodmoon     | `#dc2626`     | vendor bundle |
| crystal     | Crystal       | `#a78bfa`     | vendor bundle |
| corona      | Corona        | `#fb923c`     | vendor bundle |
| manor       | Manor         | `#94a3b8`     | vendor bundle |
| dune        | Dune          | `#fbbf24`     | vendor bundle |
| custom      | Custom        | user-defined  | vendor bundle |

## Themes added in v2 (registry-only, ship their own backdrop)

| Key      | Name    | Accent    | Vibe |
|----------|---------|-----------|------|
| aether   | Aether  | `#f5d382` | Pale gold aurora with wispy vertical mist. |
| circuit  | Circuit | `#2dd4bf` | Teal PCB traces with current pulses tracing the board. |
| moth     | Moth    | `#f5deb3` | Soft cream fluttering shapes on velvet dark. |

`dark` and `dim` themes are deprecated and no longer shipped.

## License

MIT — see `LICENSE`.
