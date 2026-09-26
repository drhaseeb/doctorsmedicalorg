# Doctors' Medical ORG — build notes

This repo (`github.com/drhaseeb/doctorsmedicalorg`) serves **doctorsmedical.org.pk** natively: the DMO
homepage/About/Contact/legal pages at the root, and the clinical tools directory merged in at `/tools`.
It used to be a standalone "Tools" app on its own host; that merge is done and complete — there is no
separate tools deployment anymore.

## Sibling sites (same organization, separate repos)

| Site | Local folder | GitHub repo | Domain |
|---|---|---|---|
| DMO home + Clinical Tools (this repo) | `doctorsmedical-tools` | `drhaseeb/doctorsmedicalorg` | doctorsmedical.org.pk (+ `/tools`) |
| Doctors' Medical Center (the clinic) | `doctorsmedical-site` | `drhaseeb/doctorsmedical-clinic` | clinic.doctorsmedical.org.pk |
| Resus Runner | `resus-runner` | `drhaseeb/resus` | resus.doctorsmedical.org.pk |
| MRCEM Expert (exam prep) | `mrcem` | `drhaseeb/expert` | mrcemexpert.drhaseeb.me |

All four are plain git repos with a GitHub remote, deployed via Cloudflare Pages' native git
integration (push to `main` → auto-deploy). There is no GitHub Actions workflow in any of them —
Cloudflare does the build itself from `npm run build`.

