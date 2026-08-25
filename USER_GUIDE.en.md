# User Guide - Belyaev Response

## How to start working with an incident card

1. Open an incident card in your SIEM
2. Click the **BR** button on the right side of the screen - the side panel opens
3. Wait for the automatic analysis (1-3 seconds)

## Panel tabs

### IOC - Indicators of Compromise
Indicators automatically extracted from the card's text. For each one:
- **Copy** - to the clipboard
- **Check** - enrichment via VirusTotal/AbuseIPDB (needs an API key in settings)

### MITRE ATT&CK
- A list of detected techniques with tactic and description
- A forecast of the next step (heuristic, a hypothesis to verify)
- Kill chain - a visual timeline

### Playbook
A response checklist automatically selected based on the incident type.
Check off steps as you complete them. Includes an asset-owner lookup
block in the CMDB/ITSM.

### Report
A structured draft report - can be exported or copied.

### Conclusion
1. Select a verdict: TP / FP / Benign / Escalation
2. Fill in: the reason, impact, actions taken, recommendations
3. Choose a format (MD/TXT/HTML)
4. **Generate** - the conclusion is copied to the clipboard with your signature
5. **Save as completed** - BelZor points are awarded

### 🎓 Mentor L1 / L2 / L3
- Step-by-step training right on the current card
- Quizzes from a bank of 101 questions across 9 categories
- L2: DFIR commands - switch between OSes (Windows/Linux/macOS/AstraLinux)
- L3: Threat hunting, GosSOPKA, escalation criteria

### 📊 Statistics
- BelZor balance and rank (7 levels from "Trainee" to "Senior L2/L3")
- Incident trends over time
- Campaign graph - cross-correlation of related incidents

## BelZor - the reward system
Awarded for closed incidents: Critical 50, High 30, Medium 15, Low 5 BelZor.
Bonuses for: MITRE context, a complete playbook, checking IOCs via TI.

## Changing the theme and language
Settings → "Appearance and language" → choose a theme (Dark/Light/High
contrast) and language → "Save". Applied immediately.

## Test bench
Popup → "🧪 SIEM test bench" - three synthetic scenarios
(Phishing/Ransomware/C2) with every type of IOC, for testing the plugin
without a real SIEM.
