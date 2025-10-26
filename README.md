# Grafana infrastructure automation

This repository provides an Ansible-based workflow to provision Grafana on Amazon EC2 instances, configure it to serve the dashboard on port 80, and apply opinionated best practices for secure, repeatable deployments. A GitHub Actions workflow is included to lint the infrastructure-as-code assets.

## Repository layout

```
ansible/
├── ansible.cfg              # Ansible configuration tuned for this project
├── inventory/
│   └── hosts.ini            # Example inventory to target EC2 instances
├── playbooks/
│   └── site.yml             # Entry-point playbook
└── roles/
    └── grafana/             # Role that installs and configures Grafana
        ├── defaults/main.yml
        ├── handlers/main.yml
        ├── tasks/main.yml
        └── templates/*.j2
.github/workflows/ci.yml     # Linting workflow (ansible-lint + yamllint)
.yamllint                    # Formatting rules for YAML files
```

## Prerequisites

* An EC2 instance running Ubuntu 24.04 LTS (tested) or a compatible distribution such as Amazon Linux 2 / other Red Hat family releases.
* Inbound TCP/80 allowed in the associated security group.
* SSH access to the instance (update `inventory/hosts.ini` accordingly).
* Local workstation with Ansible 2.15+ or execute via GitHub Actions workflow.

## Usage

1. **Clone and configure inventory**
   ```bash
   cp ansible/inventory/hosts.ini ansible/inventory/hosts.local.ini
   # edit hosts.local.ini to reference your EC2 instance and SSH key
   ```

2. **Review and override variables**
   * Default variables live in `group_vars/all/grafana.yml` and `roles/grafana/defaults/main.yml`.
   * Override sensitive values (for example `grafana_admin_password`) using [Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html).

3. **Run the playbook**
   ```bash
   cd ansible
   ansible-playbook -i inventory/hosts.local.ini playbooks/site.yml
   ```

   The role performs the following:
   * Adds the official Grafana repository using modern keyring handling (required for Ubuntu 24.04) and installs the requested version.
   * Configures Grafana to listen on port 80 and, when needed, grants the binary the capability to bind to privileged ports.
   * Applies security hardening such as disabling public sign-ups and optional anonymous access.
   * Optionally opens port 80 in `firewalld` (set `configure_firewalld: false` if you rely solely on AWS security groups).
   * Supports provisioning datasources and dashboards via `grafana_datasources` and `grafana_dashboards` variables.

4. **Access Grafana**
   Browse to `http://<ec2-public-dns>/` and log in with the credentials defined in the variables file. Update the password immediately after first login.

## Continuous integration

Every push and pull request against `main` runs the **Ansible CI** workflow located at `.github/workflows/ci.yml`. The workflow:

1. Checks out the code.
2. Installs Python 3.11, Ansible, `ansible-lint`, and `yamllint`.
3. Executes `ansible-lint` against the playbooks and role.
4. Enforces YAML style with `yamllint` (configured via `.yamllint`).

## Extending the role

* Add datasources by appending dictionaries that match Grafana's provisioning schema to `grafana_datasources`.
* Drop JSON dashboard definitions into a local directory and reference them in `grafana_dashboards` to deploy them automatically.
* Customize Grafana configuration by extending `roles/grafana/templates/grafana.ini.j2` and overriding defaults as necessary.

## Security notes

* Store administrator credentials and any secrets with Ansible Vault or an external secret manager.
* Restrict SSH access to trusted networks.
* Regularly update the EC2 AMI and Grafana version by adjusting `grafana_version`.
