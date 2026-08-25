# Belyaev Response - quick start for analysts

## 1. Installation

1. `chrome://extensions` (or `edge://extensions`, `browser://extensions` in Yandex) → turn on "Developer mode" (the toggle in the top right)
2. "Load unpacked" → select the `soc-copilot` folder from the archive
3. If the browser complains about the manifest, make sure you unpacked the whole archive, not just part of the files

## 2. Configuring the adapter for your SIEM (one time)

Without this step, the extension won't know where on the page to look for the card's title/description.

1. Open a real incident card in your SIEM in a separate tab
2. Open the extension's settings (icon → "Settings / adapters")
3. At the very top of the settings page there's a dropdown, "Tab with the open incident card." **Select exactly the tab** where the card is open
4. Find the block for the platform you need (MaxPatrol SIEM / RuSIEM / BI.ZONE) → enter a URL substring under "URL patterns" (for example, `/incidents/`)
5. For each field (Title, Description, Criticality, etc.), click "Select on page" → click the corresponding element on the tab with the card
6. Done - the selector saves automatically

## 3. Your first incident triage

1. Open an incident card → a vertical **BR** button appears on the right → clicking it opens the panel
2. **IOC** tab - automatically found indicators, the "check" button enriches them via VirusTotal/AbuseIPDB (needs your own API key in settings, optional)
3. **MITRE** tab - which tactics/techniques are already visible in the card's text
4. **Playbook** tab - a checklist of response steps
5. **🎓 Mentor** tab - if you're L1, start here: step-by-step training right on this card

## 4. Closing the incident

1. **Conclusion** tab → select a verdict (TP/FP/Benign/Escalation), fill in the reason and actions
2. "Save as completed" → BelZor points are awarded (reward for the incident)
3. With a configured Pro license, you can send the conclusion straight to Slack/Telegram/MAX/Teams/Jira

## 5. Appearance and language (if you need to change them)

1. Settings → "Appearance and language" → choose a theme/language → "Save"
2. Changes apply to all open tabs automatically - if the panel was already open, just close and reopen it to see the new look right away

## 6. Common problems

| Symptom | Solution |
|---|---|
| The BR button doesn't appear on the card | The adapter isn't configured for this URL - see step 2 |
| "Extension is not active on this page" in the popup | Same cause - the adapter's URL pattern doesn't match the current page |
| IOC/MITRE are empty | Check that the "Description"/"Container (fallback)" selector points to the block with the card's text, not an empty div |
| Pro features unavailable | For the first 30 days after install they're enabled automatically (trial). After that you need a license - see the "License" section in settings |

## 7. Where to look for more details

- `README.md` - full description of features and pricing
- `CHANGELOG.md` (or the "What's new" button in the popup) - version history and known limitations of each feature
