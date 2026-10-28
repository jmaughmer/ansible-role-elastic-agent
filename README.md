# ansible-role-elastic-agent

An Ansible role that installs Elastic Agent 9.x from Elastic's RPM or Debian package and enrolls it into **Fleet**, through the Fleet Server deployed by [ansible-role-elastic-fleet](https://github.com/jmaughmer/ansible-role-elastic-fleet), on RHEL, AlmaLinux and Rocky Linux 9 and 10, and on Ubuntu 24.04 and 26.04.

What it does:

- **Installs Elastic Agent** with ansible-role-elastic-fleet's own repository and package tasks, so both roles set up Elastic's repository, a Red Hat Satellite / Katello repository, or none, in exactly the same way.
- **Enrolls each host into an agent policy**, with the enrollment token that ansible-role-elastic-fleet published in the same run, one you set, or one looked up in Kibana.
- **Trusts Fleet Server through the CA in `ssl_certificates` or [ansible-role-elastic-tls](https://github.com/jmaughmer/ansible-role-elastic-tls)**, and can present a client certificate when Fleet Server asks for one.
- **Checks before it enrolls:** Fleet Server has to answer through that CA and report itself healthy, and the agent must not be newer than Fleet Server.
- **Upgrades in place** when `elastic_agent_version` changes, and re-enrolls only when the enrollment settings change.

## Requirements

- **Control node:** ansible-core 2.15 or later, and **ansible-role-elastic-fleet installed as `elastic_fleet`**: this role runs its package tasks (see [Package source](#package-source)).

  ```bash
  ansible-galaxy install -r requirements.yml
  ```

- **Target hosts:** RHEL, AlmaLinux or Rocky Linux 9 or 10, or Ubuntu 24.04 or 26.04, with `openssl` (installed by default). A host must not be a Fleet Server: that is already an Elastic Agent.
- **Fleet Server 9.x**, at least the version of the agents, and an **agent policy** to enroll into. ansible-role-elastic-fleet creates `agent-policy` by default (`elastic_fleet_agent_policies`); this role never creates policies.

## Supported platforms

| Distribution | `ansible_facts['distribution']` | Versions |
|---|---|---|
| Red Hat Enterprise Linux | `RedHat` | 9, 10 |
| AlmaLinux | `AlmaLinux` | 9, 10 |
| Rocky Linux | `Rocky` | 9, 10 |
| Ubuntu | `Ubuntu` | 24.04, 26.04 |

The package comes through dnf on Enterprise Linux and through apt on Ubuntu; everything else runs the same tasks on both. Satellite is Enterprise Linux only.

Preflight stops the run on any other distribution or version; add others to `elastic_agent_supported_distributions` at your own risk.

## Installation

```yaml
# requirements.yml
roles:
  - name: elastic_agent
    src: https://github.com/jmaughmer/ansible-role-elastic-agent.git
    scm: git
    version: main
  - name: elastic_fleet          # required: its package tasks are used
    src: https://github.com/jmaughmer/ansible-role-elastic-fleet.git
    scm: git
    version: main
  - name: elastic_tls            # optional, for the CA
    src: https://github.com/jmaughmer/ansible-role-elastic-tls.git
    scm: git
    version: main

collections:
  - name: community.crypto       # for elastic_tls
    version: ">=2.10.0"
```

## Quick start

Run it after the Fleet Server play, best in the same `ansible-playbook` run:

```yaml
- name: Deploy Fleet Server
  hosts: fleet_server
  become: true
  any_errors_fatal: true
  roles:
    - role: elastic_tls
    - role: elastic_fleet

- name: Install and enroll Elastic Agents
  hosts: elastic_agent
  become: true
  roles:
    - role: elastic_tls       # with elastic_tls_certificates: [], installs only the CA
    - role: elastic_agent
```

```yaml
# group_vars/elastic_agent/main.yml
elastic_agent_version: "9.5.4"
elastic_agent_policy_id: agent-policy
elastic_agent_tags: [linux]
elastic_tls_certificates: []
```

That is all: the Fleet Server URL and the enrollment token come from what ansible-role-elastic-fleet published in the first play, and the CA from ansible-role-elastic-tls. The inventory needs the `fleet_server` group (and, for elastic_tls, the `elasticsearch` group that holds its CA). [`examples/`](examples/) has a complete project.

## How a run works

Each host is handled on its own; agents do not depend on each other, so `serial` is fine, for example to upgrade a large fleet a part at a time.

1. Gather state and run preflight.
2. Configure the repository and install the package, or upgrade it in place.
3. Install the Fleet Server CA (and the client certificate, when used).
4. Decide whether the host needs enrolling: not enrolled yet, or enrolled with other settings.
5. Only for those hosts: find the enrollment token, check Fleet Server, and run `elastic-agent enroll`.
6. Start the agent, or restart it after an upgrade or new certificates, and wait until `elastic-agent status` reports it connected to Fleet and `HEALTHY`.

A run with nothing to change changes nothing, and makes no call to Kibana.

## Fleet Server URL

`elastic_agent_fleet_url` is the URL the agent enrolls through. Unless you set it, it is the first of:

1. `elastic_fleet_enrollment_urls`, which ansible-role-elastic-fleet publishes on the Fleet Server hosts when it runs earlier in the same run;
2. `elastic_fleet_public_urls`, when the inventory sets it for the first host of `elastic_agent_fleet_group` (`fleet_server`);
3. `https://<first host of fleet_server>:8220`.

After enrolling, agents use the Fleet Server hosts registered in Fleet. Through hostvars, Ansible cannot resolve a value that is itself built from `hostvars`; give `elastic_fleet_public_urls` as text in the inventory, or set `elastic_agent_fleet_url`.

## Enrollment token

Only hosts that are about to enroll need a token. It comes from, in order:

| Source | When |
|---|---|
| `elastic_agent_enrollment_token` | Set it (vault it). Any active token of the policy. |
| `elastic_fleet_enrollment_tokens[elastic_agent_policy_id]` on the first Fleet Server host | ansible-role-elastic-fleet ran earlier in the same run; it publishes one token per policy in `elastic_fleet_agent_policies`. |
| Kibana's Fleet API | Otherwise: the first active token of `elastic_agent_policy_id`, read once per run for every policy the play needs, with `elastic_agent_kibana_url` and either `elastic_agent_kibana_api_key` or `elastic_agent_kibana_username` / `_password` (a user with the **Fleet** `All` privilege). The Kibana CA defaults to the Fleet Server CA. |

`elastic-agent enroll` only takes the token on its command line, so it is visible in the process list for the moment enroll runs; the role passes it under `no_log`. An enrollment token can only enroll agents into its policy, and can be revoked in Fleet at any time.

## TLS

The role looks for the entry named `elastic_agent_ssl_certificate_name` (by default the inventory hostname) in `ssl_certificates` first and then in `elastic_tls_ssl_certificates`. The CA agents trust Fleet Server with is, in order:

1. that entry's `ca` (inline `content`, or the `path` of a file on the host);
2. `elastic_agent_ssl_ca_certificate` (PEM) or `elastic_agent_ssl_ca_certificate_path`;
3. `elastic_tls_ca_certificate`, which ansible-role-elastic-tls publishes even for hosts without certificates of their own (`elastic_tls_certificates: []`);
4. none: the system trust store, for example with a Let's Encrypt Fleet Server certificate.

It is copied to `/etc/elastic-agent/certs/fleet-ca.crt` (root only) and passed to enroll. Agents send data to Elasticsearch with the output settings of their policy, including its CA (ansible-role-elastic-fleet's `elastic_fleet_manage_output`), so nothing else is needed on the host.

**Client certificates.** With Fleet Server's `elastic_fleet_client_auth` set to `optional` or `required`, set `elastic_agent_client_certificate: true`: the entry's certificate and key (and the key's `passphrase`, passed as a file) are installed and presented. ansible-role-elastic-tls issues `clientAuth` certificates by default; keep the agents' `elastic_tls_certificates` at its default then.

**Renewal.** A renewed CA or certificate at the same path only needs a restart, which the role does.

## Enrollment and re-enrollment

The role records a salted SHA-256 of the enrollment settings (Fleet URL, policy id, CA and client certificate paths, tags, extra arguments, key passphrase) in `/etc/elastic-agent/.ansible-agent-state.json`. A host is enrolled when that record and Elastic Agent's own `/etc/elastic-agent/fleet.enc` both exist.

When the settings change, the agent is re-enrolled with `--force`. Re-enrolling gives it a **new id in Fleet**; the old one shows as offline until you remove it (or Fleet's inactivity timeout does). Set `elastic_agent_reenroll_on_change: false` to be told instead. A new token on its own does not re-enroll: any token of the same policy gives the same result. To move agents to another policy without re-enrolling them, reassign them in Fleet; the role only notices a change of `elastic_agent_policy_id`.

An agent enrolled by hand has no record, so the role enrolls it again on the first run.

## Upgrades

Change `elastic_agent_version`. Fleet cannot upgrade an agent installed from an RPM or Debian package, so the role upgrades the package with dnf or apt and restarts the agent.

- **Order:** Elasticsearch, Kibana, Fleet Server, then the agents. When the inventory sets `elastic_fleet_version` for the Fleet Servers, preflight refuses an agent newer than them.
- **Downgrades and major upgrades** are refused.
- `dnf update` never upgrades Elastic Agent behind the role's back: the repository file is written disabled and enabled only for the role's own tasks. On Ubuntu the package is held (`apt-mark hold`), so `apt upgrade` skips it; `apt-mark unhold elastic-agent` before upgrading it by hand.

## Package source

The repository and package tasks are ansible-role-elastic-fleet's `tasks/repository.yml` and `tasks/install.yml` (with its `package_version_check.yml`), run through `include_role` with this role's settings mapped onto theirs. Both roles therefore write the same repository file, `/etc/yum.repos.d/elastic-agent.repo` on Enterprise Linux or the apt source `/etc/apt/sources.list.d/elastic-9.x.sources` with the key `/etc/apt/keyrings/elastic.asc` on Ubuntu, and handle Satellite the same way. The Elasticsearch, Kibana and Logstash roles write the same apt source, so a host running several of them keeps one definition.

| `elastic_agent_repo_source` | Packages come from |
|---|---|
| `elastic` (default) | `elastic_agent_repo_baseurl`, Elastic's public repository or a mirror of it (yum or apt, to match the host) |
| `satellite` | Enterprise Linux only. `elastic_agent_satellite_repo_id`, a repository delivered by Red Hat Satellite or Katello; see ansible-role-elastic-fleet for `_repo_state`, `_repo_overrides`, `_refresh` and `_gpgkey` |
| `none` | Whatever the host has enabled |

`elastic_agent_fleet_role` names the installed role (default `elastic_fleet`).

## Slow and unreliable networks

| Variable | Default | |
|---|---|---|
| `elastic_agent_task_retries`, `elastic_agent_task_delay` | `3`, `15` | Retries for downloads and API calls |
| `elastic_agent_download_async_timeout`, `_poll` | `3600`, `15` | Package download time limit and poll interval |
| `elastic_agent_repo_timeout`, `elastic_agent_repo_retries` | `120`, `10` | dnf per-request timeout and retries (Enterprise Linux only) |
| `elastic_agent_api_timeout` | `30` | Per-request HTTP timeout |
| `elastic_agent_enroll_timeout` | `300` | How long enroll may take |
| `elastic_agent_ready_retries`, `elastic_agent_ready_delay` | `60`, `5` | Waiting for the agent to connect |

## Hardened hosts

- **FIPS.** An encrypted client key in the traditional PEM format (`Proc-Type: 4,ENCRYPTED`) cannot be read in FIPS mode; preflight refuses it. Elastic ships separate FIPS builds of Elastic Agent; install that package through `elastic_agent_repo_source: none` or a Satellite repository.
- **fapolicyd.** Nothing to do with the default rules. The package installs Elastic Agent and its components under `/var/lib/elastic-agent/data/`, where the rpm trust covers them, and the agent runs as root; tested with 9.5.4 and fapolicyd enforcing on AlmaLinux 10.
- **SELinux.** Elastic Agent runs unconfined from its packaged unit; nothing needs labelling.

## Notes and caveats

- **Check mode** is not supported for first installs: there is no package to query yet.
- **Removing a host** from the group leaves its agent enrolled and running. Unenroll it in Fleet and remove the package.
- **Integrations** belong in the agent policy, managed in Fleet; the role does not change policies.

## Variables

[`defaults/main.yml`](defaults/main.yml) documents every variable. The main ones:

| Variable | Default | Purpose |
|---|---|---|
| `elastic_agent_version` | `9.5.4` | Exact Elastic Agent release |
| `elastic_agent_fleet_group` | `fleet_server` | Inventory group of the Fleet Servers |
| `elastic_agent_fleet_url` | see [Fleet Server URL](#fleet-server-url) | URL the agent enrolls through |
| `elastic_agent_policy_id` | `agent-policy` | Agent policy to enroll into |
| `elastic_agent_enrollment_token` | `""` | Token (vault); see [Enrollment token](#enrollment-token) |
| `elastic_agent_tags` | `[]` | Tags shown in Fleet |
| `elastic_agent_ssl_certificate_name` | `{{ inventory_hostname }}` | Entry in `ssl_certificates` / `elastic_tls_ssl_certificates` |
| `elastic_agent_client_certificate` | `false` | Present a client certificate to Fleet Server |
| `elastic_agent_kibana_url` | first `kibana` host on 5601 | Kibana, to look tokens up |
| `elastic_agent_kibana_api_key` | `""` | Kibana credentials (or username and password) |
| `elastic_agent_reenroll_on_change` | `true` | Re-enroll when enrollment settings change |
| `elastic_agent_ready_states` | `[HEALTHY]` | States that count as ready |

Useful tags: `elastic_agent_install`, `elastic_agent_tls`, `elastic_agent_enroll`, `elastic_agent_service`. Fact gathering and preflight always run.

## Role layout

```
├── defaults/main.yml           # every tunable, documented
├── vars/main.yml               # internal, lazily evaluated values
├── meta/main.yml
├── requirements.yml            # ansible-role-elastic-fleet
├── tasks/
│   ├── main.yml
│   ├── facts.yml               # installed version, service and enrollment state
│   ├── preflight.yml           # validation before anything is written
│   ├── package.yml             # ansible-role-elastic-fleet's repository and install tasks
│   ├── certificates.yml        # Fleet Server CA, client certificate
│   ├── enrollment.yml          # what each host needs
│   ├── token.yml               # where the enrollment token comes from
│   ├── fleet_check.yml         # Fleet Server reachable, trusted, healthy
│   ├── service.yml             # enroll, start or restart
│   └── wait_ready.yml          # connected to Fleet and healthy
└── examples/
```

## Testing

[`molecule/default/`](molecule/default/) builds the whole chain in five systemd containers: `es-1` (Elasticsearch, single node), `kb-1` (Kibana), `fleet-1` (Fleet Server, deployed by ansible-role-elastic-fleet) and two agents, with every certificate from ansible-role-elastic-tls. [`prepare.yml`](molecule/default/prepare.yml) creates two agent policies. `agent-1` looks its token up in Kibana and presents a client certificate; `agent-2` has only the CA and takes its token the way ansible-role-elastic-fleet publishes it. The sequence is:

1. **converge**, **idempotence** and **check** (check mode on the enrolled hosts);
2. **side effect** ([`side_effect.yml`](molecule/default/side_effect.yml)): `agent-2` gets an explicit token and new tags, and must come back under a new id while `agent-1` keeps its own;
3. **verify** ([`verify.yml`](molecule/default/verify.yml)): the service, version, shared repository file, file modes, the CA, `elastic-agent status`, and, in Fleet, each agent online once, in its policy, with its tags.

[`molecule/preflight/`](molecule/preflight/) runs the checks against settings that must be refused (and some that must pass), checking each fails in the right task without printing a secret. [`molecule/docker/`](molecule/docker/) is the default scenario on Docker.

```bash
pip install -r molecule/requirements.txt
molecule test                                 # Rocky Linux 9
MOLECULE_DISTRO=rockylinux10 molecule test
molecule test -s preflight
```

### On KVM virtual machines

[`molecule/kvm/`](molecule/kvm/) runs the default scenario on real AlmaLinux 10 virtual machines
instead of containers: the same hosts, groups, playbooks and inventory, one VM per host, on local
KVM/libvirt (`qemu:///system`). Each VM has its own kernel, so SELinux is enforcing and settings such
as `vm.max_map_count` are the VM's own. Overrides for the VMs go in
`molecule/kvm/inventory/group_vars/<group>/kvm.yml`.

[`molecule/kvm-ubuntu2404/`](molecule/kvm-ubuntu2404/) and
[`molecule/kvm-ubuntu2604/`](molecule/kvm-ubuntu2604/) run the same thing on Ubuntu 24.04 and 26.04
VMs. They only swap the cloud image, and share `molecule/kvm/`'s playbooks and inventory.

```bash
for s in kvm kvm-ubuntu2404 kvm-ubuntu2604; do molecule test -s "$s" || break; done
```

The machine running Molecule needs libvirt, `virt-install`, `qemu-img` and `xorriso`, and its user
must be able to manage `qemu:///system`. Each cloud image downloads once, checksum verified, to
`/var/lib/libvirt/images/molecule`; each VM gets a throwaway disk on top of it. VM names include the
scenario, so the three scenarios can also run side by side if the machine has the memory.

## Related roles

- [ansible-role-elastic-fleet](https://github.com/jmaughmer/ansible-role-elastic-fleet): Fleet Server, the agent policies and their enrollment tokens.
- [ansible-role-elastic-tls](https://github.com/jmaughmer/ansible-role-elastic-tls): the CA agents trust Fleet Server with.
- [ansible-role-elasticsearch](https://github.com/jmaughmer/ansible-role-elasticsearch), [ansible-role-kibana](https://github.com/jmaughmer/ansible-role-kibana), [ansible-role-logstash](https://github.com/jmaughmer/ansible-role-logstash): the rest of the stack.
- [ansible-role-keepalived](https://github.com/jmaughmer/ansible-role-keepalived): a virtual address for the Fleet Server URL.

## License

MIT. See [LICENSE](LICENSE).
