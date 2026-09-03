🛡️ Belyaev Response
## 🎯 What it is

**Belyaev Response** is a browser extension (Yandex, Chrome, Edge) that
opens over an incident card in a SIEM and turns the analyst's workspace
into an integrated investigation center.

### AI second brain for SOC analysts

_A browser extension that helps investigate incidents faster and more
thoroughly - right inside the SIEM interface._

**IOC · MITRE ATT&CK · Playbooks · AI mentor · DFIR · Incident Reporting**

---

## 🔐 Release `v1.9.3` checksums

> Verify the checksum after downloading to confirm the file's integrity.

| Field | Value |
|----------|----------|
| **Version** | `v1.9.3` |
| **Build date** | September 4, 2026 |
| **Archive** | `belyaev-response-v1.9.3.zip` |
| **Size** | 199 KB (203,659 bytes) |
| **SHA-1** | `b7813eca331dfbc715a0291ad9a8dc412b7386b6` |
| **SHA-256** | `8491f15fcb0ef3488a8f4585b0ea6d966fcedba84db49e6e89ec5560834960d1` |
| **SHA-512** | `22c30172521aa13808ebbc12cf589be7700a5d863ddd8fddb917b46d04ff4ef5a6691616626f725b2e94d6aab13083bb87dc6745720834b6c20351df8310c84c` |
### Integrity check

```bash
# Linux / macOS
sha256sum belyaev-response-v1.9.3.zip
```
```powershell
# Windows PowerShell
Get-FileHash .\belyaev-response-v1.9.3.zip -Algorithm SHA256
```

⚠️ If the checksum doesn't match, do not install the file.

🔐 Security:
<img width="1774" height="744" alt="image" src="https://github.com/user-attachments/assets/be2ae5da-5354-4252-aac0-2494e50b7965" />

---

Key features:
- IOC extraction
- MITRE ATT&CK mapping
- Playbook selection
- Threat Intelligence enrichment
- Draft conclusion
- Contextual AI mentor

> Not a replacement for an expert. A second brain for the analyst: speeds
> up routine work, adds context, and leaves the final decision to the
> human.

### Supported platforms

| Platform | Support |
|-----------|-----------|
| MaxPatrol SIEM | ✅ dedicated adapter |
| RuSIEM | ✅ dedicated adapter |
| BI.ZONE | ✅ dedicated adapter |
| Security Vision SIEM, SIEM KUMA, SearchInform SIEM, R-Vision SIEM, Splunk, ArcSight | ✅ via one "Universal" adapter |
| Any other web-based SIEM | ✅ configurable universal adapter |

