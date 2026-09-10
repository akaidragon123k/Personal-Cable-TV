# Roku Personal Cable TV Companion Design

Date: 2026-09-10
Status: Approved design
Target releases: Personal Cable TV v0.4.8 + Roku Personal Cable TV Companion v0.1.0

## 1. Purpose

Build a Roku-specific Personal Cable TV companion that opens directly to a cable-style Guide and uses the existing Personal Cable TV server as its backend. The Roku app is separate from the official Jellyfin Roku app and must not modify, replace, or depend on changing the official Jellyfin Roku client.

The existing Personal Cable TV v0.4.7 behavior is the known-good rollback baseline. v0.4.8 is an additive server update that provides Roku Companion support without changing existing Jellyfin M3U/XMLTV, channel scheduling, Full Library Mode, or live-position playback behavior.

## 2. Product Names and Versioning

- Server: Personal Cable TV v0.4.8
- Roku app title: Roku Personal Cable TV Companion
- Roku package name: Roku_Personal_Cable_TV_Companion_v0.1.0.zip
- Roku GitHub repository name: Roku-Personal-Cable-TV-Companion
- Existing known-good rollback baseline: Personal Cable TV v0.4.7

## 3. High-Level Architecture

The approved architecture is:

Roku Personal Cable TV Companion -> Personal Cable TV v0.4.8 -> existing Jellyfin/media playback system

The Roku Companion connects only to Personal Cable TV. It does not ask for, receive, or store the Jellyfin API key.

Personal Cable TV remains responsible for channel definitions, schedules, categories, guide data, and live-position playback. Roku acts as a presentation and playback client.

If Roku needs information that v0.4.7 does not currently expose, v0.4.8 adds small, protected Roku Companion endpoints beside the existing working system rather than replacing or rewriting it.

## 4. Roku First-Run and Launch Flow

First launch:

1. Show a setup screen.
2. Ask for the Personal Cable TV server address, for example http://192.168.0.19:8484.
3. Ask for the Companion Password.
4. Offer Test Connection.
5. Offer Save & Open Guide.
6. On successful authorization, save the Roku device authorization locally.
7. Open the Guide.

Future launches:

- Go directly to the Guide.
- Do not show a home screen or On Now screen.
- Do not require the Companion Password again while the device remains authorized.
- A Back to Setup option remains available from the Guide so the server address can be changed later.

## 5. Guide Layout

The Guide should visually follow the approved Personal Cable TV guide concept:

- Category buttons across the top.
- Channel number and full channel name in a fixed column on the left.
- Horizontal program blocks organized by time.
- 30-minute time markers across the top.
- A red NOW line showing the current time.
- Clear focus/highlight state for the selected program.
- Current category visibly highlighted.
- A program-information panel above the Guide grid.

The design should be optimized for a 10-foot TV interface and standard Roku remote navigation.

## 6. Dynamic Categories

The category bar must not be hard-coded in the Roku app.

Roku requests the current category list from Personal Cable TV so category additions, removals, renames, and ordering remain synchronized with the server.

All Channels is available at the end of the category list.

Remote behavior:

- From the top visible guide row, pressing Up moves focus to the category bar.
- Left/Right moves between category buttons.
- OK activates the selected category.
- Down returns focus to the Guide.
- The active category remains visibly highlighted.
- All Channels is the default category on a fresh install unless a later product decision explicitly changes this.

## 7. Guide Remote Navigation

Approved Roku remote behavior:

- Up/Down in the Guide: move between channel rows.
- Left/Right in the Guide: move between program blocks.
- OK on a currently airing program: tune/play that channel at the current live position.
- OK on a future program: do not play it early; remain in the Guide.
- Replay: jump the Guide back to the current time.
- Fast Forward/Rewind: move the Guide farther forward/backward in time.
- Back from playback: return to the Guide.
- On initial Guide load, focus should land on a currently airing program in the first visible channel when available.

## 8. Guide Time Behavior

- Guide opens centered around current time.
- Time header uses 30-minute blocks.
- Red NOW line represents exact current time.
- Left/Right moves between programs.
- Fast Forward/Rewind moves farther through the schedule than a normal Left/Right press.
- Replay returns directly to NOW.
- Returning from playback restores the same category, channel, and time position used before playback.

