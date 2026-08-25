# Administrator Guide - Belyaev Response

## Centralized deployment

### Enterprise policies (chrome.storage.managed)

Configure via the Windows registry (GPO) or the Google Admin Console:

```json
{
  "LicenseKey": "BLZR-XXXX-XXXX-XXXX-XXXX",
  "CompanyNames": ["Acme Corp", "acmecorp.example"],
  "BurnoutCheckEnabled": true
}
```

The full list of policies is in `schema.json`.

### Deployment via GPO (Windows)
1. Copy `soc-copilot/` to a network share
2. Create a GPO: Computer Configuration → Administrative Templates → add the template
3. Set the path to the extension and the Extension ID
4. Push `LicenseKey` via managed storage - analysts don't enter the key manually

## Licensing server

Details: [`LICENSE_SERVER_SETUP.md`](LICENSE_SERVER_SETUP.md)

### Issuing a license
```bash
ADMIN_API_KEY=<secret> node scripts/issue-license.js \
  --org "Organization Name" --tier pro --seats 50 --days 365 \
  --url https://belyaev.expert
```

### Monitoring
```bash
# List active licenses
curl -H "X-Admin-Key: <secret>" https://belyaev.expert/api/admin/licenses

# Revoke a license
curl -X POST -H "X-Admin-Key: <secret>" \
  https://belyaev.expert/api/admin/licenses/BLZR-XXXX/revoke
```

### Auto-revocation on device limit exceeded
If activation is attempted beyond the `seats` limit, the entire key is
automatically revoked. The reason and the ID of the triggering device are
logged in the database. Issue a new key via `issue-license.js`.

## Grafana dashboard

1. Enable sync in each analyst's extension settings (or via GPO,
 `GrafanaSyncEnabled: true`)
2. Prometheus endpoint: `https://belyaev.expert/metrics?key=<license-key>`
3. Data: BelZor ranking, TP/FP/Benign trends, activity (pseudonymized)

## Team settings export/import
Settings → "Export / import team settings" → export a JSON with adapters
and playbooks → distribute to new hires. Encryption keys and license data
are not exported.

## Support
belyaev.pro@mail.ru · t.me/belyaev_security