All platform adapters are **off by default** - the analyst consciously
enables the one they need after configuring the selectors (see "How it
works" below).

---

## ⚡ Features

### Incident investigation

- IOC extraction: IPv4/IPv6, domains, URLs, MD5/SHA-1/SHA-256, email, CVE,
 file and registry paths
- Threat Intelligence: checks via VirusTotal and AbuseIPDB
- MITRE ATT&CK: automatic technique mapping, kill chain visualization
- Playbooks: 9 types by incident category, with asset-owner lookup in
 CMDB/ITSM and a 4-level criticality model (Mission/Business-critical,
 Business-operational, Office productivity)
- IOC highlighting right on the SIEM page (with a signed button in the
 panel header)
- Cross-tab correlation
- Diff of card changes
- Alert storm: detecting mass triggers
- OS-specific DFIR recommendations (Windows/Linux/macOS/AstraLinux)

### Reporting and response

- Report and conclusion - choice of format: Markdown / TXT / HTML
- Formal conclusion: TP / FP / Benign / Escalation, with the analyst's
 name and contact details at the bottom
- Auto-fill of IOCs and ATT&CK techniques
- Conclusion history, hints based on similar incidents
- Copy-paste response commands
- PDF export via print
- Integrations with Slack, Mattermost, Telegram, MAX, Jira, Teams

### SOC team training

| Level | Functionality |
|---------|-----------|
| L1 | Spotlight, step-by-step actions, quizzes, a leveling system |
| L2 | DFIR tools, remediation recommendations |
| L3 | Escalation criteria, threat hunting, GosSOPKA |
| Socratic AI | Guiding questions instead of ready-made answers |

### Detection engineering

- Draft Sigma and YARA rules
- A "second opinion" on a selected fragment
- Voice input and hotkeys
- A risk badge on the extension icon

### LLM integration

GigaChat, YandexGPT, Qwen, Ollama, DeepSeek, Mistral - as separate
options, plus a universal OpenAI-compatible mode for vLLM/GLM/
Foundation-Sec/Venice/DeepHat/RAG proxies and any other compatible
service. Local LLM support (Ollama) - data never leaves the
organization's infrastructure.

### BelZor and team metrics

- Gamification of incident handling, rewards for criticality and quality
- Bonuses for MITRE context and TI checks
- Analyst ranks and local statistics
- Weekly/monthly/quarterly/yearly reports, CSV export
- Grafana sync (Pro)

---

## 🆕 What's new in v1.9.3

- **SIEM test stand**: the panel opens again on the bundled `test-siem.html`
 stand (the BR button used to be missing); switching scenarios refreshes the
 panel contents.
- Resilient `content.js` ↔ `background.js` messaging after an extension
 update/restart (1.9.1–1.9.2).
- Client-side hardening (HTML-export escaping, cryptographically strong
 identifiers, user-input length limits).

Full changelog - `CHANGELOG.md`.

---

## 🔐 Data security

The extension is designed around the **privacy by design** principle:

- Platform adapters are off by default - the extension doesn't parse or
 analyze the content of arbitrary pages without the analyst's explicit
 choice
- API keys, tokens, webhooks, license: local storage, `AES-256-GCM`
- Secrets are only decrypted in `background.js`, never exposed to the DOM
- External requests come from the service worker, not from the SIEM's
 page context
- Automatic masking of IPs/MACs, emails, and keys before sending to an
 LLM, with a preview of the redacted text before sending
- LLM calls happen only on the analyst's explicit action and with a
 configured key
- Local LLM support via Ollama - with no external services at all
- Notifications to the team contain real IOCs (intentionally - the team
 needs real values to respond)

⚠️ Redaction is a heuristic mechanism, not a guarantee. Clear the use of
external LLMs with your security team.

---

## 🧩 How it works

```
Incident card → Adapter → IOC extraction →
                                    ↓
                        ┌───────────┼───────────┐
                        ↓           ↓           ↓
                    MITRE      Threat        Playbook
                    ATT&CK      Intel
                        └───────────┼───────────┘
                                    ↓
                        Belyaev Response side panel
                                    ↓
                        Conclusion / Report / Communication
```

### Workflow

1. Install the extension in your browser
2. Configure the adapter you need using the visual element picker and
 enable it (all adapters are off by default)
3. Open an incident card
4. Get IOCs, ATT&CK mapping, a playbook, and TI in the side panel
5. Prepare a conclusion and hand it off to the team

### Why selectors are configured manually

MaxPatrol SIEM, RuSIEM, and BI.ZONE are often deployed on-prem with
custom markup. So the extension doesn't "guess" - it lets you pick the
fields visually:

`Settings → Adapters → open incident card in a separate tab → "Select on page" → click the field`

For platforms with no dedicated adapter, there's "Universal," with
heuristic selectors by default.

---

## 🚀 Installation

### Developer mode

1. Download and unpack `belyaev-response-v1.9.3.zip`
2. Open `chrome://extensions` (or `edge://extensions`,
 `browser://extensions` in Yandex Browser)
3. Turn on **Developer mode**
4. Click **Load unpacked**
5. Select the unpacked archive folder
6. Open the extension popup → **Settings / adapters** → configure the
 platform you need

### Configurable integrations

| Integration | Purpose | Key required |
|-----------|-----------|------|
| VirusTotal | IOC enrichment | Yes |
| AbuseIPDB | IP reputation | Yes |
| Anthropic / GigaChat / YandexGPT / others | LLM summaries | Yes |
| Ollama | Local LLM | No |
| Slack / Mattermost / Telegram / MAX / Jira / Teams | Team notifications | Yes |

> Keys are stored only locally, in `chrome.storage.local`, encrypted.

---

## 💳 Pricing

| Feature | Free | Pro |
|-------------|------|-----|
| IOC extraction and highlighting | ✅ | ✅ |
| MITRE ATT&CK and playbooks | ✅ | ✅ |
| Redaction | ✅ | ✅ |
| L1/L2/L3 mentor | ✅ | ✅ |
| DFIR and security tooling | ✅ | ✅ |
| Cross-correlation | ✅ | ✅ |
| BelZor and statistics | ✅ | ✅ |
| LLM summary and forecast | - | ✅ |
| Socratic AI | - | ✅ |
| PDF conclusions | - | ✅ |
| Messengers | - | ✅ |
| Grafana and adapter marketplace | - | ✅ |

**Free** - unlimited in time.
**Pro** - 30-day trial, then activation via a license key.

---

## 🏗️ Architecture

```
manifest.json                  # Manifest V3
background.js                  # TI/LLM, tabs, context menu, webhooks
adapters.js                    # MP SIEM / RuSIEM / BI.ZONE / Universal
content.js                     # Parsing, side panel, picker, highlighting
test-siem.html + test-siem.js   # built-in test bench

utils/
├── ioc-extractor.js           # Regex-based extraction
├── mitre-attack.js            # Mapping and forecasting
├── playbooks.js               # Playbooks
├── report-template.js         # Report (MD/TXT/HTML)
├── conclusion-template.js     # Conclusion (MD/TXT/HTML)
├── redaction.js               # Redaction before sending to an LLM
├── crypto.js                  # AES-256-GCM
├── license.js                 # License and trial
├── mentor.js                  # L1/L2/L3 mentor
├── dfir-tools.js              # DFIR reference by OS
├── szi-recommendations.js     # Security-tooling recommendations
├── detection-rules.js         # Sigma/YARA
├── alert-storm.js             # Mass-trigger detector
├── campaign-graph.js          # Campaign graph
├── response-snippets.js       # Response commands
├── burnout.js                 # Fatigue indicator
├── belzor.js                  # Gamification
├── stats.js                   # Metrics and CSV
└── i18n.js                    # Translations (30 languages)

popup/                         # Popup: status, global switch, test bench
options/                       # Settings, adapters, keys, playbooks
```

---

## ⚠️ MVP limitations

- Adapters require manual CSS selector setup (and manual enabling)
- Activation is based on URL patterns; complex routes need several rules
- The free VirusTotal tier has a quota; a commercial key is needed for
 heavier load
- LLM summaries are a draft/hypothesis, not an automatic verdict
- The final decision is always made by the analyst
- Redaction is a heuristic, not a 100% guarantee

---

## 🗺️ Roadmap

- Searching for similar incidents via SIEM and ticketing system APIs
- Writing the report into a ticket field
- Centralized configuration rollout to the SOC team
- More adapters for other web consoles
- Threat hunting and detection engineering scenarios

---

## 📌 Versioning and quality

The project uses [Semantic Versioning](https://semver.org/). Full
changelog - in `CHANGELOG.md` inside the archive.

Before each release - a manual security and bug review (XSS, CSP,
injections, secret isolation, leaks in message handlers).

---

### Belyaev Response - fewer context switches. More context. A stronger SOC.

Author: **Dmitry Belyaev**
🌐 [belyaev.expert](https://belyaev.expert)
💬 Support: [belyaev.pro@mail.ru](mailto:belyaev.pro@mail.ru)