## 9. Program Information Panel

When a program is highlighted, the information area above the Guide updates automatically.

Show, when available:

- poster or backdrop
- title
- year
- runtime
- rating
- short description
- channel number
- channel name
- program start time
- program end time
- LIVE indicator when currently airing
- current progress through the airing when currently live

There is no separate program-details page in v0.1.0. The user remains on the Guide while browsing program information.

## 10. Playback Behavior

When OK is pressed on a currently airing program:

1. Roku requests an authorized playback session/address from Personal Cable TV.
2. Personal Cable TV returns a short-lived protected playback URL/session.
3. Roku begins playback.
4. Playback joins the channel at the current scheduled/live position rather than starting from the beginning.

The implementation must reuse the existing Personal Cable TV live-position behavior rather than create a second independent scheduling/playback engine.

No Start From Beginning behavior is added.

No recording controls are added.

## 11. Channel Surfing During Playback

While watching a channel:

- Up/Down tunes the previous/next Personal Cable TV channel.
- Each channel change joins the newly selected channel at its current live position.
- Show a temporary channel banner containing the channel number, channel name, and current program.
- Back returns to the Guide focused on the channel being watched.

## 12. Roku Companion Security

Security is required even for LAN use.

### 12.1 Companion Enablement

Personal Cable TV v0.4.8 adds a Roku Companion section under Settings with:

- Enable Roku Companion toggle
- Set / Change Companion Password
- LAN-only access setting, ON by default
- Authorized Roku Devices list
- Revoke action per device
- Revoke All Devices action

There is no default Companion Password.

When enabling Roku Companion for the first time, the owner must create a password before companion authorization can succeed.

Minimum password length: 8 characters.

The password is never displayed after saving.

### 12.2 Password Storage

The Companion Password must be stored as a salted password hash using an established password-hashing algorithm available in the server stack. Plaintext password storage is prohibited.

### 12.3 Device Authorization

On first successful authentication:

1. Roku sends the Companion Password over the local connection.
2. Personal Cable TV verifies it.
3. Personal Cable TV creates a cryptographically random per-device authorization token.
4. Roku stores the usable token locally in Roku persistent storage.
5. Personal Cable TV stores only a one-way hash/digest of the usable token plus device metadata needed for management.

The usable token is not stored in plaintext on the Personal Cable TV server.

An authorized Roku remains authorized until manually revoked.

Changing the Companion Password affects future authorizations but does not automatically revoke already-authorized Roku devices.

Revoke All Devices invalidates all existing Roku device tokens.

Revoking one device invalidates only that device token.

If a token is revoked, the Roku returns to the setup/password flow on the next protected request.

### 12.4 Device Metadata

The Authorized Roku Devices list may show:

- friendly device name
- device IP address
- last connected time
- authorization created time
- revoke control

Do not expose the usable device token in the UI, logs, or API responses after initial issuance.

## 13. LAN-Only Default

Roku Companion access is local-network-only by default.

- Do not intentionally expose the companion endpoints to the public Internet.
- Help & Support must warn users not to port-forward Personal Cable TV port 8484 for Roku Companion use.
- Password/device authorization remains required on the LAN.
- Optional remote companion access is out of scope for v0.4.8/v0.1.0.

The LAN-only control should be implemented conservatively so normal Docker/CasaOS local-network deployments continue to work without requiring users to understand network internals.

## 14. Protected Roku Companion API

v0.4.8 adds a dedicated protected API namespace for Roku Companion traffic. Exact path names may follow existing project routing conventions, but responsibilities are fixed:

- health/connection test
- initial password authorization
- device token validation
- dynamic categories
- channel list
- Guide/program schedule data
- current-time/NOW metadata
- program metadata needed by the top information panel
- protected live playback session/address creation
- optional device heartbeat/last-seen update

Except for the health/connection test and initial authorization endpoint, Roku Companion API requests require a valid authorized-device token.

The API must return only the information Roku needs. It must never return the Jellyfin API key or other server secrets.

## 15. Short-Lived Playback Authorization

The Roku device token should not be embedded directly into a long-lived media URL.

When playback starts, Personal Cable TV issues a short-lived playback authorization/session tied to the authorized device and selected channel.

