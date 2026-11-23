# AAP configuration-as-code baseline

A small starting point for managing a greenfield Ansible Automation Platform
(AAP) installation as code. It assumes that the AAP installation itself is
complete, but the platform has not yet been configured.

## How it works

The repository uses a short bootstrap stage followed by two configuration
stages:

1. The stage-0 collection install uses a temporary Galaxy configuration so the
   collection needed by `local.yml` is available before private Hub setup.
2. `local.yml` runs from a machine with access to the new AAP environment. It
   configures the Automation Hub sources, creates a local `ansible.cfg`, and
   applies the initial platform configuration.
3. The initial configuration creates a project and job template for this
   repository. That job template runs `site.yml` inside AAP for subsequent
   changes and day-2 operations.

The configuration is intentionally split into a common layer and use cases.
Tags allow a job template or operator to run only the requested layer.

## Prerequisites

- Ansible Automation Platform 2.7 or newer. This sample assumes the AAP 2.7
  platform-gateway API model.
- A reachable AAP installation and an AAP API token.
- Python and Ansible Core installed on the bootstrap machine.
- Access to the Red Hat Automation Hub repositories used by the example.
- A Git repository containing this project, reachable by AAP.
- The `infra.aap_configuration` collection and the collections listed in
  [`collections/requirements.yml`](collections/requirements.yml).

### The bootstrap dependency

There is an intentional chicken-and-egg step here: `local.yml` creates the
private Automation Hub configuration, but Ansible must already have
`infra.aap_configuration` installed before it can run `local.yml`. The
repository therefore uses a temporary collection source for **stage 0**.

