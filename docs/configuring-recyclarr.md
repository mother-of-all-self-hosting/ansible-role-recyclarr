<!--
SPDX-FileCopyrightText: 2020 Aaron Raimist
SPDX-FileCopyrightText: 2020 Chris van Dijk
SPDX-FileCopyrightText: 2020 Dominik Zajac
SPDX-FileCopyrightText: 2020 Mickaël Cornière
SPDX-FileCopyrightText: 2020-2024 MDAD project contributors
SPDX-FileCopyrightText: 2020-2026 Slavi Pantaleev
SPDX-FileCopyrightText: 2022 François Darveau
SPDX-FileCopyrightText: 2022 Julian Foad
SPDX-FileCopyrightText: 2022 Warren Bailey
SPDX-FileCopyrightText: 2023 Antonis Christofides
SPDX-FileCopyrightText: 2023 Felix Stupp
SPDX-FileCopyrightText: 2023 Pierre 'McFly' Marty
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Recyclarr

This is an [Ansible](https://www.ansible.com/) role which installs [Recyclarr](https://recyclarr.dev/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

Recyclarr syncs the recommended settings of the [TRaSH Guides](https://trash-guides.info/) — custom formats, quality profiles, quality sizes and media naming — to [Sonarr](https://sonarr.tv/) and [Radarr](https://radarr.video/), on a schedule.

See the project's [documentation](https://recyclarr.dev/) to learn what Recyclarr does and why it might be useful to you.

## Adjusting the playbook configuration

To enable Recyclarr with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# recyclarr                                                            #
#                                                                      #
########################################################################

recyclarr_enabled: true

########################################################################
#                                                                      #
# /recyclarr                                                           #
#                                                                      #
########################################################################
```

### Configuring Sonarr and Radarr instances

Recyclarr needs at least one Sonarr or Radarr instance to sync to, with its URL and API key. You can find the API key on Sonarr and Radarr under `Settings > General` in the `Security` section.

The instances are configured with `recyclarr_config_sonarr` and `recyclarr_config_radarr`, which take Recyclarr's `sonarr` and `radarr` sections. For example, to sync the quality sizes of the TRaSH Guides, add the following configuration to your `vars.yml` file (adapt to your needs):

```yaml
recyclarr_config_sonarr:
  sonarr:
    base_url: http://sonarr:8989
    api_key: YOUR_SONARR_API_KEY_HERE
    quality_definition:
      type: series

recyclarr_config_radarr:
  radarr:
    base_url: http://radarr:7878
    api_key: YOUR_RADARR_API_KEY_HERE
    quality_definition:
      type: movie
```

**Note**: each instance needs a name of its own, across Sonarr and Radarr alike. Recyclarr does not sync at all if a name is used for both.

Refer to [this page](https://recyclarr.dev/reference/configuration/) on the documentation for everything an instance can configure, and to [this page](https://recyclarr.dev/guide/guide-configs/) for ready-made configurations that follow the TRaSH Guides.

**Note**: Ansible cannot read the `!secret` and `!env_var` tags that Recyclarr's own examples use. Put the values in directly instead, for example with [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html).

#### Connecting to Sonarr and Radarr on the same server

If Sonarr and Radarr are installed on the same server with [their](https://github.com/mother-of-all-self-hosting/ansible-role-sonarr) [roles](https://github.com/mother-of-all-self-hosting/ansible-role-radarr), you can let Recyclarr talk to them directly over the container network, instead of going through their public hostnames. To do so, add the following configuration to your `vars.yml` file.

```yaml
# Lets Recyclarr reach Sonarr and Radarr by their container names
recyclarr_container_additional_networks_custom:
  - "{{ sonarr_container_network }}"
  - "{{ radarr_container_network }}"

# Starts Sonarr and Radarr whenever Recyclarr is started
recyclarr_systemd_wanted_services_list_custom:
  - "{{ sonarr_identifier }}.service"
  - "{{ radarr_identifier }}.service"
```

Then use `http://sonarr:8989` and `http://radarr:7878` as `base_url` (or how you/your playbook named the services, e.g. `http://mash-sonarr:8989`). If Sonarr or Radarr is served under a path prefix, add it to the URL (e.g. `http://mash-sonarr:8989/sonarr`).

### Setting the schedule (optional)

Recyclarr syncs once a day by default. To change that, add the following configuration to your `vars.yml` file (adapt to your needs):

```yaml
recyclarr_environment_variables_cron_schedule: "0 4 * * *"
```

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file
- `recyclarr_configuration_extension_yaml` for extending the configuration file which the role renders from `recyclarr_config_sonarr` and `recyclarr_config_radarr`

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

Recyclarr does not sync when it starts, only on its schedule. To sync right away, for example after changing the configuration, run the following command on the server:

```sh
docker exec recyclarr recyclarr sync
```

Replace `recyclarr` with how you/your playbook named the service, e.g. `mash-recyclarr`. Add `--preview` to see what would change without changing anything.

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu recyclarr` (or how you/your playbook named the service, e.g. `mash-recyclarr`).
