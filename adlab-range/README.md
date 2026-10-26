# adlab-range — Automated Enterprise Active Directory Pentest Lab

An infrastructure-as-code Active Directory range that builds a two-domain
enterprise topology seeded with **documented, real-world misconfigurations**,
then pairs every attack path with its detection and remediation. Built to
train full kill-chain engagements and to serve as the target environment for
professional reporting samples.

> **Lab use only.** Everything here runs on isolated private networks
> (`10.10.10.0/24`). Never run the attack steps against systems you are not
> explicitly authorized to test.

## Architecture

```
                 10.10.10.0/24 (VirtualBox host-only)
  ┌──────────────┐       ┌──────────────┐
  │ DC01         │       │ DC02         │
  │ lab.local    │◄─────►│ child.lab... │  parent/child trust
  │ 10.10.10.10  │       │ 10.10.10.11  │
  │ AD DS + ADCS │       │ AD DS        │
  └──────┬───────┘       └──────────────┘
         │ domain join
  ┌──────┴───────┐       ┌──────────────┐
  │ WS01 Win11   │       │ jump01       │
  │ 10.10.10.20  │       │ Ubuntu 22.04 │
  └──────────────┘       │ 10.10.10.50  │
                         │ attack tooling│
                         └──────────────┘
```

## Seeded vulnerabilities

| Weakness | Account/Object | Attack chain |
|---|---|---|
| SPN + weak service password (Kerberoast) | `svc_sql` | [01](attack-chains/01-kerberoasting.md) |
| `GenericAll` ACE between users | `jdoe` → `svc_sql` | [02](attack-chains/02-acl-abuse.md) |
| Kerberos pre-auth disabled (AS-REP roast) | `asrep_svc` | planned 03 |
| Enterprise AD CS (ESC1 template work) | `LAB-ISSUING-CA` | planned 04 |
| Parent/child trust | forest topology | planned 06 |
| No EDR; Sysmon only | all Windows hosts | detection practice |

## Requirements

- VirtualBox 7+ and Vagrant 2.4+
- Ansible 2.15+ on the host (`pip install ansible`)
- Collections installed automatically via `ansible/requirements.yml`
- Minimum 12 GB RAM allocated to VirtualBox

## Build it

```bash
git clone https://github.com/YOUR-GITHUB-USERNAME/adlab-range.git
cd adlab-range
vagrant up                 # ~45-90 min first run; promotes DCs, joins WS01
vagrant provision          # re-run Ansible after edits
vagrant ssh jump01         # attack host with Impacket/BloodHound preinstalled
```

Validate the domain from `jump01`:

```bash
source /opt/lab-venv/bin/activate
nxc smb 10.10.10.10 -u jdoe -p 'Welcome2026!'
```

## Repository layout

```
adlab-range/
├── Vagrantfile                  # 4-VM topology
├── ansible/
│   ├── site.yml                 # playbook entry point
│   ├── inventory
│   ├── group_vars/all.yml       # lab variables (vault secrets in real use)
│   ├── playbooks/               # DC promotion, domain join, seeding, Sysmon
│   └── files/sysmon/            # detection config
├── attack-chains/               # documented kill chains w/ ATT&CK mapping
├── detections/sigma/            # detection rules mapped to each chain
└── reports/                     # sample deliverables produced against this lab
```

## Purple-team loop (the differentiator)

Every attack chain doc contains the matching detection query and the exact
hardening step. Extend the loop by:

1. Running the attack and confirming the Sigma/Sysmon event fires in Wazuh/Elastic.
2. Applying the remediation playbook and re-running the attack to show it fails.
3. Committing screenshots of both states into `attack-chains/`.

## Cleanup

```bash
vagrant destroy -f
```
