# orchestrator-cac

> **⚠️ Work in Progress:** This project is under active development and is not yet production-ready. Roles, variables, and playbook structure are subject to change.

Ansible Configuration-as-Code (CaC) for managing resources in an Automation orchestrator instance. This project provides roles and a top-level playbook to declaratively create and manage identity providers, projects, credentials, integrations, and workflows.

## Requirements

- Ansible
- Network access to the target orchestrator API

## Usage

1. Copy `orchestrator_cac_vars.yml` and fill in your values (see [Variables](#variables)).
2. Set the required connection variables for the `authenticate` role (see role docs below).
3. Encrypt the vars file with Ansible Vault to protect sensitive values (passwords, tokens, API keys):

```bash
ansible-vault encrypt orchestrator_cac_vars.yml
```

4. Run the playbook, providing the Vault password:

```bash
ansible-playbook create_orchestrator_resources.yml -e @orchestrator_cac_vars.yml --ask-vault-pass
```

Alternatively, store the Vault password in a file and reference it with `--vault-password-file`:

```bash
ansible-playbook create_orchestrator_resources.yml -e @orchestrator_cac_vars.yml --vault-password-file ~/.vault_pass
```

To remove resources instead of creating them, set `resource_state=absent`:

```bash
ansible-playbook create_orchestrator_resources.yml -e @orchestrator_cac_vars.yml -e resource_state=absent --ask-vault-pass
```

> **Note:** Never commit an unencrypted vars file containing real credentials to version control. The provided `orchestrator_cac_vars.yml` uses placeholders — always encrypt before adding real values.

## Playbook: `create_orchestrator_resources.yml`

Runs against `localhost` and orchestrates the following sequence:

1. **Authenticate** — obtains a Bearer token from the Orchestrator API
2. **Manage identity providers** — creates AAP identity providers (looped over `aap_identity_providers`)
3. **Create projects** — creates or deletes projects (looped over `projects`)
4. **Create credentials** — creates or deletes credentials (looped over `credentials`)
5. **Manage integrations** — creates or deletes integrations (looped over `integrations`)
6. **Manage workflows** — creates workflows from JSON definition files

## Variables

Define these in your vars file (use `orchestrator_cac_vars.yml` as a template):

| Variable | Description |
|---|---|
| `aap_identity_providers` | List of AAP identity provider definitions (see role vars) |
| `projects` | List of project definitions |
| `credentials` | List of credential definitions |
| `integrations` | List of integration definitions |
| `resource_state` | `present` (default) to create, `absent` to delete |

## Roles

### `authenticate`

Logs in to the Orchestrator API and stores the Bearer token as `orchestrator_token` for use by subsequent roles.

**Defaults:**

| Variable | Default | Description |
|---|---|---|
| `orchestrator_host` | `https://orchestrator-host` | Base URL of the Orchestrator API |
| `orchestrator_username` | `""` | Orchestrator admin username |
| `orchestrator_password` | `""` | Orchestrator admin password |
| `orchestrator_validate_certs` | `false` | Whether to validate TLS certificates |

---

### `manage_identity_providers`

Creates an AAP identity provider in the Orchestrator if one does not already exist. Only runs create logic when `aap_identity_provider_state == "present"` and no identity providers are currently configured.

**Defaults:**

| Variable | Default | Description |
|---|---|---|
| `aap_identity_provider_aap_url` | `""` | URL of the AAP instance |
| `aap_identity_provider_organization` | `""` | AAP organization name |
| `aap_identity_provider_admin_username` | `""` | AAP admin username |
| `aap_identity_provider_admin_password` | `""` | AAP admin password |
| `aap_identity_provider_insecure_skip_tls_verify` | `true` | Skip TLS verification for the AAP connection |
| `aap_identity_provider_state` | `present` | `present` to create |

---

### `manage_project`

Creates or deletes a project in the Orchestrator.

**Defaults:**

| Variable | Default | Description |
|---|---|---|
| `project_name` | `""` | Name of the project |
| `project_description` | `""` | Description of the project |
| `project_labels` | `{}` | Labels to apply to the project |
| `project_state` | `present` | `present` to create, `absent` to delete |

---

### `manage_credential`

Creates or deletes a credential in the Orchestrator. Looks up the parent project by name to obtain its ID before creating the credential.

**Defaults:**

| Variable | Default | Description |
|---|---|---|
| `credential_name` | `""` | Name of the credential |
| `credential_description` | `""` | Description of the credential |
| `credential_type_name` | `""` | Credential type (e.g. `Ansible Automation Platform`, `LLM Provider`, `HTTP Bearer Token`) |
| `credential_project_name` | `""` | Name of the parent project |
| `credential_inputs` | `{}` | Type-specific credential inputs (e.g. `username`/`password`, `api_key`, `token`) |
| `credential_labels` | `{}` | Labels to apply to the credential |
| `credential_state` | `present` | `present` to create, `absent` to delete |

---

### `manage_integration`

Creates or deletes an integration in the Orchestrator. Looks up the associated credential by name before creating the integration.

**Defaults:**

| Variable | Default | Description |
|---|---|---|
| `integration_name` | `""` | Name of the integration |
| `integration_description` | `""` | Description of the integration |
| `integration_type` | `""` | Integration type (e.g. `ansible_automation_platform`, `llm_provider`, `mcp_server`) |
| `integration_credential_name` | `""` | Name of the credential to associate |
| `integration_configuration` | `{}` | Type-specific configuration (e.g. `base_url`, `provider_hint`) |
| `integration_state` | `present` | `present` to create, `absent` to delete |

---

### `manage_workflow`

Creates a workflow in the Orchestrator from a JSON definition file bundled under `roles/manage_workflow/files/`. The workflow is associated with a project looked up by name.

This role currently has no configurable defaults — the project name and workflow name are set directly in the tasks. Update `roles/manage_workflow/tasks/main.yml` to parameterize these values.
