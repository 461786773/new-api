# Remove Classic Frontend (`web/classic`) — Agent Checklist

> **Purpose:** Remove the legacy classic frontend and all runtime/build references.  
> **Target branch baseline:** `main` (line numbers verified against current tree).  
> **Do NOT** delete `web/classic/` until P0–P1 backend/build changes are complete.

---

## Execution Order

1. P0 — Go backend embed & routing (required for `go build`)
2. P1 — Dockerfile / makefile / CI
3. P2 — Theme defaults & option validation
4. P3 — Default frontend settings UI & i18n
5. P4 — Docs / gitignore / optional comment cleanup
6. P5 — `git rm -r web/classic`
7. P6 — DB migration: `theme.frontend` `classic` → `default`

---

## P0 — Go Backend (Required)

| File | Lines | Action | Symbol / Snippet |
|------|-------|--------|------------------|
| `main.go` | 43–47 | **Delete** | `//go:embed web/classic/dist`, `classicBuildFS`, `classicIndexPage` |
| `main.go` | 195–196 | **Delete** | `ClassicBuildFS`, `ClassicIndexPage` in `router.ThemeAssets{...}` |
| `main.go` | 230 | **Delete** | `classicIndexPage = bytes.ReplaceAll(classicIndexPage, ...)` (Umami) |
| `main.go` | 254 | **Delete** | `classicIndexPage = bytes.ReplaceAll(classicIndexPage, ...)` (Google Analytics) |
| `router/web-router.go` | 16 | **Edit** | Comment: "both themes" → single theme |
| `router/web-router.go` | 20–21 | **Delete** | `ClassicBuildFS`, `ClassicIndexPage` fields |
| `router/web-router.go` | 26–27 | **Delete** | `classicFS`, `NewThemeAwareFS` — use `defaultFS` directly |
| `router/web-router.go` | 32 | **Edit** | `static.Serve("/", themeFS)` → `static.Serve("/", defaultFS)` |
| `router/web-router.go` | 40–44 | **Simplify** | Remove `if common.GetTheme() == "classic"` branch; always serve `DefaultIndexPage` |
| `common/embed-file-system.go` | 45–69 | **Delete** | Entire `themeAwareFileSystem` + `NewThemeAwareFS` |
| `common/constants.go` | 24 | **Edit** | `themeValue.Store("classic")` → `"default"` |
| `common/constants.go` | 32–35 | **Edit** | `SetTheme`: remove `"classic"` acceptance |
| `setting/system_setting/theme.go` | 13 | **Edit** | `Frontend: "classic"` → `"default"` |
| `controller/option.go` | 201–208 | **Edit** | `theme.frontend` case: remove `classic` from valid values & error message |
| `controller/misc.go` | 64 | **Optional** | `"theme"` field in `/api/status` — keep or hardcode `"default"` |

### Post-P0 `router/web-router.go` shape (reference)

```go
type ThemeAssets struct {
	DefaultBuildFS   embed.FS
	DefaultIndexPage []byte
}

func SetWebRouter(router *gin.Engine, assets ThemeAssets) {
	defaultFS := common.EmbedFolder(assets.DefaultBuildFS, "web/default/dist")
	// static.Serve + NoRoute always uses DefaultIndexPage
}
```

---

## P1 — Build / CI / Dev

| File | Lines | Action | Symbol / Snippet |
|------|-------|--------|------------------|
| `makefile` | 2 | **Delete** | `FRONTEND_CLASSIC_DIR = ./web/classic` |
| `makefile` | 5 | **Edit** | Remove `build-frontend-classic`, `dev-web-classic` from `.PHONY` |
| `makefile` | 13–15 | **Delete** | `build-frontend-classic:` target |
| `makefile` | 17 | **Edit** | `build-all-frontends` — only `build-frontend` |
| `makefile` | 31–33 | **Delete** | `dev-web-classic:` target |
| `Dockerfile` | 11–19 | **Delete** | Entire `builder-classic` stage |
| `Dockerfile` | 36 | **Delete** | `COPY --from=builder-classic /build/dist ./web/classic/dist` |
| `Dockerfile.dev` | 19–21 | **Edit** | Remove `web/classic/dist` mkdir & placeholder `index.html` |
| `.github/workflows/release.yml` | 40–47 | **Delete** | Step `Build Frontend (classic)` (job 1) |
| `.github/workflows/release.yml` | 98–105 | **Delete** | Step `Build Frontend (classic)` (job 2) |
| `.github/workflows/release.yml` | 153–160 | **Delete** | Step `Build Frontend (classic)` (job 3) |
| `.gitignore` | 12 | **Delete** | `web/classic/dist` |

---

## P2 — Default Frontend (`web/default`)

