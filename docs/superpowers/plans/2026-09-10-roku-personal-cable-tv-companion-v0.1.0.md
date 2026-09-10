# Roku Personal Cable TV Companion v0.1.0 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a sideloadable Roku app named **Roku Personal Cable TV Companion v0.1.0** that authenticates to Personal Cable TV v0.4.8 and opens directly to the approved cable-style Guide.

**Architecture:** A standalone Roku SceneGraph/BrightScript client talks only to the protected `/api/roku/*` interface in Personal Cable TV v0.4.8. It stores the Personal Cable TV server address and per-device token in Roku Registry, never stores the Companion Password after authorization, renders a custom Guide optimized for remote navigation, and plays only server-issued short-lived stream URLs.

**Tech Stack:** Roku SceneGraph, BrightScript/BrighterScript `.bs`, `roUrlTransfer`, `roRegistrySection`, `Video` node, npm dev tooling (`brighterscript`, `@rokucommunity/bslint`, `roku-deploy`), Python standard-library validation scripts, Roku Developer Mode for real-device acceptance.

**Spec:** `docs/superpowers/specs/2026-09-10-roku-personal-cable-tv-companion-design.md`

## Global Constraints

- Roku app title is exactly **Roku Personal Cable TV Companion**.
- Initial Roku version is exactly **v0.1.0**.
- Release ZIP name is `Roku_Personal_Cable_TV_Companion_v0.1.0.zip`.
- App opens directly to the Guide after successful first-run authorization; there is no On Now page or normal home screen.
- First-run setup asks only for Personal Cable TV server address and Companion Password.
- Roku never receives or stores the Jellyfin API key.
- Companion Password is not retained after successful authorization.
- Per-device token is retained until revoked by Personal Cable TV.
- Categories come dynamically from Personal Cable TV; Roku does not hard-code the category list.
- OK plays only a currently airing program; future programs do not play early.
- Playback joins the current live position; no Start From Beginning and no recording controls.
- Back from playback returns to the same Guide category/channel/time position.
- Up/Down during playback surfs previous/next Personal Cable TV channels.
- Replay returns the Guide to NOW; Fast Forward/Rewind moves farther through time.
- LAN-only/server security is enforced by Personal Cable TV v0.4.8; the Roku client must clearly surface 401/403/revoked/unavailable states.
- First public testing is Developer Mode sideloading, not Roku Channel Store publication.

---

## File Structure Map

Create a new working tree named `Roku-Personal-Cable-TV-Companion/`:

- `manifest` — Roku title/version/resolution/icon/splash metadata.
- `package.json`, `package-lock.json`, `bsconfig.json`, `bslint.json` — reproducible build/lint tooling.
- `source/main.bs` — SceneGraph entry point.
- `source/Registry.bs` — server/token persistence helpers.
- `source/Device.bs` — Roku device UID/friendly name helpers.
- `source/Url.bs` — server address normalization and URL joining.
- `components/AppScene.xml`, `components/AppScene.bs` — top-level setup/guide/playback state coordinator.
- `components/SetupView.xml`, `components/SetupView.bs` — first-run address/password/Test Connection/Save & Open Guide.
- `components/tasks/ApiTask.xml`, `components/tasks/ApiTask.bs` — background JSON HTTP requests with Bearer headers.
- `components/GuideView.xml`, `components/GuideView.bs` — category bar, timeline, channel rows, focus, refresh, time state.
- `components/CategoryButton.xml`, `components/CategoryButton.bs` — reusable category control.
- `components/ChannelRow.xml`, `components/ChannelRow.bs` — channel number/name + program strip.
- `components/ProgramCell.xml`, `components/ProgramCell.bs` — program block rendering/current/focus state.
- `components/ProgramInfo.xml`, `components/ProgramInfo.bs` — selected-program details panel.
- `components/PlaybackView.xml`, `components/PlaybackView.bs` — Video node, Back behavior, channel surfing banner.
- `components/StatusOverlay.xml`, `components/StatusOverlay.bs` — unavailable/retry/revoked/offline messages.
- `images/` — Roku icon/splash raster assets derived from the existing Personal Cable TV icon branding.
- `tests/test_package_layout.py` — static package/manifest/XML checks.
- `tests/test_contract_strings.py` — validates endpoint/header/registry contract is present in source.
- `README.md` — install/setup/testing/security instructions.
- `.github/workflows/validate.yml` — compile/lint/package validation on GitHub.

