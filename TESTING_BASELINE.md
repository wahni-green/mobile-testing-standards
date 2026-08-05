# Baseline Invariants — Flutter + ERPNext

Cross-cutting checks that apply to every screen in the app, regardless of which feature it belongs to. Referenced from [TESTING_ARCHITECTURE.md](TESTING_ARCHITECTURE.md) as Milestone 0 — scaffold these before any feature-specific Track 1 work ships.

Nobody owns "does every date field format correctly" the way someone owns the Sales Order screen, so it gets skipped unless it's built once into a shared component and tested there, instead of re-checked screen by screen.

**Implementation convention:** put the shared building block in `lib/shared/` (e.g. `AppDateFormatter`, `AppCurrencyFormatter`, `AsyncActionButton`, `DeviceIntegrityService`) and its invariant test in `test/baseline/`, wired as its own required CI check — independent of and in addition to the feature-specific test suite. A screen is compliant by construction if it uses the shared component; the lint/grep checks below exist to catch the screen that didn't.

Sections marked **conditional** only apply if the app actually has that feature — include the ones matching what the app does, skip the rest.

---

## Formatting & locale

| Check | Why it matters | How to verify |
|---|---|---|
| Every date renders as `DD-MM-YYYY` | ERPNext's API defaults to `yyyy-mm-dd`; a raw API date leaking into a widget reads wrong for the org's convention and increases misread risk | Widget/golden test asserts every date-bearing widget matches `^\d{2}-\d{2}-\d{4}$`. Enforce via one shared `AppDateFormatter`; a CI grep step fails the build if a new raw `DateFormat(` call appears outside it |
| Currency amounts always show 2 decimal places — 3 for the small set of currencies actually quoted to 3 (BHD, IQD, JOD, KWD, OMR, TND) | A formatter that hardcodes 2 decimals silently truncates/mis-rounds the 3-decimal currencies, and a UI that rounds differently than ERPNext causes "the numbers don't match" tickets | Unit test the shared `AppCurrencyFormatter` against every 3-decimal currency code plus a handful of 2-decimal ones (AED, USD, EUR); enforce via one shared formatter — a CI grep step fails the build if a new `NumberFormat('#,##0.00')` or bare `.toStringAsFixed(2)` appears on an amount outside it |
| Timezone-aware fields (created/modified, scheduled times) show local time, clearly labeled | ERPNext stores in system timezone; silent UTC-vs-local drift makes appointments/deadlines look wrong | Widget test with a fixed mock clock/timezone; assert displayed value against the expected local conversion |
| No raw `null` or empty string ever rendered as visible text | Reads as broken, undermines trust in the rest of the data on screen | Snapshot test with a record missing optional fields; assert a defined empty-state string is shown, never `"null"` |
| Business-critical formatting uses the org's configured locale, not the device's default locale | A device set to `en-US` will silently reformat dates/numbers if the app trusts `Platform.localeName` / `Intl` defaults instead of an explicit app-level setting — the date-format bug reappears through a different path | Test renders formatted output with the device locale mocked to something else; assert output is unchanged |

## Interaction safety (idempotency)

| Check | Why it matters | How to verify |
|---|---|---|
| Every submit/action button disables (or debounces) itself for the duration of its request | Rapid double-tap on a slow connection is the most common cause of duplicate ERPNext records — duplicate Sales Orders, duplicate Payment Entries | Tap the button N times synchronously against a mocked API client; assert exactly 1 call fired. Build once into a shared `AsyncActionButton` — every screen using it inherits the guarantee |
| Buttons show a loading state distinct from idle/disabled | The user needs to see the tap registered, or they tap again out of doubt, which is exactly the behavior being guarded against | Widget test asserts a loading indicator appears immediately on tap, before the future resolves |
| Navigating away mid-request doesn't crash or silently orphan the result | Cancelling the request is fine; a `setState` on a disposed widget or an orphaned draft record is not | Integration test: trigger the action, immediately pop the route, assert no exception and no dangling record |
| Pull-to-refresh and other repeat-gesture actions are debounced the same way | Same failure mode as button taps — easy to forget it also applies to list-refresh and retry gestures | Same call-count assertion pattern, applied to the refresh controller |

