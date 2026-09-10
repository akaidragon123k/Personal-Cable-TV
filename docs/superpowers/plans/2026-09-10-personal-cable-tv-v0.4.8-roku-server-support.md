# Personal Cable TV v0.4.8 Roku Server Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add secure, LAN-first Roku Companion support to Personal Cable TV v0.4.8 without changing the working v0.4.7 Jellyfin Live TV, M3U/XMLTV, scheduling, media-safety, or live-position behavior.

**Architecture:** Keep v0.4.7 as the rollback baseline. Add isolated Roku persistence and security modules, a protected `/api/roku/*` HTTP surface, signed short-lived artwork/playback tickets, and a Settings/Help section. Reuse `build_guide_model()` and `CableTVApplication.stream_channel()` so Roku is an additive client rather than a new scheduling or playback engine.

**Tech Stack:** Python 3.13 standard library, SQLite, `http.server.ThreadingHTTPServer`, `hashlib.scrypt`, `secrets`, `hmac`, Jellyfin HTTP API, Docker/CasaOS.

**Spec:** `docs/superpowers/specs/2026-09-10-roku-personal-cable-tv-companion-design.md`

## Global Constraints

- Server release is exactly **Personal Cable TV v0.4.8**.
- **Personal Cable TV v0.4.7** remains the known-good rollback baseline.
- Do not modify or delete real NAS media; media remains read-only.
- Do not change existing Jellyfin M3U/XMLTV behavior, channel scheduling, Full Library Mode, seasonal behavior, or live-position playback.
- Do not expose the Jellyfin API key to Roku responses, URLs, logs, or UI.
- Roku Companion is LAN-only by default and still requires authentication.
- Companion Password minimum length is 8 characters and is stored as a salted hash, never plaintext.
- Usable Roku device tokens are stored only on Roku; the server stores only token digests.
- Existing authorized Roku devices survive password changes until individually revoked or Revoke All Devices is used.
- Roku playback URLs use short-lived signed tickets; the long-lived device token is never placed in a media URL.
- Help & Support must be updated in the same v0.4.8 release.

---

## File Structure Map

Server package working tree is created by extracting `Personal_Cable_TV_Docker_v0.4.7.zip` and renaming the root directory to `Personal_Cable_TV_Docker_v0.4.8` before implementation.

- `app/cabletv/db.py` — additive Roku tables, migration backup, Roku device/config CRUD.
- `app/cabletv/roku_security.py` — password hashing, token digests, LAN checks, authorization service.
- `app/cabletv/roku_companion.py` — guide/details serialization, Jellyfin metadata/image proxy helpers, signed temporary tickets.
- `app/cabletv/jellyfin.py` — read-only single-item metadata and image-byte fetch helpers used internally by the companion.
- `app/cabletv/web.py` — instantiate Roku services, expose `/api/roku/*` routes, Settings UI, Help & Support.
- `app/cabletv/__init__.py` — version `0.4.8`.
- `tests/test_roku_database.py` — migration, backup, config/device persistence.
- `tests/test_roku_security.py` — scrypt/password, token, LAN, revoke semantics.
- `tests/test_roku_companion.py` — guide/details/ticket payloads and secret-leak checks.
- `tests/test_roku_http.py` — route auth, JSON behavior, playback/artwork tickets.
- `tests/test_v047_regression.py` — M3U/XMLTV and live-position regression coverage.
- `README.md`, `README_FIRST.txt` — v0.4.8 Roku setup and safety notes inside the release ZIP.
- Repository root `.github/workflows/docker-publish.yml`, `casaos/docker-compose.yml`, `README.md` — public v0.4.8 publishing/install docs after local verification passes.

---

### Task 1: Add additive Roku database schema and pre-migration backup

**Files:**
- Modify: `app/cabletv/db.py:1-190`
- Create: `tests/test_roku_database.py`

**Interfaces:**
- Produces: `Database.get_roku_config() -> dict`, `Database.set_roku_config(...) -> None`, `Database.upsert_roku_device(...) -> dict`, `Database.get_roku_device(device_id: str) -> dict | None`, `Database.get_roku_device_by_token_digest(digest: str) -> dict | None`, `Database.touch_roku_device(device_id: str, ip_address: str) -> None`, `Database.list_roku_devices(include_revoked: bool=False) -> list[dict]`, `Database.revoke_roku_device(device_id: str) -> None`, `Database.revoke_all_roku_devices() -> int`.
- Produces migration backup path pattern: `<data_dir>/backups/cabletv-pre-v0.4.8-roku-YYYYMMDDTHHMMSSZ.sqlite3`.

- [ ] **Step 1: Write failing migration and CRUD tests**

