# hollowui-themes

Canonical theme registry for **HollowUI Studio** and **hollowui.com**.

The Studio app and the marketing site fetch `manifest.json` from this repo at
boot to check for new / updated themes. If the manifest's `version` matches the
client's cached copy, nothing is downloaded — the client uses its embedded
themes as-is. Otherwise it fetches only the theme files whose `hash` has
changed.

## Structure

```
manifest.json         # index of all themes (small, cached, ETag-friendly)
themes/<key>.json     # one file per theme with metadata + optional factory
```

### `manifest.json`

```json
{
  "version": "2026.09.11",
  "generated_at": "2026-09-11T00:00:00Z",
  "themes": {
    "matrix": {
      "url": "themes/matrix.json",
      "hash": "sha256:...",
      "accent": "#00ff91",
      "name": "Matrix"
    },
    ...
  }
}
```

### `themes/<key>.json`

```json
{
  "key": "matrix",
  "name": "Matrix",
  "description": "Falling green rain. Bright head, fading tail. Console.",
  "version": "1.0.0",
  "accent": { "r": 0, "g": 255, "b": 145, "hex": "#00ff91" },
  "tags": ["dark", "code", "hacker"],
  "author": "hollowui",
  "created": "2026-09-11",
  "updated": "2026-09-11"
}
```

Factories live in the app / site vendor bundles for performance. This repo is
the source of truth for **metadata** — the app already knows how to render
each theme.

## Update protocol

1. Client boots, reads local `.huisite.manifest` from cache (localStorage or
   file).
2. Client fetches `https://raw.githubusercontent.com/jinjinjojo/hollowui-themes/main/manifest.json`
   with `If-None-Match` header carrying the last-known ETag. GitHub returns
   `304 Not Modified` when nothing changed → zero payload.
3. If content changed, client compares each theme entry's `hash`. Any hash that
   differs, or any new key, triggers a fetch of the matching `themes/<key>.json`.
4. New / updated themes are added to the client's runtime registry.

## Themes shipped in v1

| Key         | Name          | Accent   |
|-------------|---------------|----------|
| matrix      | Matrix        | #00ff91  |
| galaxy      | Galaxy        | #c084fc  |
| volcano     | Volcano       | #ef5735  |
| arctic      | Arctic        | #7dd3fc  |
| mythic      | Mythic        | #facc15  |
| neongrid    | Neon Grid     | #ec4899  |
| rainforest  | Rainforest    | #22c55e  |
| sunset      | Sunset        | #ff8d50  |
| cyberpunk   | Cyberpunk     | #06b6d4  |
| ocean       | Ocean         | #14b8a6  |
| zen         | Zen           | #84cc16  |
| fireflies   | Fireflies     | #fde047  |
| aurora      | Aurora        | #67e8f9  |
| sakura      | Sakura        | #f9a8d4  |
| steampunk   | Steampunk     | #d97706  |
| abyss       | Abyss         | #22d3ee  |
| bloodmoon   | Bloodmoon     | #dc2626  |
| crystal     | Crystal       | #a78bfa  |
| corona      | Corona        | #fb923c  |
| manor       | Manor         | #94a3b8  |
| dune        | Dune          | #fbbf24  |
| custom      | Custom        | user-defined |

`dark` and `dim` themes are deprecated and no longer shipped.

## License

MIT — see `LICENSE`.
