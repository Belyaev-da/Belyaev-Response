# Belyaev Response - setup guide

For the team lead/administrator setting up the plugin for a team. All
sections are in the same order they appear on the settings page
(extension icon → "Settings").

## 1. Appearance and language
- **Theme**: Dark / Light / High contrast - chosen by the analyst,
 applied to the main panel (incident card) and the popup. The settings
 page itself follows the theme too.
- **Interface language**: 30 languages, but only translates the main
 panel's "shell" (title, tabs, buttons) - not the settings page itself
 and not the content (playbooks, the mentor, and reports stay in
 Russian).

## 2. License
Just the key field (`BLZR-XXXX-XXXX-XXXX-XXXX`) - the server URL is baked
into the code, no need to enter it (see `LICENSE_SERVER_SETUP.md` if
you're the product owner deploying your own server). 30 days of Pro
features are available right after install, with no key needed.

## 3. Grafana sync (Pro)
Sends aggregated BelZor statistics to the licensing server for
dashboards. Off by default. Only enable it if the licensing server is
already deployed and the license is active. A nickname is optional -
shown as "anonymous" by default.

## 4. Adapters for your SIEM
The most important step for a new team:
1. Open a real incident card in a separate tab
2. At the top of the settings page, select that tab from the dropdown
3. In the card for the platform you need (MP SIEM/RuSIEM/BI.ZONE), enter
 a URL pattern (a substring of the address, e.g. `/incidents/`)
4. Click "Select on page" next to each field → click the corresponding
 element on the tab with the card
5. Export the configured adapter ("Export / import settings") and
 distribute the JSON file to the rest of the analysts - no need to
 repeat the setup on every machine

**The Universal (fallback) adapter is off by default** - enable it
consciously, since it matches any site (see the warning right in the
UI).

## 5. API keys for enrichment
VirusTotal / AbuseIPDB - optional, for the "check" button on indicators.
Anthropic - for LLM features (Pro). The fields are write-only: after
saving they only show dots, never the key itself again. Keys are
encrypted (AES-256-GCM) and decrypted only inside the extension's
background process.

## 6. MITRE ATT&CK database
The "Sync" button pulls in the full official technique database instead
of the built-in curated set (~25 techniques) - do this every few months,
not at every setup.

## 7. Playbooks (JSON)
An editable list of keyword-based checklists. Adjust them to fit your
processes - the format is documented right in the text field; use
"Reset to defaults" if you break something.

## 8. Redaction before sending to an LLM
Enter your company's names/domains (one per line) - they'll be replaced
with `[[ORG_N]]` before being sent to an LLM, along with
IPs/MACs/emails/keys/account names (that part always works, no setup
needed). Do this before using any LLM features for the first time.

## 9. Team notifications
Configure one or more channels - a dedicated button for each appears in
the panel:
- **Slack/Mattermost** - Incoming Webhook URL
- **Telegram** - bot token (@BotFather) + Chat ID
- **MAX** - bot token (@MasterBot) + Chat ID
- **Teams** - the channel's Incoming Webhook URL
- **Jira** - Base URL, email, API token (id.atlassian.com), project key

## 10. Local LLM (Ollama)
For organizations that can't send data to external cloud LLMs even in
redacted form. Enable the checkbox, provide a URL (defaults to
`http://localhost:11434`) and a model. Requires a separately deployed
Ollama server in your infrastructure.

## 11. Analyst wellbeing
Gentle break reminders when signs of fatigue are detected. On by
default; only turn it off at the team's explicit request - it's not
surveillance, the data never leaves the browser.

## 12. Team settings export/import
Exports adapters + playbooks (without keys) to a file - distribute it to
new hires instead of setting things up from scratch again.

## 13. Adapter marketplace
Requires an active license. "Refresh list" → "Install" - pulls in a
ready-made adapter configured by another organization for the same SIEM
platform.

## 14. Enterprise policies (for IT, not the analyst)
Centralized settings rollout via `chrome.storage.managed` (Windows
registry / Google Admin Console) - license key, company names, webhooks,
disabled adapters. The field schema is in `schema.json` inside the
extension archive.

## Related documents in the archive
- `QUICKSTART.md` - quick start for a regular analyst
- `LICENSE_SERVER_SETUP.md` - if you're the product owner deploying your
 own server
- `PRIVACY_POLICY.md` - what is sent where
- `CHANGELOG.md` - version history and known limitations of each feature