Note: `~/Documents/expert` (Python lesson/quiz-extraction scripts that feed `mrcem`'s content) and
`~/Documents/drhaseebahsin-main/...` (an old pre-merge snapshot of this tools app, from before it had
a name) are **not** git repos and are **not** backed up anywhere. Treat anything you want to keep in
either as needing to be copied into a real repo first.

## Stack

- Vite 8 + React 19 + TypeScript, Tailwind v4 (via `@tailwindcss/vite`), `react-router-dom` v7.
- `vite-plugin-pwa` for the installable/offline app shell.
- `npm run dev` / `npm run build` (`tsc -b && vite build`) / `npm run lint` (oxlint) / `npm run preview`.
- No test suite — verification is manual (dev server + browser).

## Adding or editing a clinical calculator

Every tool is a self-contained folder under `src/tools/<slug>/`:

- `index.tsx` — the calculator itself (the interactive form/logic).
- `info.tsx` — the long-form article below it (how it works, references, FAQ). Optional; if missing,
  [ToolPage.tsx](src/pages/ToolPage.tsx) silently renders nothing for that slot.

**Routing is automatic — there is no per-tool route to register.** [ToolPage.tsx](src/pages/ToolPage.tsx)
resolves `/tools/:slug` by lazy-importing `../tools/${slug}/index.tsx` and `../tools/${slug}/info.tsx`
directly (Vite's dynamic `import()` glob), so dropping in a new folder is enough for the route to work.

What you *do* still need to do by hand for a new tool to be findable and correct:

1. Add an entry to [src/tools/registry.ts](src/tools/registry.ts) — `slug` (must match the folder name),
   `title`, `sub`, `desc`, `category` (`"clinical" | "public" | "dosing"`), `icon` (a `lucide-react` icon).
   This is what drives the `/tools` grid and the calculator's header — nothing else reads tool metadata.
2. Add its URL to [public/sitemap.xml](public/sitemap.xml), alphabetically among the `/tools/...` entries.
3. Optionally, add `<RelatedTools slugs={[...]} />` cross-links in the new tool's `info.tsx` and in
   sibling tools whose users would want to find it (e.g. the adult and paediatric head-injury tools
   link to each other).

Shared building blocks live in `src/kit/` — reuse these rather than hand-rolling form controls:
`CheckboxRow`, `NumberField`, `SegmentedField`, `OptionListField` (inputs), `ResultPanel`, `Section`
(layout), `ToolLayout` (the page chrome every tool renders inside — back link, category badge, medical
disclaimer banner, ad slot), `InfoArticle` / `ArticleTOC` / `FaqAccordion` / `References` /
`ReviewedBadge` (the info-article furniture), `RelatedTools`, `Timer`, `StimulusCard`, `InfoPopover`,
`AdSlot`.

`CheckboxRow` takes an optional `disabled` prop (grayed out, `cursor-not-allowed`) for a control whose
effect depends on another checkbox being checked first — used on the head-injury tool's delayed-
presentation toggle, which the underlying NICE guideline only actually changes anything for when the
anticoagulant/antiplatelet box is also checked.

**Clinical accuracy**: these are real bedside decision tools. Before implementing or changing scoring
logic, verify against the actual primary source (the named guideline/paper), not memory — NICE guidance
in particular is often blocked to a plain fetch from nice.org.uk directly (403); NCBI Bookshelf mirrors
and the guideline's own PDF/position-statement documents have been reliable fallbacks. When a PDF's text
comes back as raw stream/binary garbage from a fetch tool, the tool has usually still saved the file
locally — re-reading that saved path directly usually extracts it correctly.

## Design system

Tailwind tokens: `text-accent`, `bg-accent-soft`, `border-line`, `bg-surface` (see `tailwind.config`/
`index.css` for the full ramp). Shared non-tool UI atoms (`Eyebrow`, `SectionHeading`, `Card`,
`LinkButton`) live in `src/components/ui.tsx` and are what the DMO marketing pages (`src/pages/dmo/*`)
are built from — reuse these instead of new one-off styling when touching Home/About/etc.

## Content that isn't in JSX

- [src/content/org.ts](src/content/org.ts) — org name, tagline, mission, values, social links, the
  clinic/resus cross-site URLs.
- [src/content/legal.ts](src/content/legal.ts) — the single source of truth for Terms, Privacy, and
  Disclaimer body copy (each has its own `lastUpdated`). The page components (`src/pages/dmo/Terms.tsx`
  etc.) just render this through a shared `LegalPage`.

## Analytics, ads, and consent — how it actually fits together

This is the part most likely to look "broken" if you go looking without this context, because most of
the actual analytics plumbing is **not in this repo**:

- **Google Consent Mode v2 defaults** are set in a static inline `<script>` at the top of
  [index.html](index.html), before anything else: granted globally, denied for the EEA/UK/Switzerland
  region list. This has to stay the very first thing in `<head>`.
- **AdSense + the Google Funding Choices consent banner** are loaded from
  [src/lib/adsBootstrap.ts](src/lib/adsBootstrap.ts), called once from `main.tsx`. It deliberately skips
  `/privacy` — Google's rules for a registered privacy-policy page forbid it hosting the consent tag
  itself. `signalGooglefcPresent()` is the standard "let ad-blocker-detection iframes find googlefc"
  hack Funding Choices expects.
- **The actual GA4 tag is not loaded by this app at all.** It's injected by Cloudflare at the edge for
  this zone (what you'd find under the Cloudflare dashboard for `doctorsmedical.org.pk` — Zaraz or the
  newer "Tag Gateway" style first-party proxy; there's a `/tt0n/...` first-party-proxied collect
  endpoint and a directly-loaded `googletagmanager.com/gtag/js` in production). None of that
  configuration lives in this git repo or under any local file — it's Cloudflare-account-side only.
  What *is* verified is that it correctly reads the page's own `dataLayer` consent signal (the outbound
  GA hit carries a live `gcs` value that matches the actual consent state) — Cloudflare's own docs
  describe this as automatic, no extra dashboard wiring needed for the signal itself to be honored.
- The footer's "Privacy Choices" link (`reopenConsentMessage` in
  [src/layouts/DMOLayout.tsx](src/layouts/DMOLayout.tsx)) reopens the Funding Choices banner via
  `googlefc.callbackQueue`.

**Open items, as of the last time this was investigated (2026-09):**
- If the consent banner ever stops appearing for anyone (not just EU visitors), the likely cause is
  Google AdSense/Ad Manager → Privacy & messaging still being scoped to an old domain/subdomain from
  before the root-domain merge, not a code regression — that's a Google-account dashboard setting, not
  something fixable from this repo. Check the message's target "Sites" list there first.
- `config.adSlots.afterTool` / `.sidebar` in [src/config.ts](src/config.ts) are still empty strings —
  real AdSense ad units were never created/pasted in, so ads don't actually render yet even though the
  scripts load.
- A denied-consent visitor under Consent Mode still produces an anonymized "cookieless ping" to Google
  by design (no cookie, no identifier) — this is compliant, expected behavior, not evidence that consent
  is being ignored, and is a common point of confusion when eyeballing GA4 Realtime.

## Deploy

Cloudflare Pages, auto-deploy on push to `main`. `public/_redirects` handles SPA deep-link fallback.
`public/ads.txt` must keep matching `config.adClient`. No staging environment / preview builds are set
up beyond whatever Cloudflare Pages does automatically for non-main branches.
