# LeagueSphere App Review: Remediation Plan

**Date:** 2026-09-25 (last updated 2026-09-27)
**Status:** Phases 1 and 2 done and merged upstream. Phase 3 discarded by maintainer decision. Phases 4 and 5 open — Phase 4 is next.
**Source:** Two independent code reviews of the whole app group (Django backend, five React apps, container and CI config), merged and re-prioritized. Findings come from reading code and running `black --check`; no tests were run and nothing was changed on the app itself.

Read this top to bottom. Phases are ordered by risk first, then effort. Each item says **what** is wrong, **where** it lives, **how** to fix it and **how to prove it**. Effort is S (under an hour), M (half a day), L (a day or more).

---

## Where this stands (2026-09-27)

| Phase | Status | Landed as |
|---|---|---|
| 1 — Settings and edge hardening | ✅ Done, merged | [#1977](https://github.com/dachrisch/leaguesphere/pull/1977) (merge `35635b4b`) |
| 2 — Escaping, validation and access decisions | ✅ Done, merged | [#1978](https://github.com/dachrisch/leaguesphere/pull/1978) (merge `6a7e86b7`) |
| 3 — Shared cache and server-side result caching | ❌ Discarded — no Redis upstream | [#1979](https://github.com/dachrisch/leaguesphere/pull/1979) closed; superseded by [#1989](https://github.com/dachrisch/leaguesphere/pull/1989) |
| 4 — Process, docs and repo hygiene | ⬜ Open — **next** | — |
| 5 — Frontend and backend code quality | ⬜ Open | — |

How execution diverged from the plan as written:

- **Phase 1 landed reworked, not as drafted.** Three things the plan missed, all caught in review: the fail-loud env guard was bypassed whenever the app boots via the default `league_manager.settings` package (whose `__init__.py` falls back to `from .prod import *`), now handled by a shared `league_manager/settings/_env_guard.py`; the maintenance middleware catches broad `Exception` and logs it (mirroring `db_guard`) instead of only `OperationalError`, and deliberately does not cache the "off" fallback, so maintenance mode resumes as soon as the DB is back rather than after the 30 s TTL; every `add_header` moved to `server` level so `/static/` inherits it, with `proxy_hide_header` deduping the two headers Django also sends. Unplanned extra requested during review: **gzip for `application/json`** in all three confs (nginx only gzips `text/html` by default and there is no `GZipMiddleware`). `CORS_ALLOWED_ORIGINS` landed as an empty list — the vite apps use same-origin proxies. The guard broke CI's `test_backend_image`, which the maintainer fixed separately in [#2011](https://github.com/dachrisch/leaguesphere/pull/2011) and [#2012](https://github.com/dachrisch/leaguesphere/pull/2012).
- **Phase 2 landed after five review must-fixes**, each a real gap in the first cut: the nginx rate limit was bypassable without a trailing slash (now one `location ~ ^/(admin/)?login/?$`), `purge_old_journey_events --days 0` or negative was a mass-delete footgun (now `CommandError` before the cutoff is computed), `escape(None)` rendered NULL passcheck notes as the literal string `None` (now `league_manager/utils/html.py::escape_cell`), a residual `escape: False` sat in `league_table/views.py`, and the JSON-LD `</script>` breakout had no test. 2.1 also fixed three unescaped-output bugs found while adding coverage that the plan never listed: `EventsTableError`'s message, the JSON-LD payload itself, and a stray `|safe` on `GameSetup.note`.
- **2.3 is partial by design, and its follow-up is now orphaned.** The nginx `limit_req` covers only the two HTML login forms; the Knox token login `/api/accounts/auth/login/` is still unprotected. The planned `django-axes` follow-up was written as "after the shared cache" — which Phase 3 was going to provide, and won't. See N3.
- **Phase 3 was rejected, not deferred.** See the decision block in that section.
- **Stage verification is still outstanding for Phases 1 and 2.** All the work was done without access to stage, prod or the container repo's Ansible flow, and verified against `league_manager.settings.test_sqlite` rather than the LXC MariaDB test DB. Both PRs were merged with the `curl -I` and login click-through explicitly left as a post-deploy gate, not a merge gate. See N1 — this is the one open item with real risk attached.
- **1.6 landed narrower than written.** `base.py` still carries the `test_db`/`user` defaults; what changed is that `prod.py` and `stage.py` now refuse to start without the real values. Good enough in practice, but `base.py` read on its own still looks like the old silent fallback.

---

## Priority overview

| # | Item | Category | Severity | Effort | Phase | Status |
|---|------|----------|----------|--------|-------|--------|
| 1 | Clickjacking: `X_FRAME_OPTIONS = "ALLOWALL"` is an invalid value, browsers ignore it | Security | High | S | 1 | ✅ Done (#1977) |
| 2 | Prod cookies not HTTPS-only, no HSTS, no SSL redirect | Security | High | S | 1 | ✅ Done (#1977) |
| 3 | nginx sends joke headers but no security headers | Security | High | S | 1 | ✅ Done (#1977) |
| 4 | Stored XSS: pandas `to_html(escape=False)` + `\|safe` in five places | Security | High | M | 2 | ✅ Done (#1978) |
| 5 | Middleware order: maintenance middleware hits the DB before the DB guard runs | Reliability | High | S | 1 | ✅ Done (#1977) |
| 6 | Login has no brute-force protection | Security | Medium | S | 2 | ⚠️ Partial — HTML forms only, see N3 |
| 7 | `mark_safe()` on strings built from CSV rows and exception text | Security | Medium | S | 2 | ✅ Done (#1978) |
| 8 | CORS open to every origin, deprecated setting name | Security | Medium | S | 1 | ✅ Done (#1977) |
| 9 | DB silently falls back to `user`/`user`@`127.0.0.1` when env vars are missing | Reliability | Medium | S | 1 | ✅ Done (#1977) |
| 10 | `LocMemCache` with 6 gunicorn workers: six caches that disagree | Reliability / Perf | Medium | M | 3 | ❌ Discarded |
| 11 | League standings recomputed with pandas per uncached request | Performance | Medium | M | 3 | ❌ Discarded |
| 12 | Journey event API skips validation, unbounded JSON, no retention | API / Reliability | Medium | S | 2 | ✅ Done (#1978) |
| 13 | Public rosters expose name, pass number and join date, and sit in the sitemap | Privacy | Medium | S (decision) | 2 | ✅ Done (#1978) |
| 14 | `black` required by CLAUDE.md but 133 files fail and CI never runs it | Process | Medium | M | 4 | ⬜ Open |
| 15 | CLAUDE.md, README **and AGENTS.md** link 7 doc paths that moved or never existed | Docs | Medium | S | 4 | ⬜ Open |
| 16 | Repo root clutter: loose scripts, analysis MDs, four AI instruction files | Hygiene | Medium | M | 4 | ⬜ Open |
| 17 | Config drift: Django version, bumpversion, `packages.find`, env var names, dead GH workflows | Hygiene | Low | S | 4 | ⬜ Open |
| 18 | Five frontend toolchains, drifting deps, hook lint suppressions, `no-explicit-any` file | Frontend | Medium | L | 5 | ⬜ Open |
| 19 | Django nits: double route mount, GET that mutates, unused log handler, `SET_DEFAULT=1`, bare `except` | Code quality | Low | S | 5 | ⚠️ Partial — 1.7 done in #1977 |
| 20 | Tests: `assertNumQueries` in 13 of 151 files, 883-line `journey/tests.py`, `--capture=no` in addopts | Testing | Low | M | 5 | ⬜ Open |

---

## Phase 1: Settings and edge hardening (one PR, no behaviour risk) — ✅ Done ([#1977](https://github.com/dachrisch/leaguesphere/pull/1977))

Goal: close every High that is a config change. Target: a single PR touching `league_manager/settings/base.py`, `league_manager/settings/prod.py`, `league_manager/settings/stage.py` and the three nginx confs.

### 1.1 Fix `X_FRAME_OPTIONS` (item 1)
- **Where:** `league_manager/settings/base.py:230`, under `# ToDo deleteMe`.
- **Why:** `ALLOWALL` is not a valid value (only `DENY` and `SAMEORIGIN` are). Browsers drop the header, nginx adds nothing, so any site can iframe `/admin/`, the scorecard or passcheck and clickjack a logged-in staff user.
- **How:** Delete the line so Django's default `DENY` applies. If a view genuinely must be embedded, decorate that one view with `@xframe_options_exempt` or `@xframe_options_sameorigin`.
- **Verify:** `curl -I https://stage.leaguesphere.app/` shows `X-Frame-Options: DENY`. Add a test that requests `/` and asserts the header.

### 1.2 Secure cookies, HSTS and SSL redirect in prod and stage (item 2)
- **Where:** `prod.py` and `stage.py` set `SECURE_PROXY_SSL_HEADER` but nothing else. Only `demo.py` sets the cookie flags.
- **How:** Add to both files:
  ```python
  SESSION_COOKIE_SECURE = True
  CSRF_COOKIE_SECURE = True
  SECURE_SSL_REDIRECT = True
  SECURE_HSTS_SECONDS = 60 * 60 * 24 * 30  # start at 30 days, raise to a year once stable
  SECURE_HSTS_INCLUDE_SUBDOMAINS = True
  SECURE_REFERRER_POLICY = "strict-origin-when-cross-origin"
  ```
  Exclude `/health/` from the SSL redirect with `SECURE_REDIRECT_EXEMPT = [r"^health/"]` if the container healthcheck hits it over plain HTTP.
- **Verify:** `DJANGO_SETTINGS_MODULE=league_manager.settings.prod python manage.py check --deploy` reports no cookie or HSTS warnings. Deploy to stage first and confirm login still works behind Traefik.

### 1.3 Security headers in nginx (item 3)
- **Where:** `container/nginx.conf:18-20`, `container/nginx.demo.conf:18-20`, `container/nginx.staging.conf:18-20`.
- **How:** Keep the coffee joke. Add in the same `location /` block:
  ```nginx
  add_header X-Content-Type-Options "nosniff" always;
  add_header Referrer-Policy "strict-origin-when-cross-origin" always;
  add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
  ```
  Leave `X-Frame-Options` and HSTS to Django (1.1, 1.2) so there is one owner per header. A Content-Security-Policy is a follow-up: 42 jsdelivr and 2 unpkg references plus 17 templates with inline scripts need SRI hashes or nonces first.
- **Verify:** `curl -I` on stage after deploy via `./container/deploy.sh stage`. Add `container/test-compose-network.sh` style check if practical.

### 1.4 Swap the middleware order (item 5)
- **Where:** `league_manager/settings/base.py:50-51`.
- **Why:** `MaintenanceModeMiddleware` calls `SiteConfiguration.objects.first()` on a cache miss with no try/except. With the DB down and a cold cache (every worker after a restart), it raises `OperationalError` and the user sees a 500. `DatabaseGuardMiddleware`, which exists precisely for that case, never runs.
- **How:** Put `DatabaseGuardMiddleware` above `MaintenanceModeMiddleware`. Also wrap the config lookup in the maintenance middleware in a `try/except OperationalError` that falls back to `scope: off`, so it is safe regardless of order.
- **Verify:** Test with a mocked `connection.cursor` raising `OperationalError`: request `/` and assert a redirect to `/database-error/`, not a 500.

### 1.5 Tighten CORS (item 8)
- **Where:** `base.py:13`, `CORS_ORIGIN_ALLOW_ALL = True`.
- **How:** Replace with `CORS_ALLOWED_ORIGINS = [...]` listing the real front-end origins per environment (`prod.py`, `stage.py`, `demo.py`, `dev.py`). Credentials stay disabled.
- **Verify:** Preflight from an unlisted origin returns no `Access-Control-Allow-Origin`.

### 1.6 Fail loudly on missing DB config (item 9)
- **Where:** `base.py`, `DATABASES` defaults to `user`/`user`@`127.0.0.1`.
- **How:** Keep the defaults for `dev.py` and `test_sqlite.py` only. In `base.py` read the env vars without defaults; in `prod.py` and `stage.py` assert they are set (or use `os.environ[...]` so a `KeyError` stops startup). Same for `SECRET_KEY`: prod should refuse to start without one.
- **Verify:** Starting prod settings without `MYSQL_HOST` raises at import time.

### 1.7 Small settings cleanups (part of item 19)
- Remove the duplicate `django.template.context_processors.request` (`base.py:81` and `:84`).
- Remove the unused `debug.log` file handler in `LOGGING` (`base.py:214`), or attach it to a logger on purpose.
- Delete the commented-out `DEBUG_DATE` and toolbar toggles in `dev.py`.

**Phase 1 done when:** `check --deploy` is clean for prod, stage shows the new headers, all existing tests pass, and the PR is under about 60 changed lines.

**Status:** Done and merged ([#1977](https://github.com/dachrisch/leaguesphere/pull/1977)). `check --deploy` clean apart from the optional `SECURE_HSTS_PRELOAD` warning; 213 tests pass in `league_manager/`; the PR ended up well over 60 lines because of the review rework and the tests. "Stage shows the new headers" is **not** verified — see N1.

---

## Phase 2: Escaping, validation and access decisions — ✅ Done ([#1978](https://github.com/dachrisch/leaguesphere/pull/1978))

### 2.1 Stop rendering unescaped HTML from user-controlled names (item 4)
- **Where:** `gamedays/views.py:142`, `:228`, `:473`, `gamedays/service/tournament_service.py:114`, `matchreport/constants.py:30`, all `"escape": False` passed to pandas `to_html`, then `|safe` in `gamedays/templates/gamedays/statistics/league_statistics.html` and `matchreport/templates/matchreport/gameday_detail.html`.
- **Why:** Player and team names flow into those tables. A team manager (non-staff, via passcheck) can set a player name to `<img src=x onerror=...>` and it executes on the public league statistics page and in the staff match report.
- **How (TDD):**
  1. Write a test that creates a player named `<script>alert(1)</script>`, renders the statistics view and asserts the raw tag is **not** in the response body.
  2. Remove `"escape": False` everywhere. If some columns need markup (links, badges), build those cells with `django.utils.html.format_html` so only the markup is trusted and the data is escaped.
  3. Keep `|safe` only where the string is now guaranteed to come from `format_html`.
- **Verify:** The new tests pass; visually check the statistics and match report pages on stage.

### 2.2 Escape the officials import messages (item 7)
- **Where:** `officials/views.py:325`, `:390`, `:690` (`messages.success(..., mark_safe(created_entries))`, `mark_safe(f"{summary}<br>{detail}")`).
- **Why:** The strings contain official names from the DB and exception messages that echo CSV cell content (for example a `ValueError` from `int("<b>x</b>")`). Staff-only, so low reach, but it is a real stored/reflected XSS path.
- **How:** Build the message with `format_html_join("<br>", "{}", ((line,) for line in lines))` instead of string concatenation plus `mark_safe`.
- **Verify:** Test that an uploaded row containing `<b>` produces an escaped `&lt;b&gt;` in the message.

### 2.3 Rate-limit password logins (item 6)
- **Where:** `/login/` and `/admin/login/` are plain Django views (`league_manager/urls.py`); DRF's `AnonRateThrottle` covers only the API.
- **How:** Either add `django-axes` (`AXES_FAILURE_LIMIT = 5`, `AXES_COOLOFF_TIME = 1` hour, lock by username + IP) or add `limit_req_zone` in nginx for those two paths. Prefer `django-axes` because it also covers `/api/accounts/auth/login/` and logs attempts. Do this after Phase 3 if you pick axes, because it needs a shared cache to count across workers.
- **Verify:** Six wrong passwords in a row returns a lockout response; the test suite includes one such test.
- **Done as:** nginx `limit_req` on the two HTML login forms (`/login`, `/admin/login`, with and without trailing slash) only. **Follow-up (Phase 3, after the shared cache):** `django-axes` so `/api/accounts/auth/login/` (Knox token login) is covered too and attempts are logged.

### 2.4 Validate journey events (item 12)
- **Where:** `journey/views.py:19` (`JourneyEventViewSet.create` reads `request.data` directly).
- **How:** Run `self.get_serializer(data=request.data)` and `is_valid(raise_exception=True)` before creating. Add a `max_length` and an allow-list of `event_name` values, cap `metadata` size (for example 4 KB) in the serializer, and add a management command that deletes events older than 90 days.
- **Verify:** POST without `event_name` returns 400, not 500. Oversized metadata returns 400.

### 2.5 Decide on roster privacy (item 13)
- **Where:** `passcheck/templates/passcheck/roster_list.html:68-72`, `league_manager/sitemaps.py:155` (`PasscheckTeamSitemap`).
- **Why:** Anonymous visitors and search engines see first name, last name, pass number and join date of every player.
- **How:** This is a product decision, not a code bug. Options, cheapest first: (a) show the pass number column only when `is_user_or_staff`; (b) drop rosters from the sitemap and add `noindex`; (c) gate the whole roster behind login. Record the decision in `docs/topics/features/`.
- **Verify:** Anonymous GET of a roster page no longer contains pass numbers (if a or c).

**Phase 2 done when:** every `escape=False` and `mark_safe` call site has a test proving escaping, logins lock out, journey POSTs validate, and the roster decision is documented.

**Status:** Done and merged ([#1978](https://github.com/dachrisch/leaguesphere/pull/1978)), approved after one review round. Roster decision recorded in `docs/topics/features/passcheck-roster-privacy.md`. Lockout is nginx-side and verified only against a stub upstream on `nginx:stable`, never on stage — see N1. The Knox login is still uncovered — see N3.

---

## Phase 3: Shared cache and server-side result caching — ❌ Discarded (2026-09-27)

> **Decision: not doing this.** Maintainer call on [#1979](https://github.com/dachrisch/leaguesphere/pull/1979), formalised as Request Changes: no Redis is operated in any LeagueSphere environment and none will be. The six-worker `LocMemCache` split-brain described below is real — throttle counting 120 vs roughly 720/min, `/clear-cache/` reaching one worker in six, maintenance mode and DB-guard status disagreeing between workers, every cached value rebuilt six times and lost on each deploy — but none of it has caused an incident at this scale, and a Redis outage would convert a performance characteristic into a hard SPOF the team then has to operate. The suggestion PR also shipped it unsafely: no cache timeouts configured, a `127.0.0.1` default that is wrong inside Docker, and prod compose missing the `depends_on` that staging and demo got.
>
> **What replaced it:** [#1989](https://github.com/dachrisch/leaguesphere/pull/1989) `GET /api/snapshot/` (merged 2026-09-26) — one configurable, ETag'd, TTL-cached, gzippable mass endpoint. The real cost was never recompute time, it was request count: the heaviest consumer made 1 catalog call plus roughly 900 `/api/gamedays/<id>/games/` plus roughly 1600 HTML game pages per run. No cache reduces that; one endpoint does. Its TTL cache is **file-based** (`SNAPSHOT_CACHE_DIR`), which gives the cross-worker sharing 3.2 wanted with no new service.
>
> The sections below are kept as the record of what was analysed and why it was declined. Do not reopen without new evidence: an actual incident traced to cache split-brain, or a scale change that makes a shared cache load-bearing. Anything that needs cross-worker state before then should use the file-based or DB-backed route instead.

### 3.1 Replace `LocMemCache` with Redis (item 10) — ❌ Discarded
- **Where:** `base.py:62`; gunicorn runs `-w 6 --threads 2` in `deployed/docker-compose.yaml:42` and the demo and staging compose files.
- **Why:** Six workers means six private caches. Effects today: `AnonRateThrottle` "120/min" is really about 720/min; `/clear-cache/` clears one worker in six; maintenance mode can be on in one worker and off in another; the DB guard status is per worker; every cached value is rebuilt six times and lost on each deploy.
- **How:**
  1. Add a `redis` service to the three compose files (`deployed/docker-compose*.yaml`) with a named volume and a healthcheck, via the container repo's Ansible flow, never by hand on the server.
  2. Set `CACHES["default"]["BACKEND"] = "django.core.cache.backends.redis.RedisCache"` with `LOCATION` from `REDIS_URL`; keep `LocMemCache` in `dev.py` and `test_sqlite.py`.
  3. Add `REDIS_URL` to `.env_template` and `deployed/ls.env.staging.template`.
- **Verify:** On stage, toggle maintenance mode and confirm every request sees it immediately; hit `/clear-cache/` and confirm the ETag-based views recompute. Confirm throttle counts are shared by exceeding 120 anonymous requests per minute.

### 3.2 Cache computed standings and list payloads (item 11) — ❌ Discarded, partly superseded
- **Where:** `league_table/api/views.py` (`LeagueTableAPIView.get` rebuilds a pandas DataFrame per request that lacks a matching ETag); the same pattern fits `GamedayViewSet.list` and the journey progress feed.
- **How:** The ETag function already knows when data changed. Use the ETag string as the cache key: on a miss, compute, `cache.set(etag, payload, 3600)`; on a hit, return the stored JSON. Wrap this in a small helper in `league_manager/utils/` so the three endpoints share it. Only meaningful once 3.1 is in, otherwise it is six caches again.
- **Verify:** Test with `assertNumQueries`: the second request with the same ETag runs only the ETag query. Run `k6 run load-test-k6.js` before and after and compare p95 in the Grafana k6 dashboard.

**What survives of 3.2:** [#1989](https://github.com/dachrisch/leaguesphere/pull/1989) does exactly this pattern, file-based, for the snapshot endpoint. `LeagueTableAPIView.get` and `GamedayViewSet.list` still rebuild their payload for any caller that does not send a matching `If-None-Match`. That is the remaining gap; revisit it only if the logs ever show it mattering, and only Redis-free.

**Phase 3 status:** Discarded. The k6 before/after comparison it called for was never run and is not needed.

---

## Phase 4: Process, docs and repo hygiene — ⬜ Open, next up

### 4.1 Make `black` real (item 14)
- **Where:** CLAUDE.md says "REQUIRED before pushing", `black --check .` fails on **133 files** of 589 (re-measured 2026-09-27), and neither `.circleci/continue.yml` nor `.pre-commit-config.yaml` nor the workflows run black.
- **How:** One formatting-only commit (`black .`), then add a `black --check .` step to the always-on CircleCI jobs next to `scope_coverage`, and add black to `.pre-commit-config.yaml`. Add a `[tool.black]` section with an explicit `extend-exclude` for migrations if you want them left alone.
- **Verify:** CI fails on a deliberately unformatted file in a test PR.

### 4.2 Fix the documentation map (item 15)
- **Where:** `CLAUDE.md`, `README.md` **and `AGENTS.md`** (the plan missed the third). Verified mapping as of 2026-09-27:
  | Linked as | Actually at |
  |---|---|
  | `docs/guides/contributor-guide.md` | `docs/topics/guides/contributor-guide.md` |
  | `docs/guides/coding-standards.md` | `docs/topics/guides/coding-standards.md` |
  | `docs/guides/setup-guide.md` | `docs/topics/guides/setup-guide.md` |
  | `docs/guides/performance-guide.md` | `docs/topics/deployment/performance-guide.md` |
  | `docs/guides/infrastructure-policy.md` | `docs/topics/deployment/infrastructure-policy.md` |
  | `docs/arch/architecture-overview.md` | `docs/topics/architecture/architecture-overview.md` |
  | `docs/testing/` | `docs/topics/testing/` |
  | `docs/guides/infrastructure-performance-policy.md` | **does not exist anywhere** — drop the link or write the doc |
- **Also:** `CLAUDE.md` links `../container/docs/leaguesphere-environments.md` and `../PROD_TEST_GUIDE.md`, which live outside this repo on purpose. Whitelist those two in the checker rather than "fixing" them.
- **How:** Update the links. Delete `docs/AUDIT.md` and `docs/CONSOLIDATION_SUMMARY.md` (they describe a finished migration) or move them to `docs/topics/planning/history/`. Add a link checker (`lychee` or a 20-line Python script) to the always-on CI job so this cannot regress.
- **Verify:** Link checker passes.

### 4.3 Tidy the repo root (item 16)
- Move `CIRCLECI_IMPLEMENTATION.md`, `IMPLEMENTATION_CHECKLIST.md`, `RELEASE_FLOW_COMPARISON.md`, `RELEASE_PROCESS_ANALYSIS.md` into `docs/topics/planning/history/` or `docs/topics/deployment/`, as `docs/DOCUMENTATION.md` already requires.
- Move `e2e-game-test.js`, `find-game-id.js`, `step1-find-game.js`, `setup-gameday-for-today.js` and `capture_screenshot.sh` into `scripts/stage/` with a README, and remove the hard-coded username from `step1-find-game.js` (read it from an env var).
- Move `test_settings.py` to `league_manager/settings/test_e2e.py` and update `scorecard/tests/e2e/conftest.py` and `gameday_designer/tests/e2e/conftest.py`.
- Fold `GEMINI.md` and `QWEN.md` into `AGENTS.md` and make `CLAUDE.md` the Claude-specific delta only, so there is one source of truth. Decide whether the 16 module-level `AGENTS.md` / `CLAUDE.md` pairs should also collapse to one file each.
- Delete `coverage-reports/` (only a README) and the `conductor/` tracks if they are no longer used.
- Update `.circleci/scope-mapping.txt` and `scope-exclude.txt` for every move and run `python3 scripts/check_scope_coverage.py`.

### 4.4 Config drift (item 17)
- `CLAUDE.md:91` still says Django 5.2, `pyproject.toml` pins `Django==6.0.8`: fix the doc.
- `[tool.bumpversion] current_version = "4.29.0-rc.7"` against `league_manager/__init__.py`'s `__version__ = "4.31.4"`. Re-measured 2026-09-27: it has drifted two minor versions, which **proves nothing reads it** — release-please owns versioning. So this is now a straight delete of the `[tool.bumpversion]` section plus `bump2version==1.0.1` from the test deps, not a decision.
- `[tool.setuptools.packages.find]` omits `gameday_designer`, `matchreport`, `journey` (still true): add them.
- `.env.demo` uses `MYSQL_PASSWORD`/`MYSQL_DATABASE`, `base.py:196,200` reads `MYSQL_DB_NAME`/`MYSQL_PWD`: pick one naming and use it in `base.py`, `demo.py`, `.env_template` and the compose files. Fold in 1.6's leftover while here — `base.py` still defaults to `test_db`/`user`, and only `prod.py`/`stage.py` refuse to start without real values.
- The 17 GitHub Actions workflows split into 14 deactivated CI ones (triggers commented out, `workflow_dispatch` only, "deactivated in favor of CircleCI" banner) and 3 `release-please*.yaml` that **are** live — release PRs are still being cut. So: delete the 14, keep the 3.

**Phase 4 done when:** `black --check`, the link checker and `check_scope_coverage.py` all run in CI and pass, and the root holds only config files, `README.md`, `CHANGELOG.md`, `LICENSE`, `AGENTS.md` and `CLAUDE.md`.

---

## Phase 5: Frontend and backend code quality — ⬜ Open

### 5.1 One frontend toolchain (item 18)
- **Where:** five apps with five `package.json`, five ESLint, Vite and Vitest configs; exact pins in `gameday_designer` and `journey_dashboard`, carets elsewhere; `fe_template/` still carries `webpack.config.js` and a legacy `.eslintrc.json`.
- **How:** Introduce an npm workspace at the root (`package.json` `workspaces`), a shared `eslint.config.mjs` and `vite.shared.mts` that each app extends, and let Renovate group the shared deps. Delete `fe_template/` or rebuild it as the workspace template. Keep JS + Redux in liveticker and scorecard for now; converting them to TypeScript is a separate decision.
- **Verify:** `npm run eslint` and `npm run test:run` pass from the root for all apps; CircleCI node jobs still pass with the path scoping.

### 5.2 Remove lint suppressions (item 18)
- Thirteen suppressions across the three rules as of 2026-09-27 (was 12 when the plan was written — it is growing). Nine `react-hooks/exhaustive-deps` disables: `gameday_designer/src/components/ListDesignerApp.tsx:335,369,383`, `.../dashboard/GamedayDashboard.tsx:319`, `passcheck/src/components/GameOverview.tsx:32`, `PlayerModal.tsx:46`, `RosterOverview.tsx:60,74,91`. Each hides a possible stale closure. Fix by moving the effect body into `useCallback` with correct deps, or by using a ref for the value that must not retrigger.
- `passcheck/src/utils/api.ts:1` disables `no-explicit-any` for the whole file: type the API responses (the DRF serializers define the shape) and remove the directive.
- Two `react-hooks/set-state-in-effect` disables in `GamedayMetadataAccordion.tsx:224` and `useFlowState.ts:142`: derive the value during render or use a reducer.

### 5.3 Split the giant files (item 18)
- `gameday_designer/src/hooks/useFlowValidation.ts` (1462 lines) into one module per validation rule with a small orchestrator.
- `gameday_designer/src/components/list/GameTable.tsx` (1005 lines) into row, header and editing components.
- Scorecard: replace the hard-coded `/api/` string literals in `scorecard/src` (7 by current count) with one `api.js` client module.

### 5.4 Django nits (item 19)
- `league_manager/urls.py:154` and `:157` mount `journey.urls` twice; keep the API under `/api/journey/` and the HTML dashboard under `/journeys/` by splitting `journey/urls.py` into `api_urls.py` and `urls.py`.
- `ClearCacheView` mutates state on GET: make it POST with a CSRF-protected form or button.
- `RegisterAPI` inherits `IsAuthenticatedOrReadOnly`, so only logged-in users can register: either set `AllowAny` on purpose or remove the endpoint.
- `Gameday.author` and the second FK at `gamedays/models.py:89` and `:292` use `on_delete=SET_DEFAULT, default=1`: switch to `SET_NULL` with `null=True`, or `PROTECT`.
- `league_manager/middleware/db_guard.py:54` bare `except:`: catch `NoReverseMatch`.
- Rename the "journey" concept or split the app: it is both user click tracking (`Journey`, `JourneyEvent`) and the game progress feed, and the module guide only describes the second.

### 5.5 Testing hygiene (item 20)
- `assertNumQueries` is used in 13 of 151 Python test files although CLAUDE.md calls it mandatory: add it to every API list and detail test, starting with `gamedays/tests/api/test_gameday_viewset.py`. Worth prioritising `/api/snapshot/` from [#1989](https://github.com/dachrisch/leaguesphere/pull/1989) — it is now the widest single query surface in the app.
- Split `journey/tests.py` (883 lines, up from 795) into a `journey/tests/` package by view.
- Remove `--capture=no` from `pytest.ini` addopts; use `-s` locally when needed.
- `scorecard/tests/e2e/` is empty but ignored in `pytest.ini`: delete the ignore or add the tests.

---

## Next steps

N1 to N3 are carry-overs: small, but they are the only remaining items with a security or consumer impact. N4 is the next full phase.

### N1 — Verify Phases 1 and 2 on stage, then promote (S, needs stage access)
Both PRs merged with this as an explicit post-deploy gate, so the hardening is **shipped but unproven**. After `./container/deploy.sh stage`:
- `curl -I https://stage.leaguesphere.app/` → `X-Frame-Options: DENY`, `Permissions-Policy` present, and **exactly one** each of `X-Content-Type-Options` and `Referrer-Policy` (Django's copies are suppressed with `proxy_hide_header`; two of either means the dedupe broke).
- `curl -I` a `/static/` asset → same headers. That inheritance was review must-fix 3 and nothing automated covers it.
- Log in through Traefik: secure cookies, HSTS and `SECURE_SSL_REDIRECT` are all new on stage. Confirm the session survives and that `/health/` still answers over plain HTTP (`SECURE_REDIRECT_EXEMPT`).
- Six bad passwords against `/login`, `/login/`, `/admin/login`, `/admin/login/` → 429 on the sixth; `/home/` never limited.
- Only then promote to prod.

### N2 — Close the loop with the snapshot consumer (S)
[#1989](https://github.com/dachrisch/leaguesphere/pull/1989) provides the endpoint; the client still has to use it.
- **Check this first:** nginx rewrites strong ETags to weak (`W/"…"`) when it gzips, which Phase 1 turned on for `application/json`. Django's `@condition` does weak comparison on GET so revalidation should still 304, but verify on stage with `curl --compressed -H 'If-None-Match: …'` against `/api/snapshot/` before any consumer depends on it.
- Switch the snapshot bot from catalog + per-gameday fan-out to a single `/api/snapshot/?include=games,logs` call that sends `If-None-Match`.
- Agreed with the maintainer: keep the daily run off-peak, and ping before scaling the scrape up.

### N3 — Rate-limit the Knox token login, Redis-free (S)
Carried over from 2.3, whose `django-axes` follow-up assumed Phase 3's shared cache. Two options that need no new service:
- **nginx `limit_req` on `/api/accounts/auth/login/`**, reusing the existing `login` zone. Minutes of work, matches what is already there, but no attempt logging and no username-based locking.
- **`django-axes` with `AXES_HANDLER = "axes.handlers.database.AxesDatabaseHandler"`**, which counts failures in the DB rather than the cache, so the shared-cache prerequisite disappears entirely. Costs a migration and a write per failed attempt; gives logging and username+IP locking.

Recommend the nginx one now, and axes only if attempt logging is actually wanted. Either way, drop the "after the shared cache" wording from 2.3 — it no longer means anything.

### N4 — Phase 4: process, docs and hygiene
Cheapest real work left, and it protects everything after it. Suggested order, one PR each: **4.2** (docs map plus a link checker in CI — smallest, stops the rot) → **4.1** (`black .` plus the CI gate — one large mechanical commit, land it when no big PR is in flight) → **4.4** (config drift, now mostly deletes) → **4.3** (root tidy — biggest diff, most conflict-prone, needs `scope-mapping.txt` updates).

### N5 — Phase 5: code quality
Unchanged in substance. Best value per line: **5.4**'s journey double-mount and **5.2**'s suppressions (13 and growing). **5.1**'s npm workspace is the one L-effort item and can wait; **5.5**'s `assertNumQueries` work should now include `/api/snapshot/`.

---

## Working agreements for every phase

- One PR per numbered item or per phase sub-section, never one PR per phase. Conventional commit prefixes (`fix:`, `chore:`, `refactor:`).
- TDD as in CLAUDE.md: write the failing test first for every code change in Phases 1 to 3.
- Before pushing: `black .`, `pytest`, `npm run eslint` and `npm run test:run` in each touched app, `python3 scripts/check_scope_coverage.py`.
- Every change is verified on stage (`./container/deploy.sh stage`) before merging to master; infra changes go through the container repo's Ansible, never by editing a server.

## Explicitly out of scope for now

- **Redis, or any shared-cache service.** Decided 2026-09-27, not a deferral — see the Phase 3 decision block. Cross-worker state goes file-based or DB-backed.
- A full Content-Security-Policy (needs SRI or nonces for the 44 CDN references first; revisit after 5.1).
- Converting liveticker and scorecard from JavaScript + Redux to TypeScript.
- Replacing pandas in the standings and match report code paths.