---

### Task 1: Scaffold the Roku package and make it compile

**Files:**
- Create: `manifest`
- Create: `package.json`
- Create: `bsconfig.json`
- Create: `bslint.json`
- Create: `source/main.bs`
- Create: `components/AppScene.xml`
- Create: `components/AppScene.bs`
- Create: `tests/test_package_layout.py`

**Interfaces:**
- Produces SceneGraph root component `AppScene`.
- Produces npm scripts `build`, `lint`, and `package`.

- [ ] **Step 1: Write failing package-layout tests**

```python
# tests/test_package_layout.py
import unittest
from pathlib import Path

ROOT = Path(__file__).resolve().parents[1]


class PackageLayoutTests(unittest.TestCase):
    def test_manifest_identity(self):
        text = (ROOT / "manifest").read_text()
        self.assertIn("title=Roku Personal Cable TV Companion", text)
        self.assertIn("major_version=0", text)
        self.assertIn("minor_version=1", text)
        self.assertIn("build_version=0", text)
        self.assertIn("ui_resolutions=fhd", text)

    def test_scene_entrypoint_exists(self):
        self.assertTrue((ROOT / "source/main.bs").is_file())
        self.assertTrue((ROOT / "components/AppScene.xml").is_file())
        self.assertTrue((ROOT / "components/AppScene.bs").is_file())
```

- [ ] **Step 2: Run the test and verify failure**

Run: `python -m unittest tests.test_package_layout -v`

Expected: FAIL because the Roku project files do not exist.

- [ ] **Step 3: Create manifest and minimal SceneGraph entry point**

Use manifest identity:

```text
title=Roku Personal Cable TV Companion
major_version=0
minor_version=1
build_version=0
ui_resolutions=fhd
mm_icon_focus_fhd=pkg:/images/channel-poster_fhd.png
mm_icon_focus_hd=pkg:/images/channel-poster_hd.png
mm_icon_focus_sd=pkg:/images/channel-poster_sd.png
splash_screen_fhd=pkg:/images/splash-screen_fhd.png
splash_screen_hd=pkg:/images/splash-screen_hd.png
splash_screen_sd=pkg:/images/splash-screen_sd.png
splash_min_time=500
```

`source/main.bs`:

```brightscript
sub Main()
    screen = CreateObject("roSGScreen")
    port = CreateObject("roMessagePort")
    screen.SetMessagePort(port)
    scene = screen.CreateScene("AppScene")
    screen.Show()
    while true
        msg = wait(0, port)
        if type(msg) = "roSGScreenEvent" and msg.IsScreenClosed() then return
    end while
end sub
```

`AppScene.xml` extends `Scene`, declares children placeholders for setup/guide/playback/status groups, and loads `AppScene.bs` through a `<script uri="pkg:/components/AppScene.bs" />` node.

Use `brighterscript`, `@rokucommunity/bslint`, and `roku-deploy` as dev dependencies. `bsconfig.json` should compile the source tree to `out/` without renaming runtime files unexpectedly.

- [ ] **Step 4: Run layout test and BrighterScript compile**

Run: `python -m unittest tests.test_package_layout -v`

Run: `npm ci || npm install`

Run: `npx bsc --project bsconfig.json`

Expected: tests PASS and compiler exits 0.

- [ ] **Step 5: Commit**

```bash
git add manifest package.json package-lock.json bsconfig.json bslint.json source components tests
git commit -m "feat: scaffold Roku Personal Cable TV Companion"
```

---

### Task 2: Add persistent server/device authorization storage and URL normalization

**Files:**
- Create: `source/Registry.bs`
- Create: `source/Device.bs`
- Create: `source/Url.bs`
- Create: `tests/test_contract_strings.py`