## Device & session integrity

| Check | Why it matters | How to verify |
|---|---|---|
| Developer mode / USB debugging is detected and blocks (or flags) sensitive actions | A device with developer mode plus a fake-GPS app is the standard way to spoof location for check-ins and field visits | Unit test the detection service against a fake platform channel returning "dev mode on"; assert the action is blocked, not silently allowed |
| Mock/fake location providers are detected on every location read, not just once at launch | Android's mock-location flag is per-`Location` (`isFromMockProvider`), not a one-time device setting — checking only at app start misses a fake-GPS app turned on mid-session | Unit test feeds a `Location` with `isFromMockProvider = true` into the location service at read time; assert it's rejected |
| Root/jailbreak detection gates the highest-risk flows (payments, approvals) | Broader "is this device trustworthy" check beyond dev-mode/mock-location specifically | Same fake-platform-channel technique; assert the gated action refuses to proceed |
| Auth token lives in secure storage, never `SharedPreferences`/plaintext | Token theft from a rooted device or an unencrypted backup is a real ERPNext account-takeover path | Code-level check: CI greps for `SharedPreferences` anywhere near the token; fails the build if found |
| A 401 from ERPNext forces logout, not a silently retried request | A stale token retried forever just looks like "the app is broken" and masks the real auth problem | Test with a mocked 401 response; assert session is cleared and the user is routed to login |
| Device system clock manipulation is detected for time-sensitive actions (check-in/check-out) | Same fraud family as mock location — set the clock back or forward instead of spoofing GPS to fake attendance times | Unit test the integrity service against a fake platform channel reporting a clock/timezone mismatch vs. server time; assert the action is blocked or flagged |
| Runtime permissions (location, camera, storage) are re-checked at the point of use, not cached from the first grant | A user can revoke a permission mid-session from OS settings; a cached "granted" flag lets the app act on a permission it no longer has | Test the permission check is called immediately before each sensitive action, not only once at app start |
| Sensitive screens (salary, payment details, approvals) block screenshots and screen recording | Field devices are shared/lost more often than desktops; a screenshot of a payslip is a real leak | Platform test asserts the secure/no-capture flag is set (Android `FLAG_SECURE`) when a screen tagged sensitive is pushed, and cleared when it's popped |
| App content is hidden or blurred in the OS task switcher while backgrounded | The app switcher preview is effectively a screenshot the OS takes for you | Integration test backgrounds the app from a sensitive screen and asserts the privacy overlay is shown |

## Input validation & typing

| Check | Why it matters | How to verify |
|---|---|---|
| Required fields block submission until filled, mirroring each doctype's mandatory flags | Client-side validation that's out of sync with ERPNext's mandatory fields either blocks valid submissions or lets invalid ones through to fail server-side with a worse error | Test submits with each mandatory field empty in turn; assert the button stays disabled or submission is blocked client-side |
| Whitespace-only input is treated as empty, not valid | A single space bypasses naive `isNotEmpty` required-field checks | Test entering `" "` into a required field behaves identically to entering nothing |
| Numeric/currency fields use a numeric keyboard and send correctly-typed values, not stringified numbers | ERPNext expects float/int types on these fields; a stringified `"100"` sent where `100` is expected causes silent precision or validation issues | Test asserts the field's `keyboardType` and that the payload sent to the mocked API client is numeric, not a string |
| Large quantity/amount values never render in scientific notation | A known float-parsing failure mode (`1.0E7` instead of `10000000`) that reads as the app being broken | Widget test renders a large value through the shared number formatter and asserts plain decimal output |

## Search & list correctness