| File | Lines | Action | Symbol / Snippet |
|------|-------|--------|------------------|
| `web/default/src/features/system-settings/general/system-info-section.tsx` | 33 | **Edit** | `z.enum(['default', 'classic'])` → `z.literal('default')` or remove theme field |
| `web/default/src/features/system-settings/general/system-info-section.tsx` | 65–66 | **Delete** | `frontend === 'classic' ? 'classic' : 'default'` |
| `web/default/src/features/system-settings/general/system-info-section.tsx` | 83 | **Edit** | Same enum change as line 33 |
| `web/default/src/features/system-settings/general/system-info-section.tsx` | 133–162 | **Delete or simplify** | Entire `theme.frontend` `FormField` (Frontend Theme selector) |
| `web/default/src/features/system-settings/general/system-info-section.tsx` | 149–151 | **Delete** | `<SelectItem value='classic'>` |
| `web/default/src/features/system-settings/general/system-info-section.tsx` | 155–157 | **Delete** | Classic switch description `t('Switch between...')` |
| `web/default/src/features/system-settings/general/section-registry.tsx` | 19 | **Edit** | `as 'default' \| 'classic'` → `'default'` |
| `web/default/src/features/system-settings/types.ts` | 31 | **Optional** | `'theme.frontend': string` — narrow or remove |
| `web/default/src/features/system-settings/general/index.tsx` | 12 | **Keep** | `'theme.frontend': 'default'` (default is already correct) |
| `web/default/src/features/system-settings/hooks/use-update-option.ts` | 9 | **Optional** | `'theme.frontend'` in refresh keys — remove if theme option removed |

---

## P3 — i18n (Optional cleanup)

Remove unused keys from **all** locale files under `web/default/src/i18n/locales/`:

| File | Lines | Key to remove |
|------|-------|---------------|
| `en.json` | 603 | `"Classic (Legacy Frontend)"` |
| `en.json` | 3284 | `"Switch between the new frontend and the classic frontend. Changes take effect after page reload."` |
| `zh.json` | 603, 3284 | Same keys (Chinese translations) |
| `ja.json` | 603, 3284 | Same keys |
| `fr.json` | 603, 3284 | Same keys |
| `ru.json` | 603, 3284 | Same keys |
| `vi.json` | 603, 3284 | Same keys |

---

## P4 — Documentation & Agent Tools

| File | Lines | Action |
|------|-------|--------|
| `CLAUDE.md` | 38 | **Delete** line: `web/classic/ — Classic frontend ...` |
| `AGENTS.md` | 38 | **Delete** line: `web/classic/ — Classic frontend ...` |
| `.agents/skills/classic-to-default-sync/SKILL.md` | 1–end | **Delete entire file** |

---

## P5 — Optional Comment-Only References (non-blocking)

These files only mention "classic" in comments; safe to clean after main removal:

| File | Lines | Note |
|------|-------|------|
| `web/default/src/features/pricing/lib/billing-expr.ts` | 4 | Comment: "classic frontend" |
| `web/default/src/features/system-settings/models/constants.ts` | 13 | Comment: "classic frontend" |
| `web/default/src/features/usage-logs/lib/format.ts` | 169, 197 | Comments: "classic frontend" |
| `web/default/src/features/dashboard/api.ts` | 13 | Comment: "classic frontend behavior" |
| `web/default/src/features/dashboard/lib/charts.ts` | 307 | Comment: "legacy frontend" |
| `web/default/src/lib/time.ts` | 100 | Comment: "legacy frontend behavior" |
| `web/default/src/features/system-settings/models/tiered-pricing-editor.tsx` | 607 | Comment: "classic editor" |

---

## P6 — Directory Deletion (Last Step)

```bash
git rm -r web/classic
```

| Path | Action |
|------|--------|
| `web/classic/` | **Delete entire directory** (~400+ source files) |

**Pre-delete verification:**

```bash
rg "web/classic|ClassicBuild|ClassicIndex|classicFS|classicIndex" --glob '!web/classic/**' .
# Expected: no matches (except this checklist file)
```

---

## P7 — Database / Runtime Migration

No source line — apply at deploy time:

```sql
-- SQLite / MySQL / PostgreSQL (table/column names per your GORM model)
UPDATE options SET value = 'default' WHERE key = 'theme.frontend' AND value = 'classic';
```

Optional: add one-time migration in Go startup if `theme.frontend == classic`.

---

## Verification Checklist

| Step | Command / Action | Expected |
|------|------------------|----------|
| Go build | `go build -o new-api .` | Success (no embed errors) |
| Default FE build | `cd web/default && bun install && bun run build` | Success |
| Docker | `docker build -t new-api .` | Success (no builder-classic) |
| Runtime | Open `/`, `/dashboard`, `/sign-in` | Default UI loads |
| SPA refresh | Hard refresh on nested routes | No 404 |
| API | `GET /api/status` → `theme` | `"default"` |
| Settings UI | System settings | No Classic theme option |

---

## Quick Reference — All Files to Touch

```
main.go
router/web-router.go
common/embed-file-system.go
common/constants.go
setting/system_setting/theme.go
controller/option.go
controller/misc.go                          # optional
makefile
Dockerfile
Dockerfile.dev
.github/workflows/release.yml
.gitignore
web/default/src/features/system-settings/general/system-info-section.tsx
web/default/src/features/system-settings/general/section-registry.tsx
web/default/src/features/system-settings/types.ts              # optional
web/default/src/features/system-settings/hooks/use-update-option.ts  # optional
web/default/src/i18n/locales/{en,zh,ja,fr,ru,vi}.json       # optional
CLAUDE.md
AGENTS.md
.agents/skills/classic-to-default-sync/SKILL.md               # delete
web/classic/                                                  # delete last
```

---

## Agent Notes

- Go single-line comments: `//`
- Go multi-line comments: `/* */`
- Do **not** delete `web/classic/` before removing `//go:embed web/classic/dist` from `main.go` — build will fail.
- Default theme in DB may still be `classic`; change to `default` before or during deploy.
- `electron/` is unrelated to classic theme switching; no changes required unless separately scoped.