**Interfaces:**
- Produces: `GetPctvSettings() as Object` returning `{ serverAddress, deviceToken }`.
- Produces: `SavePctvServerAddress(address as String) as Void`.
- Produces: `SavePctvDeviceToken(token as String) as Void`.
- Produces: `ClearPctvDeviceToken() as Void`.
- Produces: `GetPctvDeviceIdentity() as Object` returning `{ uid, name }` from `roDeviceInfo`.
- Produces: `NormalizeServerAddress(value as String) as String`.
- Produces: `JoinServerUrl(base as String, path as String) as String`.

- [ ] **Step 1: Extend the contract test to fail first**

```python
# append to tests/test_contract_strings.py
import unittest
from pathlib import Path

ROOT = Path(__file__).resolve().parents[1]


class ContractStringTests(unittest.TestCase):
    def test_registry_never_has_password_key(self):
        source = (ROOT / "source/Registry.bs").read_text()
        self.assertIn('"serverAddress"', source)
        self.assertIn('"deviceToken"', source)
        self.assertNotIn('"password"', source.casefold())

    def test_device_identity_uses_channel_client_id(self):
        source = (ROOT / "source/Device.bs").read_text()
        self.assertIn("GetChannelClientID", source)
```

- [ ] **Step 2: Run and verify failure**

Run: `python -m unittest tests.test_contract_strings -v`

Expected: FAIL because these source files do not exist.

- [ ] **Step 3: Implement registry/device/url helpers**

Use Roku Registry section `PersonalCableTVCompanion`. Persist only `serverAddress` and `deviceToken`. Never create a registry key for the Companion Password.

Use `CreateObject("roDeviceInfo").GetChannelClientID()` for `uid` and `GetFriendlyName()` for `name`, falling back to `Roku` if no friendly name is available.

Normalize address by trimming spaces/trailing slashes; accept `http://` and `https://`. If a user enters only a host/IP and optional port, prepend `http://`. Reject any scheme other than HTTP/HTTPS in SetupView before saving.

- [ ] **Step 4: Run contract tests, lint, and compile**

Run: `python -m unittest tests.test_contract_strings -v`

Run: `npx bslint "source/**/*.bs" "components/**/*.bs"`

Run: `npx bsc --project bsconfig.json`

Expected: PASS/exit 0.

- [ ] **Step 5: Commit**

```bash
git add source/Registry.bs source/Device.bs source/Url.bs tests/test_contract_strings.py
git commit -m "feat: persist Roku companion connection settings"
```

---

### Task 3: Implement the background Personal Cable TV API task

**Files:**
- Create: `components/tasks/ApiTask.xml`
- Create: `components/tasks/ApiTask.bs`
- Modify: `tests/test_contract_strings.py`

**Interfaces:**
- Input fields: `request` object with `method`, `url`, optional `token`, optional `body`.
- Output fields: `response` object with `ok`, `status`, optional `json`, optional `error`, and `authorizationRevoked` boolean.
- All API JSON requests use `roUrlTransfer` from a Task node, never from render callbacks.

- [ ] **Step 1: Add failing contract assertions**

```python
def test_api_task_uses_json_and_bearer_header(self):
    source = (ROOT / "components/tasks/ApiTask.bs").read_text()
    self.assertIn("roUrlTransfer", source)
    self.assertIn("Authorization", source)
    self.assertIn("Bearer ", source)
    self.assertIn("application/json", source)
```

- [ ] **Step 2: Run and verify failure**

Run: `python -m unittest tests.test_contract_strings -v`

Expected: FAIL because ApiTask is absent.

- [ ] **Step 3: Implement ApiTask**

`ApiTask.xml` extends `Task`, declares `request` and `response`, and runs `executeRequest` when request changes. `ApiTask.bs` creates `roUrlTransfer`, sets `Accept: application/json`, adds `Authorization: Bearer <token>` only when a token is present, sends JSON for POST with `Content-Type: application/json`, parses a JSON object response, and maps HTTP/network failures into a stable response object.

Set `authorizationRevoked=true` for HTTP 401. Set a normal error message for 403 LAN-only failures. Do not print request bodies or tokens.

- [ ] **Step 4: Compile/lint**

Run: `npx bslint "source/**/*.bs" "components/**/*.bs"`