| Check | Why it matters | How to verify |
|---|---|---|
| Search-as-you-type is debounced before firing an API call | The same problem as button double-tap, triggered by typing instead of tapping — every keystroke firing a request floods the API and the ERPNext server | Test rapid simulated keystrokes against a mocked API client; assert only one call fires, after the debounce window |
| Pagination/infinite scroll never duplicates or drops records | A common bug when new data arrives while the user is mid-scroll, or when a page is refetched after an edit | Integration test scrolls through a mocked multi-page list; assert every record appears exactly once in the final list |

## Unsaved work & session lifecycle

| Check | Why it matters | How to verify |
|---|---|---|
| In-progress form data survives backgrounding and an OS-level low-memory kill-and-restore | Losing a half-filled form to a phone call or a backgrounded app being reaped is one of the most common field-app complaints | Integration test backgrounds and restores the app mid-form (simulating state restoration); assert entered values are still present |
| A token expiring mid-form preserves entered data and prompts re-auth, rather than discarding the form | Combines the 401-forces-logout rule above with data preservation — logging the user out is correct, losing 10 minutes of data entry on top of it is not | Test triggers a 401 mid-form-fill; assert the form state is retained and restored after re-authentication |
| Navigating away from a form with unsaved changes prompts confirmation | Prevents accidental data loss from a stray back-gesture | Test navigating back from a dirty form asserts a confirmation dialog appears; a clean (unedited) form navigates away silently |

## Document state & schema resilience

| Check | Why it matters | How to verify |
|---|---|---|
| Submitted/cancelled documents are read-only in the UI, matching ERPNext's docstatus rules | Editing a submitted Sales Invoice client-side and having the server silently reject it (or worse, partially apply it) is confusing and can corrupt state | Test renders a doc with `docstatus = 1` (submitted) and `docstatus = 2` (cancelled); assert edit controls are disabled/hidden for both |
| Child table rows (e.g. Sales Order Items) can be added, removed, and reordered without corrupting sibling rows or losing unsaved edits | Child tables are the most common source of index-related bugs — deleting row 2 of 5 should never mutate row 3's data | Widget test adds/removes/reorders rows against a known fixture; assert final state matches expected row-by-row |
| An unexpected or missing custom field from the API doesn't crash the screen | Admins add/remove custom fields in ERPNext independent of the app's release cycle; the app should degrade gracefully, not throw on `null` | Test renders a screen against a response fixture with an extra unknown field and one with a missing optional field; assert no exception in either case |
| A stale-data conflict (another user modified the record first) is surfaced to the user, not silently overwritten | ERPNext raises `TimestampMismatchError` on exactly this case; swallowing it and force-saving anyway silently loses the other user's edit | Test mocks a `TimestampMismatchError` response on save; assert the user sees a conflict message, not a silent success |

## Attachments & media

| Check | Why it matters | How to verify |
|---|---|---|
| File size and type are validated client-side against the same limits ERPNext enforces | Catching an oversized or wrong-type file before upload is a better experience than a server rejection after a slow upload completes | Test attempts an oversized/wrong-type file; assert it's rejected before any network call |
| Images are compressed/resized before upload | Field devices often have high-resolution cameras; uploading unresized photos over mobile data is slow and wastes ERPNext file storage | Test asserts the uploaded payload size is under the configured threshold for a large source image |
| Captured photo orientation (EXIF) displays correctly after upload | A well-known mobile bug where photos taken in portrait render sideways once round-tripped through storage | Test uploads a fixture image with EXIF rotation data; assert the displayed/stored orientation matches the original |
| Cancelling an in-progress upload doesn't leave an orphaned File record in ERPNext | A cancelled upload that already reached the server leaves storage cruft and a File doc pointing at nothing | Test cancels mid-upload against a mocked client; assert no attachment reference is saved |

## Accessibility baseline

