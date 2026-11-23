# AAP configuration-as-code baseline

A small starting point for managing a greenfield Ansible Automation Platform
(AAP) installation as code. It assumes that the AAP installation itself is
complete, but the platform has not yet been configured.

## How it works

The repository uses two stages to solve the configuration-as-code bootstrap
problem:

1. `local.yml` runs from a machine with access to the new AAP environment. It
   configures the Automation Hub sources, creates a local `ansible.cfg`, and
   applies the initial platform configuration.
2. The initial configuration creates a project and job template for this
   repository. That job template runs `site.yml` inside AAP for subsequent
   changes and day-2 operations.

The configuration is intentionally split into a common layer and use cases.
Tags allow a job template or operator to run only the requested layer.

## Prerequisites

- A reachable AAP installation and an AAP API token.
- Python and Ansible Core installed on the bootstrap machine.
- Access to the Red Hat Automation Hub repositories used by the example.
- A Git repository containing this project, reachable by AAP.
- The `infra.aap_configuration` collection and the collections listed in
  [`collections/requirements.yml`](collections/requirements.yml).

Install the required collections before running either playbook:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

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

Update [`vars/main.yml`](vars/main.yml) with the AAP URL, configuration-as-code
repository URL, and any environment-specific values. Update
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

