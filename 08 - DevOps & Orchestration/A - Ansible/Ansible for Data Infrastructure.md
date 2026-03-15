# Ansible for Data Infrastructure

> Ansible is an agentless automation tool that uses YAML-based playbooks to define infrastructure and deployment workflows. In data engineering, it orchestrates Docker builds, manages secrets, provisions cloud resources, and ensures repeatable deployments across environments.

---

## 1. Core Concepts

| Concept | Purpose |
|---|---|
| **Playbook** | A YAML file defining a sequence of plays (tasks applied to hosts) |
| **Role** | A reusable, self-contained unit of tasks, variables, templates, and handlers |
| **Task** | A single action (e.g., build an image, create a secret) |
| **Handler** | A task triggered only when notified by another task (e.g., restart a service) |
| **Inventory** | Defines which hosts Ansible targets and how to connect to them |
| **group_vars / host_vars** | Variable files scoped to host groups or individual hosts |
| **Collection** | A distribution format for roles, modules, and plugins (e.g., `google.cloud`) |

Ansible is **push-based** and **agentless** -- it connects over SSH or runs locally, requiring no daemon on the target. Every task should be **idempotent**: running it twice produces the same result.

---

## 2. Project Structure

A recommended layout for data engineering automation:

```
ansible/
  ansible.cfg                  # Runtime configuration
  requirements.yml             # Collection and role dependencies
  inventory/
    localhost.yml              # Local execution inventory
    dev.yml                    # Dev environment hosts
    prod.yml                   # Prod environment hosts
  group_vars/
    all.yml                    # Variables shared across all hosts
    dev.yml                    # Dev-specific overrides
    prod.yml                   # Prod-specific overrides
  playbooks/
    init.yml                   # Environment initialisation
    deploy.yml                 # Build + deploy pipeline
  roles/
    docker_build_push/
      defaults/main.yml        # Default variables (lowest precedence)
      tasks/
        main.yml               # Entry point -- includes sub-tasks
        build.yml              # Docker build logic
        tag.yml                # Tag for registry
        push.yml               # Push to Artifact Registry
    secrets_manager/
      tasks/main.yml           # Secret retrieval and storage
    env_variables/
      tasks/main.yml           # .env generation from templates
    docker_management/
      tasks/main.yml           # Container lifecycle
  templates/
    .env.j2                    # Jinja2 template for env files
  cache/                       # Fact caching (JSON files)
```

The `ansible.cfg` anchors defaults so commands work from the project root:

```ini
[defaults]
inventory = ./inventory/localhost.yml
host_key_checking = False
retry_files_enabled = False
gathering = smart
fact_caching = jsonfile
fact_caching_connection = ./cache
fact_caching_timeout = 86400
jinja2_native = True
```

Setting `jinja2_native = True` preserves Python types through Jinja2 rendering -- critical when passing structured data like JSON secrets between tasks.

See also: [[Terraform for Data Infrastructure]] for IaC patterns that complement Ansible.

---

## 3. Role-Based Design

Roles encapsulate a single responsibility. Each role lives under `roles/<name>/` and exposes configurable defaults.

### 3.1 Docker Build and Push Role

**`roles/docker_build_push/defaults/main.yml`** -- sensible defaults that callers can override:

```yaml
---
env: "dev"
project_id: "my-gcp-project"
region: "australia-southeast1"
repository: "etl-artifacts"
image_name: "etl_pipeline"
```

**`roles/docker_build_push/tasks/build.yml`** -- build with the `community.general.docker_image` module:

```yaml
- name: Build Docker image
  community.general.docker_image:
    name: "{{ image_name }}"
    tag: "{{ env }}"
    build:
      path: "{{ playbook_dir }}/../../"
      dockerfile: "docker/Dockerfile.ubuntu"
      pull: yes
      nocache: yes
```

**`roles/docker_build_push/tasks/tag.yml`** -- tag for GCP Artifact Registry:

```yaml
- name: Tag image for Artifact Registry
  ansible.builtin.command: >
    docker tag {{ image_name }}:{{ env }}
    {{ region }}-docker.pkg.dev/{{ project_id }}/{{ repository }}/{{ image_name }}:{{ env }}
```

**`roles/docker_build_push/tasks/push.yml`**:

```yaml
- name: Push image to GCP Artifact Registry
  ansible.builtin.command: >
    docker push {{ region }}-docker.pkg.dev/{{ project_id }}/{{ repository }}/{{ image_name }}:{{ env }}
```

A `main.yml` entry point imports these in order: build, tag, push.

See also: [[Docker & Container Patterns]]

### 3.2 Secrets Manager Role

```yaml
- name: Upload .env to GCP Secret Manager
  google.cloud.gcp_secret:
    name: "{{ gcp_credentials_secret_name }}"
    data: "{{ lookup('file', './env/.env') }}"
    project_id: "{{ gcp_project_id }}"
    state: present
```

The `state: present` pattern is idempotent -- the secret is created if missing and updated if changed.

### 3.3 Environment Variables Role

