<img src="img/header.jpg?raw=true" alt="Header Logo" width="100%" />
<br /><br />

# Ansible Role: `ubuntu_deb_certbot_ssl`

Safety-first Ansible role for **installing and operating Certbot** on **Debian/Ubuntu** hosts to **validate, issue, and renew** Let's Encrypt certificates using multiple challenge methods.

This role is designed to be **defensive by default**:
- Default mode is **`validate`** (preflight only; no issuance).
- Certificate issuance is **disabled** by default to avoid accidental rate limits.
- Production issuance requires an explicit **allow** switch.

---

## Key features

- **Preflight / validate mode**: run DNS and config checks without issuing certificates.
- **Multiple challenge methods** (depending on your host and environment):
  - DNS-01 (cloud DNS providers)
  - HTTP-01 `standalone`
  - HTTP-01 `http + webroot`
  - Apache plugin (`--apache`)
- **Declarative certificate list**: manage multiple certificates with a single `certbot_certs` structure.
- **Safety gates** to prevent accidental production issuance.
- **Cloud-first DNS-01 design**:
  - **GCP Cloud DNS** supported via **VM-attached Service Account** (no JSON credentials stored in Terraform state).
  - Planned evolution for AWS Route 53 and additional providers.

---

## Supported platforms

| OS Family | Versions (typical) |
|---|---|
| Ubuntu | 20.04 (focal), 22.04 (jammy), 24.04 (noble) |
| Debian | 11 (bullseye), 12 (bookworm) |

> The role validates `ansible_facts.os_family == "Debian"` (covers Debian and Ubuntu).

---

## DNS provider support status

| Provider | DNS-01 automation status | Notes |
|---|---|---|
| Google Cloud DNS (GCP) | Supported | Uses VM-attached Service Account (`auth_method: gcloud`) |
| AWS Route 53 | Not supported (planned) | Roadmap item; not implemented/validated yet |
| Cloudflare | Not supported (planned) | Roadmap item; not implemented/validated yet |s

---

## Compatibility matrix (challenge methods)

| Method | Challenge | Use case | Notes |
|---|---|---|---|
| `dns` | DNS-01 | No inbound HTTP required; ideal for private hosts | Requires DNS provider access |
| `standalone` | HTTP-01 | Simple HTTP challenge on port 80 | Needs port 80 reachable and free |
| `http` + `webroot` | HTTP-01 | Existing webserver (Apache/Nginx) serves `/.well-known` | Needs correct webroot and routing |
| `apache` | HTTP-01 | Apache-managed vhosts and automation | Requires Apache installed and reachable |

> **Currently in active cloud use:** DNS-01 with **Google Cloud DNS** via `gcloud` auth (Service Account attached to VM).  
> Other providers (Route 53, Cloudflare, etc.) are structured for future enablement.

---

## Repository layout

Below is the **project layout** used in the DevOps monorepo. It shows where inventories, playbooks and this role live, so the command examples make sense.

```
config-mgmt/ansible
.
├── ansible.cfg
├── .ansible
│   ├── collections/
│   │   └── ansible_collections/
│   └── roles/
│       └── jhonnygo.ubuntu_deb_certbot_ssl/
│           ├── CHANGELOG
│           ├── LICENSE
│           ├── README.md
│           ├── defaults/...
│           ├── img/...
│           ├── meta/...
│           ├── tasks/
│           │   ├── 01-validate.yml
│           │   ├── 02-install.yml
│           │   ├── 03-apache_preflight.yml
│           │   ├── 03-dns_preflight.yml
│           │   ├── 03-http_preflight.yml
│           │   ├── 04-hooks.yml
│           │   ├── 05-issue.yml
│           │   ├── issue_one.yml
│           │   └── main.yml
│           └── templates/...
│
├── inventories/
│   └── gcp/
│       ├── dev/
│       │   ├── gcp_compute.yml
│       │   ├── group_vars/
│       │   │   ├── all.yml
│       │   │   └── component_app.yml
│       │   └── host_vars/
│       │       └── vm-dev-01.yml
|       ├── stage/...
|       |
│       └── prod/...
│
├── playbooks/
│   ├── certbot-ssl.yml
│   └── ping.yml
└── requirements/
    ├── collections.yml
    └── roles.yml
```

