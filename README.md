# STB HG680P Ansible Automation

Automating the deployment and configuration of an HG680P Set-Top Box (STB) as a Linux-based mini server using Ansible.

This project extends the Bash automation approach by allowing the same server configuration to be managed remotely from a single control machine.

Instead of configuring each STB manually, Ansible connects to target devices over the network and executes the required configuration tasks through an Ansible playbook.

The automation covers Docker, CasaOS, Portainer, Prometheus, Node Exporter, cAdvisor, and Grafana.

> **Note:** IP addresses, hostnames, usernames, and passwords in this repository are examples only. Replace them with the actual values when deploying in your own environment. Never commit real credentials or private SSH keys to a public repository.

## Project Overview

The previous automation stage used a Bash Script to automate installation on a single STB.

Bash automation is useful when the script runs directly on the target machine. However, managing multiple devices requires another approach.

Ansible provides a centralized automation model:

```text id="d3f6fz"
                    Ansible Control Machine
                            │
                            │ SSH
                            │
                  ┌─────────┴─────────┐
                  │                   │
                  ▼                   ▼
              STB Server 1        STB Server 2
                  │                   │
                  └─────────┬─────────┘
                            │
                   Server Configuration
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
      Docker             CasaOS             Portainer
        │
        └──────────────── Monitoring ────────────
                            │
             ┌──────────────┼──────────────┐
             │              │              │
        Prometheus    Node Exporter     cAdvisor
             │
             └──────────────┐
                            ▼
                         Grafana
```

The main advantage is that the configuration can be executed from one control machine instead of logging into every target device and repeating the installation manually.

## Prerequisites

The target STB should already have:

* HG680P hardware
* Armbian Linux installed
* Network connectivity
* SSH access
* A Linux user with `sudo` privileges

The control machine requires:

* Ansible
* SSH client
* Network access to the target STB

The target environment used by this project is Ubuntu-based Armbian.

## Project Structure

```text id="e5kn6c"
stb-hg680p-ansible/
├── README.md
├── inventory.ini
└── install.yml
```

### `inventory.ini`

Defines the target devices managed by Ansible.

### `install.yml`

Contains the Ansible playbook and tasks used to configure the target server.

## Inventory

Ansible starts by identifying the target machines through an inventory file.

Example:

```ini id="q8x5ph"
[targets]

stb-server ansible_host=192.168.10.100 ansible_user=user
```

The values mean:

* `targets` — Ansible host group.
* `stb-server` — logical name of the target.
* `ansible_host` — target IP address.
* `ansible_user` — Linux user used for SSH.

Replace the example values with the configuration of the target environment.

For multiple STBs, additional hosts can be added:

```ini id="r7h0s9"
[targets]

stb-server-01 ansible_host=192.168.10.100 ansible_user=user
stb-server-02 ansible_host=192.168.10.101 ansible_user=user
```

The same playbook can then be executed against the entire `targets` group.

## Test Ansible Connectivity

Before running the installation playbook, verify that Ansible can connect to the target.

```bash id="u7yq1v"
ansible targets -i inventory.ini -m ping
```

A successful response should look similar to:

```text id="8f7y2s"
stb-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

This confirms that the Ansible control machine can communicate with the target through SSH.

## Ansible Playbook

The main automation file is:

```text id="w5x6u1"
install.yml
```

The playbook targets the `targets` inventory group and uses privilege escalation:

```yaml id="x3h8k4"
---
- name: STB Automation
  hosts: targets
  become: true
```

`become: true` allows Ansible to execute privileged tasks using `sudo`.

## Monitoring Password

The monitoring password is requested when the playbook starts:

```yaml id="r4g1k7"
vars_prompt:
  - name: monitoring_password
    prompt: "Enter monitoring password"
    private: true
```

Using `private: true` prevents the password from being displayed while it is entered.

The password is therefore not stored directly inside the playbook.

## Package Installation

The playbook updates the APT package cache:

```yaml id="t8v2p0"
- name: Update APT cache
  apt:
    update_cache: true
    cache_valid_time: 3600
```

Required packages are then installed:

```yaml id="s3c9w5"
- name: Install required packages
  apt:
    name:
      - ca-certificates
      - curl
      - gnupg
      - python3
      - python3-bcrypt
    state: present
```

These packages provide the dependencies required by the Docker installation and monitoring authentication configuration.

## Docker Installation

The playbook creates the Docker keyring directory:

```yaml id="n8k2m6"
- name: Create Docker keyring directory
  file:
    path: /etc/apt/keyrings
    state: directory
    mode: '0755'
