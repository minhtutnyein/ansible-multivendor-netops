# PA-VM01 — IKEv2/IPSec VPN (peer of Cisco BR01) via Ansible

Configures the Palo Alto side of the site-to-site tunnel whose Cisco end you
built separately. PAN-OS modules use the firewall's **XML API over HTTPS**, so
this runs with `connection: local` from the control node — no SSH to the firewall.

```
Cisco BR01 (10.1.5.7)  <==== IKEv2 / IPSec ====>  PA-VM01 (10.1.5.5)
```

## Crypto mapping (must match BR01)
| Setting            | Cisco BR01                | PA-VM01                |
|--------------------|---------------------------|------------------------|
| P1 encryption      | aes-cbc-256               | aes-256-cbc            |
| P1 integrity       | sha256                    | sha256                 |
| P1 DH group        | 20                        | group20                |
| P1 lifetime        | 28800                     | 28800 s                |
| P2 ESP encryption  | esp-aes 256               | aes-256-cbc            |
| P2 ESP auth        | esp-sha256-hmac           | sha256                 |
| P2 PFS             | group5                    | group5                 |
| PSK                | !k3v2K3y!@#               | (same, from vault)     |

## Setup
```bash
ansible-galaxy collection install -r requirements.yml
pip install pan-os-python
# edit real values, then encrypt:
ansible-vault encrypt group_vars/firewalls/vault.yml
```

## Run
```bash
ansible-playbook deploy_pa_vpn.yml --ask-vault-pass --check   # stage/preview
ansible-playbook deploy_pa_vpn.yml --ask-vault-pass           # apply + commit
```

## Things to confirm before applying
- **`pa_egress_interface` / `pa_local_ip`** — must be the interface actually
  holding 10.1.5.5. If 10.1.5.5 is a different interface, fix it in host_vars.
- **`security_zone` and `virtual_router`** must already exist (default VR does).
  A security policy permitting traffic to/from the VPN zone is NOT created here.
- **`remote_networks`** — these are placeholders. They must be the subnets that
  live *behind the Cisco BR01* (what PA should push into the tunnel), which is a
  routing-direction decision only you can confirm against your topology.
- **Proxy IDs** are set any/any (0.0.0.0/0) for Cisco VTI interop. If BR01 were
  policy-based you'd narrow these to specific local/remote pairs.

## Verify
The play runs `show vpn ike-sa` / `ipsec-sa` / `flow` at the end. You want the
IKE SA established and an IPSec SA present for peer 10.1.5.7. You can also rerun
just the check from the CLI:
```bash
ansible firewalls -m paloaltonetworks.panos.panos_op \
  -a 'provider="{{ provider }}" cmd="show vpn ipsec-sa"' --ask-vault-pass
```
