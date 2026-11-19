# Automation

Scripts and playbooks for the parts of the lab I got tired of doing by hand.

Right now this is mostly bash. Ansible is the goal — the lab has two
Proxmox nodes and enough VMs that rebuilding one by hand is a waste of
an evening.

## What's here

(nothing yet — see below)

## What's planned

- **Wazuh agent enrollment** — playbook to install the agent and register
  it with the manager on any new VM. Right now I do this manually every
  time I spin up a target box, which is the main reason this folder exists.
- **Proxmox VM provisioning** — clone from template, set the VLAN tag and
  static IP, bring it up on labnet.
- **Suricata rule deployment** — push custom rules and the decoder changes
  to the sensor without SSHing in and editing files in place.
- **Log triage helpers** — small bash scripts for the greps I run often
  (auth failures, Suricata alert counts by signature).

## Notes

I'm writing these as I hit the pain, not up front. If a playbook is here
it's because I ran the manual version enough times to know what it should
actually do.
