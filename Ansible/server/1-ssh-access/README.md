# 1-ssh-access

Bootstrap playbook for initial access setup on a fresh server: creates
users, distributes SSH keys, and configures sudo.

## What it does

1. Installs `sudo` if missing (relevant for minimal base images).
2. Generates SSH keys (ed25519) on the controller for each user —
   `~/.ssh/bootstrap-vpc/<user>/key` (+ `.pub`).
3. Creates the users on the server.
4. Configures access:
   - **service_users** (`ansible`) — passwordless sudo via
     `/etc/sudoers.d/`, no TOTP. Meant for automation.
   - **human_users** (`vaclav`, ...) — added to the `sudo` group, sudo
     access gated by TOTP (Google Authenticator) instead of a password.
5. Adds each user's public key to their `authorized_keys`.

## First run

The first login to a bare server happens as `root` (or the default
cloud-init user) via password — credentials for this live in
`host_vars/<host>.yml`:

```yaml
ansible_user: root
ansible_password: <password>
```

Run:

```bash
ansible-playbook 1-ssh-access/bootstrap.yml
```

## After the first run

- Private keys live at `~/.ssh/bootstrap-vpc/<user>/key` — use them to log
  in (`ssh -i ~/.ssh/bootstrap-vpc/vaclav/key vaclav@<host>`).
- For human users, the playbook output (`Show QR code and secret for each
  user` task) prints a TOTP secret and emergency scratch codes — enter the
  secret into an authenticator app once and store the scratch codes
  separately, since they aren't kept anywhere else.
- Once TOTP is set up, `sudo` for human users only asks for the app code —
  no password is used at all.

## Idempotency

Re-running the playbook is safe:
- Existing keys and TOTP secrets (`.google_authenticator`) are not
  regenerated (`creates:`).
- Users and sudoers files are simply reconciled to the current state.

## Known quirks

- `google-authenticator` is interactive on some package versions; the
  playbook uses `ansible.builtin.expect` with a set of prompt patterns.
  If a package update changes the wording of a prompt, the task may hang
  until its timeout — add the missing pattern to `responses` in that case.
- Requires `python3-pexpect` on the **target** host (installed
  automatically by the playbook).