```python
# tests/test_roku_database.py
import sqlite3
import tempfile
import unittest
from pathlib import Path

from cabletv.db import Database


class RokuDatabaseTests(unittest.TestCase):
    def setUp(self):
        self.tmp = tempfile.TemporaryDirectory()
        self.addCleanup(self.tmp.cleanup)
        self.root = Path(self.tmp.name)

    def test_existing_database_is_backed_up_before_roku_schema_is_added(self):
        path = self.root / "cabletv.sqlite3"
        conn = sqlite3.connect(path)
        conn.execute("CREATE TABLE settings (key TEXT PRIMARY KEY, value TEXT NOT NULL)")
        conn.execute("INSERT INTO settings(key,value) VALUES('sentinel','keep-me')")
        conn.commit()
        conn.close()

        db = Database(path)
        db.initialize()

        backups = list((self.root / "backups").glob("cabletv-pre-v0.4.8-roku-*.sqlite3"))
        self.assertEqual(len(backups), 1)
        copy = sqlite3.connect(backups[0])
        self.assertEqual(copy.execute("SELECT value FROM settings WHERE key='sentinel'").fetchone()[0], "keep-me")
        copy.close()

    def test_roku_config_defaults_disabled_and_lan_only(self):
        db = Database(self.root / "fresh.sqlite3")
        db.initialize()
        cfg = db.get_roku_config()
        self.assertFalse(cfg["enabled"])
        self.assertTrue(cfg["lan_only"])
        self.assertIsNone(cfg["password_hash"])
        self.assertIsNone(cfg["password_salt"])

    def test_roku_device_token_digest_is_persisted_and_revocable(self):
        db = Database(self.root / "fresh.sqlite3")
        db.initialize()
        row = db.upsert_roku_device("roku-client-1", "Living Room Roku", "digest-1", "192.168.1.20")
        self.assertEqual(row["device_uid"], "roku-client-1")
        self.assertEqual(db.get_roku_device_by_token_digest("digest-1")["name"], "Living Room Roku")
        db.revoke_roku_device(row["id"])
        self.assertIsNone(db.get_roku_device_by_token_digest("digest-1"))
```

- [ ] **Step 2: Run the focused test and verify failure**

Run: `PYTHONPATH=app python -m unittest tests.test_roku_database -v`

Expected: FAIL because the Roku schema/CRUD methods do not exist.

- [ ] **Step 3: Implement the additive schema, backup, and CRUD**

Add two tables to `SCHEMA`:

```sql
CREATE TABLE IF NOT EXISTS roku_companion_config (
    id INTEGER PRIMARY KEY CHECK(id=1),
    enabled INTEGER NOT NULL DEFAULT 0,
    lan_only INTEGER NOT NULL DEFAULT 1,
    password_salt TEXT,
    password_hash TEXT,
    password_n INTEGER NOT NULL DEFAULT 16384,
    password_r INTEGER NOT NULL DEFAULT 8,
    password_p INTEGER NOT NULL DEFAULT 1,
    updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
INSERT OR IGNORE INTO roku_companion_config(id) VALUES(1);
CREATE TABLE IF NOT EXISTS roku_devices (
    id TEXT PRIMARY KEY,
    device_uid TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    token_digest TEXT NOT NULL UNIQUE,
    ip_address TEXT,
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_seen_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    revoked_at TEXT
);
CREATE INDEX IF NOT EXISTS idx_roku_devices_token_digest ON roku_devices(token_digest);
```

At the beginning of `Database.initialize()`, before any v0.4.8 schema write, detect an existing non-empty database that does not yet contain `roku_companion_config`; create `backups/`, then use SQLite's online backup API to copy the database to the timestamped v0.4.8 rollback file. Do not create a migration backup for a fresh database or on subsequent v0.4.8 starts.

Add CRUD methods with `uuid.uuid4().hex` for new record IDs. `get_roku_device_by_token_digest()` must require `revoked_at IS NULL` and update `last_seen_at` only after successful authentication through the service, not as a side effect of lookup.

- [ ] **Step 4: Run the focused tests and verify pass**

Run: `PYTHONPATH=app python -m unittest tests.test_roku_database -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/cabletv/db.py tests/test_roku_database.py
git commit -m "feat: add Roku companion database storage"
```

---

### Task 2: Implement password hashing, device tokens, LAN checks, and revoke semantics

**Files:**
- Create: `app/cabletv/roku_security.py`
- Create: `tests/test_roku_security.py`
- Modify: `app/cabletv/db.py` only if a small last-seen helper is needed.

**Interfaces:**
- Consumes: Task 1 `Database` Roku CRUD.
- Produces: `is_lan_address(address: str) -> bool`.
- Produces: `RokuSecurityService.set_password(password: str) -> None`.
- Produces: `RokuSecurityService.authorize(password: str, device_uid: str, device_name: str, ip_address: str) -> dict` returning the usable token once.
- Produces: `RokuSecurityService.authenticate(token: str, ip_address: str) -> dict` returning the active device row.
- Produces: `RokuSecurityService.revoke(device_id: str) -> None` and `revoke_all() -> int`.

- [ ] **Step 1: Write failing security tests**

