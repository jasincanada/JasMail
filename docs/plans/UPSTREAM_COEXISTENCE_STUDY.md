# Study: JasMail improvements vs upstream vanilla

**Issue:** https://github.com/jasincanada/JasMail/issues/26  
**Date:** 2026-10-05  
**Status:** Recommendation ready (decision pending your OK to implement)

## Executive summary

**Recommend option E (hybrid), executed mostly as C:** keep production on **vanilla** `root-fr/jmap-webmail`, and move JasMail’s unique value (dedupe) into a **sidecar** that already partially exists (`/dockersites/email/dedupe` Python JMAP CLI). Do **not** revive a hard fork as the primary webmail until/unless capacity for weekly upstream merges returns.

Copilot could not be assigned on this repo (`copilot` assignee not available); study executed in-repo instead.

---

## Current facts

| Fact | Evidence |
|------|----------|
| Prod webmail | `rootfr/jmap-webmail:1.7.3` (switched 2026-10-05) |
| Fork base | Upstream **1.5.2**; `UPSTREAM_VERSION` says `last-merge: never` |
| Fork unique value | Dedupe scan/actions/audit + Dev OS (optional) |
| Out-of-process dedupe already exists | `email/dedupe/dedupe.py` — JMAP HTTP client, mirrors webmail match keys, moves to `dupes/` |
| In-app dedupe is entangled | Sidebar, email list/thread items, settings tab, locales, SQLite in Next instrumentation |

---

## Research answers

### 1. Minimal dedupe path set (fork-only)

From `docs/upstream/fork-only-paths.json` → `dedupe_addon`:

- `lib/mail-dedupe.ts` (~910 lines), `lib/dedupe-config.ts`, `lib/dedupe-actions/`, `lib/dedupe-audit/`
- `app/api/dedupe/`, `app/[locale]/dedupe/`
- `components/dedupe/*`, `components/settings/dedupe-settings.tsx`, `components/email/dedupe-highlight-banner.tsx`
- Stores: `dedupe-*-store.ts`, `hooks/use-dedupe-highlight.ts`

~27 dedicated source files under those trees, plus locale strings and tests.

### 2. Entanglement with core (shared hotspots)

Not isolatable without touch-ups:

| Shared file | Dedupe coupling |
|-------------|-----------------|
| `components/layout/sidebar.tsx` | Imports mail-dedupe + 3 stores; context menu “scan”; badges |
| `components/email/email-list.tsx` | Highlight store + `DedupeHighlightBanner` |
| `components/email/thread-list-item.tsx` | Per-row / thread highlight classes |
| `app/[locale]/settings/page.tsx` | `dedupe` settings tab |
| `locales/*` | `sidebar.dedupe`, settings strings |
| `instrumentation.node.ts` / `package.json` | `better-sqlite3` audit DB |
| `Dockerfile` / compose | `/data` volume, `DEDUPE_*` env |

**Implication:** Option B (thin patches on vanilla tags) is **not thin** — highlight UX alone patches list + sidebar + locales every upstream release.

### 3. Can dedupe run 100% out-of-process?

**Yes for core scan + move.** The Python CLI already:

- Opens JMAP session with basic auth
- Builds match keys via shared config concepts (`dedupe_config.py`)
- Skips trash/junk; creates/moves into `dupes/`
- Is network-bound, not tied to Next.js

**Gaps vs in-app v1.7:** interactive action picker, SQLite apply audit UI, account-wide progress in the mail chrome, highlight banners, soft-delete/`deleted/` retention UI. Those can be re-homed to ops-portal or a tiny dedicated UI without forking webmail.

### 4. Merge cadence if keeping a fork (A)

Docs already require **weekly** `npm run upstream:triage` and full Option C gates on every merge. That process was designed and then **not executed** (`last-merge: never`) while upstream shipped 1.6→1.7.3. Without a calendar owner, A recreates the same failure mode.

### 5. Should this repo become dedupe-only?

**Yes, as a medium-term rename/repurpose:** keep `jasincanada/JasMail` (or rename later to `jmap-dedupe`) as the home for:

- Python/worker service + OpenAPI
- Optional small web UI
- Port of match/action logic from `lib/mail-dedupe.ts` / `lib/dedupe-actions`

Webmail stays pure upstream images.

---

## Options comparison

| Option | Effort | Risk | UX | Fits capacity? |
|--------|--------|------|-----|----------------|
| **A Soft fork** | High ongoing | High (proven drift) | Best in-mail | No |
| **B Patch overlay** | Med–high per bump | High (shared hotspots) | Near-best | Weak |
| **C Sidecar** | Med once | Low on webmail upgrades | Separate UI | **Yes** |
| **D Upstream contrib** | High social | Low tech / high reject | Best if accepted | No (already blocked) |
| **E Hybrid (C + watch D)** | Med | Low | Good enough | **Yes — recommend** |

---

## Recommendation (production / epyc)

1. **Keep** vanilla `rootfr/jmap-webmail` pinned by tag; bump on a schedule (e.g. monthly or on CVE).
2. **Elevate** `/dockersites/email/dedupe` from profile `tools` CLI → a small always-on or cron’d **worker/API** (compose service).
3. **UI:** start with CLI + ops-portal page (or simple authenticated page); do **not** re-embed into webmail.
4. **Port** match criteria + action set from JasMail v1.7 (scan-first, confirm, retention) into the sidecar — treat web UI code as reference, not as something to merge into upstream trees.
5. **Archive** hard-fork deploy path; keep this git repo as the design/history home until code is extracted.
6. Revisit in-mail highlights only if upstream adds extension points (unlikely soon) or capacity for A returns.

### Suggested compose sketch

```yaml
# illustrative — not applied yet
dedupe-api:
  build: ./dedupe
  environment:
    - JMAP_BASE_URL=http://stalwart:8080
    - JMAP_USER=...
    - JMAP_PASSWORD=...
    - DEDUPE_AUDIT_DB_PATH=/data/dedupe-audit.db
  volumes:
    - jasmail-dedupe-data:/data
  profiles: []  # or restart + cron sidecar
```

Auth: dedicated Stalwart app password with access only to needed mailboxes; never reuse personal login in compose if avoidable.

### Update runbook (vanilla webmail)

```bash
cd /dockersites/email
# bump image tag in docker-compose.yml
docker compose pull jasmail && docker compose up -d jasmail
curl -sS http://127.0.0.1:8080/api/health
```

Dedupe upgrades are independent (`docker compose build dedupe-api && up -d`).

---

## Port vs drop

| Keep / port to sidecar | Drop (or defer) |
|------------------------|-----------------|
| Match key logic (`mail-dedupe` / `dedupe_config`) | In-list highlight CSS in fork |
| Action registry (move/trash/archive/retention) | Sidebar scan badges |
| SQLite audit schema ideas | Full Dev OS 8-reviewer gate for webmail forks |
| CLI batching / skip roles | Re-base of entire Next app onto 1.7.3 |
| Account-wide scan orchestration | Upstream PR campaign (unless they invite it) |

---

## Success criteria checklist

- [x] Comparison A–E with effort/risk/UX
- [x] Explicit production recommendation
- [x] Vanilla update runbook
- [x] Sidecar compose sketch + auth note
- [x] Port vs drop list
- [ ] **Your decision** — reply on #26 (approve E/C, or choose A/B instead)

## Non-goals honored

No production re-fork in this study; no demand on upstream maintainers; Dev OS retained only as historical process docs unless sidecar work wants a lighter gate.
