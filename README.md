# ansible_role_user_ssh

Installs the specified SSH public keys for a user, and allows SSH connection
by writing an `sshd_config.d` drop-in (`AllowUsers`) and reloading `sshd`.

## Requirements

- Ansible Core >= 2.14
- Collection `ansible.posix` (for the `authorized_key` module)
- install with: `ansible-galaxy collection install ansible.posix`
- Target hosts must support `sshd_config.d` drop-ins (OpenSSH >= 6.7) and use
 the `ssh.service` systemd unit

### Supported platforms

| Family | Distributions / versions |
| --- | --- |
| Debian | bookworm (12), trixie (13) |
| Ubuntu | jammy (22.04), noble (24.04) |

The role fails fast with a clear message on any unsupported OS.

## Role Variables

| Variable | Default | Description |
| --- | --- | --- |
| `user_ssh_name` | `null` | Existing user to manage (required). The user is **not** created by this role. |
| `user_ssh_keys` | `[]` | List of SSH public keys (full `authorized_keys` lines) to install for the user. |
| `user_ssh_keys_replace` | `false` | When `true`, replaces the whole `authorized_keys` file (any key not listed is removed). See below. |

> The handler name references `role_name`, so define it (e.g. in group/host vars)
> when using this role, e.g. `role_name: ansible_role_user_ssh`.

### `user_ssh_keys` semantics

- **Non-empty keys** → the keys are written to the user's
 `~/.ssh/authorized_keys`.
- **`user_ssh_keys: []` with `user_ssh_keys_replace: true`** → all existing keys
 are removed (`authorized_keys` is emptied). This is a deliberate "remove all
 SSH access" mode.
- **`user_ssh_keys: []` with `user_ssh_keys_replace: false`** → the role only
 ensures `~/.ssh/authorized_keys` exists (created `700`/`600` in the user's
 real home directory, resolved via `getent`), without touching existing keys.

## Dependencies

None (Galaxy role dependencies). `ansible.posix` is a collection dependency and
must be installed separately.

## Example Playbook

To add bob's SSH key

```yaml
- hosts: servers
 roles:
 - role: ansible_role_user_ssh
 vars:
 role_name: ansible_role_user_ssh
 user_ssh_name: bob
 user_ssh_keys:
 - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFoM5Wa7JbhV... bob@laptop
```

To replace all of bob's SSH keys by a new one:

```yaml
- hosts: servers
 roles:
 - role: ansible_role_user_ssh
 vars:
 role_name: ansible_role_user_ssh
 user_ssh_name: bob
 user_ssh_keys:
 - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDW7bZ... bob@workstation
 user_ssh_keys_replace: true
```

To remove all of bob's SSH keys:

```yaml
- hosts: servers
 roles:
 - role: ansible_role_user_ssh
 vars:
 role_name: ansible_role_user_ssh
 user_ssh_name: bob
 user_ssh_keys: []
 user_ssh_keys_replace: true
```

## What it does

1. Validates role inputs (types, list contents) and supported OS.
2. Checks that the `ssh` service is available.
3. Manages the user's `~/.ssh/authorized_keys` with `authorized_key`
 (`exclusive` driven by `user_ssh_keys_replace`).
4. Creates `/etc/ssh/sshd_config.d/<user_ssh_name>.conf` containing
 `AllowUsers <user_ssh_name>`.
5. Reloads `sshd` when the drop-in changes.

## Testing

The role ships a Molecule scenario (`molecule/default`) that:

- creates a container, installs `openssh-server` and the user
- runs the role
- verifies the drop-in, `sshd -T` output, and that every configured key is
 present in `authorized_keys`

Run it with:

```sh
molecule test
```

## License

MIT — see [LICENSE](LICENSE).
