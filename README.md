Localynk 1.0.0 — Desktop & Android
Private LAN file sharing for Windows, Linux and Android, with mutual-TLS device pairing. Share a folder with your phone or another PC over your local network. No cloud, no accounts, no internet required.

This is the first public release. Versions 2.1 to 2.6.0 were internal pre-releases; the version number was reset to 1.0.0. The desktop app in this release carries audit rounds R14–R29; the Android app is audited through R24.

Download
File
Platform
Notes
Localynk-1.0.0-Windows-x64-Setup.exe	Windows 10 / 11 (x64)	Installer. Recommended. No Python needed.
Localynk-1.0.0-linux-amd64.deb	Debian / Ubuntu / Zorin (x86-64)	Recommended on Debian-family distros.
Localynk-1.0.0-linux-x86_64.tar.gz	Other x86-64 Linux	Same /opt/localynk layout as the .deb.
Localynk-1.0.0-x86_64.AppImage	x86-64 Linux	Run without installing. Only present if built.
Localynk-Mobile-1.0.0-signed.apk	Android 10+	Signed release APK. Pairs by scanning the desktop's QR code.
SHA256SUMS.txt	—	Checksums for all files above

Verify before running.

Windows (PowerShell):

text
Get-FileHash .\Localynk-1.0.0-Windows-x64-Setup.exe -Algorithm SHA256
Linux:

text
sha256sum -c SHA256SUMS.txt --ignore-missing
If Windows SmartScreen warns about an unrecognized publisher, the binary is not Authenticode-signed. Check the SHA-256 hash before choosing More info → Run anyway.

Install on Android
Copy Localynk-Mobile-1.0.0-signed.apk to the phone and open it, or sideload with adb:

text
adb install -r Localynk-Mobile-1.0.0-signed.apk
Android 10 or newer (minSdk 29, target SDK 36). The signed build updates older internal test builds in place (versionCode 10000 is higher than the old internal code 84).
Start sharing on the desktop and scan the QR code shown on its Home page. The QR carries a one-time token and pins the host's certificate fingerprint. A camera is required — there is no manual 6-digit code entry on Android yet.
Everything runs over mutual TLS 1.3 on the LAN. No account, no cloud, no internet connection involved.
Install on Linux
text
# Debian / Ubuntu
sudo apt install ./Localynk-1.0.0-linux-amd64.deb
localynk
# Tarball
tar xzf Localynk-1.0.0-linux-x86_64.tar.gz
cd localynk-1.0.0-linux-x86_64 && sudo ./install.sh
# AppImage
chmod +x Localynk-1.0.0-x86_64.AppImage && ./Localynk-1.0.0-x86_64.AppImage
The .deb and tarball install to /opt/localynk with a launcher at /usr/bin/localynk. Remove with sudo apt remove localynk or sudo ./uninstall.sh.

Headless server (Linux, no GUI): ./run-server.sh --folder ~/Shared runs a TLS 1.3 file server with a short-lived pairing code and a terminal QR, mDNS discovery, the connection guard and audit logging. LAN-only binding is the default; --all-interfaces is explicit opt-in.

Highlights
Host and client in one app. Share a folder, connect to another Localynk host, or do both at once.
Pair by QR code or 6-digit code. The QR carries a one-time token and pins the host's certificate fingerprint. After pairing, each device gets its own client certificate.
Mutual TLS 1.3 only. Every later connection needs the issued client certificate. Revoke any device from Trusted Devices.
Per-device permissions: read-only, read/write or full access.
File operations. Browse, upload, download, download folders as ZIP, create folders, rename and delete, all subject to the device's permission. Transfers are verified with SHA-256.
LAN discovery over mDNS, plus Find on LAN in the Connect dialog.
Folder sandbox. Path traversal, symlinks, NTFS alternate data streams and reserved device names are rejected.
Activity audit log (SQLite): every list, upload, download, rename, delete and pairing event, with device, IP, path, size and result. Exportable to CSV.
Managed firewall rule. Localynk opens only its active port and removes the rule on stop. Windows: netsh advfirewall, Private/Domain profiles, LocalSubnet; the uninstaller removes Localynk-* rules. Linux: ufw or iptables via pkexec, limited to the local IPv4 subnet (RFC1918/link-local) unless WAN mode is on. Both can be turned off in Settings or with firewall_managed: false.
Abuse protection. Global and per-IP connection caps, plus escalating temporary bans (5 min → 20 min → 80 min, capped at 24 h) for repeated auth or protocol failures.
Android client (new). The phone gets the same mutual-TLS 1.3 pairing and LFS/1 protocol: QR pairing, browse, upload, download, rename and delete per permission, foreground-service transfers, live access-level updates, client-certificate renewal and automatic re-finding of a restarted desktop.
Security hardening in this release
Pre-release audit rounds R14–R29 are included; the Android client is audited through R24. The R14–R20 fixes:

