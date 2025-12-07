# Lab 14: DNS High Availability - Implementation Plan

## Overview

This plan implements DNS high availability with primary/secondary BIND9 servers, TSIG-authenticated zone transfers, and dynamic DNS updates using nsupdate.

## Current State Analysis

- **DNS Server**: Only `orinocoz-2` runs BIND9 (single point of failure)
- **Zone File**: Static template `db.master.j2` deployed directly
- **Resolv.conf**: Already iterates over `dns_servers` group (supports multiple DNS)
- **Prometheus**: Already scrapes `dns_servers` group for BIND metrics

---

## Task 1: Generate DNS Keys (TSIG)

### Files to Create/Modify:
- `group_vars/all.yaml` - Add vaulted secrets for `transfer_key` and `nsupdate_key`

### Steps:
1. Generate transfer key: `tsig-keygen transfer.key`
2. Generate nsupdate key: `tsig-keygen nsupdate.key`
3. Store both key secrets in Ansible Vault in `group_vars/all.yaml`:
   ```yaml
   dns_transfer_key: !vault |
     ...encrypted secret...
   dns_nsupdate_key: !vault |
     ...encrypted secret...
   ```

---

## Task 2: Create Secondary DNS with TSIG Authentication

### Files to Modify:

#### `hosts` - Add DNS subgroups:
```ini
[dns_servers]
orinocoz-2
orinocoz-1   # Add secondary DNS

[dns_primary]
orinocoz-2

[dns_secondary]
orinocoz-1
```

#### `roles/bind/templates/named.conf.options.j2` - Add transfer key:
```jinja2
key "transfer.key" {
  algorithm hmac-sha256;
  secret "{{ dns_transfer_key }}";
};
```

#### `roles/bind/templates/named.conf.local.j2` - Conditional primary/secondary:
```jinja2
{% if inventory_hostname in groups['dns_primary'] %}
zone "{{ startup_name }}" {
  type primary;
  file "db.{{ startup_name }}";
  allow-transfer { key "transfer.key"; };
};
{% elif inventory_hostname in groups['dns_secondary'] %}
zone "{{ startup_name }}" {
  type secondary;
  file "db.{{ startup_name }}";
  primaries { {{ hostvars[groups['dns_primary'][0]]['ansible_default_ipv4']['address'] }} key "transfer.key"; };
};
{% endif %}
```

#### `roles/bind/tasks/main.yaml` - Conditional zone deployment:
- Zone file only deployed on primary (using `when: inventory_hostname in groups['dns_primary']`)

---

## Task 3: Update /etc/resolv.conf

### Files to Modify:

#### `roles/resolve/templates/resolv.conf.j2` - Already supports multiple DNS:
Current template already iterates over `dns_servers` group - no changes needed once hosts file is updated.

Verify output will be:
```
nameserver <primary_ip>
nameserver <secondary_ip>
search acme.ttu
```

---

## Task 4: Rewrite Bind Role for Dynamic Updates (nsupdate)

### Files to Create/Modify:

#### `roles/bind/templates/named.conf.options.j2` - Add nsupdate key:
```jinja2
key "nsupdate.key" {
  algorithm hmac-sha256;
  secret "{{ dns_nsupdate_key }}";
};
```

#### `roles/bind/templates/named.conf.local.j2` - Allow dynamic updates:
```jinja2
zone "{{ startup_name }}" {
  type primary;
  file "db.{{ startup_name }}";
  allow-transfer { key "transfer.key"; };
  allow-update { key "nsupdate.key"; };
};
```

#### `roles/bind/templates/db.master.j2` - Minimal zone file (SOA + NS only):
```jinja2
$TTL 86400
@   IN  SOA ns-1.{{ startup_name }}. admin.{{ startup_name }}. (
            {{ ansible_date_time.epoch }}   ; Serial
       604800   ; Refresh
        86400   ; Retry
      2419200   ; Expire
       604800 ) ; Negative Cache TTL
;
{% for dns_server in groups['dns_servers'] %}
@       IN  NS  {{ dns_server }}.{{ startup_name }}.
{% endfor %}
{% for dns_server in groups['dns_servers'] %}
{{ dns_server }}    IN  A   {{ hostvars[dns_server]['ansible_default_ipv4']['address'] }}
{% endfor %}
```