```python
# tests/test_roku_security.py
import tempfile
import unittest
from pathlib import Path

from cabletv.db import Database
from cabletv.roku_security import RokuSecurityService, is_lan_address


class RokuSecurityTests(unittest.TestCase):
    def setUp(self):
        self.tmp = tempfile.TemporaryDirectory()
        self.addCleanup(self.tmp.cleanup)
        self.db = Database(Path(self.tmp.name) / "cabletv.sqlite3")
        self.db.initialize()
        self.svc = RokuSecurityService(self.db)

    def test_password_is_hashed_and_plaintext_is_not_stored(self):
        self.svc.set_password("strongpass")
        cfg = self.db.get_roku_config()
        self.assertNotEqual(cfg["password_hash"], "strongpass")
        self.assertNotEqual(cfg["password_salt"], "strongpass")

    def test_password_requires_eight_characters(self):
        with self.assertRaises(ValueError):
            self.svc.set_password("short")

    def test_device_token_is_returned_once_but_only_digest_is_in_database(self):
        self.svc.set_password("strongpass")
        auth = self.svc.authorize("strongpass", "roku-1", "Living Room Roku", "192.168.1.20")
        self.assertGreaterEqual(len(auth["token"]), 32)
        rows = self.db.list_roku_devices()
        self.assertNotIn(auth["token"], repr(rows))
        self.assertEqual(self.svc.authenticate(auth["token"], "192.168.1.20")["device_uid"], "roku-1")

    def test_password_change_does_not_revoke_existing_device(self):
        self.svc.set_password("firstpass")
        auth = self.svc.authorize("firstpass", "roku-1", "TV", "192.168.1.20")
        self.svc.set_password("secondpass")
        self.assertEqual(self.svc.authenticate(auth["token"], "192.168.1.20")["device_uid"], "roku-1")

    def test_lan_detection_blocks_public_address(self):
        self.assertTrue(is_lan_address("192.168.1.20"))
        self.assertTrue(is_lan_address("172.18.0.1"))
        self.assertTrue(is_lan_address("127.0.0.1"))
        self.assertFalse(is_lan_address("8.8.8.8"))
```

- [ ] **Step 2: Run the focused test and verify failure**

Run: `PYTHONPATH=app python -m unittest tests.test_roku_security -v`

Expected: FAIL because `roku_security.py` does not exist.

- [ ] **Step 3: Implement security primitives**

Use only Python standard-library cryptography primitives:

```python
# app/cabletv/roku_security.py
import base64
import hashlib
import hmac
import ipaddress
import secrets

SCRYPT_N = 16384
SCRYPT_R = 8
SCRYPT_P = 1
SCRYPT_DKLEN = 32


def _b64(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).decode("ascii")


def _unb64(value: str) -> bytes:
    return base64.urlsafe_b64decode(value.encode("ascii"))


def token_digest(token: str) -> str:
    return hashlib.sha256(token.encode("utf-8")).hexdigest()


def is_lan_address(address: str) -> bool:
    ip = ipaddress.ip_address(address.split("%", 1)[0])
    return ip.is_private or ip.is_loopback or ip.is_link_local
```

`set_password()` generates `secrets.token_bytes(16)` salt and derives 32 bytes with `hashlib.scrypt(password.encode(), salt=salt, n=16384, r=8, p=1, dklen=32)`; persist base64 salt/hash and parameters. Verification recomputes and uses `hmac.compare_digest()`.

`authorize()` must reject when the companion is disabled, the source is not LAN, password is wrong, or required device fields are blank. In v0.4.8 LAN-only is mandatory even though the database keeps a `lan_only` field for forward compatibility. On success create `secrets.token_urlsafe(32)`, store only `token_digest(token)`, and return `{device_id, token, device_name}`.

`authenticate()` hashes the presented token, loads only non-revoked rows, enforces LAN-only unconditionally for v0.4.8, updates last-seen/IP through `touch_roku_device()`, and returns device metadata. Never log or return the password hash, salt, or token digest.

- [ ] **Step 4: Run focused security and database tests**

Run: `PYTHONPATH=app python -m unittest tests.test_roku_security tests.test_roku_database -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/cabletv/roku_security.py app/cabletv/db.py tests/test_roku_security.py
git commit -m "feat: secure Roku companion authorization"
```

---

### Task 3: Build Roku Guide and program-details payloads without exposing Jellyfin secrets

**Files:**
- Create: `app/cabletv/roku_companion.py`
- Modify: `app/cabletv/jellyfin.py:81-220`
- Create: `tests/test_roku_companion.py`

**Interfaces:**
- Consumes: `build_guide_model(db, category, start, end, now, tz_name)`.
- Produces: `RokuCompanionService.categories() -> dict`.
- Produces: `RokuCompanionService.guide(category: str, start: datetime | None, hours: float=3.0) -> dict`.
- Produces: `RokuCompanionService.program_details(schedule_id: int) -> dict`.
- Produces: `JellyfinClient.fetch_item_details(item_id: str) -> dict`.