Run: `npx bsc --project bsconfig.json`

Expected: exit 0.

- [ ] **Step 5: Commit**

```bash
git add components/tasks/ApiTask.xml components/tasks/ApiTask.bs tests/test_contract_strings.py
git commit -m "feat: add Personal Cable TV API task"
```

---

### Task 4: Build first-run SetupView and direct-to-Guide launch behavior

**Files:**
- Create: `components/SetupView.xml`
- Create: `components/SetupView.bs`
- Modify: `components/AppScene.xml`
- Modify: `components/AppScene.bs`
- Modify: `tests/test_contract_strings.py`

**Interfaces:**
- SetupView fields: `authorized` boolean, `serverAddress` string, `errorMessage` string.
- Calls `GET /api/roku/health` for Test Connection.
- Calls `POST /api/roku/authorize` with `{password, device_uid, device_name}` for Save & Open Guide.
- AppScene checks Registry at launch: server+token present -> validate token via protected categories request -> Guide; otherwise Setup.

- [ ] **Step 1: Add failing setup contract tests**

```python
def test_setup_uses_only_server_address_and_companion_password(self):
    xml = (ROOT / "components/SetupView.xml").read_text()
    self.assertIn("Personal Cable TV Server Address", xml)
    self.assertIn("Companion Password", xml)
    self.assertIn("Test Connection", xml)
    self.assertIn("Save & Open Guide", xml)
    self.assertNotIn("Jellyfin API", xml)
```

- [ ] **Step 2: Run and verify failure**

Run: `python -m unittest tests.test_contract_strings -v`

Expected: FAIL because SetupView does not exist.

- [ ] **Step 3: Implement SetupView and launch coordinator**

Show two setup rows (Server Address and Companion Password) plus Test Connection and Save & Open Guide buttons. Selecting either setup row opens a Roku `KeyboardDialog`; the password dialog uses secure/masked entry. Test Connection normalizes the address and calls `/api/roku/health`; success must show that Roku Companion is enabled before Save can succeed.

Save & Open Guide sends the password only in the authorization request. On HTTP 200, immediately discard the password string from the view state, persist only the server address and returned device token, signal `authorized=true`, and let AppScene replace SetupView with GuideView.

On subsequent launches, AppScene goes directly to Guide when server address + token are present and valid. A 401 clears only the token and returns to Setup. A network outage must not erase a valid token.

- [ ] **Step 4: Lint/compile and package-layout tests**

Run: `python -m unittest tests.test_package_layout tests.test_contract_strings -v`

Run: `npx bslint "source/**/*.bs" "components/**/*.bs"`

Run: `npx bsc --project bsconfig.json`

Expected: PASS/exit 0.

- [ ] **Step 5: Commit**

```bash
git add components/SetupView.xml components/SetupView.bs components/AppScene.xml components/AppScene.bs tests/test_contract_strings.py
git commit -m "feat: add Roku companion first-run setup"
```

---

### Task 5: Render the cable-style Guide with dynamic categories and program information

**Files:**
- Create: `components/GuideView.xml`
- Create: `components/GuideView.bs`
- Create: `components/CategoryButton.xml`
- Create: `components/CategoryButton.bs`
- Create: `components/ChannelRow.xml`
- Create: `components/ChannelRow.bs`
- Create: `components/ProgramCell.xml`
- Create: `components/ProgramCell.bs`
- Create: `components/ProgramInfo.xml`
- Create: `components/ProgramInfo.bs`
- Modify: `components/AppScene.xml`
- Modify: `components/AppScene.bs`
- Modify: `tests/test_contract_strings.py`

**Interfaces:**
- GuideView input: `serverAddress`, `deviceToken`.
- GuideView output: `playRequest` object `{channelId, channelNumber, channelName}` and `setupRequested` boolean.
- Calls `GET /api/roku/categories` and `GET /api/roku/guide?...`.
- Calls `GET /api/roku/program/<schedule_id>` when focus changes and displays returned metadata/artwork URL.

- [ ] **Step 1: Add failing Guide contract tests**