#### `roles/bind/tasks/main.yaml` - Deploy zone file only if missing:
```yaml
- name: Configure initial zone file (primary only, if not exists)
  ansible.builtin.template:
    src: db.master.j2
    dest: /var/cache/bind/db.{{ startup_name }}
    force: no  # Don't overwrite if exists
  when: inventory_hostname in groups['dns_primary']
  notify: Reload rndc

- name: Add backup server A record
  community.general.nsupdate:
    key_name: "nsupdate.key"
    key_secret: "{{ dns_nsupdate_key }}"
    key_algorithm: "hmac-sha256"
    server: "{{ hostvars[groups['dns_primary'][0]]['ansible_default_ipv4']['address'] }}"
    zone: "{{ startup_name }}"
    record: "backup"
    type: "A"
    value: "{{ backup_server_ip }}"
  when: inventory_hostname in groups['dns_primary']
```

---

## Task 5: Create Service CNAME Records

### Files to Modify:

Create nsupdate tasks at the end of respective roles:

#### `roles/mysql/tasks/main.yaml` - Add at end:
```yaml
- name: Create db CNAME records
  community.general.nsupdate:
    key_name: "nsupdate.key"
    key_secret: "{{ dns_nsupdate_key }}"
    key_algorithm: "hmac-sha256"
    server: "{{ hostvars[groups['dns_primary'][0]]['ansible_default_ipv4']['address'] }}"
    zone: "{{ startup_name }}"
    record: "db-{{ ansible_loop.index }}"
    type: "CNAME"
    value: "{{ item }}.{{ startup_name }}."
  loop: "{{ groups['db_servers'] }}"
  loop_control:
    extended: yes
  run_once: true
  delegate_to: localhost
```

#### Similar CNAME records for other roles:
- `roles/grafana/tasks/main.yaml` → `grafana` CNAME
- `roles/loki/tasks/main.yaml` → `loki` CNAME
- `roles/prometheus/tasks/main.yaml` → `prometheus` CNAME
- `roles/haproxy/tasks/main.yaml` → `lb-1`, `lb-2` CNAMEs
- `roles/bind/tasks/main.yaml` → `ns-1`, `ns-2` CNAMEs
- `roles/agama/tasks/main.yaml` → `www-1`, `www-2` CNAMEs

#### Update service configurations to use CNAMEs:
- `roles/grafana/templates/*.j2` - Use `prometheus.{{ startup_name }}` instead of hostname
- `roles/prometheus/templates/config.yaml.j2` - Use CNAMEs for targets
- `roles/agama/templates/*.j2` - Use `db-1.{{ startup_name }}` for MySQL

---

## Task 6: Create PTR Records (Reverse DNS)

### Files to Create/Modify:

#### `roles/bind/templates/named.conf.local.j2` - Add reverse zone:
```jinja2
{% if inventory_hostname in groups['dns_primary'] %}
zone "168.192.in-addr.arpa" {
  type primary;
  file "db.168.192";
  allow-transfer { key "transfer.key"; };
  allow-update { key "nsupdate.key"; };
};
{% elif inventory_hostname in groups['dns_secondary'] %}
zone "168.192.in-addr.arpa" {
  type secondary;
  file "db.168.192";
  primaries { {{ hostvars[groups['dns_primary'][0]]['ansible_default_ipv4']['address'] }} key "transfer.key"; };
};
{% endif %}
```

#### `roles/bind/templates/db.reverse.j2` - Create reverse zone template:
```jinja2
$TTL 86400
@   IN  SOA ns-1.{{ startup_name }}. admin.{{ startup_name }}. (
            {{ ansible_date_time.epoch }}   ; Serial
       604800   ; Refresh
        86400   ; Retry
      2419200   ; Expire
       604800 ) ; Negative Cache TTL
;
{% for dns_server in groups['dns_servers'] %}
@       IN  NS  {{ dns_server }}.{{ startup_name }}.
{% endfor %}
{% for dns_server in groups['dns_servers'] %}
{{ dns_server }}    IN  A   {{ hostvars[dns_server]['ansible_default_ipv4']['address'] }}
{% endfor %}
```