- [ ] **Step 1: Write failing payload tests**

```python
# tests/test_roku_companion.py
import tempfile
import unittest
from datetime import datetime, timezone
from pathlib import Path

from cabletv.db import Database
from cabletv.roku_companion import RokuCompanionService


class RokuCompanionPayloadTests(unittest.TestCase):
    def test_categories_end_with_all_channels(self):
        db = Database(Path(tempfile.mkdtemp()) / "db.sqlite3")
        db.initialize()
        svc = RokuCompanionService(db, "UTC")
        payload = svc.categories()
        self.assertEqual(payload["categories"][-1], "All Channels")

    def test_guide_contains_channel_identity_and_now_metadata(self):
        db = Database(Path(tempfile.mkdtemp()) / "db.sqlite3")
        db.initialize()
        channel_id = db.create_channel(101, "Movies", "all_movies", False)
        now = datetime.now(timezone.utc)
        payload = RokuCompanionService(db, "UTC").guide("All Channels", now.replace(minute=0, second=0, microsecond=0), 3.0)
        self.assertIn("server_now_utc", payload)
        self.assertEqual(payload["category"], "All Channels")
        self.assertTrue(any(row["id"] == channel_id for row in payload["channels"]))

    def test_payload_never_contains_jellyfin_api_key(self):
        db = Database(Path(tempfile.mkdtemp()) / "db.sqlite3")
        db.initialize()
        db.set_setting("full_jellyfin_api_key", "SECRET-KEY-123")
        payload = RokuCompanionService(db, "UTC").categories()
        self.assertNotIn("SECRET-KEY-123", repr(payload))
```

- [ ] **Step 2: Run the focused test and verify failure**

Run: `PYTHONPATH=app python -m unittest tests.test_roku_companion -v`

Expected: FAIL because `RokuCompanionService` does not exist.

- [ ] **Step 3: Implement `RokuCompanionService` guide serialization**

Use `GUIDE_CATEGORIES` as the server-owned dynamic source; Roku never hard-codes those labels. Clamp guide windows to 3 hours to match current `build_guide_model()` behavior. Add `server_now_utc`, and preserve each channel's internal `id`, display `number`, `name`, `guide_category`, `seasonal`, and program schedule IDs/start/end/current flags.

For `program_details(schedule_id)`, locate the schedule row with a new read-only `Database.get_schedule_row(schedule_id)` helper using the same join columns as `list_schedule()`. Return local DB fields immediately. For Full Library items, call a new read-only `JellyfinClient.fetch_item_details()` internally using the saved Jellyfin server/API key, requesting only:

```python
params = {
    "Fields": "Overview,OfficialRating,CommunityRating,RunTimeTicks,ProductionYear,ImageTags"
}
```

Return `overview`, `official_rating`, `community_rating`, `year`, `runtime_seconds`, `series_name`, season/episode numbers, and an internal media ID that can later be exchanged for a temporary artwork URL. Do not return Jellyfin base URL, API key, media source ID, or raw Jellyfin playback URL.

- [ ] **Step 4: Run payload tests**

Run: `PYTHONPATH=app python -m unittest tests.test_roku_companion -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/cabletv/roku_companion.py app/cabletv/jellyfin.py app/cabletv/db.py tests/test_roku_companion.py
git commit -m "feat: add Roku guide and program payloads"
```

---

### Task 4: Add short-lived signed playback and artwork tickets

**Files:**
- Modify: `app/cabletv/roku_companion.py`
- Create: `tests/test_roku_tickets.py`

**Interfaces:**
- Produces: `RokuCompanionService.issue_playback_ticket(device_id: str, channel_id: int, base_url: str) -> dict`.
- Produces: `RokuCompanionService.verify_playback_ticket(ticket: str, channel_id: int) -> dict`.
- Produces: `RokuCompanionService.issue_artwork_ticket(device_id: str, media_id: int, base_url: str) -> dict`.
- Produces: `RokuCompanionService.verify_artwork_ticket(ticket: str, media_id: int) -> dict`.
- Playback ticket lifetime: exactly **120 seconds** for v0.4.8 initial implementation.
- Artwork ticket lifetime: exactly **300 seconds**.

- [ ] **Step 1: Write failing ticket tests**

```python
# tests/test_roku_tickets.py
import tempfile
import unittest
from pathlib import Path

from cabletv.db import Database
from cabletv.roku_companion import RokuCompanionService, TicketError


class RokuTicketTests(unittest.TestCase):
    def setUp(self):
        self.db = Database(Path(tempfile.mkdtemp()) / "db.sqlite3")
        self.db.initialize()
        device = self.db.upsert_roku_device("roku-1", "Living Room Roku", "digest-1", "192.168.1.20")
        self.device_id = device["id"]
        self.svc = RokuCompanionService(self.db, "UTC", signing_key=b"x" * 32, clock=lambda: 1_700_000_000)

    def test_playback_ticket_is_scoped_to_channel(self):
        ticket = self.svc.issue_playback_ticket(self.device_id, 7, "http://pctv:8484")
        self.assertIn("/api/roku/stream/7.ts?ticket=", ticket["url"])
        self.assertEqual(self.svc.verify_playback_ticket(ticket["ticket"], 7)["device_id"], self.device_id)
        with self.assertRaises(TicketError):
            self.svc.verify_playback_ticket(ticket["ticket"], 8)

    def test_expired_ticket_is_rejected(self):
        ticket = self.svc.issue_playback_ticket(self.device_id, 7, "http://pctv:8484")
        self.svc.clock = lambda: 1_700_000_121
        with self.assertRaises(TicketError):
            self.svc.verify_playback_ticket(ticket["ticket"], 7)
```

