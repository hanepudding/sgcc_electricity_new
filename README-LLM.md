# README-LLM.md

> Supplementary architectural document for LLMs. We assume you have already read [README.md](README.md) — this document does NOT repeat what's covered there (sensor list, config examples, deployment steps, etc.). Instead, it provides internal architecture details to help you onboard quickly.

## File Map

```
scripts/
  main.py              Entry point. Config loading, scheduling, retry loop.
  data_fetcher.py      Core scraper. Selenium automation, page navigation, data extraction.
  captcha_selenium.py  Captcha orchestrator. Detects click vs slider type, coordinates solving.
  click_captcha_solver.py  LLM vision solver for click-type captcha. Icon splitting, coordinate parsing.
  sensor_updator.py    HA REST API client. Sensor updates, cache, republish.
  vue_state.py         JS injection to extract Vue 2 component state from 95598.cn pages.
  const.py             Constants: URLs, sensor names, LLM config. Read at import time.
  notify.py            Push notifications (Pushplus, URL push, QR code upload).
  error_watcher.py     Decorator that screenshots browser on exception.
  db.py                Optional SQLite/MySQL storage.
  onnx.py              Legacy ONNX captcha solver. NOT used in active code path. Kept for reference.

config.yaml            HA add-on manifest (version, arch, UI config schema).
Dockerfile-for-github-action   Production Dockerfile. Python 3.12 + CloakBrowser.
docker-compose.yml     Standalone Docker deployment.
example.env            All env vars with comments.
DOCS.md                HA add-on documentation (displayed by Supervisor UI).
```

## Data Flow

```
main.py
 ├─ Load config: .env or /data/options.json → os.environ
 ├─ Try republish() from cache → if success, skip initial scrape
 ├─ schedule: run twice daily (JOB_START_TIME ± 10min, +12h)
 │   └─ schedule: republish() every 5 minutes
 └─ run_task(fetcher) with retry (default 5 attempts)

data_fetcher.fetch()
 ├─ _get_webdriver()         # CloakBrowser Chromium + stealth args, headless in Docker
 ├─ _login(driver)
 │   ├─ Navigate to 95598.cn, fill phone + password
 │   ├─ captcha_selenium.solve_captcha_in_browser()
 │   │   ├─ Detect type via JS injection (_detect_captcha_type_js)
 │   │   ├─ Click type → click_captcha_solver.solve()
 │   │   │   ├─ Download reference strip → split into 3 icons (PIL)
 │   │   │   ├─ Download main image → convert to data URI
 │   │   │   ├─ Single LLM call: find all 3 icons, return normalized coords
 │   │   │   └─ Parse JSON {"coords":[[x,y],[x,y],[x,y]]}
 │   │   ├─ Slider type → LLM identifies gap ratio → simulate_drag()
 │   │   └─ Scale coords to DOM element → ActionChains click
 │   └─ Fallback: QR code login (notify URL → user scans)
 ├─ _get_user_ids(driver)    # Supports multiple 户号 under one login
 └─ For each user_id:
     ├─ _get_electric_balance()       # BALANCE_URL page, DOM scrape
     ├─ vue_state.normalize_balance() # Enhanced: 应交金额 via Vue injection
     ├─ _get_yearly_data()            # ELECTRIC_USAGE_URL, yearly tab
     ├─ _get_month_usage()            # Monthly table, numpy reshape
     ├─ _get_yesterday_usage()        # Daily tab, first row
     ├─ vue_state.normalize_usage()   # TOU breakdown (valley/flat/peak/tip)
     ├─ _save_user_data() to DB       # If DB_TYPE configured
     └─ sensor_updator.update_one_userid()
         ├─ _save_to_cache()          # /data/sgcc_cache.json
         └─ POST /api/states/<entity_id> for each sensor
```

## Config Injection (Internal)

Three paths, evaluated in order:

1. **`.env`** — `python-dotenv` loads this when `PYTHON_IN_DOCKER` is NOT set (local dev).
2. **`/data/options.json`** — HA Supervisor writes add-on UI config here. `main.py` reads the file and sets `os.environ` entries **manually** (NOT automatic env injection). Then explicitly refreshes `const.LLM_*` attributes.
3. **`os.environ`** — Docker `-e` flags, `docker-compose.yml` environment block.

