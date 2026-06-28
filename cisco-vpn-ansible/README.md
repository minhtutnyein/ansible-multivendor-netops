# Cisco IOS IKEv2/IPSec VPN — Ansible Deployment

Deploys the IKEv2 proposal/policy/keyring/profile, IPSec transform-set + profile,
the Tunnel interface, and static routes onto Cisco IOS / IOS-XE branch routers,
then saves the config (the `do wr mem` step) and verifies the tunnel.

## Topology

```
  Cisco CSR BR01 (10.1.5.145)  <==IKEv2/IPSec (tunnel.1)==>  Palo Alto pa-fw01 (10.1.5.5)
  Tunnel1 : 11.11.11.11/32                                     tunnel.1 : 111.111.111.111/32
  IKEv2   : AES-256-CBC / SHA-256 / DH group20 / 28800 s
  IPSec   : AES-256-CBC / SHA-256 / PFS group5 / 3600 s
```

## Layout
```
ansible.cfg
inventory.ini
deploy_vpn.yml
group_vars/branch_routers/vars.yml     # non-secret group vars
group_vars/branch_routers/vault.yml    # secrets — ENCRYPT THIS
host_vars/br01.yml                      # all BR01 site values
templates/ikev2_ipsec_tunnel.j2         # the config, parameterised
requirements.yml
```

## One-time setup
```bash
# 1. Install the Cisco collection
ansible-galaxy collection install -r requirements.yml

# 2. Put real secrets in the vault and encrypt it
#    (edit the placeholders first, then:)
ansible-vault encrypt group_vars/branch_routers/vault.yml
```

## Deploy
```bash
# Dry run — show what WOULD change, change nothing on the box
ansible-playbook deploy_vpn.yml --ask-vault-pass --check --diff

# Apply for real
ansible-playbook deploy_vpn.yml --ask-vault-pass

# Just one site
ansible-playbook deploy_vpn.yml --ask-vault-pass --limit br01
```

## Adding another branch (BR02, BR03 ...)
1. Add the host to `inventory.ini` under `[branch_routers]`.
2. Copy `host_vars/br01.yml` to `host_vars/br02.yml` and edit the values.
The template and playbook do not change.

## Notes
- **Saving config:** handled by `save_when: modified` in the playbook (idempotent),
  not a literal `do wr mem`. Set `save_config_when: always` in group vars to force it.
- **`no shut`** is sent as `no shutdown` (the full IOS keyword).
- **Idempotency caveat:** Cisco does not echo the pre-shared key back in the
  running-config in plaintext, so the two `pre-shared-key` lines may report as a
  change on every run even when unchanged. This is expected and harmless. Same can
  apply to a few `crypto` lines depending on IOS version.
- **`match address local`** in your IKEv2 policy assumes IOS-XE syntax; on some
  platforms this is `match fvrf ... match address local`. Adjust the template line
  if your platform differs.
- Run with `--check --diff` first against one device to confirm the generated
  commands before a wide rollout.