---

## Installation

### Option A: Install from Ansible Galaxy

```bash
ansible-galaxy role install jhonnygo.ubuntu_deb_certbot_ssl
```

### Option B: Install from Git (recommended during development)

Repository:

```text
https://github.com/jhonnygo/ansible-role-ubuntu-deb-certbot-ssl.git
```

Install:

```bash
ansible-galaxy role install git+https://github.com/jhonnygo/ansible-role-ubuntu-deb-certbot-ssl.git
```

Or pin a branch/tag:

```bash
ansible-galaxy role install git+https://github.com/jhonnygo/ansible-role-ubuntu-deb-certbot-ssl.git,main
```

---

## Recommended usage layout (example)

This README documents the same layout used in your repo structure (trimmed to what matters for this role):

```text
ansible/
  playbooks/
    certbot-ssl.yml
  inventories/
    gcp/
      dev/
        gcp_compute.yml
        group_vars/
          component_app.yml
```

---

## Playbook example: `playbooks/certbot-ssl.yml`

A minimal playbook that targets the group `component_app` and runs the role:

```yaml
- name: DNS preflight (no certbot issuance)
  hosts: component_app
  gather_facts: true
  become: true
  serial: 1

  roles:
    - role: jhonnygo.ubuntu_deb_certbot_ssl
```

---

## Inventory example (GCP dynamic inventory)

`inventories/gcp/dev/gcp_compute.yml`:

```yaml
plugin: google.cloud.gcp_compute

# Adds a prefix to host variables
vars_prefix: "gcp_"

projects:
  - "your-gcp-project"

auth_kind: application

zones:
  - europe-west1-b

# Filter instances by labels
filters:
  - labels.environment = dev
  - status = RUNNING

# Auto-generate groups by labels
keyed_groups:
  - key: labels.environment
    prefix: env
    separator: "_"
  - key: labels.role
    prefix: role
    separator: "_"
  - key: labels.component
    prefix: component
    separator: "_"

compose:
  ansible_host: networkInterfaces[0].accessConfigs[0].natIP
```

### How to run

From the `ansible/` directory:

```bash
ansible-playbook -i inventories/gcp/dev/gcp_compute.yml playbooks/certbot-ssl.yml
```

---

## Configuration example: `group_vars/component_app.yml`

This file drives the role behavior. The defaults below represent a **safe preflight-only setup**:

```yaml
# Modes: validate | staging | production
certbot_mode: validate

# Safety: default to never issue
certbot_issue_enabled: false

# Extra safety: block production unless explicitly allowed
certbot_allow_production: false

# DNS defaults (Google Cloud DNS)
certbot_dns:
  google:
    project: "your-gcp-projct-id"   # Where live DNS
    zone: "domain-com"              # Managed Zone NAME (not the DNS name)
    auth_method: "gcloud"           # gcloud | json
    propagation_seconds: 30

# Declarative certificates to manage
certbot_certs: []
```

---

## Core variables

### Global behavior

| Variable | Type | Default | Description |
|---|---:|---:|---|
| `certbot_mode` | string | `validate` | `validate` (no issuance), `staging`, or `production` |
| `certbot_issue_enabled` | bool | `false` | Must be `true` to issue/renew |
| `certbot_allow_production` | bool | `false` | Must be `true` if issuing in `production` |
| `certbot_email` | string | (required when issuing) | Email required for issuance |
| `certbot_force_renew` | bool | `false` | Force renewal even if not due |
| `certbot_skip_issue_if_present` | bool | `true` | Skip issuance if cert already exists |
| `certbot_dns_preflight_enabled` | bool | `true` | Enables safe DNS preflight checks |

### Crypto defaults

| Variable | Default | Notes |
|---|---|---|
| `certbot_key_type` | `ecdsa` | `ecdsa` or `rsa` |
| `certbot_ecdsa_curve` | `secp256r1` | ECDSA curve when `ecdsa` |
| `certbot_rsa_key_size` | `4096` | RSA size when `rsa` |

---

## Certificate declaration: `certbot_certs`

`certbot_certs` is a list of certificate definitions. Each item typically includes:

- `name`: logical identifier (used for filenames/logging)
- `method`: `dns` / `standalone` / `http` / `apache`
- `domains`: list of domain names for the certificate

