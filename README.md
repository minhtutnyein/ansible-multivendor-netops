# Network Automation — Ansible Projects

This bundle contains three related Ansible projects built for the multi-vendor
lab (Arista vEOS, Cisco IOSv, Palo Alto PAN-OS). Each is self-contained with its
own inventory, vars, vault, templates and README.

```
network-automation/
├── multitier-fabric-ansible/   # ★ the full data-centre fabric (start here)
├── cisco-vpn-ansible/          # Cisco BR01 branch-router IKEv2/IPSec VPN
└── palo-alto-vpn-ansible/      # standalone Palo Alto side of the BR01 VPN
```

## Architecture 
![alt text](ANSIBLE-MULTIVENDOR.jpeg)

## 1. multitier-fabric-ansible  ★ main project
End-to-end automation for the whole fabric, with every device config transcribed
from the live running-configs (idempotent on re-run):

- **vEOS1 / vEOS2** — MLAG domain 12, VARP, per-tier VRFs (WEB/APP/DB), default routes
- **vIOS3** — L2 transit switch (Po3 to leaves, Gi0/2 trunk to the firewall)
- **vIOS4** — L2 access switch (Po4 to leaves, server access ports)
- **pa-fw01** — interfaces + transit subinterfaces, zones, tags, address objects,
  vRouter01 static routes, source-NAT, the full 9-rule security policy, AND the
  BR01/BR02 site-to-site IPSec VPNs

Run it all with `site.yml`. See `multitier-fabric-ansible/README.md` for detail.

## 2. cisco-vpn-ansible
The branch-router end of the site-to-site tunnel — Cisco IOS/IOS-XE IKEv2 proposal,
policy, keyring, profile, IPSec transform-set/profile, the tunnel interface and
static routes. Template-driven, reusable per branch (BR01, BR02, …).

**BR01 (CSR):** `10.1.5.145` → PA outside `10.1.5.5` | IKEv2 AES-256/SHA-256/DH20 | IPSec AES-256/SHA-256/PFS5

### cisco-vpn-deploy task (`deploy_vpn.yml`)

Pushes the full IKEv2/IPSec configuration to every host in `[branch_routers]`,
saves the running-config (idempotent `save_when: modified`), then verifies the
tunnel state. Three tasks run in order:

| # | Task | What it does |
|---|------|--------------|
| 1 | **Push config** | Renders `templates/ikev2_ipsec_tunnel.j2` via `cisco.ios.ios_config` and applies all crypto + tunnel + static-route lines |
| 2 | **Show applied commands** | Prints the IOS commands that actually changed (skipped when list is empty) |
| 3 | **Verify** | Runs `show interfaces Tunnel`, `show crypto ikev2 sa`, `show crypto ipsec sa` and prints the output — skipped in `--check` mode |

**Key variables (host_vars/br01.yml):**
```
peer_ip / tunnel_source / tunnel_destination   — endpoint IPs
tunnel_id / tunnel_ip / tunnel_mask            — Tunnel interface
ike_encryption / ike_integrity / ike_dh_group  — Phase 1 crypto
esp_cipher / esp_hmac / pfs_group              — Phase 2 crypto
static_routes[]                                — network/mask/next_hop list
save_config_when (group_vars)                  — modified | always | never
```

**Run the deploy:**
```bash
cd cisco-vpn-ansible

# One-time: install collection and encrypt vault
ansible-galaxy collection install -r requirements.yml
ansible-vault encrypt group_vars/branch_routers/vault.yml

# Dry run — preview commands, touch nothing
ansible-playbook deploy_vpn.yml --ask-vault-pass --check --diff

# Apply to all branch routers
ansible-playbook deploy_vpn.yml --ask-vault-pass

# Target a single site
ansible-playbook deploy_vpn.yml --ask-vault-pass --limit br01
```

**Adding a new branch (BR02, BR03 …):**
1. Add the host to `inventory.ini` under `[branch_routers]`.
2. Copy `host_vars/br01.yml` → `host_vars/br02.yml` and update the site values.
3. Re-run the playbook — the template and tasks are unchanged.

## 3. palo-alto-vpn-ansible
A focused, standalone playbook for just the Palo Alto end of the BR01 VPN
(IKE/IPSec crypto, gateway, tunnel, proxy-id). This is now also covered inside the
main fabric project's `pa-fw01.yml`; keep this one if you want the VPN managed
separately from the full firewall build.

## Common setup (per project)
```bash
cd <project>
ansible-galaxy collection install -r requirements.yml
pip install pan-os-python            # only for the Palo Alto projects
ansible-vault encrypt group_vars/**/vault.yml   # after putting real creds in
ansible-playbook <playbook>.yml --ask-vault-pass --check --diff   # preview
ansible-playbook <playbook>.yml --ask-vault-pass                  # apply
```

Collections used across the bundle: `arista.eos`, `cisco.ios`,
`paloaltonetworks.panos`, `ansible.netcommon`.

## Notes
- All configs validated and template-rendered locally; run `--check --diff`
  against one device of each type before a full rollout — they were not run
  against live gear here.
- Secrets live in each project's vault file; encrypt before use. The IPSec PSK
  must match across the Cisco and Palo Alto ends.
- Directory structure matters for Ansible auto-loading (`group_vars/<group>/`,
  `host_vars/<host>/`) — extract this archive rather than copying files loose.