```python
def test_guide_loads_categories_from_server_not_literal_list(self):
    source = (ROOT / "components/GuideView.bs").read_text()
    self.assertIn("/api/roku/categories", source)
    self.assertNotIn('"Anime Movies", "Anime Shows"', source)


def test_guide_contains_now_line_and_program_info_components(self):
    xml = (ROOT / "components/GuideView.xml").read_text()
    self.assertIn("nowLine", xml)
    self.assertIn("ProgramInfo", xml)
```

- [ ] **Step 2: Run and verify failure**

Run: `python -m unittest tests.test_contract_strings -v`

Expected: FAIL because Guide components do not exist.

- [ ] **Step 3: Implement visual structure**

Use FHD 1920x1080 coordinates. The top area contains ProgramInfo, beneath it a horizontally scrollable category row, then the time header and Guide grid. Keep a fixed left column for channel number/name and a clipped program timeline to the right. Calculate program-cell X/width from `start_utc/end_utc` relative to the 3-hour guide window; 30-minute headers share the same scale. Calculate the red NOW line from `server_now_utc` relative to `start_utc/end_utc` so its position matches program cells.

Use charcoal/dark gray panels, white text, and blue only for active/focus accents, matching Personal Cable TV's approved UI direction. Do not add unrelated Roku/Jellyfin branding.

Category controls are built exclusively from the server response. On fresh setup select `All Channels` if present; otherwise select the final returned category.

When focus moves to a program, update ProgramInfo with title, year, runtime, rating, description, channel, times, LIVE indicator, and progress when present. If `artwork_url` exists, set the Poster URI to that temporary server URL; otherwise hide the poster area cleanly.

- [ ] **Step 4: Lint/compile**

Run: `npx bslint "source/**/*.bs" "components/**/*.bs"`

Run: `npx bsc --project bsconfig.json`

Expected: exit 0.

- [ ] **Step 5: Commit**

```bash
git add components/GuideView* components/CategoryButton* components/ChannelRow* components/ProgramCell* components/ProgramInfo* components/AppScene* tests/test_contract_strings.py
git commit -m "feat: add cable-style Roku guide"
```

---

### Task 6: Implement approved Roku remote navigation and Guide time behavior

**Files:**
- Modify: `components/GuideView.bs`
- Modify: `components/CategoryButton.bs`
- Modify: `components/ChannelRow.bs`
- Modify: `components/ProgramCell.bs`
- Create: `tests/test_remote_contract.py`

**Interfaces:**
- Guide remote behavior: Up/Down channel rows, Left/Right program blocks, Up above top row -> categories, Down categories -> Guide, OK category -> filter, OK current program -> `playRequest`, future program -> no play, Replay -> NOW, Rewind/Fast Forward -> larger time jump.

- [ ] **Step 1: Write failing source-level remote contract test**

```python
# tests/test_remote_contract.py
import unittest
from pathlib import Path

ROOT = Path(__file__).resolve().parents[1]


class RemoteContractTests(unittest.TestCase):
    def test_guide_handles_required_remote_keys(self):
        source = (ROOT / "components/GuideView.bs").read_text().casefold()
        for key in ("up", "down", "left", "right", "ok", "replay", "rewind", "fastforward"):
            self.assertIn(f'"{key}"', source)

    def test_future_program_is_not_sent_to_playback(self):
        source = (ROOT / "components/GuideView.bs").read_text()
        self.assertIn("is_current", source)
```

- [ ] **Step 2: Run and verify failure**

Run: `python -m unittest tests.test_remote_contract -v`

Expected: FAIL until all approved key paths are implemented.

- [ ] **Step 3: Implement deterministic focus/time state**

Keep explicit state variables: `m.categoryIndex`, `m.channelIndex`, `m.programIndex`, `m.windowStartUtc`, `m.focusArea` (`"categories"` or `"guide"`).

Behavior:

```text
Up: previous channel; from top visible row -> category bar
Down: next channel; from category bar -> selected/current channel row
Left/Right: previous/next program in selected row
OK on category: reload guide for that category
OK on program with is_current=true: emit playRequest
OK on future program: remain focused, no API playback request
Replay: set window start to current half-hour and reload
Rewind: subtract 90 minutes from window start and reload
FastForward: add 90 minutes to window start and reload
```

