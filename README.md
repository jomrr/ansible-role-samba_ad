# Ansible Role: samba_ad

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-samba_ad)
![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-samba_ad)
![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-samba_ad)
[![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-samba_ad/dev.yml?branch=dev&label=dev)](https://github.com/jomrr/ansible-role-samba_ad/actions/workflows/dev.yml?query=branch%3Adev)
[![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-samba_ad/main.yml?branch=main&label=main)](https://github.com/jomrr/ansible-role-samba_ad/actions/workflows/main.yml?query=branch%3Amain)

Ansible role for managing Samba AD users, groups, organizational units, DNS
zones, and static records.

## Scope

### Managed

- Reserved DNS names.
- AD-integrated forward and reverse DNS zones, aging options, and individual
  static DNS records.
- Domain users, organizational units, groups, and additive or authoritative
  group memberships.

### Not Managed

- DC installation, provisioning, joins, and service configuration.
- Windows LAPS schema and delegation.
- Domain password policies and Password Settings Objects (PSOs).
- Automatic PTR creation and replacement of complete DNS record sets.

## Requirements

- An existing Samba AD domain with working DNS and Kerberos discovery.
- Native Samba Python bindings for the interpreter executing the role.
- Administrator credentials supplied through Ansible Vault or another secret
  store.
- RFC2307 enabled in the domain when managing POSIX attributes.
- jomrr.samba >=2.1.0 for zone aging options and record TTL management.

## Dependencies

```yaml
collections:
  - name: community.general
    version: '>=12.0.0'
  - name: jomrr.samba
    version: '>=2.1.0'
```

## Role Variables

### `samba_ad_server`

Type: `str`. Required: `true`.

DNS hostname of the existing Samba AD domain controller.

### `samba_ad_realm`

Type: `str`. Required: `true`.

Kerberos realm of the existing domain.

### `samba_ad_admin_password`

Type: `str`. Required: `true`.

Administrator password for directory operations, supplied through a secret
store.

### `samba_ad_dns_reserved_names`

Type: `list`. Required: `false`.

DNS names reserved with administrator-owned loopback A records; an empty list
stops management.

Default:

```yaml
samba_ad_dns_reserved_names:
  - wpad
  - isatap
```

### `samba_ad_dns_zones`

Type: `list`. Required: `false`.

AD-integrated forward or reverse DNS zones; omitted entries are left unmanaged.

Default:

```yaml
samba_ad_dns_zones: []
```

### `samba_ad_dns_records`

Type: `list`. Required: `false`.

Individual static DNS records; other values at the same name are preserved.

Default:

```yaml
samba_ad_dns_records: []
```

### `samba_ad_ous`

Type: `list`. Required: `false`.

Organizational units in parent-before-child order; deletion uses reverse order
and requires empty OUs.

Default:

```yaml
samba_ad_ous: []
```

### `samba_ad_users`

Type: `list`. Required: `false`.

Domain user accounts; omitted entries are left unmanaged.

Default:

```yaml
samba_ad_users: []
```

### `samba_ad_groups`

Type: `list`. Required: `false`.

Domain groups and optional memberships; all groups are created before resolving
nested memberships.

Default:

```yaml
samba_ad_groups: []
```

### `samba_ad_user_update_password`

Type: `str`. Required: `false`.

User password update policy; always intentionally changes passwords on every
run. Overridable per user.

Default:

```yaml
samba_ad_user_update_password: on_create
```

### `samba_ad_group_members_purge`

Type: `bool`. Required: `false`.

Remove unlisted group members when members is supplied; overridable per group.

Default:

```yaml
samba_ad_group_members_purge: false
```

## Check Mode

Check mode previews changes to an existing domain. DNS record checks require an
existing zone: a zone predicted for creation is not available to subsequent
record tasks in check mode. A normal run can create new zones and their records
together.

## Security Notes

- The role reserves wpad and isatap as administrator-owned A records pointing to
  127.0.0.1. Configure samba_ad_dns_reserved_names to select the names; removing
  a name stops management and preserves the record. Existing record ownership
  and ACLs are not changed.

## Operational Notes

- Directory and DNS operations authenticate as Administrator using
  samba_ad_admin_password against samba_ad_server. Run the role once per domain;
  AD replicates the resulting changes to other DCs.
- samba_ad_dns_zones and samba_ad_dns_records default to empty lists. Zones are
  managed first, followed by DNS reservations and static records, before
  directory objects. Forward and reverse zones are selected by their names.
  replication accepts domain (default) or forest and applies only when creating
  a zone. Omitted aging, norefresh_interval, and refresh_interval settings
  remain unmanaged; intervals are in hours, with 0 selecting the DC default.
  Supply aging options only with state: present.
- Records support A, AAAA, CNAME, PTR, MX, NS, SRV, and TXT. name is relative to
  zone; @ means the zone root. MX requires preference; SRV requires priority,
  weight, and port. ttl defaults to 900 seconds and is updated on the existing
  record. PTR records must be declared separately.
- Removing a DNS list entry stops management and preserves the object. state:
  absent explicitly deletes it. Records are managed individually, preserving
  other values at the same name. To replace a value, declare the old value
  absent and the new value present. Deleting a zone removes all its records;
  remove all separate record declarations for that zone, including absent
  entries. DNS reservations also require their domain zone to remain present.
- samba_ad_ous, samba_ad_users, and samba_ad_groups manage only listed objects.
  Removing an item stops management; state: absent explicitly deletes it. List
  OUs in parent-before-child order. Empty OUs marked absent are removed in
  reverse order after users and groups are managed. OU deletion never removes
  unlisted child objects recursively.
- All groups are created before membership is reconciled, so nested groups may
  appear in any order. Omitted members leaves membership unmanaged.
  samba_ad_group_members_purge defaults to false (additive); true makes a
  supplied members list authoritative. Each group may override members_purge. An
  empty members list removes all members only in authoritative mode.
- New users require a password supplied through a secret store.
  samba_ad_user_update_password defaults to on_create and may be overridden per
  user with update_password. always deliberately resets a supplied password on
  every run and is not idempotent. Optional user and group attributes remain
  unchanged when omitted, except enabled, scope, category, and location, which
  use the documented module defaults. Omitted path places or moves users and
  groups to the default Users container.

## Supported Platforms

| OS Family | Distribution | Version | Container Image |
| --------- | ------------ | ------- | --------------- |
| RedHat | AlmaLinux | latest | [jomrr/molecule-almalinux:latest](https://hub.docker.com/r/jomrr/molecule-almalinux) |
| Debian | Debian | latest | [jomrr/molecule-debian:latest](https://hub.docker.com/r/jomrr/molecule-debian) |
| RedHat | Fedora | latest | [jomrr/molecule-fedora:latest](https://hub.docker.com/r/jomrr/molecule-fedora) |
| Suse | OpenSuse Tumbleweed | latest | [jomrr/molecule-opensuse-tumbleweed:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-tumbleweed) |
| Debian | Ubuntu | latest | [jomrr/molecule-ubuntu:latest](https://hub.docker.com/r/jomrr/molecule-ubuntu) |

## Example Playbook

### Manage an existing domain

```yaml
---

- name: SAMBA_AD | Manage directory objects and DNS
  hosts: dc1
  gather_facts: false
  roles:
    - role: jomrr.samba_ad
      samba_ad_server: dc1.ad.example.com
      samba_ad_realm: AD.EXAMPLE.COM
      samba_ad_admin_password: "{{ vault_samba_ad_admin_password }}"

```

### Manage forward and reverse DNS

```yaml
samba_ad_dns_zones:
  - name: apps.example.com
    replication: domain
    aging: false
    norefresh_interval: 168
    refresh_interval: 168
  - name: 2.0.192.in-addr.arpa
    replication: forest
samba_ad_dns_records:
  - zone: apps.example.com
    name: mail
    type: A
    value: 192.0.2.10
    ttl: 3600
  - zone: 2.0.192.in-addr.arpa
    name: '10'
    type: PTR
    value: mail.apps.example.com
  - zone: apps.example.com
    name: '@'
    type: MX
    value: mail.apps.example.com
    preference: 10
  - zone: apps.example.com
    name: _submission._tcp
    type: SRV
    value: mail.apps.example.com
    priority: 0
    weight: 100
    port: 587
```

### Replace one DNS value

Other values at the same name remain unchanged.

```yaml
samba_ad_dns_records:
  - zone: apps.example.com
    name: mail
    type: A
    value: 192.0.2.10
    state: absent
  - zone: apps.example.com
    name: mail
    type: A
    value: 192.0.2.20
    ttl: 3600
```

### Manage directory objects

Group entries support `name`, `path`, `scope`, `category`, `description`,
`gid_number`, `members`, `members_purge`, and `state`. `scope` accepts
`global` (default), `domain_local`, or `universal`; `category` accepts
`security` (default) or `distribution`. All user and OU object options are
exposed in the role argument schema as well. Directory operations use
Administrator and `samba_ad_admin_password` against the configured DC.

```yaml
samba_ad_ous:
  - name: Staff
    path: DC=ad,DC=example,DC=com
  - name: Engineering
    path: OU=Staff,DC=ad,DC=example,DC=com
samba_ad_users:
  - username: jdoe
    path: OU=Engineering,OU=Staff,DC=ad,DC=example,DC=com
    given_name: Jane
    surname: Doe
    password: "{{ vault_jdoe_password }}"
samba_ad_groups:
  - name: engineers
    path: OU=Engineering,OU=Staff,DC=ad,DC=example,DC=com
    members: [jdoe]
  - name: announcements
    scope: universal
    category: distribution
    members: [engineers]
```

## References

- [jomrr.samba collection](https://github.com/jomrr/ansible-collection-samba)

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
