# Privacy Policy - Belyaev Response

**Document version:** 1.0 · **Date:** August 15, 2026 · **Applies to product version:** 1.5.3+

## 1. Who we are

Belyaev Response is a browser extension for SOC analysts, developed
based on the concept of Dmitry Belyaev (vCISO, independent expert,
belyaev.expert). This policy describes what data the extension and its
companion licensing server process.

## 2. General principle

The extension is designed around a **data locality** principle: by
default, all processing happens in the analyst's browser, and nothing is
sent to the developer's servers, because the developer only operates a
licensing server.

## 3. What data is processed, and where

### 3.1. Data that NEVER leaves the browser
- Indicators of compromise (IOCs) extracted from the incident card, and
 the title/description text - processed locally to display in the
 panel, for MITRE mapping, and for generating draft reports and
 conclusions.
- API keys (VirusTotal, AbuseIPDB, Anthropic), bot tokens (Telegram,
 MAX), webhook URLs - stored in `chrome.storage.local` in encrypted form
 (AES-256-GCM), decrypted only in the extension's background context
 (`background.js`) at the moment of an actual network request.
- Event/incident history, mentor progress, BelZor balance - stored in
 the browser's local storage, not synced to the developer's servers.

### 3.2. Data sent to external TI services (on the analyst's explicit action)
When the "check" button next to a specific indicator is clicked, that
indicator's value (IP/domain/hash) is sent to VirusTotal and/or
AbuseIPDB using the API key the analyst entered in settings themselves.
This only happens on click, never automatically.

### 3.3. Data sent to an LLM (Anthropic API or a local model)
Before sending text to an LLM (buttons marked "✨"/Pro), the extension
**automatically replaces** IP/MAC addresses, emails, SSH/AWS keys, JWT
tokens, account names (by keyword and by the `DOMAIN\user` format), and
user-defined company names with placeholders. Before sending, the
analyst sees an exact preview of what will be sent ("🔒 Redacted before
sending"). The model never receives the original values - only
placeholders like `[[IP_1]]`; substituting the real values back happens
locally, when the response is displayed.

For organizations that can't send data to external cloud LLMs even in
redacted form, the extension supports a local LLM (Ollama-compatible
API) - in that case, data never leaves the organization's infrastructure
at all.

### 3.4. Data sent to the licensing server (only with an active Pro license)
The licensing server is not a service run by the developer - it's
infrastructure that the organization deploys and administers itself
(see `/license-server` in the distribution). The extension exchanges the
following data with it:

| Feature | What's sent |
|---|---|
| License activation | the license key, a random device identifier (generated locally) |
| Grafana sync (optional, off by default) | the analyst's nickname (set by the user, doesn't have to be a real name), aggregated counters (BelZor balance, number of events/incidents, breakdown by verdict) - **no card contents, no IOCs, no incident text** |
| Adapter marketplace | adapter configuration (URL patterns, CSS selectors) - technical data, containing no incident information |
| Term glossary | the term/definition text the analyst chose to publish themselves, linked to the organization (not the individual) via the license key |

Grafana sync is **off by default** and requires explicit enabling in
settings. The analyst may leave the nickname used for ranking blank or
arbitrary - the extension never requires a real name.

### 3.5. Data sent to team notifications (Slack/Telegram/MAX/Teams/Jira)
Only when the "send conclusion" button is explicitly clicked - a short
summary (incident title, verdict, reason, link) is sent to the
channel/service that the organization itself configured with its own
credentials. This is internal communication with colleagues, not with
the developer.

## 4. What the extension does NOT do

- It does not collect or send to the developer any browsing history,
 visited sites, or personal user data outside of SIEM incident cards.
- It does not activate automatically on arbitrary sites - only on pages
 matching configured adapters (the universal fallback adapter, capable
 of matching any site, is **off by default** and requires deliberate
 enabling with an explicit warning about the consequences - see
 CHANGELOG v1.5.0).
- It does not sell or share any data with third parties.
- It does not show advertising.
- It does not perform any automated destructive actions (IP blocking,
 host isolation, etc.) - all response command templates are provided
 only as text for the analyst to copy and run manually.

## 5. Data storage and deletion

- All local data is stored in the analyst's browser's
 `chrome.storage.local` and is deleted when the extension is removed,
 or manually by clearing the extension's data in the browser's settings.
- Data on the licensing server (if used) is stored encrypted
 (AES-256-GCM) and is governed by the organization's own retention
 policy - contact your SOC/security administrator for details of data
 retention within your organization.

## 6. Analyst rights

- Redaction before sending to an LLM always applies to the relevant
 features and cannot be disabled by the user - it's a deliberate
 protective constraint, not a setting.
- Using external TI services, LLMs, and messenger notifications requires
 explicitly providing your own API keys/tokens - without them, the
 corresponding features simply don't work; automatic sending without a
 key is not possible.
- A nickname for the team ranking is optional and doesn't have to be a
 real name.

## 7. Changes to this policy

This policy is updated alongside product changes that require
clarifying how data is handled. The current version ships with the
extension in the `PRIVACY_POLICY.md` file.

## 8. Contact

For product-related questions: **belyaev.expert**, Telegram:
**t.me/belyaev_security**.
For questions about data processing within your organization, contact
your company's SOC/security administrator.