After any reload, restore focus to the nearest matching channel ID/program schedule ID where possible. Initial load focuses a currently airing program on the first visible channel when available.

- [ ] **Step 4: Run remote contract, lint, compile**

Run: `python -m unittest tests.test_remote_contract -v`

Run: `npx bslint "source/**/*.bs" "components/**/*.bs"`

Run: `npx bsc --project bsconfig.json`

Expected: PASS/exit 0.

- [ ] **Step 5: Commit**

```bash
git add components/GuideView.bs components/CategoryButton.bs components/ChannelRow.bs components/ProgramCell.bs tests/test_remote_contract.py
git commit -m "feat: add Roku guide remote navigation"
```

---

### Task 7: Add live playback, Back-to-Guide state restoration, and channel surfing

**Files:**
- Create: `components/PlaybackView.xml`
- Create: `components/PlaybackView.bs`
- Modify: `components/AppScene.xml`
- Modify: `components/AppScene.bs`
- Modify: `components/GuideView.bs`
- Modify: `tests/test_contract_strings.py`
- Modify: `tests/test_remote_contract.py`

**Interfaces:**
- AppScene receives GuideView `playRequest`, calls `POST /api/roku/playback` with Bearer token, then passes returned temporary URL to PlaybackView.
- PlaybackView fields: `streamUrl`, `channelList`, `currentChannelId`, `returnRequested`, `channelChangeRequested`.
- Playback uses Roku `Video` node.

- [ ] **Step 1: Add failing playback contract tests**

```python
def test_playback_uses_server_issued_temporary_url(self):
    app = (ROOT / "components/AppScene.bs").read_text()
    playback = (ROOT / "components/PlaybackView.bs").read_text()
    self.assertIn("/api/roku/playback", app)
    self.assertIn("streamUrl", playback)
    self.assertNotIn("X-Emby-Token", app + playback)
```

Add remote assertions that PlaybackView handles `up`, `down`, and `back`.

- [ ] **Step 2: Run and verify failure**

Run: `python -m unittest tests.test_contract_strings tests.test_remote_contract -v`

Expected: FAIL because PlaybackView is absent.

- [ ] **Step 3: Implement playback coordinator**

On current-program OK, AppScene POSTs `{channel_id:<internal-id>}` to `/api/roku/playback` using the long-lived device token in the Authorization header. The response URL itself contains only the 120-second server-issued media ticket. Set that URL as the `Video` node content URL and begin playback immediately.

Before leaving Guide, snapshot `{category, channelId, scheduleId, windowStartUtc, channelIndex, programIndex}` in AppScene memory.

Back from PlaybackView stops the Video node and restores GuideView from that snapshot without returning to setup.

Up/Down during playback requests previous/next channel based on the current filtered Guide channel list, asks `/api/roku/playback` for a fresh temporary URL, switches Video content, and shows a temporary banner with channel number, channel name, and current program title. Each switch relies on the server's existing live-position stream behavior.

- [ ] **Step 4: Lint/compile**

Run: `python -m unittest tests.test_contract_strings tests.test_remote_contract -v`

Run: `npx bslint "source/**/*.bs" "components/**/*.bs"`

Run: `npx bsc --project bsconfig.json`

Expected: PASS/exit 0.

- [ ] **Step 5: Commit**

```bash
git add components/PlaybackView.xml components/PlaybackView.bs components/AppScene.xml components/AppScene.bs components/GuideView.bs tests
git commit -m "feat: add Roku live playback and channel surfing"
```

---

### Task 8: Add unavailable/offline/revoked states without losing authorization unnecessarily

**Files:**
- Create: `components/StatusOverlay.xml`
- Create: `components/StatusOverlay.bs`
- Modify: `components/AppScene.xml`
- Modify: `components/AppScene.bs`
- Modify: `components/GuideView.bs`
- Create: `tests/test_error_contract.py`

**Interfaces:**
- `StatusOverlay` supports modes `offline`, `unavailable`, `revoked`, `lanBlocked`.
- Offline/unavailable actions: Retry and Back to Setup.
- Revoked action clears token and returns to Setup.

