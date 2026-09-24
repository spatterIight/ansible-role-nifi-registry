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
SPDX-FileCopyrightText: 2025 - 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Apache NiFi Registry

This is an [Ansible](https://www.ansible.com/) role which installs [Apache NiFi Registry](https://nifi.apache.org/projects/registry/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

Apache NiFi Registry is a complementary application for [Apache NiFi](https://nifi.apache.org/) that provides a central location for storing and managing versioned flows (and extension bundles) shared across one or more NiFi instances.

See the project's [documentation](https://nifi.apache.org/docs/nifi-registry-docs/) to learn what Apache NiFi Registry does and why it might be useful to you.

## How this role secures Apache NiFi Registry

Unlike Apache NiFi, Apache NiFi Registry has no built-in username/password login. Its only ways of authenticating users are client TLS certificates, LDAP, Kerberos, or OpenID Connect, and any of them requires running it over HTTPS with a keystore, a truststore and an authorizer to match.

This role therefore runs Apache NiFi Registry **unsecured, over plain HTTP, inside its own container network**, and protects the public side with Traefik:

- **Your browser** reaches it at `https://nifi-registry.example.com` through Traefik, which requires HTTP basic authentication (`nifi_registry_basic_auth_username` / `nifi_registry_basic_auth_password`) before passing any request on. Requests to `/` are redirected to the web interface at `/nifi-registry/`.
- **Your Apache NiFi instances** reach it directly over the container network at `http://nifi-registry:18080`, bypassing Traefik. Apache NiFi's Registry client cannot send basic-auth credentials, which is why it does not go through Traefik.

The consequence is that **anything attached to the Apache NiFi Registry container network has full read and write access to it**. Only attach containers you trust — in practice, the Apache NiFi instances that use it.

Since Apache NiFi Registry sees every request as anonymous, it does not know who changed what: version comments in Apache NiFi record the NiFi user, but the Registry's own audit trail says `anonymous`.

## Prerequisites

To deploy Apache NiFi Registry using this role it is necessary that:

1. The [community.general](https://github.com/ansible-collections/community.general) collection be installed. This is needed to support modifying XML configuration files.
2. The [passlib](https://pypi.org/project/passlib/) Python library (with [bcrypt](https://pypi.org/project/bcrypt/)) be installed wherever Ansible runs. This is needed to hash the basic-auth password.

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

# The credentials Traefik asks for before letting a request through to Apache NiFi Registry.
# Generate a password using `pwgen -s 32 1`, or some other way
nifi_registry_basic_auth_username: admin
nifi_registry_basic_auth_password: ""

########################################################################
#                                                                      #
# /nifi-registry                                                       #
#                                                                      #
########################################################################
```

The password is bcrypt-hashed with a fixed salt (`nifi_registry_basic_auth_salt`), so that re-running the playbook does not produce a different hash every time and needlessly restart the service. The consequence is that the same password produces the same hash on every installation of this role, so choose a password with enough entropy to stand on its own.

To let more than one person in, add further `username:hash` entries (e.g. generated via `htpasswd -nbB user password`) to `nifi_registry_container_labels_traefik_basic_auth_users_custom`.

### Using a different authentication method

If something else in front of Traefik already authenticates requests (for example a forward-auth / single sign-on middleware), you can attach it and turn off basic authentication:

```yaml
nifi_registry_container_labels_traefik_basic_auth_enabled: false

nifi_registry_container_labels_traefik_additional_middlewares_custom:
  - my-sso@file
```

The role refuses to run with basic authentication enabled but no credentials configured, so it cannot end up exposed without authentication by accident.

### Connecting Apache NiFi

Apache NiFi needs to be attached to the Apache NiFi Registry container network. If you use the [Apache NiFi role](https://github.com/spatterIight/ansible-role-nifi) on the same host:

```yaml
nifi_container_additional_networks_custom:
  - "{{ nifi_registry_container_network }}"
```

Then, in Apache NiFi, open **Controller Settings** → **Registry Clients**, add a **NifiRegistryFlowRegistryClient**, and set its **URL** to `http://nifi-registry:18080` (use your `nifi_registry_identifier` as the hostname if you changed it, e.g. `http://mash-nifi-registry:18080`).

Alternatively, you can publish the port on the host with `nifi_registry_container_http_host_bind_port` (e.g. `127.0.0.1:18080`). Anything that can reach that port has full access, so bind it to a loopback or otherwise private address.

### Adjusting the Apache NiFi Registry configuration

The role ships Apache NiFi Registry's own configuration files verbatim in [`files/conf/`](../files/conf/) and applies adjustments on top of them at install time. Take a look at [`defaults/main.yml`](../defaults/main.yml) for the `nifi_registry_*_replacements` variables that you can customize via your `vars.yml` file, for example:

- `nifi_registry_properties_replacements_custom` for `nifi-registry.properties`
- `nifi_registry_providers_xml_replacements` for `providers.xml` (e.g. to store flows in Git or in the database rather than on the file system)
- `nifi_registry_bootstrap_conf_replacements` for `bootstrap.conf` (e.g. to change the JVM heap size)

Upstream's `NIFI_REGISTRY_*` and `AUTH` environment variables have no effect with this role, since its container startup script does not rewrite the configuration from them.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, Apache NiFi Registry becomes available at the specified hostname like `https://nifi-registry.example.com`. Log in with the basic-auth credentials you configured.

Flows are stored under `nifi_registry_data_path` (`/nifi-registry/data` by default): their metadata in the `database` directory, and the flow contents themselves in `flow_storage`. Back up both together.

## Troubleshooting

User guide is available on [this page](https://nifi.apache.org/docs/nifi-registry-docs/html/user-guide.html).

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu nifi-registry` (or how you/your playbook named the service, e.g. `mash-nifi-registry`).