Requirements:

- expiration is enforced server-side
- authorization is scoped to playback needs
- expiration is short enough to reduce replay risk while long enough to start normal Roku playback reliably
- revoking a Roku device prevents that device from creating new playback sessions

The exact expiration duration is an implementation detail to be selected during implementation and tested on real Roku hardware; it must not weaken the requirement that playback URLs are temporary.

## 16. Connection and Error Handling

If Personal Cable TV becomes unavailable:

- Do not crash.
- Keep the Guide visible when cached/previously loaded data exists.
- Show Personal Cable TV is unavailable.
- Offer Retry and Back to Setup.

If Guide refresh fails but old Guide data exists:

- Continue showing the last loaded Guide.
- Mark it Offline / Last Updated.
- Retry when the user requests or when normal refresh logic succeeds later.

If device authorization is revoked or invalid:

- Show This Roku is no longer authorized.
- Clear the unusable local authorization.
- Return to the setup/password flow.

If the server address is invalid or unreachable during first-run setup:

- Show a clear connection error.
- Do not save the setup as successful.

## 17. Data Persistence on Roku

Persist locally:

- Personal Cable TV server address
- per-device authorization token
- friendly device identity as needed
- last selected category/channel/time state as needed for normal same-session return behavior

Do not persist:

- Jellyfin API key
- Companion Password after successful authorization
- plaintext server-side secrets

## 18. Personal Cable TV v0.4.8 Database Changes

Database changes must be additive only.

Do not rewrite or repurpose the existing channel, programming, Jellyfin, library, guide, or media tables solely for Roku support.

Create separate Roku Companion storage for:

- companion security settings
- Companion Password hash and hash parameters/salt as required by the chosen password hashing implementation
- authorized Roku device records
- device-token hashes/digests
- created/last-seen timestamps
- short-lived playback authorization records only if persistence is required by the implementation; prefer ephemeral server-side state when safe and practical

Before applying the v0.4.8 database migration, create a backup of the existing Personal Cable TV database.

If the Roku feature is disabled, all normal Personal Cable TV and Jellyfin functionality must continue to work as before.

## 19. Existing Functionality That Must Remain Unchanged

The Roku work must not intentionally change:

- Jellyfin M3U tuner output
- XMLTV guide output
- existing channel numbering
- existing channel ordering behavior beyond already-approved Personal Cable TV rules
- existing channel creation/edit/delete behavior
- Full Library Mode
- Jellyfin library sync
- Personal Cable TV programming/schedules
- holiday/seasonal scheduling behavior
- current live-position playback behavior
- read-only media safety
- Jellyfin API key handling for the server

## 20. Media and Server Safety

- Never delete, rename, move, overwrite, or modify real NAS media.
- Existing read-only media behavior remains intact.
- Roku Companion code must not require writable media mounts.
- Do not modify Jellyfin stock web files.
- Do not modify Jellyfin data except through existing supported Personal Cable TV/Jellyfin interfaces already used by the application.

## 21. Help & Support Update

Personal Cable TV v0.4.8 must update Help & Support in the same release.

Add Roku Companion documentation covering:

- what the Roku Companion is
- requirement for Personal Cable TV v0.4.8 or newer
- enabling Roku Companion
- setting/changing the Companion Password
- LAN-only default
- warning not to port-forward port 8484
- authorized device management and revocation
- first-run Roku setup
- server address example
- test connection
- what happens when authorization is revoked
- Roku Developer Mode sideload instructions for the test release
- basic connection/playback troubleshooting

## 22. Roku Packaging and Distribution

First testing release:

- Roku Personal Cable TV Companion v0.1.0
- package as a Roku sideload ZIP
- publish the test ZIP to GitHub
- test through Roku Developer Mode and Roku Development Application Installer

The initial goal is sideload testing, not immediate Roku Channel Store publication.

Known Roku development constraint: only one development/sideloaded app can occupy the development slot on a Roku at a time; this must be mentioned in testing documentation.

Broader Roku distribution is a later project phase after real-device testing passes.

## 23. Testing Strategy

### 23.1 Personal Cable TV v0.4.8 Server Tests

Add automated tests for:

- Roku Companion disabled by default unless existing project policy explicitly requires otherwise
- enabling requires a valid Companion Password
- minimum password length
- password is stored hashed, not plaintext
- correct password authorizes a device
- incorrect password is rejected
- token generation uniqueness
- server stores token hash/digest rather than usable token
- valid token can access protected Guide endpoints
- missing/invalid/revoked token is rejected
- password change does not revoke existing devices
- single-device revoke works
- Revoke All Devices works
- Guide/category responses do not contain Jellyfin API keys or other secrets
- playback authorization expires
- disabled Roku Companion rejects protected companion use
- existing v0.4.7 behavior remains covered by regression tests
- database migration creates a pre-migration backup and preserves existing data

### 23.2 Roku App Tests

Test logic/components for:

- first-run setup
- server address persistence
- successful/failed Test Connection
- password authorization
- token persistence
- direct-to-Guide launch after authorization
- dynamic category rendering
- category focus navigation
- Guide directional navigation
- Replay return-to-NOW behavior
- future-program OK does not start playback
- current-program OK starts playback
- Back from playback restores Guide focus/state
- Up/Down channel surfing during playback
- revoked authorization returns to setup
- unavailable server error state
- stale Guide/offline state

### 23.3 Real Roku Acceptance Test

Before v0.1.0 is considered working, test on an actual Roku device:

1. Sideload v0.1.0.
2. Connect to a v0.4.8 Personal Cable TV test server.
3. Enter server address and Companion Password.
4. Confirm authorization persists across app restart.
5. Confirm app opens directly to Guide.
6. Confirm top categories load dynamically.
7. Confirm Up/Down/Left/Right/OK navigation.
8. Confirm category bar is reachable by pressing Up above the top guide row.
9. Confirm current program playback starts.
10. Watch for at least five seconds, leave playback, wait, then re-enter and confirm playback joins farther ahead rather than restarting.
11. Confirm Back returns to the same Guide location.
12. Confirm Up/Down channel surfing while playing.
13. Confirm Replay returns Guide to NOW.
14. Revoke the Roku in Personal Cable TV Settings and confirm the Roku is forced back to setup on the next protected request.
15. Restart Personal Cable TV and confirm authorized-device state and normal Personal Cable TV data survive as intended.
16. Confirm existing Jellyfin Live TV M3U/XMLTV Guide and playback still work after v0.4.8.

## 24. Rollback Strategy

v0.4.7 remains the known-good baseline.

Before v0.4.8 migration/deployment:

- back up the existing Personal Cable TV database
- preserve the v0.4.7 application/package/image needed for rollback

If v0.4.8 fails acceptance testing:

- stop using the v0.4.8 application build
- restore the v0.4.7 application baseline
- restore the pre-v0.4.8 database backup if the database migration makes rollback otherwise incompatible

Roku v0.1.0 is additive and separate, so removing the Roku app must not affect Jellyfin or normal Personal Cable TV operation.

## 25. Out of Scope for v0.4.8 / Roku v0.1.0

- modifying the official Jellyfin Roku app
- injecting the web/LG companion into the official Roku client
- public Internet/remote Roku Companion access
- Roku Channel Store publication
- recording/DVR controls
- Start From Beginning
- separate On Now page
- separate Roku home screen
- future-program reminders
- user profiles/multiple Companion passwords
- changes to existing Jellyfin M3U/XMLTV behavior

## 26. Success Criteria

The project is successful when:

- Personal Cable TV v0.4.8 adds Roku support without breaking the v0.4.7 baseline behavior.
- Roku Personal Cable TV Companion v0.1.0 installs through Roku Developer Mode for testing.
- First-run setup requires server address + Companion Password.
- Authorized Roku devices remain authorized until revoked.
- Roku launches directly to the approved cable-style Guide.
- Categories are dynamic from Personal Cable TV.
- Guide navigation works with the Roku remote.
- Current programs play at the current live position.
- Returning to a channel later joins farther ahead as time advances.
- Channel surfing works with Up/Down while playing.
- No Jellyfin API key is exposed to Roku.
- Revocation works correctly.
- Normal Jellyfin Live TV M3U/XMLTV and Personal Cable TV functionality continue working.
- Real NAS media remains untouched.