- [ ] **Step 1: Write failing error-state tests**

```python
# tests/test_error_contract.py
import unittest
from pathlib import Path

ROOT = Path(__file__).resolve().parents[1]


class ErrorContractTests(unittest.TestCase):
    def test_required_messages_exist(self):
        text = (ROOT / "components/StatusOverlay.xml").read_text() + (ROOT / "components/StatusOverlay.bs").read_text()
        self.assertIn("Personal Cable TV is unavailable", text)
        self.assertIn("This Roku is no longer authorized", text)
        self.assertIn("Retry", text)
        self.assertIn("Back to Setup", text)
        self.assertIn("Offline", text)
        self.assertIn("Last Updated", text)
```

- [ ] **Step 2: Run and verify failure**

Run: `python -m unittest tests.test_error_contract -v`

Expected: FAIL because StatusOverlay is absent.

- [ ] **Step 3: Implement error handling**

GuideView retains its last successful Guide payload in memory. A refresh/network error leaves existing cells visible and shows `Offline · Last Updated <time>`. Retry reissues the failed request.

A network failure never clears the saved device token. HTTP 401 sets revoked mode, displays `This Roku is no longer authorized`, clears the token, and returns to Setup after OK. HTTP 403 displays a LAN-only message and offers Back to Setup without claiming the password is wrong.

Back to Setup does not immediately erase the working server address or token. If the user saves a different server address, clear the old local device token before authorizing the new server. Do not revoke the old server-side device automatically; it remains visible to the owner for manual revoke.

- [ ] **Step 4: Run all static tests and compile**

Run: `python -m unittest discover -s tests -p 'test_*.py' -v`

Run: `npx bslint "source/**/*.bs" "components/**/*.bs"`

Run: `npx bsc --project bsconfig.json`

Expected: all tests PASS and compile/lint exit 0.

- [ ] **Step 5: Commit**

```bash
git add components/StatusOverlay.xml components/StatusOverlay.bs components/AppScene* components/GuideView.bs tests/test_error_contract.py
git commit -m "feat: add Roku companion offline and revoke handling"
```

---

### Task 9: Add branded Roku raster assets, README, CI validation, and build the v0.1.0 ZIP

**Files:**
- Create: `images/channel-poster_fhd.png`
- Create: `images/channel-poster_hd.png`
- Create: `images/channel-poster_sd.png`
- Create: `images/splash-screen_fhd.png`
- Create: `images/splash-screen_hd.png`
- Create: `images/splash-screen_sd.png`
- Create: `README.md`
- Create: `.github/workflows/validate.yml`
- Modify: `tests/test_package_layout.py`

**Interfaces:**
- Build output: `Roku_Personal_Cable_TV_Companion_v0.1.0.zip` with `manifest` at ZIP root.

- [ ] **Step 1: Extend package test for required artwork and ZIP-root rules**

```python
def test_manifest_artwork_files_exist(self):
    for name in (
        "channel-poster_fhd.png", "channel-poster_hd.png", "channel-poster_sd.png",
        "splash-screen_fhd.png", "splash-screen_hd.png", "splash-screen_sd.png",
    ):
        self.assertTrue((ROOT / "images" / name).is_file(), name)
```

- [ ] **Step 2: Run and verify failure**

Run: `python -m unittest tests.test_package_layout -v`

Expected: FAIL until artwork files exist.

- [ ] **Step 3: Produce Roku raster assets from the approved existing Personal Cable TV icon branding**

Use the existing Personal Cable TV icon as the visual source; preserve its charcoal/dark-gray/blue TV/play branding rather than inventing unrelated branding. Generate required FHD/HD/SD poster and splash PNG sizes appropriate for the manifest. Do not ship the source SVG if it is not needed at Roku runtime.

- [ ] **Step 4: Write README and GitHub validation workflow**

README must state: requires Personal Cable TV v0.4.8+, first launch server address + Companion Password, no Jellyfin API key on Roku, LAN-only default, Developer Mode sideload steps, one-sideloaded-development-app limitation, direct-to-Guide behavior, remote key map, revocation behavior, and that this is separate from the official Jellyfin Roku app.