**Critical:** `const.py` reads `LLM_*` at **import time**. After `/data/options.json` loads, `main.py` must refresh:
```python
const.LLM_API_KEY = os.getenv('LLM_API_KEY', '').strip()
const.LLM_BASE_URL = os.getenv('LLM_BASE_URL', '').strip()
const.LLM_MODEL = os.getenv('LLM_MODEL', '').strip()
```
Other modules reference `const.LLM_*` (module-level attribute lookup), so they see the refreshed values.

## Browser Stack

- **CloakBrowser**: Chromium fork with C++ source-level anti-detection. Binary installed at Docker build time (`python3 -m cloakbrowser install`).
- **Auto-update disabled**: `ENV CLOAKBROWSER_AUTO_UPDATE=false` in Dockerfile. Without this, `ensure_binary()` checks PyPI + GitHub on every startup (~206MB download).
- **Stealth args**: `cloakbrowser.get_default_stealth_args()` returns fingerprint randomization flags passed to ChromeOptions.
- Headless in Docker (`--headless=new`), headed locally.

## Captcha Pipeline

95598.cn uses Tencent captcha. Two types detected via JS injection:

**Click type**: Reference strip (3 icons) + main grid image → LLM returns center coords for each icon → ActionChains click in order → confirm button.

**Slider type**: Background image with puzzle gap → LLM returns gap X-ratio (0~1) → mapped to pixel distance → simulated human drag (3-5 segments, random pauses).

If wrong type detected, auto-refreshes captcha up to 5 times.

## HA Sensor Update Internals

- `POST /api/states/{entity_id}` with JSON body (state, attributes including unit/device_class/state_class/icon/last_reset).
- **Dedup**: `should_update()` GETs current state before POSTing. Skips if unchanged.
- **Cache**: `/data/sgcc_cache.json`. Written after every successful fetch. `republish()` every 5min re-POSTs cached data. Prevents data loss on HA restart.
- **Startup guard**: If cache exists and republish succeeds, skips initial scrape. Protects account from frequent logins on container restarts.

## Build & CI

- **Dockerfile**: `python:3.12.11-slim-bookworm`. System deps for headless Chromium, pip install from `requirements.txt`, `cloakbrowser install`.
- **CI**: `.github/workflows/docker-image.yml`. Triggers on `release: published` or `workflow_dispatch`. Matrix build: amd64 + aarch64 (QEMU). Pushes to `ghcr.io/hanepudding/sgcc_electricity-{arch}:{version}`.
- **Version flow**: Release tag → CI reads tag → updates `config.yaml` version → commits → builds images.

## Known Issues / Gotchas

1. **db.py incomplete API**: `data_fetcher._save_user_data()` calls `upsert_user()`, `insert_daily_data()`, `insert_monthly_data()`, `insert_yearly_data()`, `insert_balance_log()`, `cleanup_old_data()` — these methods do NOT exist on `SqliteDB` / `MysqlDB` classes. Code only survives because default `DB_TYPE=NONE` means `self.db` is `None` and the entire save path is skipped. **Enabling DB will crash.**
2. **`send_url` Content-Type typo**: `sensor_updator.py` has `"Content-Type": "application-json"` (missing `/`). Works because `requests` overrides the header when `json=` param is used.
3. **`input()` in debug mode**: `data_fetcher._login()` calls `input("请输入手机验证码")` when `DEBUG_MODE=true`. Will hang in Docker (no stdin).
4. **`onnx.py` unused**: Legacy ONNX solver. Not imported anywhere in active code. Safe to ignore.
5. **January edge case**: `_get_yearly_data()` and `_get_month_usage()` switch to previous year when `month == 1`.
6. **`RETRY_WAIT_TIME_OFFSET_UNIT` dominates runtime**: Used as `time.sleep()` between almost every Selenium operation. At default 15s, a full scrape takes 5-10 minutes per user.