Depending on `method`, additional keys apply (examples below).

### Example 1: DNS-01 (GCP Cloud DNS with gcloud)

```yaml
certbot_certs:
  - name: "dns-dev-domain-com"
    method: "dns"
    dns_provider: "google"
    dns_auth_method: "gcloud"
    domains:
      - "dev.domain.com"
```

### Example 2: Standalone HTTP-01

```yaml
certbot_certs:
  - name: "standalone-dev-domain-com"
    method: "standalone"
    domains:
      - "dev.domain.com"
```

### Example 3: HTTP-01 via webroot

```yaml
certbot_certs:
  - name: "http-dev-domain-com"
    method: "http"
    http_auth: "webroot"
    webroot: "/var/www/html"
    domains:
      - "dev.domain.com"
```

### Example 4: Apache plugin

```yaml
certbot_certs:
  - name: "apache-dev-domain-com"
    method: "apache"
    domains:
      - "dev.domain.com"
```

---

## GCP DNS-01 prerequisites (current supported cloud DNS provider)

At this time, the role supports DNS-01 automation with **Google Cloud DNS** using
`auth_method: gcloud` and a **VM-attached Service Account**. This approach avoids
storing Service Account JSON keys on disk or inside Terraform state (security best practice).

AWS Route 53 and other DNS providers are part of the planned roadmap but are **not
supported** for automated DNS-01 operations in the current version.

### Why avoid JSON credentials?
Embedding a Service Account JSON key into IaC (e.g., Terraform state) is a common security risk. Instead, this design assumes:
- The target VM has a **Service Account attached**
- The Service Account has the minimal permissions required to update Cloud DNS

### Required permissions (high level)
Grant the attached Service Account permissions to manage DNS records in the relevant Cloud DNS zone (principle of least privilege).

### Runtime expectation
The VM must be able to authenticate via the metadata server and execute `gcloud` operations (or equivalent provider tooling) as required by the DNS plugin workflow.

---

## Safety model (important)

This role is intentionally strict:

1. **`certbot_mode: validate` never issues.**
2. **`certbot_issue_enabled: false` by default** (anti rate-limit safety).
3. **Production issuance requires explicit approval**:
   - `certbot_issue_enabled: true`
   - `certbot_mode: production`
   - `certbot_allow_production: true`

If you enable issuance, you must also set a valid `certbot_email`.

---

## Typical workflows

### 1) DNS preflight only (safe)
```yaml
certbot_mode: validate
certbot_issue_enabled: false
certbot_dns_preflight_enabled: true
```

### 2) Staging issuance (recommended before production)
```yaml
certbot_mode: staging
certbot_issue_enabled: true
certbot_email: "you@example.com"
```

### 3) Production issuance (explicit allow required)
```yaml
certbot_mode: production
certbot_issue_enabled: true
certbot_allow_production: true
certbot_email: "you@example.com"
```

---

## Testing and verification checklist

| Check | DNS | Standalone | Webroot | Apache |
|---|:--:|:--:|:--:|:--:|
| DNS propagation preflight | ✅ | N/A | N/A | N/A |
| Requires inbound port 80 | ❌ | ✅ | ✅ | ✅ |
| Works behind private networks | ✅ | ❌ (usually) | ❌ (usually) | ❌ (usually) |
| Requires existing web server | ❌ | ❌ | ✅ | ✅ (Apache) |

---

## Troubleshooting

- **Role says Debian/Ubuntu only**: confirm `ansible_facts.os_family` is `Debian`.
- **Staging/Production blocked**: verify `certbot_issue_enabled` and `certbot_allow_production` (production only).
- **DNS-01 fails on GCP**:
  - Confirm the VM Service Account has the required Cloud DNS permissions.
  - Confirm the Cloud DNS zone name (`zone`) is the *Managed Zone NAME* (not the domain string).
  - Increase `propagation_seconds` for slow DNS propagation.

---

## License

MIT-0 (see `LICENSE`).

---

## Author / Maintainer

Jhonny Gomez (JhonnyGO)  
Company: Jhoncy Tech  
Contact: `soporte@jhoncytech.com`

<img src="img/happy-coding.jpg?raw=true" alt="Footer Logo" />