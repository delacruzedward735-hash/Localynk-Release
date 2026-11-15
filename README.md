<!--
GitHub release form:
  Tag:     v1.0.0
  Title:   Localynk Desktop 1.0.0 — Private LAN file sharing
  Target:  main
  Assets:  Localynk-1.0.0-Windows-x64-Setup.exe
           Localynk-1.0.0-linux-amd64.deb
           Localynk-1.0.0-linux-x86_64.tar.gz
           Localynk-1.0.0-x86_64.AppImage   (only if built)
           SHA256SUMS.txt
-->

# Localynk Desktop 1.0.0

**Private LAN file sharing for Windows and Linux, with mutual-TLS device pairing.** Share a folder with your phone or another PC over your local network. No cloud, no accounts, no internet required.

This is the first public release. Versions 2.1 to 2.6.0 were internal pre-releases; the version number was reset to **1.0.0**.

## Download

| File | Platform | Notes |
|---|---|---|
| `Localynk-1.0.0-Windows-x64-Setup.exe` | Windows 10 / 11 (x64) | Installer. Recommended. No Python needed. |
| `Localynk-1.0.0-linux-amd64.deb` | Debian / Ubuntu / Zorin (x86-64) | Recommended on Debian-family distros. |
| `Localynk-1.0.0-linux-x86_64.tar.gz` | Other x86-64 Linux | Same `/opt/localynk` layout as the `.deb`. |
| `Localynk-1.0.0-x86_64.AppImage` | x86-64 Linux | Run without installing. Only present if built. |
| `SHA256SUMS.txt` | — | Checksums for all files above |

Verify before running.

Windows (PowerShell):

```powershell
Get-FileHash .\Localynk-1.0.0-Windows-x64-Setup.exe -Algorithm SHA256
```

Linux:

```bash
sha256sum -c SHA256SUMS.txt --ignore-missing
```

> If Windows SmartScreen warns about an unrecognized publisher, the binary is not Authenticode-signed. Check the SHA-256 hash before choosing **More info → Run anyway**.

### Install on Linux

```bash
# Debian / Ubuntu
sudo apt install ./Localynk-1.0.0-linux-amd64.deb
localynk

# Tarball
tar xzf Localynk-1.0.0-linux-x86_64.tar.gz
cd localynk-1.0.0-linux-x86_64 && sudo ./install.sh

# AppImage
chmod +x Localynk-1.0.0-x86_64.AppImage && ./Localynk-1.0.0-x86_64.AppImage
```

The `.deb` and tarball install to `/opt/localynk` with a launcher at `/usr/bin/localynk`. Remove with `sudo apt remove localynk` or `sudo ./uninstall.sh`.

**Headless server (Linux, no GUI):** `./run-server.sh --folder ~/Shared` runs a TLS 1.3 file server with a short-lived pairing code and a terminal QR, mDNS discovery, the connection guard and audit logging. LAN-only binding is the default; `--all-interfaces` is explicit opt-in.

## Highlights

- **Host and client in one app.** Share a folder, connect to another Localynk host, or do both at once.
- **Pair by QR code or 6-digit code.** The QR carries a one-time token and pins the host's certificate fingerprint. After pairing, each device gets its own client certificate.
- **Mutual TLS 1.3 only.** Every later connection needs the issued client certificate. Revoke any device from **Trusted Devices**.
- **Per-device permissions:** read-only, read/write or full access.
- **File operations.** Browse, upload, download, download folders as ZIP, create folders, rename and delete, all subject to the device's permission. Transfers are verified with SHA-256.
- **LAN discovery** over mDNS, plus **Find on LAN** in the Connect dialog.
- **Folder sandbox.** Path traversal, symlinks, NTFS alternate data streams and reserved device names are rejected.
- **Activity audit log** (SQLite): every list, upload, download, rename, delete and pairing event, with device, IP, path, size and result. Exportable to CSV.
- **Managed firewall rule.** Localynk opens only its active port and removes the rule on stop. Windows: `netsh advfirewall`, Private/Domain profiles, `LocalSubnet`; the uninstaller removes `Localynk-*` rules. Linux: `ufw` or `iptables` via `pkexec`, limited to the local IPv4 subnet (RFC1918/link-local) unless WAN mode is on. Both can be turned off in **Settings** or with `firewall_managed: false`.
- **Abuse protection.** Global and per-IP connection caps, plus escalating temporary bans (5 min → 20 min → 80 min, capped at 24 h) for repeated auth or protocol failures.

