# aap-config-as-code-baseline
A base configuration as code approach for greenfield AAP installations

## Usage
This repository starts when the installation playbook is complete, but nothing has been configured yet.

## High Level workflow
There's a chicken-and-egg problem for running Ansible configuration as code. This approach solves the problem by running a two-stage deployment.
First, there's a local playbook that configures automation hub for collection content, and the initial configuration of the platform.
This creates a job template which can then be run to apply further configurations, including day-2 operations, etc.


### Step 1
The first playbook to run is local.yml, which is meant to be run from the command line with
`ansible-playbook -i localhost local.yml`
This playbook sets up automation hub with a content sync, and configures AAP to have a job template present to apply additional configuration as code through.

