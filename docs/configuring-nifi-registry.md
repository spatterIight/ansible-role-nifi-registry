<!--
SPDX-FileCopyrightText: 2020 - 2024 MDAD project contributors
SPDX-FileCopyrightText: 2020 - 2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2020 Aaron Raimist
SPDX-FileCopyrightText: 2020 Chris van Dijk
SPDX-FileCopyrightText: 2020 Dominik Zajac
SPDX-FileCopyrightText: 2020 Mickaël Cornière
SPDX-FileCopyrightText: 2022 François Darveau
SPDX-FileCopyrightText: 2022 Julian Foad
SPDX-FileCopyrightText: 2022 Warren Bailey
SPDX-FileCopyrightText: 2023 Antonis Christofides
SPDX-FileCopyrightText: 2023 Felix Stupp
SPDX-FileCopyrightText: 2023 Julian-Samuel Gebühr
SPDX-FileCopyrightText: 2023 Pierre 'McFly' Marty
SPDX-FileCopyrightText: 2024 Thomas Miceli
SPDX-FileCopyrightText: 2024 - 2025 Suguru Hirahara
SPDX-FileCopyrightText: 2025 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Apache NiFi Registry

This is an [Ansible](https://www.ansible.com/) role which installs [Apache NiFi Registry](https://nifi.apache.org/registry.html) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

Apache NiFi Registry is a complementary application for Apache NiFi that provides a central location for storage and management of shared resources across one or more instances of NiFi and/or MiNiFi.

See the project's [documentation](https://nifi.apache.org/docs/nifi-registry-docs/) to learn what Apache NiFi Registry does and why it might be useful to you.

## Prerequisites

To deploy Apache NiFi Registry using this role it is necessary that:

1. The [community.general](https://github.com/ansible-collections/community.general) collection be installed. This is needed to support modifying XML configuration files.
2. The [community.crypto](https://github.com/ansible-collections/community.crypto) collection be installed. This is needed to create the self-signed HTTPS certificate for Apache NiFi Registry.
3. The `keytool` program be available on the target host. This can be installed via `apt install default-jre` on Debian systems.

## Adjusting the playbook configuration

To enable Apache NiFi Registry with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# nifi-registry                                                        #
#                                                                      #
########################################################################

nifi_registry_enabled: true

nifi_registry_hostname: nifi-registry.example.com

# A passphrase used to generate a self-signed certificate
# which is used to serve Apache NiFi Registry via HTTPS internally
# Generate one using `pwgen -s 64 1`, or some other way
nifi_registry_self_signed_cert_passphrase: ""

# A passphrase used to encrypt sensitive values inputted into NiFi
# Generate one using `pwgen -s 64 1`, or some other way
nifi_registry_sensitive_props_key: ""

# The default login credentials to configure
# The password must be at least 12 characters, generate one using `pwgen -s 32 1`, or some other way
# The salt must be exactly 22 characters, and does not necessarily need to be changed from default value
nifi_registry_login_username: admin
nifi_registry_login_password: my-secure-password

########################################################################
#                                                                      #
# /nifi-registry                                                       #
#                                                                      #
########################################################################
```

### Adjusting the Traefik configuration

Since the Apache NiFi Registry container only supports listening via HTTPS it is necessary to configure Traefik to skip verifying Apache NiFi Registry's HTTPS certificate (since it self-signed).

To do this a custom `serversTransports` must be defined in Traefik's **dynamic** configuration.

```yaml
http:
  serversTransports:
    insecure-nifi-registry-transport:
      insecureSkipVerify: true
```

Or, if you are using the [MASH Traefik](https://github.com/mother-of-all-self-hosting/mash-playbook) role.

```yaml
traefik_provider_configuration_extension_yaml: |
  http:
    serversTransports:
      {{ nifi_registry_traefik_serverstransport }}:
        insecureSkipVerify: true
```

You can use the `nifi_registry_traefik_serverstransport` variable to reference the name dynamically, or just hard-code the value `insecure-nifi-registry-transport`.

### Adjusting the Apache NiFi Registry configuration

There are some additional things you may wish to configure about the component.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for the `nifi_registry_conf_*` variables that you can customize via your `vars.yml` file.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, Apache NiFi Registry becomes available at the specified hostname like `https://example.com`.

To get started, open the URL with a web browser to log in to the instance.

To log in to the instance use your configured credentials, or the randomly generated ones printed to the log.

Log Example:

```bash
Nov 20 13:20:24 server mash-nifi-registry[1990462]: Generated Username [d4a56f91-fcaf-4f6b-b83f-b21ea177c25b]
Nov 20 13:20:24 server mash-nifi-registry[1990462]: Generated Password [tlRhxiDso0cvWrLROwtkagpmk1Qwx1Rt]
```

## Troubleshooting

User guide is available on [this page](https://nifi.apache.org/docs/nifi-registry-docs/html/user-guide.html).

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu nifi-registry` (or how you/your playbook named the service, e.g. `mash-nifi-registry`).