```

It then downloads the Docker GPG key:

```yaml id="p2d7r4"
- name: Add Docker GPG key
  get_url:
    url: https://download.docker.com/linux/ubuntu/gpg
    dest: /etc/apt/keyrings/docker.asc
    mode: '0644'
```

The Docker repository is added dynamically based on the target architecture and Ubuntu release.

The required Docker components are then installed:

```text id="m7v4q9"
docker-ce
docker-ce-cli
containerd.io
docker-buildx-plugin
docker-compose-plugin
```

Finally, the Docker service is enabled and started:

```yaml id="f5j8k2"
- name: Enable Docker
  service:
    name: docker
    state: started
    enabled: true
```

## CasaOS

The playbook checks whether CasaOS is already installed:

```yaml id="a9w3c6"
- name: Check CasaOS
  stat:
    path: /usr/bin/casaos
  register: casaos_binary
```

CasaOS is installed only when the binary does not already exist:

```yaml id="b6x1r8"
- name: Install CasaOS
  shell: curl -fsSL https://get.casaos.io | bash
  when: not casaos_binary.stat.exists
```

This prevents the installer from being executed unnecessarily when CasaOS is already present.

## Portainer

The playbook creates the Portainer directory:

```yaml id="k4q9z2"
- name: Create Portainer directory
  file:
    path: /opt/portainer
    state: directory
    mode: '0755'
```

It then creates a Docker Compose configuration:

```yaml id="c8m5p1"
services:
  portainer:
    image: portainer/portainer-ce:lts
    container_name: portainer
    restart: unless-stopped
    ports:
      - "9443:9443"
      - "8000:8000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./portainer_data:/data
```

The container is started from `/opt/portainer`:

```yaml id="z1h7v3"
- name: Start Portainer
  command:
    cmd: docker compose up -d
    chdir: /opt/portainer
```

Portainer can then be accessed through:

```text id="h6q2n8"
https://IP-STB:9443
```

## Prometheus Authentication

The playbook generates a bcrypt password hash using Python and the `bcrypt` module:

```yaml id="r9m4x7"
- name: Generate Prometheus bcrypt password
  shell: |
    python3 -c "import bcrypt, os; print(
    bcrypt.hashpw(
    os.environ['PASSWORD'].encode(),
    bcrypt.gensalt()
    ).decode())"
  environment:
    PASSWORD: "{{ monitoring_password }}"
  register: prometheus_password_hash
  changed_when: false
  no_log: true
```

Two important options are used here.

### `changed_when: false`

The task generates a value but does not modify the system state, so Ansible does not report it as a changed configuration task.

### `no_log: true`

The password and generated hash are hidden from Ansible output.

This is important because the playbook handles authentication data.

## Monitoring Directory

The monitoring directory is created at:

```text id="p8w3k6"
/opt/monitoring
```

Prometheus data is stored under:

```text id="v2c9n5"
/opt/monitoring/monitoring-data/prometheus_data
```

The directory is assigned to UID and GID `65534` for compatibility with the Prometheus container configuration.

## Prometheus

The playbook generates a Prometheus configuration with the following scrape targets:

```yaml id="y6r1t4"
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets:
          - prometheus:9090

  - job_name: node-exporter
    static_configs:
      - targets:
          - node-exporter:9100

  - job_name: cadvisor
    static_configs:
      - targets:
          - cadvisor:8080
```

The monitoring stack therefore collects metrics from:

* Prometheus
* Node Exporter
* cAdvisor

## Prometheus Web Authentication

The generated Prometheus web configuration uses the bcrypt hash:

```yaml id="u4n8k1"
basic_auth_users:
  monitor: "<GENERATED_BCRYPT_HASH>"
```

The actual hash is generated during playbook execution and should never be hard-coded into a public repository.

## Grafana

The playbook creates Grafana provisioning directories:

```text id="q3m7x9"
/opt/monitoring/grafana/provisioning/datasources
```

A Prometheus datasource is then configured automatically:

```yaml id="b8k2w6"
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    basicAuth: true
    basicAuthUser: monitor
    basicAuthPassword: "{{ monitoring_password }}"
    isDefault: true