| Check | Why it matters | How to verify |
|---|---|---|
| Tap targets are at least 48×48dp | Below this, field users wearing gloves or using the app one-handed mis-tap adjacent controls | Widget test asserts the rendered size of interactive elements meets the minimum |
| Icon-only buttons carry a semantic label | A screen-reader user — or an automated a11y test — has nothing to go on otherwise | Test asserts `Semantics`/`tooltip` is present on every icon-only `IconButton` |
| Layout survives the OS's maximum text-scale setting | Users with visual impairments scale system text up; a layout that clips or overlaps at 200% scale is unusable for them | Widget test renders key screens with `textScaleFactor: 2.0`; assert no overflow errors |
| Color choices meet baseline contrast (WCAG AA) for text and status indicators | Low-contrast status pills/labels are the most common accessibility miss in dashboards | Automated contrast check against the app's defined color tokens, not per-screen eyeballing |

## Stability baseline

| Check | Why it matters | How to verify |
|---|---|---|
| App cold-launches without crashing on every OS version the team supports | The most basic possible regression, and the easiest to miss if CI only runs against one emulator image | Smoke test in the device matrix: launch to the login/home screen on each supported OS version, assert no crash |
| Repeated navigation to and from the same screen doesn't leak memory or duplicate listeners | A `StreamSubscription` or controller not disposed on `dispose()` accumulates silently until the app degrades hours into a shift | Integration test navigates to/from a screen N times; assert listener/controller count returns to baseline, not growing |

## Conditional — only where the feature exists

| Check | Applies when | Why it matters | How to verify |
|---|---|---|---|
| Queued offline actions replay exactly once on reconnect | The app supports offline capture/queueing | The offline-sync equivalent of the button-debounce problem — a retried queue is a duplicate-record risk, just delayed | Test queues an action offline, simulates reconnect twice in a row; assert the mocked API receives exactly one call |
| Rapid duplicate barcode/QR/NFC scans within a short window are deduped | The app scans barcodes, QR codes, or NFC tags (inventory, asset tracking) | A shaky hand or a reflective surface can register the same physical scan twice in quick succession | Test feeds two identical scan events within the debounce window; assert only one action is triggered |
| An ERPNext maintenance-mode or 503 response is shown as a proper message, not a JSON-parse crash | The backend has scheduled maintenance windows the app may hit | ERPNext returns an HTML maintenance page, not JSON, when in maintenance mode — a naive JSON decode throws instead of showing a message | Test with a mocked HTML/503 response in place of JSON; assert a defined "service unavailable" UI state, not an unhandled exception |
| A push notification tapped from a cold start routes to the correct, authenticated screen | The app supports push notifications | Cold-start notification handling is a different code path than in-app navigation and is easy to leave untested | Integration test launches the app via a mocked notification payload; assert it lands on the expected screen, authenticated |
| A deep link to a doctype the user lacks permission for shows a proper error, not a crash | The app supports deep links | A shared link opened by someone without access is a normal occurrence, not an edge case | Test opens a deep link with a mocked 403 permission response; assert a defined "no access" screen, not an unhandled exception |
| Biometric/PIN re-authentication is required after the app has been backgrounded past a configured threshold | The app implements an app-lock feature | Verifies the lock actually re-engages rather than only prompting once at cold start | Integration test backgrounds the app past the threshold and resumes it; assert the lock screen appears before any content |
| API calls respect the currently selected company context, with no cross-company data leakage | The ERPNext instance has multiple companies | Mixing data across companies is a real data-integrity and confidentiality issue, not just a display bug | Test switches company context and asserts subsequent API calls and cached data are scoped to the newly selected company only |
| Text doesn't overflow or truncate awkwardly, and RTL layouts mirror correctly | The app supports multiple languages, including a right-to-left one | Translated strings are often longer than English, and RTL is easy to get half-mirrored | Widget test renders key screens in the longest supported translation and in an RTL locale; assert no overflow and correct mirroring |