#### `roles/bind/tasks/main.yaml` - Add PTR record creation:
```yaml
- name: Create PTR records for all hosts
  community.general.nsupdate:
    key_name: "nsupdate.key"
    key_secret: "{{ dns_nsupdate_key }}"
    key_algorithm: "hmac-sha256"
    server: "{{ hostvars[groups['dns_primary'][0]]['ansible_default_ipv4']['address'] }}"
    zone: "168.192.in-addr.arpa"
    record: "{{ hostvars[item]['ansible_default_ipv4']['address'].split('.')[3] }}.{{ hostvars[item]['ansible_default_ipv4']['address'].split('.')[2] }}"
    type: "PTR"
    value: "{{ item }}.{{ startup_name }}."  # Note trailing dot!
  loop: "{{ groups['all'] }}"
  when: inventory_hostname in groups['dns_primary']
```

---

## Task 7: Grafana Dashboard for Secondary DNS

### Files to Modify:

#### `roles/prometheus/templates/config.yaml.j2`:
Already iterates over `dns_servers` group - will automatically pick up secondary once added to hosts.

#### `roles/grafana/files/main.json` (or create `dns.json`):
Add panels for secondary DNS metrics:
- `bind_resolver_queries_total` per server
- `bind_zone_transfer_success_total`
- `bind_up` for both DNS servers

---

## Post Task: Create name.txt

### Files to Create:
```
/home/doc/ica0002/name.txt
```
Content format:
```
Real Name:github_username:discord_username
```

---

## Implementation Order

1. **Task 1**: Generate TSIG keys and add to vault
2. **Task 2**: Update hosts file, modify bind role for primary/secondary
3. **Task 3**: Verify resolv.conf (should work automatically)
4. **Task 4**: Rewrite bind role for nsupdate dynamic records
5. **Task 5**: Add CNAME records to each service role
6. **Task 6**: Add reverse zone and PTR records
7. **Task 7**: Update Grafana dashboard
8. **Post**: Create name.txt

---

## File Change Summary

| File | Action | Description |
|------|--------|-------------|
| `hosts` | Modify | Add dns_primary, dns_secondary groups |
| `group_vars/all.yaml` | Modify | Add dns_transfer_key, dns_nsupdate_key (vaulted) |
| `roles/bind/templates/named.conf.options.j2` | Modify | Add TSIG keys |
| `roles/bind/templates/named.conf.local.j2` | Modify | Primary/secondary zones + reverse zone |
| `roles/bind/templates/db.master.j2` | Modify | Minimal zone (SOA + NS only) |
| `roles/bind/templates/db.reverse.j2` | Create | Reverse zone template |
| `roles/bind/tasks/main.yaml` | Modify | Conditional deployment + nsupdate tasks |
| `roles/mysql/tasks/main.yaml` | Modify | Add db CNAME records |
| `roles/grafana/tasks/main.yaml` | Modify | Add grafana CNAME |
| `roles/loki/tasks/main.yaml` | Modify | Add loki CNAME |
| `roles/prometheus/tasks/main.yaml` | Modify | Add prometheus CNAME |
| `roles/haproxy/tasks/main.yaml` | Modify | Add lb CNAMEs |
| `roles/agama/tasks/main.yaml` | Modify | Add www CNAMEs |
| `roles/grafana/files/main.json` | Modify | Add secondary DNS panels |
| `name.txt` | Create | Identification file |

---

## Verification Commands

After implementation, verify with:
```bash
# Test zone transfer from secondary
dig @<secondary_ip> acme.ttu AXFR

# Test resolution from secondary
dig @<secondary_ip> orinocoz-1.acme.ttu

# Test CNAME resolution
dig @<primary_ip> grafana.acme.ttu

# Test reverse lookup
dig @<primary_ip> -x <host_ip>

# Test HA (stop primary, verify secondary works)
ansible orinocoz-2 -m service -a "name=bind9 state=stopped" --become
dig @<secondary_ip> orinocoz-1.acme.ttu
```
