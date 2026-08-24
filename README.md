# HA Tutorial, Ansible Playbooks

Ansible automation for the [CSC Pouta High Availability tutorial](https://docs.csc.fi/cloud/pouta/tutorials/high-availability/).

Sets up a five-VM stack on cPouta:

| VM | Role |
|----|------|
| HAProxy-1 | Primary load balancer, jump host, floating-IP holder |
| HAProxy-2 | Backup load balancer (Keepalived VRRP failover) |
| Frontend-1 | Flask application node |
| Frontend-2 | Flask application node |
| Monitoring | Prometheus + Grafana |

---

## Prerequisites

- Ansible ≥ 2.14 on your local machine
- OpenStack credentials sourced (`source <project>-openrc.sh`)
- An SSH key pair registered in Pouta, with the private key available locally
- Python `openstackclient` available locally (for ad-hoc queries)
- A working Database. We recommend [Pukki DBAAS](https://docs.csc.fi/cloud/dbaas/), but any postgres database will work. you need to record the following data base connection information: `db_host`, `db_port`, `db_user`, `db_password` and `db_name`.

Install the required Ansible collection:

```bash
ansible-galaxy collection install -r requirements.yml
```

---

## Configure variables

Edit `group_vars/all.yml` and replace every `REPLACE_WITH_*` placeholder:

| Variable | Description |
|----------|-------------|
| `project_cidr` | Your Pouta project network CIDR (e.g. `192.168.1.0/24`) |
| `keepalived_auth_pass` | Shared VRRP password (choose any strong password) |
| `os_application_credential_id` / `os_application_credential_secret` | OpenStack application credentials for the failover script |
| `db_host` / `db_password` | Pukki DBaaS connection details |

**Note**: The floating ip id will be replaced in step 2.

### Local variables

Copy `local.yml.example` to `local.yml` and fill in your personal values:

```bash
cp local.yml.example local.yml
```

| Variable | Description |
|----------|-------------|
| `key_name` | SSH key pair name as registered in the Pouta dashboard |
| `network` | Your project's internal network name |

`local.yml` is gitignored and never committed — each user keeps their own copy.

---

## Provision infrastructure

```bash
ansible-playbook create_infra.yml
```

It reads `key_name` and `network` from `local.yml` (see above). When it finishes it prints a summary like:

```
inventory.ini
  haproxy1  ansible_host=<FLOATING_IP>
  haproxy2  ansible_host=<HAPROXY_2_PRIVATE_IP>
  frontend1 ansible_host=<FRONTEND_1_PRIVATE_IP>
  frontend2 ansible_host=<FRONTEND_2_PRIVATE_IP>
  monitoring ansible_host=<MONITORING_PRIVATE_IP>

group_vars/all.yml
  frontend_1_ip: <FRONTEND_1_PRIVATE_IP>
  frontend_2_ip: <FRONTEND_2_PRIVATE_IP>
  floating_ip_id: (run: openstack floating ip list)
```

Get the floating IP UUID:

```bash
openstack floating ip list
```

Edit `group_vars/all.yml` and replace every `REPLACE_WITH_*` placeholder:

| Variable | Description |
|----------|-------------|
| `floating_ip_id` | UUID of the floating IP (see Step 2) |

HAProxy-1 acts as the SSH jump host for all other VMs. The `ProxyJump` settings in `inventory.ini` handle this automatically.

---

## Configure the stack

```bash
ansible-playbook -i inventory.ini site.yml
```

This configures all five VMs in three plays:

1. **HAProxy play**, installs HAProxy, Keepalived, and the OpenStack CLI; deploys the load-balancer config and the VRRP failover script.
2. **Frontend play**, clones the [rahti-ha-tutorial](https://github.com/CSCfi/rahti-ha-tutorial) Flask app and runs it as a systemd service.
3. **Monitoring play**, installs Prometheus and Grafana.

---

## Teardown

To destroy all provisioned resources, set `state: absent` in `group_vars/all.yml` and re-run:

```bash
ansible-playbook -i inventory.ini create_infra.yml
```

---

## File reference

```
.
├── create_infra.yml       # Provision VMs, security groups, and floating IP
├── site.yml               # Configure all VMs
├── inventory.ini          # Host list and SSH settings (auto-generated)
├── requirements.yml       # Ansible collection dependencies
├── local.yml.example      # Template for personal variables (copy to local.yml)
├── group_vars/
│   └── all.yml            # Shared variables (fill in REPLACE_* values)
└── templates/
    ├── haproxy.cfg.j2     # HAProxy load-balancer config
    ├── keepalived.conf.j2 # VRRP config (master/backup priority)
    ├── failover.sh.j2     # Keepalived notify script (reassigns floating IP)
    ├── clouds.yaml.j2     # OpenStack credentials for the failover script
    ├── ha-tutorial.env.j2 # Flask app environment variables
    ├── ha-tutorial.service.j2  # systemd unit for the Flask app
    └── prometheus.yml.j2  # Prometheus scrape config
```
