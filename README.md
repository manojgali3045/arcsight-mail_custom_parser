# arcsight-mail_custom_parser
ArcSight FlexConnector parser and runbooks for a custom email-platform syslog source
 Email Platform SIEM Onboarding — ArcSight FlexConnector
 
Documentation and deliverables from onboarding an email platform (Postfix, Cyrus IMAP, Amavisd-new, ClamAV, SpamAssassin, OpenDKIM, OpenDMARC, Roundcube, saslauthd) into ArcSight/Logger.
 
**Connector type:** Syslog-based FlexConnector (UDP)
**Status:** Custom submessage-format parser live in production — 0 unparsed events validated across a 12-hour, ~4.8M-event capture.
 
## Contents
 
| File | Purpose |
|---|---|
| `ArcSight_Connector_Log_Filtering_Runbook.md` / `.docx` | Runbook: reducing noisy log volume at the connector level using `customeventsfilter.regex.*` (not the ESM-style `agents[0].filter[...]`, which doesn't work on Logger-only connectors) |
| `Parser_Creation_Runbook.docx` | Runbook: building a FlexConnector submessage-format parser from raw syslog capture through deployment and field-level Logger verification |
 
## Key facts for anyone picking this up
 
- Parser file format must be **submessage** (`regex=` + `submessage[i].pattern[j].*`), never the flat `regex.count`/`regex[N]` format — the latter loads but silently fails at runtime (NullPointerException flood) on the syslog sub-agent.
- Correlation fields: one custom string field for Queue ID (links events within one Postfix hop), another for Message ID (links a mail across its full lifecycle, including across multiple Postfix hops).
- Connector-level noise filtering uses `customeventsfilter.regex.enabled` / `customeventsfilter.regex.pattern.exclude`, filtering raw text before parsing — confirmed to cut EPS by roughly 90% with zero loss of security-relevant events.
- `agents[0].resolvehostnames=false` prevents internal syslog hostnames resolving to unrelated public IPs in dvchost/dvc.
 
## Open items
 
- [ ] Deploy identical parser + config to standby connector
- [ ] EICAR test to validate the Amavis Blocked/INFECTED path on live traffic
- [ ] Remove legacy syslog forwarding once the new parsed port is confirmed stable
- [ ] Build ESM correlation use cases (SASL spray, repeated single-account failures, non-webmail-source IMAP login, DMARC fail, VIP mailbox filter changes)
 
## General findings pattern (details withheld — internal report only)
 
During onboarding, log review surfaced typical mail-platform operational gaps worth checking on any similar deployment: missing/incomplete DKIM DNS records, backend auth (LDAP) bind failures, blocklist (RBL) lookup failures, clock drift across mail servers, malformed sender domain SPF records, webmail proxies masking real client source IPs, incomplete auth-log forwarding coverage, and accounts showing sustained automated-pattern login failures that warrant a credential check. Specific hostnames, domains, and account names are documented separately in the internal (non-public) version of this report.