```

This allows Grafana to use Prometheus automatically after deployment.

## Monitoring Docker Compose

The playbook creates the monitoring Compose configuration with four components:

```text id="v9x2c5"
cAdvisor
Node Exporter
Prometheus
Grafana
```

The images used are:

```text id="z7m4q1"
gcr.io/cadvisor/cadvisor:v0.47.1
prom/node-exporter:v1.7.0
prom/prometheus:v3.0.0
grafana/grafana:11.5.0
```

The exposed ports are:

| Component     | Port |
| ------------- | ---: |
| cAdvisor      | 8080 |
| Node Exporter | 9100 |
| Prometheus    | 9090 |
| Grafana       | 3000 |

The monitoring stack is started with:

```yaml id="e5r8k3"
- name: Start Monitoring
  command:
    cmd: docker compose up -d
    chdir: /opt/monitoring
```

## Run the Playbook

After the inventory and playbook have been prepared, test the connection:

```bash id="g2k7v5"
ansible targets -i inventory.ini -m ping
```

If the connection is successful, run the complete automation:

```bash id="c6p1z9"
ansible-playbook -i inventory.ini install.yml
```

Ansible will request the monitoring password before executing the tasks.

The playbook then performs the deployment in sequence:

```text id="m8q3w6"
APT Update
    ↓
Required Packages
    ↓
Docker
    ↓
CasaOS
    ↓
Portainer
    ↓
Prometheus
    ↓
Node Exporter
    ↓
cAdvisor
    ↓
Grafana
```

## Play Recap

At the end of the playbook execution, Ansible displays a `PLAY RECAP`.

Example:

```text id="t9x4k2"
PLAY RECAP

stb-server : ok=21 changed=3 unreachable=0 failed=0 skipped=1
```

The most important values are:

* `ok` — tasks completed successfully.
* `changed` — tasks that changed the target system.
* `unreachable` — hosts that could not be contacted.
* `failed` — tasks that failed.
* `skipped` — tasks that were intentionally skipped.

A `failed=0` result indicates that no task failed during the playbook execution.

## Verify the Deployment

After the playbook completes, connect to the target STB and check the running containers:

```bash id="x7m2q5"
docker ps
```

The expected containers include:

```text id="p4v8n1"
portainer
cadvisor
node-exporter
prometheus
grafana
```

All containers should report a running status.

## Access the Services

Replace `IP-STB` with the actual target address.

| Service    | Access                |
| ---------- | --------------------- |
| CasaOS     | `http://IP-STB`       |
| Portainer  | `https://IP-STB:9443` |
| Prometheus | `http://IP-STB:9090`  |
| Grafana    | `http://IP-STB:3000`  |

## Automation Model

The overall automation model is:

```text id="n6x3r8"
              Ansible Control Machine
                       │
                       │ SSH
                       ▼
                 Target STB
                       │
          ┌────────────┼────────────┐
          │            │            │
       Docker        CasaOS     Portainer
          │
          └──────── Monitoring ────────┐
                                      │
                ┌─────────────────────┼─────────────────┐
                │                     │                 │
           Prometheus           Node Exporter       cAdvisor
                │
                ▼
             Grafana
```

This approach allows one control machine to manage one or more STBs using the same playbook.

## Bash vs Ansible

Bash and Ansible serve different purposes in the automation workflow.

### Bash

Bash is useful for:

* Local automation
* Sequential installation
* Single-server provisioning
* Custom shell commands

The previous project used Bash to automate the setup of one STB.

### Ansible

Ansible is useful for:

* Remote configuration
* Multiple target machines
* Repeatable provisioning
* Centralized management
* Configuration consistency

The Bash implementation therefore becomes the foundation for a more scalable Ansible-based approach.

## Security Considerations

Because this repository is public, do not commit:

* Real IP addresses
* SSH private keys
* Passwords
* API tokens
* Production credentials
* Private infrastructure information

The inventory should contain example values only.

The monitoring password is requested interactively through `vars_prompt`.

Sensitive Ansible task output is hidden using:

```yaml
no_log: true
```

Even so, always review the playbook and generated configuration before deploying it in a production environment.

## Result

After successful execution, the HG680P STB is configured as a Linux-based mini server with:

* Docker
* Docker Compose
* CasaOS
* Portainer
* Prometheus
* Node Exporter
* cAdvisor
* Grafana

The important difference from the previous Bash implementation is that the deployment can be controlled remotely from one Ansible control machine.

This makes the setup easier to repeat across multiple compatible STB devices.

## Next Stage

This Ansible project completes the server automation stage.

The next step is to move from server provisioning into application deployment using container image versioning and CI/CD.

The following project focuses on deploying versioned Docker images with **Docker Compose and GitLab CI/CD**.