## Security hardening in this release

Pre-release audit rounds R14–R20 are included. Notable fixes:

- Sandbox checks every path component *before* resolving it, so symlink and junction escapes are blocked. ZIP creation no longer follows symlinks or junctions.
- The manual 6-digit code locks after 10 wrong guesses in total, so an attacker cannot cancel a pairing by spamming bad codes.
- Pairing can no longer overwrite an existing trusted-device record or reuse the host's own device ID.
- Uploads are written to temp files and committed atomically after checksum validation. They can no longer overwrite a file created concurrently.
- TLS handshake timeout reduced from 60 s to 5 s. Deeply nested JSON is rejected instead of crashing the handler.
- The server TLS key is re-encrypted with a random in-memory password before it touches disk. New enrollment CAs use `path_length=0`.
- Firewall rules are limited to the local subnet. The Rust firewall helper no longer searches `PATH` for `netsh`, `ufw` or `iptables`.
- Firewall removal is now verified (ufw re-reads rules, iptables `-C`). A dismissed elevation prompt reports failure and the rule stays tracked so the next Stop retries. A `ufw` rule Localynk did not create is never overwritten or deleted, and `80/tcp` no longer matches `8080/tcp`.
- IPv6 peers are rate-limited per /64, while link-local, ULA and loopback peers are keyed by full address, so one bad device cannot get the whole LAN banned.
- Pillow upgraded to 13.x after `pip-audit` flagged advisories on 12.3.0.

## Reliability and performance

- Large directories are paged (`offset` / `limit` / `total`) and listed off the event loop, with a 250,000-entry scan cap.
- Cancelled or disconnected ZIP downloads stop the build and delete the partial archive. Stale temp files are cleaned at start.
- Free disk space is checked before downloads and uploads. Host-supplied sizes and checksums are validated.
- Remote Files now navigates folders whose names start with a dot (`.git`, `.config`) correctly. Rename, delete, upload and new-folder could previously act on the wrong folder.
- Adaptive chunked I/O, cheaper ZIPs and `scandir` listings (see `PERFORMANCE-R18.md`).

## Build and packaging

- The Rust security library and firewall helper are bundled into the Windows standalone build and installer, and into the Linux standalone build, `.deb` and tarball. Nuitka's data-dir option skips binaries, so they are copied explicitly.
- zeroconf is built as pure Python for frozen builds on every platform. This avoids a Cython/Nuitka class-size mismatch that appeared only after freezing. The build repairs an existing environment automatically.
- The release gate runs the frozen executable's self-tests before any installer or package is produced. The Linux builder reads the finished `.deb` back and verifies its critical members.

## Upgrading from a 2.x pre-release

Uninstall the old build first. Windows: **Settings → Apps → Localynk → Uninstall**. Debian/Ubuntu: `sudo apt remove localynk` (APT treats 1.0.0 as a downgrade from 2.x). Paired devices and the trusted-device database live in the per-user data folder and survive the uninstall.

The `LFS/1` wire protocol is unchanged, so existing Android clients keep working. The firewall helper must match the app version. The installer and packages ship both, so do not mix files from different releases.

## Requirements

- Windows 10 or 11, 64-bit; or x86-64 Linux with a desktop session (Debian/Ubuntu-family for the `.deb`)
- `ufw` or `iptables` plus `pkexec` on Linux if you want Localynk to manage the firewall rule
- A local network where devices can reach each other. Client isolation / "AP isolation" on some routers will block connections.

## Known limitations

- Client-certificate renewal is not implemented. Certificates last 825 days.
- Audit-log purge can be re-sealed by someone who controls the local database. A full fix needs an external anchor.
- Android legacy HTTP client removal and R8 re-enable are still pending on the Android side.
- Windows was verified on the build machine (installer and frozen self-tests). Linux packages and broader target-machine and device acceptance testing are ongoing, so please report anything unexpected.
- Linux: ARM64 packages and macOS builds are not part of this release.

## Reporting issues

Open an issue with your OS and version, the Localynk version (**About** page) and the log file from the Localynk data folder. For security issues, see `SECURITY.md` and report privately rather than in a public issue.