- [ ] **Step 2: Run and verify failure**

Run: `PYTHONPATH=app python -m unittest tests.test_roku_tickets -v`

Expected: FAIL because ticket methods do not exist.

- [ ] **Step 3: Implement stateless HMAC tickets**

At service construction, generate an ephemeral 32-byte signing key with `secrets.token_bytes(32)` unless a test key is injected. Ticket payload is compact JSON containing `kind`, `device_id`, resource ID, `exp`, and a random nonce; encode with URL-safe base64 without padding and append an HMAC-SHA256 signature. Verification must use `hmac.compare_digest`, enforce `kind`, resource ID, expiration, and verify that `device_id` still refers to a non-revoked device before allowing the request.

Do not persist the signing key: server restart intentionally invalidates only temporary media URLs, not device authorization.

- [ ] **Step 4: Run ticket and security tests**

Run: `PYTHONPATH=app python -m unittest tests.test_roku_tickets tests.test_roku_security -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/cabletv/roku_companion.py tests/test_roku_tickets.py
git commit -m "feat: add temporary Roku media tickets"
```

---

### Task 5: Expose the protected Roku HTTP API and protected stream/artwork routes

**Files:**
- Modify: `app/cabletv/web.py:33-60, 879-1015`
- Modify: `app/cabletv/roku_companion.py`
- Modify: `app/cabletv/jellyfin.py`
- Create: `tests/test_roku_http.py`

**Interfaces:**
- Public LAN endpoint: `GET /api/roku/health`.
- Public LAN authorization endpoint: `POST /api/roku/authorize` JSON.
- Protected bearer endpoints: `GET /api/roku/categories`, `GET /api/roku/guide`, `GET /api/roku/program/<schedule_id>`, `POST /api/roku/playback`.
- Signed temporary endpoints: `GET /api/roku/stream/<channel_id>.ts?ticket=...`, `GET /api/roku/artwork/<media_id>?ticket=...`.
- Device auth header: `Authorization: Bearer <device-token>`.

- [ ] **Step 1: Write failing HTTP contract tests**

```python
# tests/test_roku_http.py
import json
import threading
import urllib.error
import urllib.request
import tempfile
import unittest
from pathlib import Path

from cabletv.web import CableTVApplication, create_server


class RokuHttpTests(unittest.TestCase):
    def setUp(self):
        root = Path(tempfile.mkdtemp())
        self.app = CableTVApplication(root, root / "media")
        self.server = create_server("127.0.0.1", 0, self.app)
        self.thread = threading.Thread(target=self.server.serve_forever, daemon=True)
        self.thread.start()
        self.base = f"http://127.0.0.1:{self.server.server_address[1]}"
        self.addCleanup(self.server.shutdown)
        self.addCleanup(self.server.server_close)

    def request_json(self, path, method="GET", body=None, token=None):
        headers = {"Accept": "application/json"}
        data = None
        if body is not None:
            data = json.dumps(body).encode("utf-8")
            headers["Content-Type"] = "application/json"
        if token:
            headers["Authorization"] = f"Bearer {token}"
        req = urllib.request.Request(self.base + path, data=data, headers=headers, method=method)
        with urllib.request.urlopen(req, timeout=2) as response:
            return response.status, json.loads(response.read().decode("utf-8"))

    def test_health_does_not_leak_secrets(self):
        self.app.db.set_setting("full_jellyfin_api_key", "TOPSECRET")
        status, payload = self.request_json("/api/roku/health")
        self.assertEqual(status, 200)
        self.assertNotIn("TOPSECRET", repr(payload))

    def test_protected_route_rejects_missing_bearer_token(self):
        with self.assertRaises(urllib.error.HTTPError) as ctx:
            self.request_json("/api/roku/categories")
        self.assertEqual(ctx.exception.code, 401)
```

- [ ] **Step 2: Run HTTP tests and verify failure**

Run: `PYTHONPATH=app python -m unittest tests.test_roku_http -v`

Expected: FAIL/404 because Roku routes do not exist.

- [ ] **Step 3: Add HTTP helpers and routes**

In `_Handler`, add `_json_body()` that requires `application/json`, decodes a JSON object, and raises `ValueError` on malformed/non-object bodies. Add `_bearer_token()` that parses the `Authorization` header and returns the token or `None`.

