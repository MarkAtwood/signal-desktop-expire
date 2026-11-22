# signal-desktop-expire

Delete Signal Desktop messages older than N days.

Signal Desktop lacks automatic message expiration. This tool fills that gap by deleting old messages and their attachments directly from the local database.

## Supported Platforms

| Platform | Key Storage | Status |
|----------|-------------|--------|
| macOS | Keychain | Tested |
| Linux (GNOME) | libsecret (GNOME Keyring) | Supported |
| Linux (KDE) | KWallet | Supported |
| Linux (Flatpak) | libsecret or KWallet | Supported |
| Linux (Snap) | libsecret or KWallet | Supported |
| Windows | DPAPI | Supported |

## Requirements

- Python 3.10+
- [sqlcipher](https://github.com/sqlcipher/sqlcipher) CLI
- Python cryptography library

### macOS

```bash
brew install sqlcipher
pip install cryptography
```

### Linux (Debian/Ubuntu)

For GNOME:
```bash
sudo apt install sqlcipher libsecret-tools
pip install cryptography
```

For KDE:
```bash
sudo apt install sqlcipher kwalletcli
pip install cryptography
```

### Windows

Download sqlcipher from the [official releases](https://github.com/nickhsharp/sqlcipher/releases) or build from source. Ensure it's in your PATH.

```bash
pip install cryptography
```

## Installation

```bash
git clone https://github.com/MarkAtwood/signal-desktop-expire.git
cd signal-desktop-expire
pip install cryptography
chmod +x signal-desktop-expire  # Unix only
```

## Usage

**Important**: Quit Signal Desktop before running. The database is locked while Signal is running.

```bash
# See what would be deleted (safe, no changes made)
./signal-desktop-expire --days 90 --dry-run

# Delete messages older than 90 days
./signal-desktop-expire --days 90

# Delete messages but keep attachment files
./signal-desktop-expire --days 90 --no-attachments

# Override database location
./signal-desktop-expire --days 90 --db /path/to/db.sqlite --config /path/to/config.json
```

On Windows, use `python signal-desktop-expire` instead of `./signal-desktop-expire`.

## Options

| Option | Description |
|--------|-------------|
| `--days N, -d N` | Delete messages older than N days (required) |
| `--dry-run, -n` | Show what would be deleted without making changes |
| `--no-attachments` | Don't delete attachment files, only database records |
| `--db PATH` | Override path to Signal's db.sqlite |
| `--config PATH` | Override path to Signal's config.json |
| `--attachments-dir PATH` | Override path to attachments.noindex directory |
| `--force, -f` | Run even if Signal appears to be running (may fail) |

## How it works

1. Retrieves the database encryption key from the system secret store:
   - macOS: Keychain
   - Linux: libsecret (GNOME Keyring)
   - Windows: DPAPI
2. Decrypts Signal's SQLCipher database key from config.json
3. Queries for messages and attachments older than N days
4. Deletes messages (attachment records cascade automatically)
5. Deletes orphaned attachment files from disk
6. Vacuums database to reclaim space

## Signal Desktop Data Locations

| Platform | Path |
|----------|------|
| macOS | `~/Library/Application Support/Signal/` |
| Linux (native) | `~/.config/Signal/` |
| Linux (Flatpak) | `~/.var/app/org.signal.Signal/config/Signal/` |
| Linux (Snap) | `~/snap/signal-desktop/current/.config/Signal/` |
| Windows | `%APPDATA%\Signal\` |

## Automation

### macOS (launchd)

Create `~/Library/LaunchAgents/com.signal.expire.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.signal.expire</string>
    <key>ProgramArguments</key>
    <array>
        <string>/path/to/signal-desktop-expire</string>
        <string>--days</string>
        <string>90</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Day</key>
        <integer>1</integer>
        <key>Hour</key>
        <integer>3</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>/tmp/signal-expire.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/signal-expire.log</string>
</dict>
</plist>
```

Then: `launchctl load ~/Library/LaunchAgents/com.signal.expire.plist`

### Linux (systemd timer)

Create `~/.config/systemd/user/signal-expire.service`:

```ini
[Unit]
Description=Expire old Signal Desktop messages

[Service]
Type=oneshot
ExecStart=/path/to/signal-desktop-expire --days 90
```

Create `~/.config/systemd/user/signal-expire.timer`:

```ini
[Unit]
Description=Monthly Signal message expiration

[Timer]
OnCalendar=monthly
Persistent=true

[Install]
WantedBy=timers.target
```

Then:
```bash
systemctl --user daemon-reload
systemctl --user enable --now signal-expire.timer
```

### Windows (Task Scheduler)

Use Task Scheduler to create a monthly task running:
```
python C:\path\to\signal-desktop-expire --days 90
```

**Note**: Signal must be quit for automated jobs to succeed.

## Safety

- Always use `--dry-run` first to see what would be deleted
- The tool refuses to run if Signal Desktop is running (unless `--force`)
- Database is vacuumed after deletion to reclaim space
- Attachment files are only deleted if they exist on disk

## Troubleshooting

### "Failed to get Signal password from secret storage" (Linux)

For GNOME:
1. Install `secret-tool`: `sudo apt install libsecret-tools`
2. Signal Desktop must have been run at least once (to create the key)
3. Your keyring must be unlocked (happens automatically on login)

For KDE:
1. Install `kwallet-query` or `kwalletcli`: `sudo apt install kwalletcli`
2. Signal Desktop must have been run at least once
3. KWallet must be unlocked

### "Database locked" error

Quit Signal Desktop completely. On Linux, check for background processes:
```bash
pkill -f signal-desktop
```

## License

MIT
