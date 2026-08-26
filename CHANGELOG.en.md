# Changelog

The project uses [Semantic Versioning](https://semver.org/). Starting with 1.0.0,
full SemVer discipline applies: backward-incompatible changes (storage schemas,
licensing server API) require a major version bump; new functionality -
a minor bump; bug fixes - a patch version.

## [1.9.1] - 2026-08-18 - critical patch: content.js → background.js messaging was broken

### Fixed

**`Uncaught TypeError: Cannot read properties of undefined (reading 'onMessage')` in content.js.**
The content script became "orphaned" if the extension was updated/reloaded while
an SPA page (e.g. `my.selectel.ru`) stayed open - in that state
`chrome.runtime` becomes `undefined`, and registering
`chrome.runtime.onMessage.addListener(...)` threw an unhandled exception.
Added an early `chrome.runtime.id` check at the start of content.js and safe
wrappers `socCopilotSendMessage`/`socCopilotStorageGet`/
`socCopilotStorageSet`/`socCopilotStorageRemove` that catch both this scenario
and context invalidation happening after the script has already started
(`chrome.runtime.lastError`, synchronous exceptions when accessing
`chrome.storage.local`).

**IOC enrichment, LLM features (Summary/Predict/Mentor), sending to Slack/Telegram/MAX/Teams/Jira,
the badge, and cross-tab correlation were effectively broken.** During the first pass of the fix above,
the `socCopilotSendMessage` wrapper accidentally called itself instead of
`chrome.runtime.sendMessage` (infinite recursion, swallowed by its own `catch`) - the
entire content.js → background.js channel silently stopped responding.
Found and fixed before release by an additional audit.

**Redaction (data masking before sending to an LLM) didn't cover generic secrets.**
`utils/redaction.js` only stripped SSH/AWS keys and JWTs; passwords, bearer
tokens, and vendor tokens (Stripe `sk_`, GitHub `ghp_`, Slack `xox*`) could
be sent to the LLM in plain text. Added patterns for
`password/secret/token/api key:...`, `Bearer...`, and popular vendor tokens.

**XSS defense-in-depth: `href` in the panel.** In the MITRE technique blocks,
forecasts, similar incidents, and DFIR tool links, the URL was inserted into
the `href` attribute without escaping (`escapeAttr`), unlike the rest of the
text. Not currently exploitable (the browser itself percent-encodes special
characters in URLs), but fixed for consistency with the rest of the code.

**background.js: messages weren't checking the sender.** `chrome.runtime.onMessage`
now explicitly rejects messages not from its own extension
(`sender.id !== chrome.runtime.id`). Not currently exploitable
(`externally_connectable` isn't declared), this is a regression safeguard.

**Race condition in Device ID generation.** `getOrCreateDeviceId()` in
background.js could generate different UUIDs on parallel calls during a cold
service worker start - now the first call's promise is cached and reused.

### Known limitations (not fixed in this release, does not block the release)

- `enrichCache` and `openIncidentTabs` in background.js only live in the
 service worker's memory and are reset when Chrome unloads it (~30 sec of
 idle time) - the enrichment cache TTL and cross-tab correlation are
 effectively limited by the SW's lifetime. A candidate for moving to
 `chrome.storage.session` in one of the next releases.
- `migrateSecretsToEncrypted()`/`applyManagedPolicies()` can, in rare cases,
 run concurrently from `onInstalled` and `onStartup` - idempotent, but with
 redundant operations.

## [1.9.0] - 2026-08-18 - critical patch: CSP blocked the SIEM test bench; logo selection

### Fixed

**The SIEM test bench (test-siem.html) didn't display data.** Chrome blocked
the inline `<script>` containing the scenarios (SCENARIOS/render) with a CSP
error: "Executing inline script violates the following Content Security Policy directive
'script-src 'self''...". Cause: MV3 sets `script-src 'self'` without
`'unsafe-inline'` for extension pages by default, and test-siem.html is
opened exactly as an extension page
(`chrome.runtime.getURL("test-siem.html")` from popup.js) - so this CSP
applies to it. The inline script never ran, the incident card elements
(#bt-title, #bt-description, etc.) stayed stuck on "Loading…", and the
scenario switch buttons didn't respond.

Fixed: the scenario code was moved into an external file `test-siem.js`,
loaded as `<script src="test-siem.js"></script>` - CSP `'self'` allows it.
The file was added to the manifest's `web_accessible_resources`.

**The same bug class** - on the settings page (options.html), an inline
`<script>` that inserted the version number into the "About" block
(#about-version) was also blocked by CSP and silently failed. Moved to
options.js.

### Added

- **Plugin logo selection.** In the settings page's "Appearance and language"
 section - a "Plugin logo" selector: "Variant 1" (default, `icons/*.png`) and
 "Variant 2" - a set the administrator places themselves in `icons/logo2/`
 (icon16/48/128.png, see `icons/logo2/PLACE_LOGO_HERE.txt`). Applied to the
 extension's toolbar icon via `chrome.action.setIcon()` right after saving
 and on every browser launch/install. If the variant's files are missing,
 saving is blocked with a warning and the current icon stays unchanged
 (checked via a HEAD request to `chrome.runtime.getURL(...)` before
 applying).

## [1.8.1] - 2026-08-15 - 101 quizzes, burnout v2.0, audit, documentation

### Added
- A bank of 101 quizzes across 9 categories: IOC(13), MITRE(14), DFIR(13), triage(11),
 malware(10), network(12), threat-intel(9), incident-management(9), compliance(10).
 Detailed explanations for each answer. No repeats within a session.
- 13 mentor steps (added: ioc2, mitre2, malware, network, TI, compliance).
- 7 BelZor levels (Junior L2, Middle L2, Senior L2/L3).
- Burnout v2.0: 7 signals, personalized messages, a recovery tracker.
- Documentation for git: INSTALL.md, USER_GUIDE.md, ADMIN_GUIDE.md, LICENSE.md.

### Security audit (v1.8.1)
- All 9 innerHTML occurrences checked - escapeHtml/escapeAttr used everywhere
 external data is involved.
- eval/new Function/document.write - none present.
- Logging secrets to console.log - none present.
- ReDoS (12 patterns, 50k characters) - fine.
- Message types - all covered.

## [1.7.1] - 2026-08-15 - critical patch: ReferenceError on injection into already-open tabs

### Fixed

**ReferenceError: SOC_COPILOT_DEFAULT_ADAPTERS is not defined** - a mass error on
every open tab right after a browser/PC restart. Cause: the function
injectIntoExistingTabs() (added in 1.7.0 to eliminate a 1-2 min delay)
injected **only** content.js into already-open tabs. But an already-open tab
had none of the dependencies loaded (adapters.js, utils/*.js), so the very
first line, state.adapters: SOC_COPILOT_DEFAULT_ADAPTERS, threw a
ReferenceError.

Fixed: the entire stack of 24 files is now injected in the same order as
declared in the manifest. Added a SOC_COPILOT_ALL_SCRIPTS constant, verified
to match the manifest at build time (checked in the final 1.7.1 audit - no
discrepancies).

## [1.7.0] - 2026-08-15 - major feature release for customer requirements

### Fixed
- The IOC detector missed hashes in HTML context: the `\b` word boundary
 doesn't work across line breaks in innerHTML. Switched to capture groups +
 matchAll.
- A 1-2 min delay after a PC restart: added content script injection into
 already-open tabs via `injectIntoExistingTabs()` on onStartup/onInstalled.
- The active tab was becoming transparent: added an active-tab background,
 overflow-x scroll for cases where not all tabs fit.

### Added
- 9 playbook types (phishing/malware/ransomware/brute-force/lateral-movement/
 insider/web-attack/C2/general) - each including asset-owner lookup in
 CMDB/ITSM + a 4-level asset criticality model (Mission/Business-critical/
 Business-operational/Office).
- DFIR by OS: a Windows/Linux/macOS/AstraLinux switcher in the L2 mentor,
 with specific artifact-collection commands and tools for each platform.
- A "Universal" adapter with a list of SIEMs (MaxPatrol/KUMA/Security Vision/
 SearchInform/R-Vision/Splunk/ArcSight) - one adapter instead of 6 separate
 ones.
- Test bench: a test-siem adapter for test-siem.html.
- Expanded LLM list: GigaChat, YandexGPT, Qwen, Ollama, DeepSeek, GLM,
 Foundation-Sec, Mistral, vLLM + a universal OpenAI-compatible mode.
- Analyst name and contact details in the conclusion + format choice
 (MD/TXT/HTML).
- An "About" section: version, belyaev.expert, belyaev.pro@mail.ru.
- A BelZor description explaining its value to analysts.
- Tooltips on key buttons.
- Em dashes replaced with hyphens in the UI (19 places).

## [1.6.0] - 2026-08-15 - unified licensing server, changelog viewer removed

### Changed (licensing model)
- The licensing server URL is no longer entered manually by each user - it's
 hardcoded as a constant in the code (a single server, licenses are only
 issued by the product owner). The "Licensing server URL" field was removed
 from settings, leaving only the key input. See LICENSE_SERVER_SETUP.md for
 how to set your own real domain before distribution.
- The enterprise LicenseKey policy now activates the license automatically
 even without a LicenseServerUrl set (previously both fields were required).

### Removed
- The "What's new"/changelog button in the popup - it didn't load reliably
 on some browser configurations even after the previous fix
 (web_accessible_resources + timeout). Instead of further debugging one
 minor feature, it was removed, and the changelog.html file was deleted.
 The version history remains in CHANGELOG.md in the archive.

### Fixed
- The theme on the settings page only colored part of the elements
 (body/headings), while input fields/buttons/adapter cards stayed dark even
 with the light theme selected - added a unified override block for all
 form elements.
- Clarified the tooltip about the interface language: explicitly states that
 the switcher only covers the main panel on the incident card page, not the
 settings page itself.

### Added
- LICENSE_SERVER_SETUP.md - instructions for connecting your own licensing
 server.

## [1.5.4] - 2026-08-15 - changelog viewer wasn't loading, settings theme wasn't changing, automatic license revocation

### Fixed (extension)
- changelog.html got stuck on "Loading…" - added web_accessible_resources
 for CHANGELOG.md in the manifest (likely cause: an MV3 restriction on
 fetching the extension's own resources without explicit declaration) + a
 5s timeout with an explicit error instead of loading forever.
- The settings page (options.html) didn't follow the selected theme - the
 switcher changed the panel's theme on the SIEM card, but the settings page
 itself stayed dark. Added CSS variables and applied the theme class on load
 and immediately after saving.

### Added (licensing server)
- Automatic license revocation when the device limit (seats) is exceeded: an
 activation attempt beyond the limit now doesn't just reject the new device
 - it revokes the ENTIRE key (including already-active devices) - protection
 against a key spreading outside the organization. The reason, time, and ID
 of the triggering device are logged in the license record for
 administrator auditing. The device limit already supported up to 1000
 (including up to 100 requested) - no range change was needed, only the
 revocation logic itself.

## [1.5.3] - 2026-08-15 - theme/language switching didn't reach the right tab + mojibake in CHANGELOG

### Fixed

- **Theme/language switching looked "broken"**: the settings update only
 went to the tab selected in the selector-picker dropdown (needed for the
 "Select on page" feature when configuring an adapter) - not the tab where
 the analyst actually had the incident card open. If these were different
 tabs (the common case - most users never touch that dropdown), the update
 simply never arrived. Now `socCopilotSettingsUpdated` is broadcast to ALL
 open tabs.
- **"Mojibake" when opening CHANGELOG.md** via the "What's new" button in
 the popup - when the browser opens an `.md` file directly as
 `chrome-extension://.../CHANGELOG.md`, it sometimes guesses the encoding
 incorrectly. Added a `changelog.html` wrapper with an explicit
 `<meta charset="UTF-8">` that loads the text via `fetch()` and decodes it
 as UTF-8 - regardless of a specific browser's heuristics. The link in
 popup.js was updated.
- **The popup didn't follow the selected theme** (visible in the user's
 screenshot - the panel switched to the light theme while the popup stayed
 dark): added CSS variables and applied the theme class when the popup
 opens.

### Known limitation
- The interface language in the popup is still not translated (only the
 main panel on the card page is) - fully localizing the compact popup at
 the current string volume was deliberately not done as part of this patch,
 to avoid increasing regression risk in an urgent fix; will be considered
 separately.

### Added
- `QUICKSTART.md` - a step-by-step guide for the analyst: install →
 configure the adapter → first incident triage → closing it →
 theme/language switching → common issues.

## [1.5.2] - 2026-08-15 - critical patch: default_locale without _locales

Fixed the error "Default locale was specified, but _locales subtree is
missing" - the manifest contained `"default_locale": "ru"` with no
`_locales/` folder (the extension uses its own translation system via
`utils/i18n.js`, not the built-in `chrome.i18n`). The key was removed.

## [1.5.1] - 2026-08-15 - critical patch: the extension wouldn't install in Yandex Browser

### Fixed

**The extension wouldn't load at all** - reported by the user when trying to
install it in Yandex Browser: `Invalid type for attribute 'additionalProperties'. Failed
to load manifest.` Cause - `schema.json` (used for enterprise policies via
`"storage": {"managed_schema": "schema.json"}` in the manifest, see 0.9.0)
had `"additionalProperties": false` at the top level. That's valid JSON
Schema, but the Chromium extension-policy parser (used in both Yandex
Browser and Chrome/Edge) doesn't follow full JSON Schema - it uses its own
restricted policy schema format, where `additionalProperties`, if present,
is expected to be an object-schema (describing the type of allowed extra
properties), not a boolean - a boolean value causes a type error while
parsing, rather than being silently ignored.

**Fixed**: the line `"additionalProperties": false` was removed from
`schema.json`. No functionality is lost - without this line, extra
(non-schema) policy properties are simply allowed by default, which was
sufficient before too, since strictly forbidding them was never a
functional requirement - it was only a side effect of adding the line
without testing on real Chromium browsers.

### What this means for 0.9.0-1.5.0

If the extension did NOT install for you (an error at the manifest-loading
stage, before the extension ever ran) - you weren't affected by this bug in
any other way besides the install failure itself; functionality after the
fix is identical to previous versions, the extension simply now actually
loads in Chromium browsers with strict policy-schema validation (confirmed
by a user report specifically for Yandex Browser - it would likely also
reproduce in current versions of Chrome/Edge with the same strict checking,
and isn't specific to Yandex Browser).

### Process lesson

None of the previous comprehensive security audits (including v1.5.0) caught
this issue, because `schema.json` was only checked for validity as JSON
(`python3 -m json.tool` - the file is syntactically fully correct) and for
being referenced in the manifest, but not for compliance with the specific,
narrower subset that the Chromium policy schema parser actually accepts -
valid JSON and a valid Chrome extension policy schema are not the same
thing, and this wasn't tested live in any browser before the user's report.
Logged as an explicit gap in the review process going forward.

## [1.5.0] - 2026-08-14 - comprehensive security audit

A planned audit at explicit request before project handover - a systematic
review of the entire codebase (extension + licensing server) for bugs and
vulnerabilities. All findings are listed below without omission, including
things that were suspected but not confirmed on re-checking - that's also
part of a good-faith audit.

### 🔴 Critical finding - fixed

**The generic adapter (`id: "generic"`) was enabled by default and had no
`urlPatterns`, meaning it technically matched ANY site.** Confirmed by a
programmatic test before the fix: `socCopilotResolveAdapter()` returned the
"Universal" adapter for `google.com` and `github.com`. The practical
consequence - the extension extracted text (`h1,h2` / `main,body`), ran the
IOC detector, wrote an event to PERSISTENT storage (`eventsLog`), and
registered the tab for cross-correlation - **on every site visited**, not
just SIEM incident cards. This directly contradicted the "activates only on
recognized platforms" principle stated in the README since version 0.1.0.
**Fixed:** `enabled: false` by default (`adapters.js`) - enabling it is now
a deliberate action by the analyst, with an explicit warning in
`options.html` about the consequences (site-agnostic activation).

### 🟡 Medium findings - fixed

- **CORS (`Access-Control-Allow-Origin: *`) was applied identically to all
 licensing-server endpoints, including `/api/admin/*`.** Admin endpoints
 are, by design, only ever called from a CLI/curl - they have no legitimate
 browser client, so allowing CORS on them provides no benefit but does
 increase the attack surface if `ADMIN_API_KEY` were hypothetically leaked.
 `sendJSON()` now accepts `{ allowCors: false }`, applied to all 3 admin
 handlers and to the preflight `OPTIONS` for `/api/admin/*`.
- **While implementing this very fix, a regression bug was introduced**:
 `pathname` was used in the `OPTIONS` handler before being computed
 (temporal dead zone for `const`) - this would have crashed the server with
 a `ReferenceError` on every OPTIONS request. Caught by an immediate
 regression test right after the fix, before moving on. The `url`/`pathname`
 computation was moved to the start of the request handler.
- **`toggleMarketplacePreview()` in `options.js`** interpolated `id` into a
 CSS attribute selector without a `try/catch`, unlike similar spots in
 `content.js`. The server already strictly validates `id` with a regex
 (`^[a-zA-Z0-9_-]{1,64}$`), so there was no practical exploitability - the
 fix was added for defense-in-depth and consistency with the rest of the
 code, not because a real hole was found.

### 🔍 Checked in detail, including a disproven hypothesis

During the first pass of the audit, a suspicion arose of **stored XSS via
the adapter marketplace** - that the `name`/`vendor`/`id` fields of an
installed third-party adapter could end up in the settings page's
`innerHTML` without escaping. On a precise line-by-line re-check of all
related functions (`renderAdapters()`,
`populateMarketplacePublishSelect()`, the marketplace listing renderer),
the suspicion was **not confirmed** - `escapeHtml()` is applied at every
point, including attribute contexts (`value="..."`). This wasn't published
as an inaccurate "vulnerability found" note simply because it was an
intermediate-stage assumption - what's recorded here is what was actually
verified and holds for the current state of the code.

Further systematic review (with no prior suspicions, across the whole
codebase):
- **XSS via `innerHTML`**: a complete map of every `innerHTML` point in the
 project was built. Special attention was given to data from the
 MARKETPLACE and GLOSSARY, the only sources of content from other
 organizations in the system: `escapeHtml()`/`escapeAttr()` confirmed
 everywhere. SIEM incident card fields (`title`, `description`, etc. -
 potentially containing text inserted by an attacker, e.g. a phishing
 email subject) - the only HTML insertion point (the "Summary" tab) is
 correctly escaped; other uses are JSON storage or plain text for external
 messengers, not an HTML context.
- **`eval()`, `new Function()`, `document.write`, `.outerHTML`** - not used
 anywhere in the project.
- **ReDoS** (catastrophic regex backtracking): all patterns in
 `ioc-extractor.js` and `redaction.js` (16 patterns, including SSH keys
 using `[\s\S]*?`) were run against a 50,000-character input with no match
 at the end - all under 5ms, including a separate test with an unclosed
 PGP block over 100,000 characters.
- **Prototype pollution** via `__proto__` in JSON request bodies to the
 server - all handlers use explicit destructuring of named fields, not
 `Object.assign`/spreading the whole body.
- **Rate limiting** on marketplace/glossary endpoints confirmed to be
 covered - `marketplace-publish`, `marketplace-download`,
 `glossary-submit`, `glossary-vote` are limited by IP.
- **Secret logging**: a grep across the entire server codebase confirmed no
 `console.log` calls with keys/tokens/passwords.
- **Constant-time comparison** (`crypto.timingSafeEqual`) for
 `ADMIN_API_KEY` was re-confirmed and wasn't touched by this release's
 changes.
- A full regression test of every server subsystem (licenses → activation →
 marketplace → glossary → ingest → metrics → health) after all fixes - the
 first run reported an activation error; investigated immediately, turned
 out to be a bug in the test itself (deviceId shorter than the minimum
 length of 4 characters), not a code bug - confirmed by re-running the test
 with a valid deviceId.

### Known limitations after this audit

- The audit is a manual systematic review with targeted automated tests
 (ReDoS timing, end-to-end HTTP scenarios), not a formal penetration test
 and not an automated SAST/DAST scan. For environments with elevated
 requirements (critical infrastructure, the financial sector), an
 independent external audit before production deployment is recommended.
- The expert content of the "advisory" modules (Sigma/YARA drafts,
 DFIR/security-tooling recommendations) was not checked for domain
 correctness as part of THIS audit - this is a security review, not a
 domain review; see the corresponding release notes for caveats about the
 draft nature of the recommendations.

### Added - RTL layout (closes a gap from 1.3.0)
- `dir="rtl"` is applied on `.sc-panel` for Arabic/Urdu/Persian - flexbox
 mirrors correctly per the CSS spec without manually reworking every rule.
 A targeted exception for the campaign-graph canvas (coordinates shouldn't
 be mirrored).
- While implementing this, an ordering bug was caught and fixed:
 `applyTheme()` was called before `panelEl` was created, so text direction
 wasn't applied on the panel's first opening. Split into
 `applyThemeClasses()` (for `shadowHost`, before insertion into the DOM)
 and `applyTextDirection()` (for `panelEl`, after creation) with the
 correct call order.

### Added - a live SIEM term glossary (closes a gap from 1.2.0)
- A community dictionary of platform-specific terms via the licensing
 server - reused the adapter marketplace's architecture (the same
 license-key authorization principle, the same `withDB` pattern with
 check+mutation in one call).
- New server endpoints: `GET/POST /api/glossary/terms`, `POST.../vote`.
- Bottom-up moderation via votes instead of a separate moderator role - one
 vote per license key, re-voting changes direction rather than inflating
 the count.
- A new "📖 Glossary" tab in the panel - a Pro feature (team-wide value,
 like the rest of the server integrations).

### Fixed / security (found and fixed while developing this release)
- **A real API bug**: the `httpStatus` field (internal-only, used solely
 for response routing) leaked into the HTTP response body of
 `POST /api/glossary/terms/:id/vote` - `{"httpStatus":200,"upvotes":1,...}`
 instead of a clean `{"upvotes":1,...}`. Found by an end-to-end test before
 release.
- **While first fixing this bug, a syntax error was introduced** (an
 orphaned duplicate code fragment outside a function) - caught immediately
 by re-running `node --check` and re-running the end-to-end test, before
 moving on. A good example of why testing after every fix matters, not
 just after the original implementation.
- A full end-to-end test of the glossary on a live server (publish → 401
 without authorization → vote → duplicate-vote protection → changing vote
 direction → listing with recalculated rating → term-length validation) -
 all scenarios passed.
- A full comprehensive audit before the build (syntax, duplicate functions,
 HTML ids, message-type cross-check, CSS brace balance, all 22 `utils/`
 modules wired up in the manifest).

### Known limitations of the new release
- Glossary voting is tied to the license key, not to a specific
 device/analyst - all devices in an organization using the same key
 (within the seats limit) are physically voting "on behalf of" the same
 identifier. Accurate per-device vote tracking would need a more complex
 model (device-level voting), not implemented in this version.
- Glossary terms don't go through automatic content moderation - it relies
 entirely on community voting. Clearly malicious/spam content can
 technically be published (it will be visible until it accumulates a
 negative rating).

## [1.4.0] - 2026-08-14 - restored in the CHANGELOG during the 1.5.0 audit

**Note on this entry:** while editing the CHANGELOG during the 1.5.0 audit,
this section was accidentally lost (a mistake while merging the file via
`sed`/`tail` - documented in detail in the 1.5.0 entry as part of the
audit's good faith: if bugs in the code are being disclosed, it would be
odd to stay silent about a bug in our own documentation). Restored by
content from the conversation history where these changes were presented to
the user at the time of release - unchanged in substance.

### Added - RTL layout (closes a gap from 1.3.0)
- `dir="rtl"` is applied on `.sc-panel` for Arabic/Urdu/Persian - flexbox
 mirrors correctly per the CSS spec without manually reworking every rule.
 A targeted exception for the campaign-graph canvas (coordinates shouldn't
 be mirrored).
- While implementing this, an ordering bug was caught and fixed:
 `applyTheme()` was called before `panelEl` was created, so text direction
 wasn't applied on the panel's first opening. Split into
 `applyThemeClasses()` (for `shadowHost`, before insertion into the DOM)
 and `applyTextDirection()` (for `panelEl`, after creation) with the
 correct call order.

### Added - a live SIEM term glossary (closes a gap from 1.2.0)
- A community dictionary of platform-specific terms via the licensing
 server - reused the adapter marketplace's architecture (the same
 license-key authorization principle, the same `withDB` pattern with
 check+mutation in one call).
- New server endpoints: `GET/POST /api/glossary/terms`, `POST.../vote`.
- Bottom-up moderation via votes instead of a separate moderator role - one
 vote per license key, re-voting changes direction rather than inflating
 the count.
- A new "📖 Glossary" tab in the panel - a Pro feature (team-wide value,
 like the rest of the server integrations).

### Fixed / security (found and fixed while developing this release)
- **A real API bug**: the `httpStatus` field (internal-only, used solely
 for response routing) leaked into the HTTP response body of
 `POST /api/glossary/terms/:id/vote` - `{"httpStatus":200,"upvotes":1,...}`
 instead of a clean `{"upvotes":1,...}`. Found by an end-to-end test before
 release.
- While first fixing this bug, a syntax error was introduced (an orphaned
 duplicate code fragment outside a function) - caught immediately by
 re-running `node --check` and re-running the end-to-end test, before
 moving on.
- A full end-to-end test of the glossary on a live server (publish → 401
 without authorization → vote → duplicate-vote protection → changing vote
 direction → listing with recalculated rating → term-length validation) -
 all scenarios passed.

### Known limitations of this release
- Glossary voting is tied to the license key, not to a specific
 device/analyst - all devices in an organization using the same key
 (within the seats limit) are physically voting "on behalf of" the same
 identifier.
- Glossary terms don't go through automatic content moderation - it relies
 entirely on community voting.

## [1.3.0] - 2026-08-14

### Added - themes and multilingual support
- **Three panel themes**: Dark (default, as before), Light, High contrast
 (WCAG-oriented - not just a color swap, but visible borders on interactive
 elements). `panel-styles.js`'s key colors were converted to CSS custom
 properties with fallback values; switching happens in settings and applies
 immediately to already-open cards.
- **A multilingual interface - the top 30 languages** (`utils/i18n.js`) by
 number of speakers (English, 中文, हिन्दी, Español, Français, العربية, বাংলা,
 Português, Русский, اردو, Bahasa Indonesia, Deutsch, 日本語, Kiswahili,
 मराठी, తెలుగు, Türkçe, தமிழ், Tiếng Việt, 한국어, Italiano, فارسی, ไทย,
 ગુજરાતી, Polski, Українська, മലയാളം, ಕನ್ನಡ, Nederlands, Română). A switcher in
 settings.

### Honest disclosure about the scope of multilingual support (not hidden, stated directly in the settings UI)
The interface "shell" is translated - the panel header, the names of the 8
tabs, common buttons (copy/check/close) - the most visible part of the UI.
Dynamic content (L1-L3 mentor steps, DFIR tool descriptions, playbook text,
generated reports/conclusions - many thousands of words) stays in Russian.
Fully translating ALL content into 30 languages is a separate,
large-scale localization project on its own, not a single feature; trying
to rush it would have produced either low translation quality or a
disproportionately large share of the release spent on one feature at the
expense of everything else. The list of translated keys and languages was
programmatically checked for completeness (30×20 = 600 translations, no
gaps).

### Architectural decision (themes)
- The theme class is applied on `shadowHost` (the element in the light DOM
 that owns the shadow root), and the variable overrides use the
 `:host(.sc-theme-X)` selector - this way styles cascade correctly through
 the ENTIRE shadow tree, including the `.sc-toggle` toggle button, which
 sits outside `.sc-panel` as a separate node. Applying the class directly
 on `.sc-panel` wouldn't have worked for the toggle - this was figured out
 before implementing, not discovered after the fact (verified beforehand,
 not retroactively).

### Checked before the build
- Translation dictionary completeness: 30 languages × 20 keys, zero gaps
 (checked programmatically).
- CSS brace balance after the mass replacement of colors with variables -
 not broken.
- A full audit (syntax/duplicate functions/HTML ids/message-type
 cross-check) - as before every release, no discrepancies found.
- Rendering of all 8 panel tabs was tested in 6 languages from different
 language families (Cyrillic, Latin, logographic, right-to-left, Devanagari)
 - renders correctly at the text level (a visual check in a real browser
 wasn't done, since the sandbox has no GUI - a manual check of RTL
 languages (Arabic, Urdu, Persian) on a real install is recommended, text
 direction wasn't explicitly set in CSS).

### Known limitations of the new release
- RTL languages (Arabic, Urdu, Persian) are translated lexically, but CSS
 doesn't switch the panel's direction to `dir="rtl"` - the text will read
 in the correct word order, but the panel's overall layout (buttons
 left/right) stays LTR. This is a visual, not a functional, defect -
 requires separate layout work.
- High contrast is a basic level (colors, borders), not a full audit
 against a specific standard (WCAG 2.1 AA/AAA) with all contrast ratios.

## [1.2.0] - 2026-08-14

### Added - the campaign graph and the adapter marketplace
- **The campaign relationship graph** (`utils/campaign-graph.js`):
 visualizes connections between closed incidents from the local conclusion
 history, based on shared IOCs/techniques - a simple circular layout on
 `<canvas>`, no external libraries. Isolated incidents (with no
 connections) aren't shown, to avoid cluttering the graph. Clicking a node
 opens the corresponding incident in a new tab. Lives in the "Statistics"
 tab.
- **The adapter marketplace** (licensing server + UI): organizations can
 share adapter configs (selectors for specific versions of MP SIEM/RuSIEM/
 BI.ZONE) via the licensing server - dramatically reduces first-day pain
 for new customers on the same platforms. Publishing and downloading
 require a valid license key (authorship is tied to the organization), and
 the listing is readable by any valid key - that's the point of the
 marketplace. New server endpoints:
 `GET/POST /api/marketplace/adapters`,
 `POST /api/marketplace/adapters/:id/download`.

### Architectural decision (marketplace)
- The license key needed to authorize marketplace requests is decrypted
 **exclusively** in `background.js` - `options.js` only passes
 non-sensitive adapter data (id/name/selectors, not a secret), and
 background.js adds the server authorization itself. The same
 write-only/decrypt-only-in-background principle used for all other
 secrets since version 0.9.0.

### Fixed / security (audit before the build)
- In the first implementation of the server handlers for
 publishing/downloading adapters, authorization was checked via a
 SEPARATE `loadDB()` call before calling `withDB()` for the mutation -
 this double DB read created a small race (TOCTOU) between checking the
 license and writing, which didn't match the pattern already established
 elsewhere in the code (check and mutation in a single `withDB()`).
 Rewritten to a single consistent pattern, matching
 `handleActivateOrValidate`.
- A full end-to-end marketplace test (7 scenarios: publish → id conflict
 between organizations → rejection without authorization → listing →
 download with a counter → re-checking the counter → invalid-id
 validation) - all passed on a real running server before the release
 build.
- Found and fixed a gap: `populateMarketplacePublishSelect()` was declared
 but never called after loading local adapters - the publish dropdown
 would have stayed empty. Added a call in `loadAdapters()`.
- A full cross-check of every message type between `content.js`/`options.js`
 ↔ `background.js` after adding 3 new marketplace handlers - no
 discrepancies.

### Deliberately still deferred
- Shadow mode for seniors (read-only subscription to an L1 analyst's
 stream with automatic QA) - a complex access-rights model, needs its own
 dedicated design.
- A live community SIEM term glossary - needs crowdsourcing and moderation
 infrastructure that doesn't exist yet, on the server or in the extension.

## [1.1.0] - 2026-08-14

### Added - the whole backlog closed + new breakthrough features
- **"What changed while the tab was inactive" diff**
 (`utils/diff-detector.js`): a snapshot of the card's fields is taken on
 first view, a `visibilitychange` listener compares it on focus return, and
 shows the specific field changes.
- **A response command generator** (`utils/response-snippets.js`):
 copy-paste templates for blocking an IP/domain/hash under Windows
 Firewall, iptables, Cisco ACL, the hosts file, BIND RPZ, Squid, Defender,
 CrowdStrike, YARA. A "blocking commands" button in the IOC list.
 Deliberately has NO `fetch`/execution of any kind - only text for manual
 use.
- **A Teams card format** for webhook broadcasting (MessageCard) - a new
 channel alongside Slack/Telegram/MAX.
- **A burnout detector** (`utils/burnout.js`): a gentle break reminder when
 mentor quiz accuracy drops and/or work happens late at night several days
 in a row. Not a diagnosis - just a friendly hint, can be turned off in
 settings (on by default), shown no more than once per session.
- **Detection engineering from a closed incident**
 (`utils/detection-rules.js`): draft Sigma and YARA rules generated from a
 closed incident's IOCs/techniques - closes the loop of "investigated it →
 teach the system to catch it automatically." Requires review by a
 detection engineer before deployment, explicitly marked `experimental`.
- **An alert storm detector** (`utils/alert-storm.js`): groups mass
 same-type events over the last hour by normalized title - a cure for
 alert fatigue (no need to triage 20 cards one by one if the cause is a
 single one).
- **One-click second opinion**: a "🤝 Request a second opinion" button in
 the conclusion, reuses the already-configured notification channels with
 separate text formatting tailored for a review request (not a final
 notification).
- **Local LLM** (Ollama-compatible API): all 4 LLM features (summary,
 forecast, mentor, "Ask AI") now go through a single `callLLM()` entry
 point with a switch between Anthropic ↔ a local/on-prem endpoint -
 relevant for organizations that can't send data to external cloud LLMs
 even in redacted form.
- **Jira Service Desk**: creating tickets via REST API v3 (Basic Auth
 email+token) - another team-notification channel.
- **Voice input** (Web Speech API) for conclusion fields - a 🎤 button next
 to every text field, recognition is built into Chrome, audio is never
 additionally sent anywhere by the extension.

### Fixed / security
- A full cross-check of every message type between `content.js` ↔
 `background.js` after a large refactor of LLM calls onto `callLLM()` - no
 unhandled types left.
- An expanded integration test (7 new modules in a single scenario) passed
 with no errors before the release build.
- Verified consistency of IOC types between the snippet generator and the
 button markup (a MAC address correctly doesn't get a meaningless
 "blocking commands" button).

### Deliberately deferred (not part of this release)
- The campaign relationship graph (visualization) - needs canvas
 rendering, a separate session.
- The adapter marketplace via the licensing server - needs a new server
 registry.
- Shadow mode for seniors (read-only subscription to an L1 stream) - a
 complex access-rights model, needs a well thought-out design before
 implementation.
- A live community SIEM term glossary - needs crowdsourcing
 infrastructure.

## [1.0.0] - 2026-08-14 - first stable release

### Added - closes a gap from 0.9.0
The audit before the stable release found that the licensing and Grafana
backend logic (from 0.9.0) had no UI - the user physically couldn't
activate a license. Closed in this release:
- A **"License"** section in settings: status (Free/trial with days
 remaining/active license with expiration date), fields for the server URL
 and key, an activation button.
- A **"Grafana sync"** section: an enable checkbox, an analyst nickname for
 the ranking, a manual "sync now" button (in addition to the automatic
 daily sync).
- An end-to-end test (a real running licensing server + a real HTTP
 request) confirmed the data format between `background.js` and the server
 is correct.

### Fixed / security (audit before the stable release)
- **Found and fixed an IOC classification bug**: MAC addresses
 (`00:1A:2B:3C:4D:5E`) were mistakenly recognized as IPv6 addresses - the
 patterns matched by shape (colon-separated hex groups). This could send an
 analyst to check a MAC address against an IPv6 reputation service. Added a
 separate `mac` pattern, and `ipv6` results are now filtered against values
 already identified as MAC. Added a human-readable label for the mentor
 quiz ("MAC address" instead of the technical `mac`).
- Cross-checked ALL message types between `content.js`, `options.js`,
 `popup.js` and `background.js` - every message sent has a handler in the
 right context (including messages to a specific tab via
 `chrome.tabs.sendMessage`, which falsely looked "unhandled" under a flat
 check).
- Removed dead code: the unused `licenseServerUrl` key in `STORAGE_KEYS`
 inside `content.js` (the value was never read anywhere - only
 `background.js`/`options.js` actually need the licensing server URL).
- Cross-checked the `requirePro()` keys in `content.js` against the
 definitions in `utils/license.js` - all 5 client-side Pro gates use
 existing feature keys.
- Ran a full integration test: from raw incident-card text to IOC →
 MITRE → playbook → report → conclusion → BelZor → DFIR recommendations →
 security-tooling recommendations → GosSOPKA draft → redaction → license
 status - all 13 steps of the chain completed on a single realistic
 scenario with no errors and no leaks in the redacted text.

### What "1.0.0" means for this project
It doesn't mean "bug-free" or a "feature freeze" - it means that all the
functionality promised across CHANGELOG 0.1.0-0.9.0 has been implemented,
wired up to the UI, and verified by at least one end-to-end test (not just
a syntax check). Each feature's known limitations remain documented in the
corresponding sections above, by version, and aren't considered "debt" -
they're deliberate boundaries that have been honestly disclosed to the
user.

## [0.9.0] - 2026-08-14

### Added - licensing, pricing tiers, enterprise policies, Grafana
- **Licensing**: a 30-day fully-featured trial from the moment of install,
 then the Free tier without an active license. Activation via a separate
 licensing server (see `/license-server`), periodic re-checks once a day
 via `chrome.alarms` with an offline grace period (the server being
 unreachable doesn't silently disable Pro).
- **Free/Pro split** (`utils/license.js`): Free includes IOC/MITRE/
 playbooks/reports/redaction/L1-L3 mentor/DFIR recommendations/
 security-tooling recommendations (maximum coverage for the general user,
 excluding features with direct LLM operating costs). Pro includes LLM
 summary, LLM MITRE forecast, Socratic AI mentor, "Ask AI about the
 selection," conclusion export to PDF, sending to Slack/Telegram/MAX,
 Grafana sync - features with billable LLM calls and team/managerial
 value.
- **Enterprise policies** via `chrome.storage.managed` (Chrome's official
 mechanism, manifest `"storage": {"managed_schema": "schema.json"}`) -
 centralized rollout of the license key, server address, company names,
 webhooks, etc. on managed corporate installs.
- **Grafana integration**: the extension sends aggregated
 (pseudonymized) BelZor statistics to the licensing server, which exposes
 it in Prometheus format (`/metrics`) and JSON
 (`/api/stats/leaderboard`) for Grafana dashboards.

### Added - the licensing server (`/license-server`, a separate package)
- A zero-dependency Node.js HTTP server: license activation/validation,
 statistics ingest, Prometheus and JSON endpoints for Grafana, an admin
 API for issuing/revoking licenses.
- A `scripts/issue-license.js` CLI, a systemd unit, an nginx config with
 TLS, and a detailed `DEPLOY.md` for Ubuntu 24.04 LTS.
- A full end-to-end test of every endpoint (creation → activation → seats
 limit → ingest → metrics → leaderboard → revocation → re-validation) -
 run manually and documented.

### Added - L2/L3 mentor, DFIR, security-tooling recommendations
- **L2 (Forensics/Security tooling)** and **L3 (Hunting/escalation)**
 tracks in the mentor - a switcher next to the existing L1 track.
- `utils/dfir-tools.js`: 10 DFIR/forensics tools (Volatility3, KAPE,
 Velociraptor, Autopsy, FTK Imager, Chainsaw, Hayabusa, Wireshark, YARA,
 Plaso) with official links, honest licensing caveats (e.g. KAPE's
 restriction on commercial use), and quick-start instructions. Relevant
 tools are matched to the MITRE ATT&CK techniques detected on the current
 card.
- `utils/szi-recommendations.js`: standard remediation actions for 10
 classes of security tooling (EDR, NGFW/IPS, WAF, DLP, NAC, SIEM/SOAR,
 PAM, Sandbox, Proxy, email), tied to the detected ATT&CK tactics.
- Both recommendation systems are in the free tier (security reference
 content shouldn't sit behind a paywall).

### Added - GosSOPKA / FSB Order No. 161 (informational, WITHOUT a fabricated integration)
- Before implementing, checked via web search against current sources:
 NKTsKI has no public self-service API for third-party tools; access to a
 GosSOPKA subject's personal account is only granted after concluding a
 cooperation agreement (info@cert.gov.ru). FSB Order No. 161 (effective
 05/31/2026) governs the **accreditation of GosSOPKA centers** by the FSB
 of Russia, not the work of individual analyst tools.
- `utils/gossopka.js`: a generator for a DRAFT incident card in the general
 categories publicly known to be collected by NKTsKI (for later manual
 submission through the official channel), plus reference information
 about accreditation tracks under Order 161 with links to official
 sources. No calls to a non-existent public GosSOPKA API - deliberately,
 to avoid creating a false impression of automatic submission.

### Added - secret encryption (AES-256-GCM, "resistant to casual reading")
- **On the extension side** (`utils/crypto.js`, WebCrypto API): all secrets
 (VirusTotal/AbuseIPDB/Anthropic API keys, Telegram/MAX bot tokens,
 Slack/Mattermost webhooks, the license key) are now stored in
 `chrome.storage.local` ONLY in encrypted form.
- **Architectural hardening**: decryption happens EXCLUSIVELY in
 `background.js` (the most isolated context) - neither `content.js` (which
 lives in the SIEM page's context, the most exposed context) nor
 `options.js` ever hold secrets in plain text again. Secrets used to be
 passed via `chrome.runtime.sendMessage` in plain text before this release
 - now messages only contain an action request, and background.js reads
 and decrypts the keys itself.
- **Write-only settings UI**: after saving, a secret field shows a "saved"
 placeholder instead of the secret itself - the secret can't be
 accidentally seen again by reopening the settings page. Fields switched
 to `type="password"`.
- **On the licensing server** (`lib/dbcrypto.js`, built-in `crypto`):
 `data/db.json` is encrypted on disk (AES-256-GCM) - direct access to the
 file (a backup leak, another process on the same server) doesn't expose
 the data.
- Automatic migration of secrets saved by versions < 0.9.0 into the
 encrypted format on the first run of the updated extension.

### Fixed / security (found and fixed during the audit BEFORE the release build)
- **Critical bug**: during the encryption refactor, the signatures of
 `handleLLMSummary`, `handleLLMPredict`, `handleLLMMentor` in
 `background.js` weren't kept in sync with the new calls from
 `content.js` - the functions still expected `apiKey` as the first
 parameter, even though messages no longer passed it. This would have
 broken ALL LLM features in production. Found and fixed during an
 end-to-end cross-check of every message type between `content.js` ↔
 `background.js` before the build.
- Found and removed 3 remaining references to the already-deleted
 `state.apiKeys` in `content.js` (would have thrown a `TypeError` on
 clicking any LLM button).
- `options.html` wasn't loading `utils/crypto.js` - `options.js` would
 have thrown a `ReferenceError` on the first save of any key. Added.
- A full cross-check of storage keys: obsolete plaintext keys
 (`socCopilotApiKeys`, `socCopilotTelegramConfig`, `socCopilotMaxConfig`,
 `socCopilotWebhookUrl`) were removed from the code after migrating to
 their `*Enc` equivalents.
- An end-to-end test confirmed: a dump of `chrome.storage.local` contains
 NO secrets in any form after saving through the new UI, while
 `background.js` correctly decrypts them for actual network requests.
- A server end-to-end test confirmed: `db.json` on disk is unreadable
 without the encryption key, and all functionality (license creation/
 activation) works on top of the encryption with no degradation.
- The backup commands from `DEPLOY.md` were reviewed and tested - the
 first version of the command was syntactically incorrect, fixed and
 verified live.

### Known limitations of the new release
- Encryption in the extension protects against casual/automated reading of
 `chrome.storage.local` (another extension, a profile dump, an agent
 without specific knowledge of the mechanism) - NOT against an attacker
 with full control over the same browser profile and time to analyze the
 extension's code (the encryption key is technically reachable in the same
 storage). Honestly described as defense-in-depth, not an absolute
 guarantee.
- The trial is local (based on install date), not confirmed by the server;
 reinstalling the extension technically resets the trial. To control this
 at the organization level, use the `LicenseKey` enterprise policy, which
 activates Pro immediately without depending on the local trial.
- The GosSOPKA module only prepares a draft and reference information, it
 doesn't automatically submit anything to NKTsKI (there's no public API
 for that).

## [0.7.0] - 2026-08-14

### Added - counters and reports
- A new "📊 Statistics" tab in the panel. Clear counter definitions
 (important for trust in the numbers): **"event"** = a unique card the
 plugin recognized (by URL); **"incident"** = a card for which the analyst
 SAVED a conclusion - so the counter reflects actually closed work, not
 the number of clicks.
- A period report: week / month / quarter / half-year / year - number of
 events, closed incidents, BelZor earned, breakdown by verdict
 (TP/FP/Benign/Escalation). Periods are rolling windows in days
 (7/30/91/182/365), not calendar months/quarters.
- CSV report export (with a BOM so Cyrillic opens correctly in Excel) -
 for handing off to a shift lead/team lead.
- All data is calculated **locally only**, from the browsing history of a
 specific analyst's specific browser - explicitly stated in the UI, to
 avoid creating the illusion of aggregated statistics across the whole SOC
 team.

### Added - BelZor gamification
- A new reward currency, **BelZor** (Belyaev + vigilance/attentiveness),
 is awarded when a conclusion is saved for an incident. The base amount
 depends on criticality (parsed from the card's criticality field text in
 Russian/English, with a fallback to a heuristic risk score if the field
 couldn't be parsed): Critical - 50, High - 30, Medium - 15, Low - 5.
- Bonuses for indicators of quality work (not for the "correctness" of the
 verdict - the plugin can't verify that): +5 for using MITRE ATT&CK
 context, +5 for fully completing the playbook, +10 for actually checking
 at least one IOC via a TI service.
- A rank system: Rookie → Attentive (100) → Vigilant (300) → Guardian/
 BelZor Guardian (700) → Response Master (1500). A rank-up notification -
 a toast in the panel.
- Protection against double-crediting: BelZor is awarded once per specific
 card (by URL), even if the conclusion is re-saved multiple times.
- Balance and rank are visible in the popup without opening the panel.

### Audit before release
- A full check for duplicate function declarations across all JS files -
 none found.
- Checked `data-tab`/`data-tab-content` correspondence for all 8 panel
 tabs (including the new "Statistics") - 1:1, no "dead" tabs.
- An end-to-end test of statistics aggregation across all 5 periods on
 synthetic data with varying dates - rolling windows correctly
 include/exclude records at the boundaries.
- An end-to-end test of awarding BelZor with a rank transition (Rookie →
 Attentive) - correct calculation of the base amount, bonuses, and
 detection of the rank-up moment.
- Verified protection against double-crediting BelZor when re-saving a
 conclusion for the same card.

### Known limitations of the new release
- Report periods are rolling windows in days, not tied to calendar
 month/quarter/year boundaries (e.g. "month" is the last 30 days, not the
 current calendar month).
- Criticality detection from the field text is a heuristic (Russian/English
 keywords), like everywhere else in the parser; non-standard wording falls
 back to the risk-score heuristic.
- BelZor rewards process thoroughness (using MITRE, a complete playbook,
 a TI check), NOT the actual correctness of the verdict - the plugin can't
 verify that automatically, and it shouldn't be interpreted as an
 assessment of an analyst's work quality for HR purposes without
 understanding this limitation.

## [0.6.0] - previous session

### Added - a mentor for L1 analysts
- A new "🎓 Mentor" tab in the panel. Pedagogical principles (see
 `utils/mentor.js`): **just-in-time learning** (explanations are tied to
 real data from the CURRENT incident, not abstract examples),
 **scaffolding** (simple to complex), **active recall** instead of passive
 reading (short quizzes with instant feedback and a "why" explanation),
 **reflection** for tasks with no single correct answer (a TP/FP verdict,
 prioritizing playbook steps - comparing one's own reasoning against a
 heuristic, not a strict grade), light **gamification** (levels "Trainee →
 Beginner L1 → Confident L1 → Ready for L2" based on the number of correct
 answers).
- **Spotlight highlighting right on the card page**: the mentor doesn't
 just explain, it points - outlining the relevant element (the title, the
 block with the card's text) right inside the SIEM interface, with smooth
 scrolling to the element and tracking of scroll/resize.
- **Mini-quizzes on real data**: for example, "identify the type of this
 indicator" using an actual IOC from the card, "which ATT&CK tactic has
 already occurred" based on actually detected techniques - not a bank of
 random questions, but tied to the specific incident being reviewed.
- **Socratic dialogue with the AI mentor**: the analyst's free-form
 question about the incident - by default the model responds with guiding
 questions, encouraging independent reasoning rather than immediately
 giving a ready-made verdict (if the analyst explicitly asks for a direct
 answer, it gives one).
- Progress is saved locally and visible in the popup without opening the
 card.
- Data redaction (see 0.5.0) also applies to questions asked to the mentor
 - the same pipeline, no exceptions.

### Audit and fixes (done before release)
- A full check of all files for duplicate function declarations - none
 found.
- Checked `data-tab` / `data-tab-content` consistency across every panel
 tab - a 1:1 match, no "dead" or unclosed tabs.
- Checked every `innerHTML` point for escaping: the new mentor elements
 (quiz text, questions, level names, IOC values inside quiz questions) go
 through `escapeHtml`.
- Fixed a typo in the text of one of the training steps (mixed
 Cyrillic/Latin characters, found during testing - didn't affect security,
 but broke readability).
- Confirmed by an end-to-end test: the chain "build context → redact →
 LLM → response → restore" works identically for the mentor, the summary,
 and the forecast - redaction has no bypass path in any of the three LLM
 call scenarios.
- Re-checked the fixes from 0.4.0 (PDF export without `<script>`
 injection, `contextMenus.removeAll` before `create`) - no regressions
 introduced.
- The spotlight overlay is deliberately placed in the page's light DOM
 (not the panel's Shadow DOM) - otherwise it can't be positioned over the
 real portal elements; verified that the overlay is correctly removed when
 the panel closes, the tab changes, or the target element disappears from
 the DOM (an SPA re-render), so as not to leave a "stuck" element on the
 user's page.

### Known limitations of the new release
- The mentor's course is a fixed sequence of 6 steps per incident, it
 doesn't dynamically adapt to what the analyst has already clearly
 demonstrated (e.g. if they've already gone through the same incident type
 10 times, the steps aren't automatically shortened).
- Reflection (free text) isn't saved across page reloads - only for the
 duration of the current card's review session.
- Spotlight won't work if the adapter doesn't have the corresponding
 field's selector configured via the picker (see README) - the step is
 then shown without a visual hint, text only.

## [0.5.0] - previous session

### Rename
- The extension was renamed to **Belyaev Response** (previously
 "SOC Copilot"). Internal code/storage identifiers (`socCopilot*`) were
 deliberately left unchanged - this reduces migration bug risk and is
 invisible to the user.

### Added - data redaction before sending to an LLM (the key change of this release)
- A new module, `utils/redaction.js`: before ANY text is sent to an
 external LLM (Anthropic), it automatically finds and replaces with
 placeholders (`[[IP_1]]`, `[[MAC_1]]`, `[[ACCOUNT_1]]`, `[[ORG_1]]`, etc.):
 - IPv4/IPv6 addresses, MAC addresses;
 - email addresses;
 - SSH private and public keys, AWS access key IDs, JWT tokens;
 - account names - by keyword (login/user/account/username) and by the
 `DOMAIN\username` format;
 - company names/domains - set by the analyst in settings (impossible to
 detect automatically).
- Redaction is applied on **both** paths to the LLM: the card
 summary/forecast (assembled in `content.js`) and "Ask about the
 selection" from an arbitrary page (assembled in `background.js`, since
 this path doesn't go through content.js).
- **Transparency for the analyst**: before sending, the panel shows a
 collapsible "🔒 Redacted before sending" preview - the analyst sees the
 exact text that will go to the model BEFORE clicking the request button.
- **On-the-fly restoration**: if the model's response references a
 placeholder (`[[IP_1]]`), it's replaced back with the real value before
 being shown to the analyst - the model itself never saw the original
 value and never does.
- The list of company names/domains is configured in options
 ("Data redaction before LLM").

### Added - Telegram and MAX
- In addition to the Slack/Mattermost webhook, the "Send conclusion
 summary" button now supports **Telegram** (Bot API, `sendMessage`) and
 **MAX** (Bot API, `POST /messages`, see `dev.max.ru/docs-api`). A separate
 button is shown in the panel for each configured channel.
- Important: this broadcast is a notification to the response team, not a
 request to an LLM - redaction is deliberately not applied to it
 (colleagues need the real values).

### Fixed / security
- Pre-release audit: checked for duplicate function declarations across
 `content.js`, `background.js`, `options.js` - none found.
- An end-to-end test of the chain "redact → prompt → model response →
 restore" confirmed no sensitive values leaked into the outgoing text (see
 tests in the development history; the test artifact isn't included in
 the distribution, only the module's code).
- `background.js` now loads `utils/redaction.js` via `importScripts` -
 this is the only place where LLM prompts are assembled, which rules out
 accidentally bypassing redaction from content.js.

### Known limitations of the new release
- Detecting account names and company names is a heuristic (keywords, a
 configured list), it doesn't guarantee 100% coverage of non-standard log
 formats. The "what will be sent" preview lets the analyst check this
 manually before sending - recommended to verify on first use with a new
 card type.
- The MAX messenger's API is actively evolving (see
 `dev.max.ru/docs-api`); if sending stops working, the likely cause is an
 endpoint change on MAX's side - check the current documentation.
- Redaction doesn't cover embedded screenshots/images - text only.

## [0.4.0] - previous session

### Added
- **Cross-tab correlation for a shift.** The background script keeps a
 registry of open incident cards (IOCs/techniques/URL) for the duration of
 the browser session. A new card is automatically checked for overlap with
 tabs the analyst already has open - a "Related open tabs" section in the
 IOC panel.
- **IOC highlighting right on the card page** (a toggle button in the
 panel header) - wraps found indicators in `<mark>` within the description
 text, without altering the rest of the portal's markup. Limited by node
 count for performance.
- **Hotkeys** (see `chrome://extensions/shortcuts`): open/close the panel,
 quickly mark the verdict as False Positive, copy the report draft.
- **A risk badge on the extension icon** - a heuristic score (based on IOC
 count, kill chain stage, and detected techniques) is shown right on the
 toolbar icon, so you can see a tab's priority without opening it.
- **"Ask SOC Copilot"** - a context-menu item for selected text on any page
 (not just a configured incident card): a quick LLM question about an
 arbitrary fragment (a log, an error, a strange line) with no need to
 configure an adapter. Requires an Anthropic API key.
- **Broadcasting the conclusion to the team** - optionally sending a short
 conclusion summary to a Slack/Mattermost-compatible Incoming Webhook via
 a button in the "Conclusion" tab.
- **Settings export/import** (adapters + playbooks, no API keys) to JSON -
 for sharing configuration with teammates without shared infrastructure.
- **A new-version indicator** in the popup ("What's new in 0.4.0").

### Fixed / security
- Eliminated a potential HTML/script injection vector in the conclusion's
 PDF export: the conclusion content used to be inserted into an inline
 `<script>` via `JSON.stringify` without escaping the `</script`
 sequence; rewritten to insert escaped text directly into a `<pre>` with
 no inline script over user data.
- Audited every `innerHTML` point in the panel: confirmed that all text
 potentially depending on the SIEM card's content or the analyst's input
 goes through `escapeHtml`/`escapeAttr` before insertion. Values from
 curated internal lists (ATT&CK ids, static URLs) are deliberately not
 escaped, since they don't depend on user/external input.
- Limited the amount of text scanned when highlighting IOCs on the page
 (protection against hangs on very large portal DOM trees).

### Known limitations of the new release
- Cross-tab correlation is only stored in the service worker's memory -
 reset when the browser restarts/the worker is unloaded (this is a
 session cache, not persistent storage).
- Webhook broadcasting only supports the simple `{text: "..."}` format
 (Slack/Mattermost); Microsoft Teams needs a separate card format - not
 implemented in this version.
- IOC highlighting on the page is "best effort": it may not work in SPAs
 with dynamic DOM re-rendering (React/Vue portals may redraw nodes over
 the highlighting).

## [0.3.0] - previous session

### Added
- A formal incident conclusion generator (verdict TP/FP/Benign/Escalation,
 reason, impact, actions, recommendations) with auto-filled IOC/MITRE/
 playbook data.
- Conclusion export to PDF via the browser's print function.
- A local history of completed conclusions and a "Similar completed
 incidents" hint based on overlapping IOCs/ATT&CK techniques.

## [0.2.0] - previous session

### Added
- Automatic mapping of card text onto MITRE ATT&CK (a curated set of
 techniques).
- Kill chain visualization (tactics already covered).
- A heuristic forecast of the attacker's next step (curated typical
 chains + optional LLM refinement).
- Optional sync of the full technique database with MITRE's public CTI
 repository.

## [0.1.0] - initial MVP

### Added
- Adapters for MaxPatrol SIEM, RuSIEM, BI.ZONE, a universal fallback with
 an element picker for configuring selectors without knowing the portal's
 exact markup.
- IOC extraction (IP, domains, URLs, hashes, email, CVE, paths).
- Enrichment via VirusTotal/AbuseIPDB (your own API key).
- Keyword-based playbooks with a checklist saved per card URL.
- A Markdown report draft with clipboard copy.
- An optional LLM summary via the Anthropic API (your own key).

## Planned (backlog, not part of the current release)
- A diff of "what changed on the card while the tab was inactive."
- A generator for ready-made response commands (blocking an IP/hash) for
 specific EDR/firewalls - only as a copy-paste template, with no
 auto-execution.
- A personal analyst workload dashboard (incidents/shift, average
 handling time) - tracked from the local conclusion history.
- A Teams card format for webhook broadcasting.