The temporary source is not the private Hub that `local.yml` creates. It uses
Red Hat Automation Hub at `console.redhat.com` and Ansible Galaxy as a
fallback. This follows Ansible's [Galaxy server-list configuration](https://docs.ansible.com/projects/ansible/latest/collections_guide/collections_installing.html#configuring-the-ansible-galaxy-client),
and the `infra.aap_configuration` collection is available through [Ansible
Galaxy](https://galaxy.ansible.com/ui/collections/infra/aap_configuration/).

Copy the secret-free example to a temporary location, replace
`REPLACE_ME` with the Red Hat Automation Hub token, and protect the file:

```bash
cp ansible-bootstrap.cfg.example /tmp/aap-bootstrap-ansible.cfg
chmod 600 /tmp/aap-bootstrap-ansible.cfg
$EDITOR /tmp/aap-bootstrap-ansible.cfg
```

Then install the collections using that configuration:

```bash
ANSIBLE_CONFIG=/tmp/aap-bootstrap-ansible.cfg \
  ansible-galaxy collection install -r collections/requirements.yml
```

The `ANSIBLE_CONFIG` prefix is temporary and applies only to that command. Do
not copy the token into the repository's [`ansible.cfg`](ansible.cfg), and do
not pass it directly on the command line. Ansible documents the server-list
ordering and token settings in its [Galaxy client configuration guide](https://docs.ansible.com/projects/ansible-core/devel/collections_guide/collections_installing.html#configuring-the-ansible-galaxy-client).

If policy requires all content to come from the private Hub, an AAP
administrator must first sync the required collections—including
`infra.aap_configuration`—into a private repository using the [AAP 2.7 remote
repository procedure](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/administer-configure_remote_repositories_to_sync_automation_content_).
In that case, point the temporary config at the private Hub instead of
`console.redhat.com`.

## AAP 2.7+ URLs and tokens

This repository assumes AAP 2.7 or newer. In AAP 2.7, API access is routed
through the platform gateway; direct controller, Hub, or Event-Driven Ansible
component URLs and component-level authentication are no longer supported.
See Red Hat's [AAP 2.7 API changes](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/upgrade-con_upgrade_api_changes_27)
and [authentication changes](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/upgrade-con_auth_changes_27)
for the migration details.

Set `automation_hub_url` in [`vars/main.yml`](vars/main.yml) to the base URL
of your AAP platform gateway, for example `https://gateway.example.com`.
Use the gateway hostname supplied by your AAP administrator; do not substitute
a separate `controller.example.com` or `hub.example.com` hostname.

The values used by this sample come from these locations:

- `vault_aap_token`: Create a token in the AAP platform gateway at **Access →
  Tokens**, or create an OAuth application under **Access → Applications**.
  The [AAP 2.7 token documentation](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/secure-ref_controller_token_session_management)
  covers token management. Use a token belonging to a user or service account
  with the permissions required to create the initial configuration.
- `vault_rh_token`: Open the [Red Hat Hybrid Cloud Console Automation Hub token
  page](https://console.redhat.com/ansible/automation-hub/token), choose
  **Load token**, and copy the offline token. Red Hat's [AAP 2.7 Hub token
  procedure](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/secure-token_management_hub)
  explains the same process. This token is used to sync validated and
  certified content from `console.redhat.com`.
- `automation_hub_url`: The AAP platform gateway URL for the private Hub/API
  endpoint used by your installation. The AAP administrator or installation
  inventory should provide this hostname.
- `config_as_code_scm_url`: The clone URL for the Git repository containing
  this project, such as `https://github.example.com/team/config-as-code.git`.
- `scm_username` and `vault_scm_password`: The SCM username and a repository
  access token/password that can read the configuration repository. Store the
  secret only in the encrypted vault file.

The sample also contains demonstration credentials in
[`common/vars/credentials.yml`](common/vars/credentials.yml). Replace those
values or remove that example before using the repository in a real
environment.

## Configure secrets and environment values

Copy the example vault file and replace every placeholder with an appropriate
value:

```bash
cp vars/vault.yml.example vars/vault.yml
ansible-vault encrypt vars/vault.yml
```

The real `vars/vault.yml` is ignored by Git. Run the playbook with the vault
password prompt or with the vault password mechanism used by your environment:

```bash
ansible-playbook --ask-vault-pass -i localhost, local.yml
```

Update [`vars/main.yml`](vars/main.yml) with the platform gateway URL,
configuration-as-code repository URL, and any environment-specific values. Set
`aap_validate_certs: true` in production when your gateway uses a certificate
that is trusted by the execution environment. Update
[`automation_hub/vars/main.yml`](automation_hub/vars/main.yml) if the Hub
repositories or collection requirements differ from the example.

Do not commit API tokens, passwords, or generated credentials. Any credentials
that have been exposed outside the intended environment should be revoked and
replaced.

## Bootstrap the platform

Run the local bootstrap playbook from the repository root:

```bash
ansible-playbook --ask-vault-pass -i localhost, local.yml
```

The playbook applies, in order:

- Automation Hub collection remotes and repositories.
- The local `ansible.cfg` needed to consume those repositories.
- The initial AAP configuration, including the configuration-as-code project
  and job template.

The generated repository [`ansible.cfg`](ansible.cfg) is the post-bootstrap
configuration. It points collection installation at the private Hub gateway
and is intended to replace the temporary configuration for later installs.
Remove the temporary file when the bootstrap is complete.

The `-i localhost,` trailing comma is important: it supplies an inline
inventory containing `localhost`.

## Apply configuration in AAP

After bootstrap, the AAP job template should run [`site.yml`](site.yml). The
playbook loads the common variables and then dispatches the selected
configuration layers:

- `common/` contains shared credentials and projects.
- `use_case_1/` contains the first example job-template configuration.

Run the same playbook locally for validation or troubleshooting:

```bash
ansible-playbook --ask-vault-pass -i localhost, site.yml
ansible-playbook --ask-vault-pass -i localhost, site.yml --tags common
ansible-playbook --ask-vault-pass -i localhost, site.yml --tags use_case_1
```

The survey defined in [`vars/main.yml`](vars/main.yml) documents the currently
available configuration layers (`common` and `use_case_1`). The job template
can also be launched with tags directly. Add a new use case by creating a
directory with a `dispatch.yml` task list and a `vars/main.yml` file, then add
its include, tag, and survey choice to the corresponding files.

## Repository layout

```text
local.yml                       Bootstrap playbook run outside AAP
site.yml                        Configuration playbook run by AAP
ansible.cfg                     Example local Galaxy configuration
ansible-bootstrap.cfg.example   Temporary stage-0 Galaxy configuration
vars/main.yml                   Shared bootstrap and project variables
vars/vault.yml.example          Secret-value template; copy and encrypt it
automation_hub/vars/main.yml   Automation Hub remotes and repositories
common/                         Shared AAP configuration
use_case_1/                     Example use-case configuration
collections/requirements.yml    Required Ansible collections
```

## Validate changes

Use syntax checks before committing changes:

```bash
ansible-playbook --syntax-check local.yml
ansible-playbook --syntax-check site.yml
```

For a real run, use `--check` only after confirming that the installed
collection versions support check mode for the resources being managed.