Route behavior:

```text
GET  /api/roku/health
  200 {service:"roku-companion", server_version:"0.4.8", enabled:<bool>, auth_required:true, lan_only:true}

POST /api/roku/authorize
  JSON {password, device_uid, device_name}
  200 {device_id, device_name, token}

GET /api/roku/categories
  Bearer token required
  200 {categories:[...]}

GET /api/roku/guide?category=All%20Channels&start=<ISO8601>&hours=3
  Bearer token required
  200 guide payload

GET /api/roku/program/<schedule_id>
  Bearer token required
  200 program payload; include a 300-second signed `artwork_url` when artwork is available

POST /api/roku/playback
  Bearer token required
  JSON {channel_id}
  200 {url, expires_at}
```

For `/api/roku/stream/<id>.ts`, verify the 120-second ticket, verify the device is still active, verify the channel is active, then delegate byte production to the existing `self.app.stream_channel(channel_id)` logic. Do not duplicate playback resolution.

For `/api/roku/artwork/<media_id>`, verify the artwork ticket, load the media row, and proxy the Primary Jellyfin image through a new `JellyfinClient.fetch_primary_image_bytes(item_id, max_width=480)` read-only helper. Send the returned image MIME type and bytes; return 404 when no artwork is available.

Error status contract: 400 malformed request, 401 missing/invalid/revoked device token, 403 non-LAN source address, 404 missing resource, 410 expired media ticket, 503 unavailable scheduled/Jellyfin playback.

- [ ] **Step 4: Run HTTP, ticket, payload, and regression tests**

Run: `PYTHONPATH=app python -m unittest tests.test_roku_http tests.test_roku_tickets tests.test_roku_companion tests.test_roku_security -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/cabletv/web.py app/cabletv/roku_companion.py app/cabletv/jellyfin.py tests/test_roku_http.py
git commit -m "feat: expose protected Roku companion API"
```

---

### Task 6: Add Roku Companion Settings controls and device management UI

**Files:**
- Modify: `app/cabletv/web.py:676-865, 974-1015`
- Create: `tests/test_roku_settings_ui.py`

**Interfaces:**
- POST `/api/roku/settings` form fields: `enabled`, optional `password`. v0.4.8 displays LAN-only as required/ON and does not accept a form value that disables it.
- POST `/api/roku/devices/<device_id>/revoke`.
- POST `/api/roku/devices/revoke-all`.

- [ ] **Step 1: Write failing UI tests**

```python
# tests/test_roku_settings_ui.py
import tempfile
import unittest
from pathlib import Path
from cabletv.web import CableTVApplication


class RokuSettingsUiTests(unittest.TestCase):
    def test_settings_page_contains_roku_companion_security_controls(self):
        root = Path(tempfile.mkdtemp())
        app = CableTVApplication(root, root / "media")
        page = app.dashboard_html("http://127.0.0.1:8484")
        self.assertIn("Roku Personal Cable TV Companion", page)
        self.assertIn("Enable Roku Companion", page)
        self.assertIn("LAN-only access", page)
        self.assertIn("Authorized Roku Devices", page)
        self.assertIn("Revoke All Devices", page)
        self.assertNotIn("password_hash", page)
        self.assertNotIn("token_digest", page)
```

- [ ] **Step 2: Run UI test and verify failure**

Run: `PYTHONPATH=app python -m unittest tests.test_roku_settings_ui -v`

Expected: FAIL because the new UI is absent.

- [ ] **Step 3: Implement the Settings card and POST handlers**

Under the existing Settings section, keep the existing Jellyfin Companion card and add a separate **Roku Personal Cable TV Companion** card. Show enable state, `LAN-only access: On (required in v0.4.8)` as a non-editable security status, password field labeled `Set / Change Companion Password` with no existing value filled in, authorized device rows with name/IP/last connected, per-device Revoke button, and Revoke All Devices.

Rules enforced server-side:

- Enabling when no password hash exists requires a new password of at least 8 characters.
- Leaving password blank preserves the current password.
- Changing password does not revoke devices.
- Disabling Roku Companion blocks protected requests but does not silently delete authorization records.
- v0.4.8 must not provide a UI or API path that disables LAN-only enforcement; remote companion access remains out of scope.
- Revoke actions are POST-only.

- [ ] **Step 4: Run Settings and security tests**

Run: `PYTHONPATH=app python -m unittest tests.test_roku_settings_ui tests.test_roku_security tests.test_roku_http -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/cabletv/web.py tests/test_roku_settings_ui.py
git commit -m "feat: add Roku companion settings and device controls"
```

---

### Task 7: Update Help & Support and version to v0.4.8

**Files:**
- Modify: `app/cabletv/web.py:793-804`
- Modify: `app/cabletv/__init__.py`
- Modify: `README.md`
- Modify: `README_FIRST.txt`
- Create: `tests/test_roku_help.py`

**Interfaces:**
- No new runtime interface; documentation must match the shipped behavior.