```yaml
- name: Generate .env file from template
  ansible.builtin.template:
    src: "templates/.env.j2"
    dest: "./env/.env"
```

The `template` module renders Jinja2 with the current variable context, allowing secrets and project IDs to flow into config files without hardcoding.

---

## 4. Playbook Patterns

### 4.1 Initialisation Playbook

Sets up the local environment, wires secrets, and generates config:

```yaml
---
- name: Initialize Environment
  hosts: localhost
  gather_facts: false

  vars:
    gcp_credentials_secret_name: "gcp-etl-env"
    env_file_path: "./env/.env"
    env_template: "templates/.env.j2"

  roles:
    - env_variables
    - docker_management
    - secrets_manager

  tasks:
    - name: Retrieve GCP credentials from Secret Manager
      gcp_secret_manager_secret:
        name: "{{ gcp_credentials_secret_name }}"
        gcp_project_id: "{{ gcp_project_id }}"
        state: present
      register: retrieved_secret

    - name: Decode secret payload
      set_fact:
        secret_data: "{{ retrieved_secret.secret.data | b64decode | from_json }}"

    - name: Generate .env file from template
      template:
        src: "{{ env_template }}"
        dest: "{{ env_file_path }}"
      vars:
        secret_data: "{{ secret_data }}"
```

Key patterns:
- **`register`** captures module output for downstream tasks
- **`set_fact`** creates runtime variables from dynamic data
- **Jinja2 filters** (`b64decode`, `from_json`) transform API responses inline

### 4.2 Deployment Playbook

Orchestrates infrastructure setup and image builds:

```yaml
- name: Deploy Infra + Image
  hosts: localhost
  gather_facts: no
  vars:
    env: "{{ env | default('dev') }}"
  tasks:
    - name: Setup GCP Resources
      include_tasks: playbook/setup_gcp_resources.yml

    - name: Build and Push Docker Image
      include_tasks: playbook/build_push_image.yml
```

The `default()` filter ensures the playbook works even without an explicit `--extra-vars` flag. Override at runtime:

```bash
ansible-playbook playbooks/deploy.yml -e "env=prod"
```

### 4.3 Idempotent Task Design

- Use **`state: present`** / **`state: absent`** rather than imperative shell commands
- Prefer dedicated modules (`community.general.docker_image`) over raw `shell`/`command`
- When `command` is unavoidable, add **`creates`** or **`changed_when`** to prevent unnecessary re-runs

```yaml
- name: Authenticate Docker to Artifact Registry
  ansible.builtin.command: gcloud auth configure-docker --quiet
  changed_when: false  # This command is always safe to re-run
```

---

## 5. GCP Integration

### 5.1 Required Collections

Declare dependencies in `requirements.yml`:

```yaml
collections:
  - name: google.cloud
    version: ">=3.0"
  - name: community.general
    version: "2.0.0"
```

Install with:

```bash
ansible-galaxy collection install -r requirements.yml
```

### 5.2 Common GCP Modules

| Module | Use Case |
|---|---|
| `google.cloud.gcp_compute_instance` | Provision VMs for ETL workloads |
| `google.cloud.gcp_secret` | Store/retrieve secrets in Secret Manager |
| `google.cloud.gcp_storage_bucket` | Create GCS buckets for data landing zones |
| `google.cloud.gcp_compute_network` | Set up VPC networks |

### 5.3 Authentication

GCP modules authenticate via a service account JSON key or Application Default Credentials:

```yaml
# In group_vars/all.yml
gcp_service_account_file: "/tmp/service-account.json"
gcp_project_id: "my-gcp-project-dev"
gcp_region: "us-central1"
```

For CI/CD, prefer workload identity federation over long-lived keys. See [[GCP Authentication Patterns]].

---

## 6. Docker Automation

### 6.1 Build Pipeline

The build-tag-push workflow maps cleanly to separate task files within a role:

1. **Build** -- `community.general.docker_image` with `nocache: yes` for reproducible builds
2. **Tag** -- apply the Artifact Registry path prefix
3. **Push** -- ship to `<region>-docker.pkg.dev/<project>/<repo>/<image>:<tag>`

### 6.2 Container Lifecycle

The `docker_management` role handles running containers. Common tasks include:

```yaml
- name: Stop existing container
  community.general.docker_container:
    name: "{{ container_name }}"
    state: absent

- name: Start updated container
  community.general.docker_container:
    name: "{{ container_name }}"
    image: "{{ docker_repo }}/{{ image_name }}:{{ tag }}"
    state: started
    restart_policy: unless-stopped
    env_file: "./env/.env"
```

See also: [[Docker & Container Patterns]]

---

## 7. Variable Management

### 7.1 Precedence (Low to High)

1. Role `defaults/main.yml` -- fallback values
2. `group_vars/all.yml` -- shared across all hosts
3. `group_vars/<env>.yml` -- environment overrides
4. Playbook `vars:` block -- play-scoped
5. `--extra-vars` on the CLI -- highest precedence, always wins

### 7.2 group_vars Example