CI workflow runs:

```yaml
- run: npm ci
- run: python -m unittest discover -s tests -p 'test_*.py' -v
- run: npx bslint "source/**/*.bs" "components/**/*.bs"
- run: npx bsc --project bsconfig.json
```

- [ ] **Step 5: Build the sideload ZIP with manifest at root**

Run: `npx roku-deploy --package --out-dir dist --out-file Roku_Personal_Cable_TV_Companion_v0.1.0.zip`

Expected: `dist/Roku_Personal_Cable_TV_Companion_v0.1.0.zip` exists and contains `manifest`, `source/`, `components/`, and `images/` at ZIP root.

- [ ] **Step 6: Validate ZIP and checksum**

Run: `unzip -l dist/Roku_Personal_Cable_TV_Companion_v0.1.0.zip`

Run: `sha256sum dist/Roku_Personal_Cable_TV_Companion_v0.1.0.zip`

Expected: no `.git`, tests, node_modules, passwords, device tokens, or Jellyfin API keys in package.

- [ ] **Step 7: Commit**

```bash
git add images README.md .github/workflows/validate.yml tests/test_package_layout.py manifest package.json package-lock.json bsconfig.json bslint.json source components
git commit -m "release: prepare Roku companion v0.1.0 test package"
```

---

### Task 10: Real Roku Developer Mode acceptance test against Personal Cable TV v0.4.8

**Files:**
- No source changes unless a failure is reproduced first and fixed in a new test-first commit.

**Interfaces:**
- Input package: `Roku_Personal_Cable_TV_Companion_v0.1.0.zip`.
- Test backend: Personal Cable TV v0.4.8 with Roku Companion enabled.

- [ ] **Step 1: Enable Roku Developer Mode and sideload the v0.1.0 ZIP**

Use Roku's Development Application Installer. Confirm any previously sideloaded development app can be replaced because Roku provides one development sideload slot.

- [ ] **Step 2: Complete first-run setup**

Enter the Personal Cable TV address, for example `http://192.168.0.19:8484`, enter the Companion Password, run Test Connection, then Save & Open Guide. Confirm no Jellyfin API key is requested.

- [ ] **Step 3: Verify authorization persistence**

Exit and reopen the app. Expected: it opens directly to Guide without asking for the password again.

- [ ] **Step 4: Verify Guide visuals and categories**

Expected: top ProgramInfo, dynamic category buttons, fixed channel number/name column, 30-minute time headers, program blocks, red NOW line, active/focus styling, and All Channels default on fresh setup.

- [ ] **Step 5: Verify remote behavior**

Test Up/Down channels, Left/Right programs, Up from top row to category buttons, Left/Right category selection, OK category activation, Down back to Guide, Replay to NOW, Rewind/Fast Forward time jumps, and future-program OK doing nothing except keeping focus.

- [ ] **Step 6: Verify live-position playback**

Press OK on a currently airing program, watch at least five seconds, press Back, wait about one minute, tune that channel again, and confirm playback joins farther ahead instead of restarting or resuming the old join point.

- [ ] **Step 7: Verify Back state restoration and channel surfing**

Expected: Back restores the same Guide category/channel/time focus; Up/Down during playback changes channels and shows the temporary channel banner.

- [ ] **Step 8: Verify offline and revoked-device handling**

Temporarily make Personal Cable TV unavailable and confirm the Guide does not crash, Retry/Back to Setup appear, and cached Guide remains marked Offline/Last Updated when available. Restore server, then revoke this Roku in Personal Cable TV Settings and confirm the next protected request shows `This Roku is no longer authorized` and returns to setup.

- [ ] **Step 9: Verify existing Jellyfin Live TV still works**

In the normal Jellyfin client, confirm existing M3U/XMLTV Guide and a known channel still play correctly after Personal Cable TV v0.4.8 is installed.

- [ ] **Step 10: Accept or rollback**

Only mark Roku Companion v0.1.0 and Personal Cable TV v0.4.8 as accepted working baselines after all steps pass. If a failure occurs, keep v0.4.7 as the Personal Cable TV rollback baseline and keep the Roku app labeled a test build until the failure is fixed and retested.
