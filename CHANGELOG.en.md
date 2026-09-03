# Changelog

This project follows [Semantic Versioning](https://semver.org/).
This file tracks the client side (the browser extension).

## [1.9.3] — 2026-09-04

### Fixed
- **SIEM test stand.** The side panel did not open on the bundled
  `test-siem.html` stand: the **BR** button never appeared and the popup showed
  "Extension is not active on this page". The stand now loads the panel scripts
  directly, so the panel opens there too.
- Switching scenarios on the stand (Phishing / Ransomware / C2) rescans the card
  and refreshes the panel again.

### Improved
- Client-side hardening: full HTML escaping in the conclusion HTML export, a
  cryptographically strong generator for draft-rule identifiers, and input-length
  limits in the redaction module.

## [1.9.2] — 2026-08-19

### Fixed
- `Cannot read properties of undefined (reading 'onMessage')` in `content.js`
  when scripts were injected twice into already-open tabs after a browser or
  extension restart. Added an "already active" check and idempotent init.

## [1.9.1] — 2026-08-18

### Fixed
- Messages between `content.js` and `background.js` were lost (the
  `reading 'onMessage'` error in an orphaned content script after the extension
  was updated while a page stayed open). `chrome.*` calls are now guarded against
  an invalidated context.

## [1.9.0] — 2026-08-18

### Fixed
- Manifest V3 CSP blocked inline scripts on extension pages, so the bundled test
  stand and the "About" block did not render. Scripts moved to external files.

### Added
- Extension icon variant selection.

## [1.8.0] — 2026-08

### Changed
- Platform adapters (MaxPatrol SIEM / RuSIEM / BI.ZONE) are now off by default —
  the extension no longer activates out of the box on incident-like pages.
- Security Vision, SIEM KUMA, SearchInform, R-Vision, Splunk and ArcSight are
  merged into one "Universal" adapter.

### Added
- A master extension toggle in the popup — instantly removes the panel from all tabs.
- Unobtrusive contextual hints on the panel tabs.
- A bundled `test-siem.html` stand with synthetic incidents.

### Fixed
- Manifest V3 CSP restrictions and a leak in the message handler.
- Panel widened (380 → 440 px); clipped mentor buttons fixed.

## Earlier versions (1.6.x–1.7.x)

Core functionality: IOC extraction, MITRE ATT&CK mapping, playbook selection,
conclusion and report drafts, the L1/L2/L3 AI mentor, per-OS DFIR commands
(Windows/Linux/macOS/AstraLinux), BelZor, three themes, 30 UI languages,
data redaction before LLM calls, and messenger integrations.
