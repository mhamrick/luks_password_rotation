# luks_passphrase_rotation

Ansible role to rotate LUKS passphrases on remote machines and store the new passphrase securely in a password manager (1Password or Keeper), with ansible-vault-encrypted local backups.

## Workflow

```
Generate passphrase -> Discover LUKS devices -> Rotate on devices -> Upload to 1Password/Keeper -> Save encrypted local backup
```

## Prerequisites

- **Ansible** >= 2.10
- **community.crypto** collection: `ansible-galaxy collection install community.crypto`
- **cryptsetup** installed on target hosts (the role installs it automatically)
- **1Password CLI** (`op`) — if using 1Password integration, with `OP_SERVICE_ACCOUNT_TOKEN` env var set
- **Keeper CLI** (`keeper`) — if using Keeper integration, with config file
- **Vault password file** — if using local backup encryption

## Role Variables

### LUKS settings

| Variable | Default | Description |
|---|---|---|
| `luks_rotation_no_log` | `true` | Suppress passphrase values from task output |
| `luks_rotation_current_passphrases` | *required* | List of current/old passphrases on the LUKS devices |

### Passphrase generation

| Variable | Default | Description |
|---|---|---|
| `luks_rotation_passphrase_length` | `64` | Length of the generated passphrase |
| `luks_rotation_passphrase_charset` | `"alphanumeric"` | Character set: `"alphanumeric"`, `"ascii_printable"`, or `"hex"` |

### Vault integration

| Variable | Default | Description |
|---|---|---|
| `luks_rotation_vault_integration` | `"none"` | `"none"`, `"1password"`, `"keeper"`, or `"both"` |
| `luks_rotation_vault_title_template` | `"LUKS Passphrase - {{ inventory_hostname }}"` | Title for password manager entries |
| `luks_rotation_vault_fail_on_upload_error` | `false` | Fail the play if vault upload errors |
| `luks_rotation_1password_vault` | `"Infrastructure"` | 1Password vault name |
| `luks_rotation_1password_tags` | `["luks", "encryption-key"]` | Tags for 1Password items |
| `luks_rotation_keeper_folder_path` | `"LUKS Keys"` | Keeper folder for records |
| `luks_rotation_keeper_config_path` | `"~/.keeper/config.json"` | Keeper CLI config path |

### Local backup

| Variable | Default | Description |
|---|---|---|
| `luks_rotation_local_backup_enabled` | `true` | Write ansible-vault-encrypted backup to localhost |
| `luks_rotation_local_backup_dir` | `"./passphrase_backups"` | Backup directory on the controller |
| `luks_rotation_vault_password_file` | *required when backup enabled* | Path to ansible-vault password file |

## Example Playbook

```yaml
- hosts: luks_servers
  become: true
  vars:
    luks_rotation_current_passphrases:
      - "{{ vault_current_luks_passphrase }}"
    luks_rotation_vault_integration: "1password"
    luks_rotation_local_backup_enabled: true
    luks_rotation_vault_password_file: "~/.vault_pass"
  roles:
    - role: luks_passphrase_rotation
```

## Security Notes

- All tasks handling passphrase values use `no_log: true` by default. Set `luks_rotation_no_log: false` only for debugging on non-production systems.
- Local backup files are encrypted with ansible-vault immediately after writing. If encryption fails, the plaintext file is deleted before the task errors out.
- The backup directory is created with mode `0700`.
- Password manager uploads use `delegate_to: localhost` and never expose credentials to remote hosts.

## Recovering a Local Backup

```bash
ansible-vault decrypt --vault-password-file ~/.vault_pass passphrase_backups/myhost_luks_passphrase.yml
cat passphrase_backups/myhost_luks_passphrase.yml
```

The decrypted file contains:

```yaml
luks_passphrase: "<the passphrase>"
luks_devices: ["/dev/sda3", "/dev/sdb1"]
rotated_date: "2025-01-15T12:00:00Z"
```

## License

MIT