Sandbox checks every path component before resolving it, so symlink and junction escapes are blocked. ZIP creation no longer follows symlinks or junctions.
The manual 6-digit code locks after 10 wrong guesses in total, so an attacker cannot cancel a pairing by spamming bad codes.
Pairing can no longer overwrite an existing trusted-device record or reuse the host's own device ID.
Uploads are written to temp files and committed atomically after checksum validation. They can no longer overwrite a file created concurrently.
TLS handshake timeout reduced from 60 s to 5 s. Deeply nested JSON is rejected instead of crashing the handler.
The server TLS key is re-encrypted with a random in-memory password before it touches disk. New enrollment CAs use path_length=0.
Firewall rules are limited to the local subnet. The Rust firewall helper no longer searches PATH for netsh, ufw or iptables.
Firewall removal is now verified (ufw re-reads rules, iptables -C). A dismissed elevation prompt reports failure and the rule stays tracked so the next Stop retries. A ufw rule Localynk did not create is never overwritten or deleted, and 80/tcp no longer matches 8080/tcp.
IPv6 peers are rate-limited per /64, while link-local, ULA and loopback peers are keyed by full address, so one bad device cannot get the whole LAN banned.
Pillow upgraded to 13.x after pip-audit flagged advisories on 12.3.0.
R21–R29 (desktop)
Client-certificate renewal (RENEW_CERTIFICATE, R21): 825-day client certificates renew automatically within 90 days of expiry, same key only, over the existing mTLS channel. A different key is refused, a revoked device cannot renew, and renewal failures never break the user's request.
Revocation and permission changes bind immediately (R22/R23, verified in R29): every request re-loads the device record, and revoking a device aborts all of its live sessions, mid-transfer included.
The audit-log hash chain can no longer fork under concurrent writers (BEGIN IMMEDIATE, R22).
Python dependency locks are generated, fully pinned and SHA-256 hashed (R21).
Windows firewall rework (R21): one persistent rule Localynk-App-LAN scoped to Localynk.exe, Private/Domain profiles, LocalSubnet only, verified read-only at runtime — so Start/Stop/Exit need no UAC prompt and leave no per-port rule behind. Honest diagnostics when the helper is untrusted, a warning when every active network is classified Public, and the firewall helper must report the exact app version before anything is elevated.
Production-readiness audit (R29, desktop + Android): no exploitable issue found. One defense-in-depth fix: relocation candidates must now be LAN-scoped IP literals (private, CGNAT/Tailscale, link-local, loopback) — a rogue LAN device can no longer make the client dial an arbitrary advertised address. Documentation corrected on TLS 1.3 client-certificate timing: the chain must verify against the pinned CA before anything is sent, and no request or secret reaches an unverified host.
Android (R20–R24)
Pairing/TLS audited sound (R20): TLS 1.3 only; the trust-all manager exists only on the pairing socket and only until the QR SHA-256 fingerprint matches — before any token or CSR is sent. Host-id SAN, serverAuth EKU and SPKI pinning on every connection; the client key is AES-GCM-wrapped by an AndroidKeyStore key; the QR parser accepts private LAN literals only; VPN/cellular are refused; backups are off.
Client-certificate renewal, same rules as the desktop (R20): same key only, best effort, throttled, never blocks an operation.
Cleartext HTTP is removed as a path (R20): the manifest and network_security_config.xml forbid cleartext and accept system CAs only.
Parity with desktop R27/R28 (R21): Forget asks the desktop to revoke the phone (UNPAIR_DEVICE) and always wipes local credentials afterwards; a failed folder load searches the LAN for the saved host id and reconnects automatically after the same TLS/CA/SAN/key-pin verification — an impostor with the right id but a different CA fails, and nothing is saved. Only idempotent folder loads relocate; uploads, downloads, renames and deletes are never repeated automatically. At most one search per desktop per 20 s.
Live access levels (R21 + device-testing follow-up): the phone re-reads GET_DEVICE_INFO every 10 s while connected and after any "not permitted" refusal, and announces "Access changed by the desktop: now …". Polling stops in the background and when the desktop reports the phone revoked.
Human-readable sizes: "117.7 MB" instead of raw byte counts, in file rows and transfer progress.
Production audit R24, no code change needed: one exported component (launcher activity), allowBackup=false, downloads are length- and SHA-256-verified and a bad file is deleted, uploads sanitise filenames, and no Log call anywhere can leak a secret.
Reliability and performance
Large directories are paged (offset / limit / total) and listed off the event loop, with a 250,000-entry scan cap.
Cancelled or disconnected ZIP downloads stop the build and delete the partial archive. Stale temp files are cleaned at start.
Free disk space is checked before downloads and uploads. Host-supplied sizes and checksums are validated.
Remote Files now navigates folders whose names start with a dot (.git, .config) correctly. Rename, delete, upload and new-folder could previously act on the wrong folder.
Adaptive chunked I/O, cheaper ZIPs and scandir listings (see PERFORMANCE-R18.md).
Auto-reconnect after the host restarts (R28): the desktop host picks a new port on every start, so saved addresses went stale. A failed folder load now searches mDNS for the saved host id, verifies it, and reconnects automatically ("Host moved: reconnected at ip:port").
No more blank error dialogs (R24): background loads show an inline banner with Retry / Reconnect / Details instead of a modal; user actions get a titled dialog with cause and next step; cancellations are silent; uncaught GUI errors are logged and toasted instead of vanishing.
Users and Trusted Devices tables redraw only when the data changes — no more blinking buttons (R23); Trusted Devices groups repeated pairings from the same device under one row, with one-click "revoke older" (R26).
Rotating file log, single-instance guard, the preferred_interface setting is honored by the QR code, overview, mDNS record and firewall subnet, and a LAN-only server warns when its LAN address changes (R21).
Build and packaging
The Rust security library and firewall helper are bundled into the Windows standalone build and installer, and into the Linux standalone build, .deb and tarball. Nuitka's data-dir option skips binaries, so they are copied explicitly.
zeroconf is built as pure Python for frozen builds on every platform. This avoids a Cython/Nuitka class-size mismatch that appeared only after freezing. The build repairs an existing environment automatically.
The release gate runs the frozen executable's self-tests before any installer or package is produced. The Linux builder reads the finished .deb back and verifies its critical members.
The Linux→Windows cross-build script fails closed on an unverified Python installer, and the firewall helper must match the app version before the app will elevate it (R21).
Upgrading from a 2.x pre-release
Uninstall the old desktop build first. Windows: Settings → Apps → Localynk → Uninstall. Debian/Ubuntu: sudo apt remove localynk (APT treats 1.0.0 as a downgrade from 2.x). Paired devices and the trusted-device database live in the per-user data folder and survive the uninstall.