```yaml
# group_vars/all.yml
gcp_project_id: "my-gcp-project-dev"
gcp_region: "us-central1"
gcp_artifact_repo: "{{ gcp_region }}-docker.pkg.dev/{{ gcp_project_id }}/etl-artifacts"
docker_image_name: "etl-app"
docker_tag: "{{ ansible_date_time.epoch | default('latest') }}"
```

Variables can reference other variables -- Ansible resolves them lazily at runtime.

### 7.3 Vault Encryption

Encrypt sensitive variable files:

```bash
ansible-vault encrypt group_vars/prod.yml
ansible-playbook playbooks/deploy.yml --ask-vault-pass
# Or with a password file for CI/CD:
ansible-playbook playbooks/deploy.yml --vault-password-file ~/.vault_pass
```

### 7.4 Jinja2 Templating in Variables

The `.env.j2` template pulls structured secret data:

```jinja2
GCP_PROJECT_ID={{ gcp_project_id }}
DB_HOST={{ secret_data.db_host }}
DB_PASSWORD={{ secret_data.db_password }}
ENVIRONMENT={{ env }}
```

---

## 8. Inventory Patterns

### 8.1 Localhost Execution

Most data engineering Ansible runs locally (building images, calling APIs):

```yaml
# inventory/localhost.yml
all:
  hosts:
    localhost:
      ansible_connection: local
      ansible_python_interpreter: "{{ ansible_playbook_python }}"
```

Setting `ansible_python_interpreter` to `ansible_playbook_python` avoids Python version mismatches.

### 8.2 Dynamic Inventory for Cloud

For managing remote VMs, use the GCP dynamic inventory plugin:

```yaml
# inventory/gcp.yml
plugin: google.cloud.gcp_compute
projects:
  - my-gcp-project-dev
regions:
  - us-central1
keyed_groups:
  - key: labels.env
    prefix: env
```

This auto-discovers Compute Engine instances and groups them by label.

### 8.3 Multi-Environment Targeting

```bash
# Dev (default)
ansible-playbook playbooks/deploy.yml

# Production -- override vars and inventory
ansible-playbook playbooks/deploy.yml \
  -i inventory/prod.yml \
  -e "env=prod gcp_project_id=my-gcp-project-prod"
```

---

## 9. Best Practices

- **Idempotency** -- Every task should be safe to re-run. Use modules with `state:` parameters instead of shell scripts where possible.
- **Tags for selective execution** -- Tag roles and tasks so you can run subsets:
  ```yaml
  roles:
    - role: docker_build_push
      tags: [docker, build]
    - role: secrets_manager
      tags: [secrets]
  ```
  ```bash
  ansible-playbook playbooks/init.yml --tags "docker"
  ```
- **Check mode (dry-run)** -- Preview changes without applying them:
  ```bash
  ansible-playbook playbooks/deploy.yml --check --diff
  ```
- **Error handling** -- Use `block/rescue/always` for graceful failures:
  ```yaml
  - block:
      - name: Push image
        command: docker push {{ image_uri }}
    rescue:
      - name: Re-authenticate and retry
        command: gcloud auth configure-docker --quiet
      - name: Retry push
        command: docker push {{ image_uri }}
  ```
- **`gather_facts: false`** -- Disable for localhost/API-only playbooks to speed up execution
- **Fact caching** -- Use `jsonfile` caching (`ansible.cfg`) to avoid redundant fact gathering across runs

---

## 10. Integration with CI/CD

### 10.1 GitHub Actions

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install Ansible + collections
        run: |
          pip install ansible
          ansible-galaxy collection install -r ansible/requirements.yml
      - name: Run deployment
        env:
          GCP_SA_KEY: ${{ secrets.GCP_SA_KEY }}
        run: |
          echo "$GCP_SA_KEY" > /tmp/service-account.json
          cd ansible
          ansible-playbook playbooks/deploy.yml -e "env=prod"
```

### 10.2 Containerised Runner

Package Ansible and its dependencies into a Docker image for reproducible CI runs:

```dockerfile
FROM python:3.11-slim
RUN pip install ansible google-auth requests docker
COPY ansible/ /ansible/
WORKDIR /ansible
RUN ansible-galaxy collection install -r requirements.yml
ENTRYPOINT ["ansible-playbook"]
```

```bash
docker run --rm \
  -v /tmp/service-account.json:/tmp/service-account.json \
  ansible-runner playbooks/deploy.yml -e "env=prod"
```

See also: [[GitLab CI-CD for dbt]], [[Docker & Container Patterns]], [[Terraform for Data Infrastructure]]

---

## Quick Reference

```bash
# Install dependencies
ansible-galaxy collection install -r requirements.yml

# Init environment
ansible-playbook playbooks/init.yml

# Deploy to dev
ansible-playbook playbooks/deploy.yml

# Deploy to prod with overrides
ansible-playbook playbooks/deploy.yml -e "env=prod" -i inventory/prod.yml

# Dry-run
ansible-playbook playbooks/deploy.yml --check --diff

# Run only docker tasks
ansible-playbook playbooks/deploy.yml --tags "docker"

# Encrypt prod secrets
ansible-vault encrypt group_vars/prod.yml
```
