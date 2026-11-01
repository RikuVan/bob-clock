# Bob Clock — macOS menu bar app for starting and stopping the workday in Bob

Status: draft for approval (revision 2, after review) · Date: 2026-11-01

## Intent

A minimal macOS menu bar app that records the start and end of the workday in
Bob (HiBob) attendance. It replaces opening Bob's web or mobile app twice a day.
It is distributed through the author's own Homebrew tap,
[RikuVan/homebrew-formulae](https://github.com/RikuVan/homebrew-formulae).

[Doist/bob-cli](https://github.com/Doist/bob-cli) is the reference for talking to
Bob (service-user Basic auth against `https://api.hibob.com/v1`), but it has no
attendance support. No code from it is reused.

## Goals

- One click to start the day, and one click to stop it, both recorded in Bob at the current time.
- The menu is refreshed from Bob whenever it opens and before every action, so actions always use Bob's current state, including clock-ins made on the web or phone. The menu bar icon shows the result of the latest successful refresh. It is not continuously synced (see Lifecycle contract).
- Optional reminders (nudges) to start and to stop, with configurable times and days.
- Credentials can be entered once, are checked before saving, and are stored securely.
- Installed with `brew install --cask rikuvan/formulae/bob-clock`.

## Non-goals

- More than one start/stop pair per day. Breaks are handled by the company's attendance policy.
- Projects, tasks, comments or reason codes on entries.
- Duration-based attendance reporting.
- Viewing history, or editing or fixing past entries. That is done in Bob.
- Skipping reminders on holidays or time off. This would need an extra time-off permission, and the user chose to ignore it.
- Continuous background polling just to keep the icon current.
- Windows, Linux, and Intel Macs.

## Anti-goals (failure even if it "works")

- Writing incorrect hours to Bob without telling the user. Examples: closing an overnight entry with a 20-hour duration, closing a break when the user meant to stop work, or replaying a write that may already have succeeded.
- Assuming a state the app hasn't confirmed. For example, treating "no data" or "couldn't load" as "not started".
- A shared or company-wide token. Each user has their own service user, scoped to themselves.
- Storing the API token anywhere except the Keychain.
- A Dock icon, a main window, or anything that demands attention beyond the menu and the opt-in nudges.

## Prerequisites (documented in the in-app help and README)

- The company has Bob's paid **Time and Attendance** module.
- The user's attendance policy uses **start/end (clock in/out) reporting** and does **not require a project or task** on entries. Attendance policies aren't exposed through the public API ([Explore Attendance API](https://apidocs.hibob.com/docs/explore-attendance-api)), so the app can't check this in advance. It recognizes data it can't handle and refuses to write when it sees it (see Unsupported).
- The user has their own service user with *People's data > Time attendance > Manage time attendance sheets* ([attendance reference](https://apidocs.hibob.com/reference/attendance.md)). Its *Access data for* setting is *Select people by name*, with only themselves selected ([service users §3.3](https://apidocs.hibob.com/docs/api-service-users)).

A successful validation proves the token can read attendance for that employee. It does not prove policy compatibility or that later writes will succeed.

## Constraints (from Bob's API docs)

- **Auth:** HTTP Basic with `serviceUserId:token`.
- **Employee ID:** a service user isn't tied to an employee, so the user provides their Bob employee ID. This is the backend ID in their profile URL, not the company employee number.
- **Clock in:** `POST /attendance/employees/{employeeId}/entries/clock-in` with body `{"clockInTime": …}`, `entryType` `work` sent explicitly. The date part must be today.
- **Clock out:** `PATCH /attendance/employees/{employeeId}/entries/clock-out` with body `{"clockOutTime": …}`. It closes the employee's **most recent open entry, whatever its type**. The endpoint can't target a specific entry ID. The date part must be today. 404 means "no open clock-in" or "feature not enabled".
- **Search:** `POST /attendance/entries/search` ([reference](https://apidocs.hibob.com/reference/post_attendance-entries-search.md)):
  - requires both a `fields` projection and `filters`: `employeeId` `equals`, plus `clockInDate` `from`/`to` (at most 33 days per request)
  - returns only the fields that were requested
  - is paginated: `limit` defaults to 50 (max 100), `response_metadata.next_cursor` gives the next page, and `null` means there are no more
  - has no documented ordering
  - can return `work` or `break` entries, and start/end or duration entries
- **Entry rules:** no future entries, and no overlaps. For start/end entries, the clock-in and clock-out must be on the same date. Only entries in **running** timesheet cycles can be accessed through the public API. Entries in submitted, approved or locked cycles can't be changed ([Explore Attendance API](https://apidocs.hibob.com/docs/explore-attendance-api)).
- **Rate limits:** 100 requests per minute for clock-in/out, 200 for search.
- **Timestamps:** the docs' examples are offset-free (`2026-02-08T09:01:00`), and they don't say which time zone interprets them or defines "today". This is a gate (see Spikes).

## Approaches considered

| Approach | Tradeoff | Outcome |
|---|---|---|
| **Native Swift / SwiftUI** | Apple frameworks cover the menu bar, Keychain, notifications with action buttons, and launch at login. A few MB, low memory. Swift 6.4 is installed. Xcode is not, so it builds with SwiftPM and a bundling script. | **Chosen by the user** |
| Tauri (Rust + TypeScript) | TypeScript UI, closer to bob-cli. About 10 MB. Rust isn't installed. Keychain, notification actions and login items all come from plugins. | Rejected |
| Electron | About 150 MB for two buttons. | Ruled out |
| SwiftBar/xbar plugin | Requires installing another app. No action buttons in notifications. | Ruled out |

Distribution:

| Option | Tradeoff | Outcome |
|---|---|---|
| Own tap, unsigned (ad-hoc signed) | Free. The cask clears quarantine in `postflight`. Users trust the tap and skip Apple's malware check. | Rejected once the user decided to renew their Apple Developer subscription |
| **Signed + notarized**, in own tap | Apple Developer Program ($99/yr, user is renewing). Notarization in CI. No quarantine workaround. Also keeps the official Homebrew cask repo open as an option later (it requires Gatekeeper-passing apps since 2026-09-01). | **Chosen by the user** |

## Product behaviour (decided with the user)

- **One start and one stop per day. "Done" means done.** A mistaken stop is fixed in Bob.
- **Start and Stop always use the current time.** No scheduled, future or backdated times.
- **The app never edits or closes entries from earlier days.** For anything unusual, it explains and offers **Open Bob**.
- **Mutations are not retried automatically.** Reads may be.

### First run and Settings

- On first launch, or whenever credentials are missing, the Settings window opens.
- It has three fields: **Service user ID**, **Token**, **Bob employee ID** (digits only, labelled "not your employee number"). It assumes the user already has them.
- A **"How do I get these?"** link opens a help panel built into the app. The panel explains:
  - what a service user is
  - what to ask a Bob admin for: the permission and access scope listed under Prerequisites
  - the policy prerequisites
  - where to find the employee ID: open your profile in Bob and copy the long number after `/employee-profile/` in the URL ([Bob docs](https://apidocs.hibob.com/docs/update-employee-data))
- **Save** runs the validation check. Credentials are only stored if the check passes. Otherwise the window stays open and shows the reason.
- Settings also contains the reminder settings, notification permission status and **Launch at login** (see Lifecycle contract).

### Validation check

Validation runs the same state derivation as a refresh (below). Writes nothing.

| Result | Message |
|---|---|
| Derivation completes | Valid. Store credentials, show the derived state. |
| 401 | ID or token is wrong. |
| 403 | Missing the *Manage time attendance sheets* permission, or not scoped to this employee. Links to the help panel. |
| 404 | Attendance isn't enabled for this company in Bob. |
| Network / decode error | Can't confirm. Nothing is stored. |

If a token scoped to *other* people returns an empty list rather than 403, validation passes. The first Start then fails and shows Bob's error message. Spike 2 settles which happens.

### Nudges (decided with the user)

- In Settings:
  - **Remind me to start:** on/off + time (default 09:30)
  - **Remind me to stop:** on/off + time (default 18:00)
  - **Days:** weekday checkboxes (default Mon–Fri), shared by both reminders
- A nudge is a macOS notification with one action button: **Start day** or **Stop day**.
- At most one start nudge and one stop nudge per day. No snooze, no repeats.
- Delivery and catch-up are defined in the Lifecycle contract.

## Contract 1: State derivation

### What a refresh reads

- **Fields requested:** `/attendanceEntry/id`, `type`, `clockInDate`, `clockInTime`, `clockOutDate`, `clockOutTime`, `duration`.
- **Coverage:** two search windows, each ≤ 33 days, together covering the last **62 days** up to today. Each window follows `next_cursor` until it's `null`, with `limit: 100`. (Designer's call: one monthly cycle plus slack. Cycle boundaries aren't exposed through the API, so this is a fixed, documented horizon rather than a guarantee.)
- **Coverage is all-or-nothing.** If any page of any window fails, or doesn't decode, the result is **Unknown**. The app never falls back to a partial result.
- **Known limits, stated in the help panel:**
  - An open entry older than 62 days, or in a cycle the API can't access, is invisible to the app.
  - In that case Start may be offered, and Bob is expected to reject a clock-in that would overlap. That expectation is unverified (Spike 4). Bob's error message is then shown.
  - The app doesn't claim "no open entry exists". It claims "none found in the last 62 days of accessible data".

### Reduction

The app combines **all** entries from every window and page, without relying on their order. It then applies the first matching rule:

| # | Condition | State | Actions |
|---|---|---|---|
| 1 | Any window incomplete, failed, or undecodable | **Unknown** (shows the reason) | Refresh, Open Bob, Settings |
| 2 | Any entry that has neither a start/end pair nor an open start (e.g. duration-only, missing `clockInTime`) dated today, **or** more than one open entry of any type | **Unsupported** ("Bob has entries this app can't safely handle") | Open Bob |
| 3 | An open `break` entry (any date) | **OnBreak** ("A break is open in Bob. End it there.") | Open Bob |
| 4 | An open `work` entry with `clockInDate` before today | **StaleOpen** ("Clocked in since Mon 09:12. Fix it in Bob.") | Open Bob |
| 5 | Exactly one open `work` entry dated today | **Running(entryId, clockInTime)** | Stop day |
| 6 | No open entries, and ≥ 1 closed `work` entry today | **Done(earliest in, latest out)** | none |
| 7 | Otherwise | **NotStarted** | Start day |

- An "open" entry has a `clockInTime` but no `clockOutTime`. Since `clockOutTime` is always requested, a missing value means it really is absent.
- "Today" and "before today" use the attendance calendar settled in Spike 1, and the same calendar is used for the search date boundaries.
- Closed `break` entries and closed entries from earlier days don't affect the state.

### Menu and icon per state

| State | Menu | Icon |
|---|---|---|
| Checking (no successful derivation yet) | "Checking Bob…" | outline clock, dimmed |
| NotStarted | **Start day** | outline clock |
| Running | "Started 09:02", **Stop day** | filled clock |
| Done | "Done today, 09:02–17:41" | check mark |
| StaleOpen / OnBreak / Unsupported | explanation, **Open Bob** | warning |
| Unknown | reason, **Refresh**, **Open Bob** | warning |
| Uncertain (see Contract 2) | "Couldn't confirm whether Bob recorded this", **Refresh**, **Open Bob** | warning |

Every state also shows **Settings…** and **Quit**.

## Contract 2: Action safety

- **One command path.** The menu items and the notification buttons all go through the same `perform(action, expectation)` routine.
- **One mutation at a time.** While a mutation is running, the state is **Busy**: all actions are disabled, and a second request (from the menu or a notification) is rejected locally without a network call.
- **Every action is checked before it writes:**
  1. Run a full refresh (Contract 1).
  2. Compare the result with the action's expectation:
     - **Start** expects NotStarted, on the same attendance date, for the same employee.
     - **Stop** expects Running with the **same `entryId`**, on the same attendance date, for the same employee.
  3. If the state doesn't match, make **no write**. Show "Bob has changed since this was offered" and the fresh state.
  4. If it matches, send exactly one write request.
- **Notification actions carry their expectation.** The `userInfo` holds `{action, employeeId, attendanceDate, entryId?}`. Clicking an old notification goes through the same check, so a stale Stop from yesterday can never close today's session.
- **Notifications are cleaned up.** When the state changes, any delivered nudge whose expectation no longer matches is removed (`removeDeliveredNotifications`).
- **After a successful write:** refresh immediately.
- **Uncertain writes.** If the request times out or the connection drops after it was sent, the outcome is unknown. The app then:
  - enters **Uncertain** and **does not replay the write**
  - runs one read to reconcile: if the refresh shows the expected result (Running for Start, Done for Stop), the write is treated as recorded; any other derived state is shown as-is
  - stays in Uncertain if that read also fails, until a manual Refresh or the next lifecycle trigger
- **Definite rejections.** A 4xx from the write shows Bob's `error` message, then the app refreshes.
- **Stale responses are ignored.** Each refresh carries a sequence number. A response older than the latest completed mutation, or the latest applied refresh, is dropped, so an earlier slow read can't overwrite newer confirmed state.
- **Residual race, accepted:** Bob's API has no conditional writes. Between the check and the write, another client (the web app or phone) could change the state. With one person using it, this window is about a second, and it's accepted.

## Contract 3: Lifecycle

### Refresh triggers

A refresh runs on:
- launch (including at login)
- wake from sleep (`NSWorkspace.didWakeNotification`)
- the menu opening (the existing menu shows "Checking…" next to the current state until the refresh returns)
- before any action, and after any write
- day change (`NSCalendarDayChanged`)
- time zone change (`NSSystemTimeZoneDidChange`)
- the reminder evaluator (below)

There is **no periodic polling**. A clock-out made on the phone while the Mac stays awake and the menu stays closed only shows up at the next trigger. Opening the menu always shows current state. On day change, a stored Done or NotStarted becomes Checking until the refresh completes.

### Reminders

- **Evaluated by the running app**, not by scheduled local notifications, because those can't run a Bob check just before they appear ([Apple](https://developer.apple.com/documentation/usernotifications/scheduling-a-notification-locally-from-your-app)).
- **When it runs:** the app keeps one timer for the next reminder time. It also re-evaluates at launch, wake and day change.
- **Which states trigger a nudge:**
  - The **start** reminder is due on enabled days, from its time until the stop-reminder time (or 23:59 if the stop reminder is off), and only in NotStarted.
  - The **stop** reminder is due from its time until 23:59, and only in Running.
  - So a missed morning nudge isn't shown in the evening, and nothing is shown for any other state.
- **Order:** when a reminder is due, the app refreshes. If the state matches, it posts an **immediate** notification carrying the action's expectation (Contract 2), then records `lastDelivered[kind] = attendanceDate` in `UserDefaults`. Because this is saved, restarting the app doesn't produce a duplicate.
- **If the Bob check fails:** no nudge, and nothing is recorded. The app re-evaluates every 15 minutes while the reminder is still within its window. These are read retries only.
- **Enabling a reminder, or changing its time, after today's time has already passed:** treated as already delivered today, so the change takes effect from the next enabled day.
- **While the app is quit:** no reminders. On the next launch, the rules above apply, so a reminder still within its window fires once.
- **Notification permission:**
  - It's requested when a reminder is first enabled.
  - Settings shows the real status from `UNUserNotificationCenter.notificationSettings()`.
  - If it's denied or turned off later, Settings shows "Notifications are off in System Settings" with a button to open them. The reminder toggles stay as saved but are marked inactive.

### Launch at login

- **`SMAppService.mainApp.status` is the source of truth** for what Settings shows. A saved boolean is not.
- On first run, the app calls `register()` once (the default is on), and stores only that this was attempted.
- The toggle calls `register()` or `unregister()`.
- Errors are shown inline.
- `.requiresApproval` shows "Allow Bob Clock in System Settings > Login Items" with a button that calls `SMAppService.openSystemSettingsLoginItems()` ([Apple](https://developer.apple.com/documentation/servicemanagement/smappservice/status-swift.enum/requiresapproval)).
- Status is re-read whenever Settings opens.

### Storage

- **Credentials:**
  - Service user ID, token and employee ID are stored together as one generic-password item in the **file-based login keychain** (no `kSecUseDataProtectionKeychain`). That way no keychain-access-group entitlement or provisioning profile is needed.
  - The item's access list trusts the app's designated requirement (Developer ID team + bundle ID), so it survives release upgrades. (Designer's call. Spike 5 verifies it.)
  - Local development builds use a different bundle ID (`dev.richardvancamp.bobClock.dev`), so a differently signed build never shares or prompts for the release item.
- **Everything else** goes in `UserDefaults`: reminder settings, `lastDelivered`, and the attempted-login-item flag.

## Spikes (gates before building state logic)

These run against the user's real Bob account, through a throwaway script or test target, before Contract 1 is implemented. Their results are written back into this document.

1. **Time zone and "today."**
   - Which time zone interprets offset-free timestamps?
   - Are explicit offsets accepted and honored?
   - Which calendar defines "today" for the must-be-today rule, and for `clockInDate`?
   - Test with the Mac's time zone set **different** from the employee's Bob/site time zone.
   - The outcome fixes a single `AttendanceCalendar`, used for writes, search bounds and reduction. If Bob's time zone can't be known without extra permissions, the fallback is device-local time plus a documented limitation.
2. **Wrongly scoped token:** does search return 403 or an empty list?
3. **Closing an overnight entry:** does clock-out of an entry from an earlier day get rejected? The design refuses it either way. This only tells us which message to show.
4. **Clock-in with an existing open entry** (work or break, including one outside the search window): is it rejected? This decides how much the 62-day horizon can be relied on.
5. **Signed build:**
   - Run one manual Developer ID sign → notarize → staple.
   - Check that the notification action buttons and `SMAppService` registration work in the notarized build.
   - Check that the Keychain item stays readable after installing a newer signed build over the old one.

## Platform and build

- Swift, SwiftUI `MenuBarExtra` (menu style), `LSUIElement` (no Dock icon).
- `UserNotifications` with one category per action.
- `SMAppService.mainApp`.
- `NSWorkspace` and calendar notifications as triggers.
- Minimum macOS 14. Apple silicon only.
- **Domain logic is pure and has its own target:** state reduction, action expectation checks, reminder eligibility, and the calendar. HTTP, clock, Keychain and notifications are injected, so all of it can be tested without Bob.
- Built with SwiftPM (no Xcode project). A script:
  - assembles `Bob Clock.app` (`Info.plist` with bundle ID, `LSUIElement`, version)
  - signs it with the **Developer ID Application** certificate, with hardened runtime and a secure timestamp (`codesign --force --options runtime --timestamp`). No sandbox. No extra entitlements are expected, given the file-based keychain choice; Spike 5 confirms this
  - zips it with `ditto -c -k --keepParent`, submits it with `xcrun notarytool submit --wait`, staples it with `xcrun stapler staple`, then zips the stapled app again for release
  - also supports an ad-hoc signed dev build (`--sign -`, `.dev` bundle ID, no notarization)
- Product name `Bob Clock`, slug `bob-clock`, bundle ID `dev.richardvancamp.bobClock`. This follows the existing `dev.richardvancamp.pinboardWizard` (designer's call, for the user to confirm). It must not change after the first release.

## Distribution

- **App repo:** [`RikuVan/bob-clock`](https://github.com/RikuVan/bob-clock), public, so that Homebrew can download release assets without auth.
  - Pushing a `v*` tag runs GitHub Actions on `macos-latest` (arm64).
  - The workflow imports the Developer ID certificate into a temporary keychain, then builds, bundles, signs, notarizes, staples and zips.
  - It attaches `bob-clock.zip` to the GitHub release.
  - It then sends a `repository_dispatch` (`update-bob-clock`, payload `version`) to `RikuVan/homebrew-formulae`.
- **App repo secrets:**
  - `DEVELOPER_ID_P12` (base64 certificate + private key) and `DEVELOPER_ID_P12_PASSWORD`
  - a **team** App Store Connect API key with the Developer role, for `notarytool --key/--key-id/--issuer`: `NOTARY_KEY_P8`, `NOTARY_KEY_ID`, `NOTARY_ISSUER_ID`
  - `HOMEBREW_TAP_TOKEN`: a fine-grained token that can send dispatch events to the tap
  - An API key is used rather than an Apple ID + app-specific password because it isn't tied to a personal login and can be revoked separately (designer's call, for the user to confirm).
- **Tap:** add `Casks/bob-clock.rb`, following `pinboard-wizard.rb`:
  - `version`, `sha256`, `url` to the release zip, `name`, `desc`, `homepage`
  - `depends_on macos: ">= :sonoma"`, `depends_on arch: :arm64`
  - `app "Bob Clock.app"`
  - `uninstall quit:` with the bundle ID
  - `zap trash:` for preferences. The Keychain item is left for the user.
- **Tap workflow:** add a separate `update-bob-clock.yml`, modelled on the Pinboard-specific `update-cask.yml` (which stays unchanged). It triggers on `update-bob-clock`, updates the cask, runs `brew audit`, and opens a PR.
  - It uses the **tap's own** secrets. Dispatching an event doesn't pass secrets across repos.
  - The existing `HOMEBREW_TAP_TOKEN` in the tap is reused if it's still valid.

## Testing expectations

Unit tests on the pure domain target. Each one is a consumer-visible boundary:

- **Reduction:**
  - each state in Contract 1
  - an open break → OnBreak, with no Stop offered
  - an open work entry 40 days back (second window) → StaleOpen, not NotStarted
  - a failed second window, or a failed later page → Unknown
  - the same entries split across pages and shuffled → same state
  - a duration-only entry today → Unsupported, with no times shown
  - two open entries → Unsupported
- **Actions:**
  - a stale Stop notification (yesterday's date, or a different `entryId`) → no write
  - a Start while Busy → rejected locally
  - a timed-out write → Uncertain, followed by one read and no replay
  - an older refresh response arriving after a mutation → dropped
- **Reminders:**
  - restart after delivery → no duplicate
  - start reminder past the stop-reminder time → not delivered
  - Bob check fails → not recorded, re-evaluated later
  - enabling a reminder after its time → takes effect the next day
- **Calendar:** midnight rollover, and the device time zone differing from the attendance calendar → consistent day classification.
- **Launch at login:** the status mapping (`enabled`, `notRegistered`, `requiresApproval`, `notFound`) → the Settings text. Revoking permission in System Settings is checked manually.

Manual smoke test against real Bob before the first release:
- validation with a bad token, and with a token missing the permission
- Start → Stop, then check that both appear in *My attendance*
- clock in from the web → the menu shows Running
- start a break in Bob → the menu shows OnBreak

## Left to the builder

- Code structure and file layout within the targets above. The test framework (XCTest or Swift Testing).
- Icon artwork (SF Symbols are fine).
- Exact wording of messages and of the help panel.
- The release workflow details beyond the contract above.