- [ ] **Step 1: Write failing documentation tests**

```python
# tests/test_roku_help.py
import tempfile
import unittest
from pathlib import Path
from cabletv import __version__
from cabletv.web import CableTVApplication


class RokuHelpTests(unittest.TestCase):
    def test_version_is_048(self):
        self.assertEqual(__version__, "0.4.8")

    def test_help_documents_roku_security_and_sideloading(self):
        root = Path(tempfile.mkdtemp())
        page = CableTVApplication(root, root / "media").dashboard_html("http://192.168.1.2:8484")
        for text in (
            "Roku Personal Cable TV Companion",
            "Companion Password",
            "LAN-only",
            "do not port-forward",
            "Developer Mode",
            "one sideloaded development app",
            "This Roku is no longer authorized",
        ):
            self.assertIn(text, page)
```

- [ ] **Step 2: Run help tests and verify failure**

Run: `PYTHONPATH=app python -m unittest tests.test_roku_help -v`

Expected: FAIL because version/help still describe v0.4.7/Roku Coming Soon.

- [ ] **Step 3: Update version and documentation**

Set `__version__ = "0.4.8"`.

Update Device Support from `Roku TV — Coming Soon` to `Roku TV — Supported with Roku Personal Cable TV Companion v0.1.0` without implying the official Jellyfin Roku app is modified.

Help must explain: enabling the companion, setting/changing password, LAN-only requirement, never port-forwarding port 8484 for this feature, first-run Roku server address + password, Test Connection, persistent authorization, per-device revoke, Revoke All, revoked-device behavior, Developer Mode sideloading, one-development-app limitation, and basic playback troubleshooting. Keep the existing M3U/XMLTV and LG/Web Companion instructions intact.

- [ ] **Step 4: Run help/UI tests**

Run: `PYTHONPATH=app python -m unittest tests.test_roku_help tests.test_roku_settings_ui -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/cabletv/__init__.py app/cabletv/web.py README.md README_FIRST.txt tests/test_roku_help.py
git commit -m "docs: add Roku companion setup for v0.4.8"
```

---

### Task 8: Add v0.4.7 regression tests and run the complete server suite

**Files:**
- Create: `tests/test_v047_regression.py`
- Modify runtime files only if a regression is found; do not weaken tests to fit changed behavior.

**Interfaces:**
- Verifies existing `/iptv/channels.m3u`, `/iptv/guide.xml`, `stream_channel()`, channel ordering, and media safety remain unchanged.

- [ ] **Step 1: Write regression tests around existing behavior**

```python
# tests/test_v047_regression.py
import tempfile
import unittest
from pathlib import Path

from cabletv.db import Database
from cabletv.feeds import build_m3u
from cabletv.streamer import safe_media_path, StreamSafetyError


class V047RegressionTests(unittest.TestCase):
    def setUp(self):
        self.root = Path(tempfile.mkdtemp())
        self.db = Database(self.root / "db.sqlite3")
        self.db.initialize()

    def test_channel_listing_stays_sorted_by_number(self):
        self.db.create_channel(105, "Five", "all_movies", False)
        self.db.create_channel(101, "One", "all_movies", False)
        self.assertEqual([c["number"] for c in self.db.list_channels()], [101, 105])

    def test_m3u_still_uses_existing_iptv_stream_route(self):
        channel_id = self.db.create_channel(101, "Movies", "all_movies", False)
        m3u = build_m3u(self.db, "http://pctv:8484", tz_name="UTC")
        self.assertIn(f"/iptv/channel/{channel_id}.ts", m3u)
        self.assertNotIn("/api/roku/stream/", m3u)

    def test_media_safety_rejects_path_outside_allowed_root(self):
        media_root = self.root / "media"
        media_root.mkdir()
        outside = self.root / "outside.mkv"
        outside.write_bytes(b"x")
        with self.assertRaises(StreamSafetyError):
            safe_media_path(outside, media_root)
```

- [ ] **Step 2: Run the complete suite**

Run: `PYTHONPATH=app python -m unittest discover -s tests -p 'test_*.py' -v`

Expected: all tests PASS. Any failure in existing behavior is a blocker.

- [ ] **Step 3: Run syntax compilation**

Run: `PYTHONDONTWRITEBYTECODE=1 python -m compileall -q app`

Expected: exit 0.

- [ ] **Step 4: Build the Docker image locally**

Run: `docker build -t personal-cable-tv:0.4.8 .`

Expected: exit 0.

- [ ] **Step 5: Run container smoke tests with temporary writable app data and read-only media**

```bash
mkdir -p .smoke-data .smoke-media
docker run --rm -d --name pctv-048-smoke \
  -p 18484:8484 \
  -v "$PWD/.smoke-data:/app/data" \
  -v "$PWD/.smoke-media:/media:ro" \
  personal-cable-tv:0.4.8
sleep 3
python -c "import urllib.request; print(urllib.request.urlopen('http://127.0.0.1:18484/health').read().decode())"
python -c "import urllib.request; print(urllib.request.urlopen('http://127.0.0.1:18484/api/roku/health').read().decode())"
docker stop pctv-048-smoke
```