Android: internal 2.x test builds update in place — versionCode 10000 is higher than the old internal code 84, and paired desktops survive the update.

The LFS/1 wire protocol is unchanged, so existing Android clients keep working. The firewall helper must match the app version. The installer and packages ship both, so do not mix files from different releases.

Requirements
Windows 10 or 11, 64-bit; or x86-64 Linux with a desktop session (Debian/Ubuntu-family for the .deb)
ufw or iptables plus pkexec on Linux if you want Localynk to manage the firewall rule
A local network where devices can reach each other. Client isolation / "AP isolation" on some routers will block connections.
Android 10 or newer for the phone app, with a camera for QR pairing (no manual code entry yet).
Known limitations
Client certificates last 825 days and renew automatically within the final 90 days (desktop R21+ / Android R20+). A certificate that has already expired cannot be renewed in place — remove the pairing and pair again.
Audit-log purge can be re-sealed by someone who controls the local database. A full fix needs an external anchor.
Android: the legacy cleartext HTTP stack is still compiled (it fails closed — the platform refuses cleartext, nothing is sent) and is slated for removal in a follow-up release. R8/minification stays off for this release (it only obfuscates; the APK embeds no secrets) pending a full device pass.
Android pairing requires the camera; there is no manual 6-digit code entry yet. mDNS/NSD results are untrusted hints — pairing always requires the QR fingerprint.
Windows was verified on the build machine (installer and frozen self-tests). Linux packages and broader target-machine and device acceptance testing are ongoing, so please report anything unexpected.
Linux: ARM64 packages and macOS builds are not part of this release.
Reporting issues
Open an issue with your OS and version, the Localynk version (About page) and the log file from the Localynk data folder. For security issues, see SECURITY.md and report privately rather than in a public issue.
