# nk-tokens-pipeline-demo

A **runnable, end-to-end demo** of the Novakid design-token pipeline — from a Figma
Variable to a `--nk-*` CSS custom property (and Dart + TS), with semantic aliases
resolving to primitive hexes. Built to show the dev team *how it works* in ~5 minutes.

> Grammar & architecture are **locked and eng-signed** (Confluence "Tokens arch &
> naming", `NPA-9291-foundations-spec.md`). This repo doesn't re-open them — it
> *demonstrates* them. The token values come straight from `final-token-tree.md`.

---

## The chain (what this demo proves)

```
┌─────────────┐   Tokens Studio    ┌──────────────┐   git push    ┌─────────────┐
│   Figma     │   plugin (export)  │ tokens.json  │  to GitHub    │   GitHub    │
│ Variables   │ ─────────────────> │  (DTCG)      │ ────────────> │   repo      │
│ (Foundations│   slash names →    │ this file is │   single      │  tokens/    │
│   file)     │   nested groups,   │ the contract │   file        │             │
└─────────────┘   aliases → refs   └──────────────┘               └──────┬──────┘
                                                                          │
                              npm run build:tokens (Style Dictionary)     │
                              ┌───────────────────────────────────────────┘
                              ▼
        ┌───────────────────────────────────────────────────────────┐
        │ build/css/variables.css   --nk-color-background-brand-...   │  Web  (parent-mf, Landing)
        │ build/dart/nk_colors.dart  NkColors.colorBackgroundBrand... │  Mobile (Flutter)
        │ build/ts/tokens.ts         tokens.color.background.brand... │  TypeScript (shared)
        └───────────────────────────────────────────────────────────┘
                              │
                              ▼  import in the consumer
        parent-mf: `import 'variables.css'` → components read var(--nk-*)
```

**The one naming rule** (Figma name *is* the contract): `/` → `-`, lowercase, prefix
`--nk-`. Style Dictionary applies it per platform — designers never write
platform-specific names.

| Figma Variable | Web (CSS) | Mobile (Dart) | TypeScript |
|---|---|---|---|
| `color/background/brand/default` | `var(--nk-color-background-brand-default)` | `NkColors.colorBackgroundBrandDefault` | `tokens.color.background.brand.default` |

---

## Run it (30 seconds)

```bash
npm install
npm run build:tokens
```

Outputs land in `build/`. Spot-check the round-trip:

```bash
# semantic alias resolved to the primitive hex (not a var(), not a copied name):
grep background-brand-default build/css/variables.css
#  --nk-color-background-brand-default: #6d46fc;   ← = color/violet/600

# the load-bearing lemon rule baked in:
grep on-lemon build/css/variables.css
#  --nk-color-text-neutral-on-lemon: #1a1a1a;      ← grey/900, never white
```

**162 CSS vars** (83 colour = 35 primitives + 48 semantic, + 16 size + 33 typography +
30 shadow), **83 Dart colors**, one typed TS tree. No `px`/`var()`/`--` leaks into any
**name** (units live in values only).

---

## What's in here

| Path | Role |
|---|---|
| `tokens/tokens.json` | The **right-sized DTCG build set** (final-token-tree §7.1) — the shape Tokens Studio pushes from Figma. Two token sets — **`primitives`** (35 colours + space/radius/stroke + atomic typography + atomic shadow) and **`semantic`** (48 aliases) — so the plugin creates two Figma collections (Primitives + Semantic/Product). Dimensions are bare numbers (px added on output). Style Dictionary deep-merges the two sets back to one root via the `nk/flatten-sets` preprocessor, so `--nk-*` names/hexes are unchanged. |
| `tokens/gated.tokens.jsonc` | The **5 gated** semantic tokens, documented but not built (4 `*/warning/*` → unbuilt amber; 1 `accent` → not in the intent dict). Not loaded by the build. |
| `style-dictionary.config.mjs` | Platform config: CSS / Dart / TS. |
| `build-tokens.mjs` | Build runner + the two custom formats (`nk/dart-colors`, `nk/ts-nested`). |
| `build/` | **Generated** outputs (gitignored). Proof of round-trip. |
| `.github/workflows/build-tokens.yml` | CI: on push to `tokens/`, rebuild + upload `build/` as an artifact. |

---

## Tokens Studio — the manual half (free Starter, €0)

The designer side, done once in the Figma file (no code):

1. **Figma Foundations file** → install the **Tokens Studio** plugin (free Starter — no seat upgrade).
2. The plugin reads the Figma Variables. Slash names (`color/violet/600`) export as
   **nested JSON groups** (`color.violet.600`); aliases export as **dot-path references**
   (`{color.violet.600}`), *not* copied hexes — this is why one hex change ripples everywhere.
3. **Settings → Sync → add a GitHub provider** → point at this repo, **single token set**,
   branch `main`, file path `tokens/tokens.json`. Authenticate (OAuths as you).
4. **Push to Git.** Tokens Studio writes `tokens/tokens.json`. That push is what triggers CI below.

That's the whole designer loop: edit a Variable in Figma → Push → code rebuilds.

---

## Plugging into parent-mf (Web consumer)

`parent-mf` is **MUI 6.4.4** (theme in `src/theme/*`). Integration is additive and
**non-colliding** — our prefix is `--nk-*`, MUI's is `--mui-*`, so they never clash:

```ts
// once, at the app root
import 'nk-tokens-pipeline-demo/build/css/variables.css'; // injects :root { --nk-* }
```

```tsx
// then components read tokens directly — in sx, styled(), or plain CSS
<Button sx={{
  bgcolor: 'var(--nk-color-background-brand-default)',   // #6D46FC
  color:   'var(--nk-color-text-brand-on-brand)',        // #FFFFFF
  borderRadius: 'var(--nk-size-radius-200)',             // 8px
}} />
```

Migration path (later, not this demo): point MUI `theme.palette.primary.main` at
`var(--nk-color-background-brand-default)` so existing MUI components inherit tokens
without touching every call-site.

---

## Free-tier limits & where the paywall triggers

Verified against Tokens Studio docs/pricing (2026-06-01). The Phase-0 MVP stays at **€0**:

| Free on Starter (what we use) | Paywalled (Starter Plus ~€39/mo per extra editor) |
|---|---|
| color, dimension (space/radius/stroke), number | **Multi-file split** (Landing/Games/Mobile syncing separately) |
| **atomic** typography & shadow Variables | **Themes** / theme switching (e.g. shipping Dark as a Theme) |
| single token set, single-file Git sync | **Composite** token types (composite typography, boxShadow, border, composition) |
| one GitHub provider | branch switching, non-local Variables/Styles |

**Why typography & shadow are atomic here:** composite token types are paid. So we keep
them **atomic** in `tokens.json` (`typography/title/xl/{size,font-weight,line-height}`,
`effect/shadow/xl/{offset-x,offset-y,blur,spread,color,opacity}`) and **bundle the
composites as Figma Styles** (a Text Style / Effect Style per role) — that bundling is
free and lives in Figma, not in the token file. Style Dictionary can re-compose them
downstream if a platform wants a single `box-shadow` string.

---

## Scope notes (honoring the locked spec)

- **Light mode only.** Dark is deferred (§7) — added later as additive rows, no rename.
- **`amber` / `warning`** — namespace provisioned, **no rows built**. The four `*/warning/*`
  semantic aliases are omitted (they'd dangle on an unbuilt ramp). Gated on brand approval.
- **`accent`** (lemon's natural semantic home) is **not in the closed intent dict** →
  `background/accent/default` omitted until the axis is extended in Confluence.
- **`lemon/300`** reserved (legacy `#FFE357` folds into `lemon/400`); **lemon 600–900** not built.
- **`size/radius/300`** (=12px) reserved, not built.
- These omissions are deliberate: every alias in `tokens.json` resolves to a real
  primitive, so the build is clean and honest.