Expected: both health endpoints return HTTP 200; Roku health says disabled by default; container stops cleanly.

- [ ] **Step 6: Commit regression tests**

```bash
git add tests/test_v047_regression.py
git commit -m "test: protect v0.4.7 behavior during Roku update"
```

---

### Task 9: Package and publish Personal Cable TV v0.4.8 after verification

**Files:**
- Create release ZIP: `Personal_Cable_TV_Docker_v0.4.8.zip`
- Modify repository: `.github/workflows/docker-publish.yml`
- Modify repository: `casaos/docker-compose.yml`
- Modify repository: `README.md`

**Interfaces:**
- Docker image tags: `ghcr.io/akaidragon123k/personal-cable-tv:0.4.8` and `latest`.

- [ ] **Step 1: Create a clean ZIP with no database, credentials, caches, or test artifacts**

Run from the parent directory:

```bash
find Personal_Cable_TV_Docker_v0.4.8 -type d -name __pycache__ -prune -exec rm -rf {} +
rm -rf Personal_Cable_TV_Docker_v0.4.8/.smoke-data Personal_Cable_TV_Docker_v0.4.8/.smoke-media
zip -r Personal_Cable_TV_Docker_v0.4.8.zip Personal_Cable_TV_Docker_v0.4.8 \
  -x '*/data/*.sqlite3' '*/data/backups/*' '*/.env' '*/tests/*' '*/.git/*'
```

Expected: ZIP contains source/runtime docs only; no `cabletv.sqlite3`, API key, password, tokens, `.env`, or `__pycache__`.

- [ ] **Step 2: Verify package contents and checksum**

Run: `unzip -l Personal_Cable_TV_Docker_v0.4.8.zip`

Run: `sha256sum Personal_Cable_TV_Docker_v0.4.8.zip`

Expected: clean listing and a recorded SHA256 for the release notes.

- [ ] **Step 3: Update public GitHub release references only after the local package passes**

Change `.github/workflows/docker-publish.yml` from `0.4.7` ZIP/context/tags to `0.4.8`.

Change `casaos/docker-compose.yml` image to `ghcr.io/akaidragon123k/personal-cable-tv:0.4.8`, CasaOS version to `0.4.8`, and release notes to describe Roku Companion support/security without changing the read-only media mount or other safety controls.

Change the root `README.md` current public version to v0.4.8 and document that Roku support requires the separate **Roku Personal Cable TV Companion v0.1.0**; keep Jellyfin M3U/XMLTV setup instructions.

- [ ] **Step 4: Publish the v0.4.8 ZIP and trigger the existing GHCR workflow**

Upload `Personal_Cable_TV_Docker_v0.4.8.zip`, commit the workflow/Compose/README changes, and wait for GitHub Actions to complete successfully before calling v0.4.8 published.

- [ ] **Step 5: Verify the public image reference**

Confirm the workflow conclusion is `success` and the CasaOS Compose references exactly `ghcr.io/akaidragon123k/personal-cable-tv:0.4.8`.

- [ ] **Step 6: Commit/publish checkpoint**

Commit message: `release: publish Personal Cable TV v0.4.8`.

---

### Task 10: Real NAS rollback-safe acceptance test

**Files:**
- No source changes unless a failure is reproduced and fixed through a new test-first task.

**Interfaces:**
- Test server: the user's existing CasaOS Personal Cable TV installation.
- Rollback source: v0.4.7 application/image plus the pre-v0.4.8 database backup created by Task 1.

- [ ] **Step 1: Before installing v0.4.8, confirm the existing v0.4.7 database/app backup is present**

Record the current channel count, library count, and a known working Jellyfin channel before upgrade.

- [ ] **Step 2: Install v0.4.8 through the same CasaOS Compose path used for v0.4.7**

Keep `/app/data` on `/DATA/AppData/personal-cable-tv/data`, keep `/media` read-only, and do not change Jellyfin storage.

- [ ] **Step 3: Verify existing data survived**

Confirm existing libraries, channels, schedules, Jellyfin connection, M3U, XMLTV, and channel ordering still appear.

- [ ] **Step 4: Enable Roku Companion and create the Companion Password**

Confirm no password is displayed after save and Authorized Roku Devices is initially empty.

- [ ] **Step 5: Run the Roku v0.1.0 acceptance plan against this server**

Do not publish v0.4.8 as the accepted baseline until Roku auth, Guide, playback, revoke behavior, and the existing Jellyfin Live TV regression test all pass on the real NAS.

- [ ] **Step 6: If acceptance fails, use the rollback rule instead of modifying live data in place**

Stop v0.4.8, restore v0.4.7 application/image, and restore the pre-v0.4.8 database backup only if the migration prevents clean rollback. Never touch NAS media during rollback.
