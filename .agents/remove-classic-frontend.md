# Remove Classic Frontend (`web/classic`) — Agent Checklist

> **Status:** ✅ Completed (2026-06-10)  
> **Purpose:** Remove the legacy classic frontend and all runtime/build references.  
> **Result:** Only `web/default` remains; runtime theme switching removed.

---

## Completion Summary

| Phase | Status | Notes |
|-------|--------|-------|
| P0 — Go backend embed & routing | ✅ Done | `ThemeAssets` simplified; `NoRoute` always serves `DefaultIndexPage` |
| P1 — Dockerfile / makefile / CI | ✅ Done | `builder-classic` stage removed; CI classic build steps removed |
| P2 — Default frontend settings UI | ✅ Done | Frontend Theme selector removed from system settings |
| P3 — i18n cleanup | ✅ Done | Classic-related keys removed from 6 locale files |
| P4 — Docs / agent tools | ✅ Done | `CLAUDE.md`, `AGENTS.md` updated; `classic-to-default-sync` skill deleted |
| P5 — Comment cleanup | ✅ Done | 7 files updated |
| P6 — Directory deletion | ✅ Done | `git rm -r web/classic` |
| P7 — DB migration | ✅ Done | Auto-migration in `model/option.go` (`migrateClassicThemeOption`) |

---

## Key Post-Change Architecture

### `router/web-router.go`

```go
type ThemeAssets struct {
	DefaultBuildFS   embed.FS
	DefaultIndexPage []byte
}

func SetWebRouter(router *gin.Engine, assets ThemeAssets) {
	defaultFS := common.EmbedFolder(assets.DefaultBuildFS, "web/default/dist")
	router.Use(static.Serve("/", defaultFS))
	router.NoRoute(/* ... always serve assets.DefaultIndexPage */)
}
```

### Theme defaults

- `common/constants.go`: default theme `"default"`; `SetTheme` only accepts `"default"`
- `setting/system_setting/theme.go`: `Frontend: "default"`
- `controller/option.go`: `theme.frontend` only accepts `"default"`

### DB migration (startup)

`model/option.go` → `migrateClassicThemeOption()`:

- Runs after `loadOptionsFromDatabase()`
- If `theme.frontend == "classic"`, updates to `"default"` via `UpdateOption`

Manual SQL (optional, if migration already ran in app):

```sql
UPDATE options SET value = 'default' WHERE key = 'theme.frontend' AND value = 'classic';
```

---

## Files Changed

### Backend

| File | Change |
|------|--------|
| `main.go` | Removed classic embed vars and analytics injection |
| `router/web-router.go` | Single-theme static serving |
| `common/embed-file-system.go` | Removed `themeAwareFileSystem` / `NewThemeAwareFS` |
| `common/constants.go` | Default theme `"default"` |
| `setting/system_setting/theme.go` | Default `Frontend: "default"` |
| `controller/option.go` | `theme.frontend` validation: `default` only |
| `model/option.go` | Added `migrateClassicThemeOption()` |

### Build / CI

| File | Change |
|------|--------|
| `makefile` | Removed `FRONTEND_CLASSIC_DIR`, `build-frontend-classic`, `dev-web-classic` |
| `Dockerfile` | Removed `builder-classic` stage and classic dist COPY |
| `Dockerfile.dev` | Removed `web/classic/dist` placeholder |
| `.github/workflows/release.yml` | Removed classic frontend build steps (3 jobs) |
| `.gitignore` | Removed `web/classic/dist` |

### Frontend (`web/default`)

| File | Change |
|------|--------|
| `system-info-section.tsx` | Removed theme field and Frontend Theme selector |
| `section-registry.tsx` | Removed `theme.frontend` from `SystemInfoSection` props |
| `i18n/locales/{en,zh,ja,fr,ru,vi}.json` | Removed Classic theme i18n keys |
| 7 comment-only files | Cleaned "classic frontend" references |

### Docs / tools

| File | Change |
|------|--------|
| `CLAUDE.md` | Removed `web/classic/` from architecture tree |
| `AGENTS.md` | Removed `web/classic/` from architecture tree |
| `.agents/skills/classic-to-default-sync/SKILL.md` | Deleted |

### Deleted

| Path | Change |
|------|--------|
| `web/classic/` | Entire directory removed (~400+ files) |

---

## Left Unchanged (intentional)

| Item | Reason |
|------|--------|
| `controller/misc.go` — `"theme"` in `/api/status` | Kept; returns `system_setting.GetThemeSettings().Frontend` (always `"default"` after migration) |
| `web/default/.../types.ts` — `'theme.frontend': string` | Kept for API compatibility |
| `use-update-option.ts` — `'theme.frontend'` refresh key | Kept; harmless if option never updated from UI |
| `change.patch` | Historical patch file still references `web/classic`; not part of runtime |

---

## Verification Checklist

| Step | Command / Action | Expected |
|------|------------------|----------|
| Go build | `go build -o new-api .` | Success (no embed errors) |
| Default FE build | `cd web/default && bun install && bun run build` | Success |
| Frontend typecheck | `cd web/default && bun run typecheck` | Success ✅ (verified) |
| Docker | `docker build -t new-api .` | Success (no builder-classic) |
| Runtime | Open `/`, `/dashboard`, `/sign-in` | Default UI loads |
| SPA refresh | Hard refresh on nested routes | No 404 |
| API | `GET /api/status` → `theme` | `"default"` |
| Settings UI | System settings → General | No Classic theme option |
| Grep | `rg "web/classic\|ClassicBuild\|classicFS" --glob '!change.patch' .` | No matches (except this file) |

---

## Agent Notes

- Go single-line comments: `//`
- Go multi-line comments: `/* */`
- **Do not** re-add `//go:embed web/classic/dist` — `web/classic/` no longer exists.
- Browser tab title comes from **System Name** (`SystemName` option), set via `document.title` in `web/default/src/main.tsx` from `/api/status` → `system_name`.
- `electron/` is unrelated to classic theme switching; no changes required unless separately scoped.
