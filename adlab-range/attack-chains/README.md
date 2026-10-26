# Attack Chains

Each document follows the same structure: recon → exploitation → impact →
detection → remediation, and maps to MITRE ATT&CK.

| # | Chain | ATT&CK | Misconfig seeded by |
|---|---|---|---|
| 01 | [Kerberoasting](01-kerberoasting.md) | T1558.003 | SPN on `svc_sql` + weak password |
| 02 | [GenericAll ACL abuse](02-acl-abuse.md) | T1098 / T1069 | Weak ACE granting `jdoe` control of `svc_sql` |

## Planned additions (build these next and commit incrementally)

- 03 — AS-REP roasting (`asrep_svc`, pre-auth disabled) — T1558.004
- 04 — AD CS ESC1 against the seeded Enterprise CA — T1649
- 05 — LLMNR poisoning + SMB relay from the workstation segment — T1557
- 06 — Child-to-parent trust abuse (`child.lab.local` → `lab.local`) — T1558
- 07 — Persistence: DCSync ACL + golden ticket — T1003.006 / T1558.001
