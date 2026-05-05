# SSH Setup Handoff — Lee PC → NUC-PC

## Goal
Configure Claude Code on the **Lee PC** (personal Windows computer) to SSH into the **NUC-PC** (a second Windows laptop on the same Wi-Fi) so Claude can drive that machine remotely.

## Current state — what's WORKING
- OpenSSH Server is installed, running, and set to auto-start on **NUC-PC**
- Windows Firewall rule `sshd` allows TCP port 22 inbound on NUC-PC
- NUC-PC's IPv4: `192.168.5.197` (LAN only)
- An ed25519 keypair was generated on NUC-PC at `C:\Users\leehu\.ssh\id_ed25519` (with a passphrase)
- The matching public key is installed on NUC-PC at `C:\ProgramData\ssh\administrators_authorized_keys` with correct ACLs (Administrators:F, SYSTEM:F, inheritance disabled)
- The private key was `scp`'d from NUC-PC to Lee PC at `C:\Users\Lee\.ssh\id_ed25519` (444 bytes, file size shows 1 KB on disk — normal cluster rounding)
- **From Lee PC, manual `ssh -i C:\Users\Lee\.ssh\id_ed25519 leehu@192.168.5.197` works** — passphrase prompt, then drops into a `leehu@NUC-PC C:\Users\leehu>` shell on NUC-PC

## Current state — what's FAILING
Claude Code on Lee PC, with these SSH-connection values:
- Friendly name: `NUC_PC`
- SSH host: `192.168.5.197`
- SSH port: `22`
- Identity file: `C:\Users\Lee\.ssh\id_ed25519`
- Username: `leehu`

Returns: **"Connection failed: All configured authentication methods failed"** and "Something went wrong / Select a folder first."

## Diagnosis (most likely)
Claude's SSH client cannot interactively prompt for the **key passphrase**, so when it tries to use the passphrase-encrypted private key, it falls through to other methods, none of which succeed → "All configured authentication methods failed". Manual `ssh` works because PowerShell can prompt for the passphrase.

## Proposed fixes (pick one)

### Path 1 — ssh-agent (recommended, keeps passphrase security)

On **Lee PC**:

1. Admin PowerShell:
   ```powershell
   Get-Service ssh-agent | Set-Service -StartupType Automatic
   Start-Service ssh-agent
   ```
2. Normal PowerShell:
   ```powershell
   ssh-add C:\Users\Lee\.ssh\id_ed25519
   ```
   Enter passphrase once. Should report `Identity added`.
3. Test (no `-i`, no prompt expected):
   ```powershell
   ssh leehu@192.168.5.197
   ```
4. Re-try Claude's SSH connection.

### Path 2 — regenerate key without passphrase (faster, slightly less secure)

On **Lee PC**:

1. Generate new key, no passphrase:
   ```powershell
   ssh-keygen -t ed25519 -C "lee-pc-to-nuc" -f $env:USERPROFILE\.ssh\id_ed25519_nopass
   ```
2. Print public key:
   ```powershell
   type $env:USERPROFILE\.ssh\id_ed25519_nopass.pub
   ```
3. On **NUC-PC** (admin PowerShell), append that public key to `C:\ProgramData\ssh\administrators_authorized_keys`:
   ```powershell
   Add-Content -Path "C:\ProgramData\ssh\administrators_authorized_keys" -Value "<paste public key line here>"
   icacls "C:\ProgramData\ssh\administrators_authorized_keys" /inheritance:r /grant "Administrators:F" /grant "SYSTEM:F"
   Restart-Service sshd
   ```
4. Update Claude's identity-file field to `C:\Users\Lee\.ssh\id_ed25519_nopass` and retry.

## Reference values

| Item | Value |
|---|---|
| Server (target) hostname | NUC-PC |
| Server IP (LAN) | 192.168.5.197 |
| Server username | leehu |
| Server SSH port | 22 |
| Server public key file | C:\ProgramData\ssh\administrators_authorized_keys |
| Client (Lee PC) username | Lee |
| Client private key path | C:\Users\Lee\.ssh\id_ed25519 |
| Public key currently installed | `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBvYZ9122ZKRZD0uCy81dkrDQkOOnyhYW8VRJTYq2t2O NUC_963` |

## Other notes
- Both machines must be on the same Wi-Fi for the LAN IP to work. For remote-network access later, consider Tailscale.
- "Select a folder first" is unrelated to auth — it's Claude asking you to pick a working directory on NUC-PC after a successful connection.
- Windows OpenSSH for **administrator** users requires keys be in `C:\ProgramData\ssh\administrators_authorized_keys`, not `~/.ssh/authorized_keys`. Already handled.

## Suggested next step for fresh Claude Code session
Confirm `Get-Service ssh-agent` status on Lee PC, then walk the user through Path 1. If ssh-agent doesn't make Claude's connection succeed, fall back to Path 2.
