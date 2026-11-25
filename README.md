# CC Auto APT archive


Kho APT của dự án cung cấp gói `cc-auto` bản Go cho amd64 và arm64.
Chạy lệnh dưới đây từ tài khoản sẽ sử dụng Claude Code:

```sh
sudo install -d -m 0755 /etc/apt/keyrings
key=$(mktemp)
curl -fsSL https://cuongnguyengit.github.io/cc-auto-apt/apt/cc-auto-archive-keyring.gpg -o "$key" &&
    printf '%s  %s\n' 'ffdf76f4e3e1580b722f97f23ab203170b6d0b1cafdf14124ab06ccc9c7bba80' "$key" | sha256sum -c - &&
    sudo install -m 0644 "$key" /etc/apt/keyrings/cc-auto.gpg &&
    printf '%s\n' 'deb [signed-by=/etc/apt/keyrings/cc-auto.gpg] https://cuongnguyengit.github.io/cc-auto-apt/apt ./' |
    sudo tee /etc/apt/sources.list.d/cc-auto.list >/dev/null
rm -f "$key"
sudo apt update
sudo apt install cc-auto
cc-auto-setup
```

Cài Claude Code CLI riêng theo [hướng dẫn của Anthropic](https://code.claude.com/docs/en/setup).
Lệnh `cc-auto-setup` cấu hình shell, VS Code và systemd cho user hiện tại và
có cùng các tùy chọn như `install.sh` bên dưới. Mở terminal mới sau khi setup.
Sau khi cập nhật gói, chạy lại `cc-auto-setup` để làm mới shell integration và
khởi động lại router. Trước khi gỡ gói, tắt dịch vụ đã bật bằng
`systemctl --user disable --now claude-account-router.service`.


## Go 1.3.0

Native Go binaries; Python is not required. Existing profiles are preserved.

```sh
sudo apt update
sudo apt install --only-upgrade cc-auto
cc-auto-setup
# Reopen your terminal after setup.
cc-account --help
cc-auto --help
cc-auto check --watch -n 0.1
cc-account doctor
```

Includes complete CLI examples, per-account 5h/week routing limits, weekly pacing, priority/sticky/round-robin/least-used strategies, adaptive/static forecasting, and diagnostics. Watch refreshes the display while respecting quota polling and backoff.

See [the CLI guide](docs/cli.md). Windows, macOS, and Linux archives are available in this repository's Releases.

## macOS via Homebrew

CC Auto 1.3.1 fixes Keychain-backed quota monitoring on macOS (Apple Silicon and Intel):

```sh
brew install cuongnguyengit/tap/cc-auto
cc-auto-setup
```

To update: `brew update`, `brew upgrade cuongnguyengit/tap/cc-auto`, then `cc-auto-setup`.
Reopen your terminal and run `cc-auto check`. Monitoring preserves profiles; per-process routing remains independent of the configured VS Code profile shown as Active.

## Go 1.3.2: resumed-session recovery fix

APT and Homebrew provide 1.3.2. Resumed Claude terminal/VS Code conversations reclaim recovery state from exited managed launchers; live owners stay protected. New OS lifetime locks support crash recovery. Existing profiles and conversations are preserved.

Ubuntu: `sudo apt update && sudo apt install --only-upgrade cc-auto`, then `cc-auto-setup`.
Mac: `brew update && brew upgrade cuongnguyengit/tap/cc-auto`, then `cc-auto-setup`.

Reopen terminals/new VS Code processes after updating integrations. Legacy retained terminal directories without a lifetime lock remain protected if liveness is unknown. See the CLI guide for details.

## Go 1.3.3: legacy Linux ownership and diagnostics

APT and Homebrew now provide 1.3.3. Linux verifies same-user processes before reclaiming retained folders from older versions without lifetime locks. A surviving Claude child remains protected even if its launcher exited. Unknown ownership is distinguished from verified live ownership. `cc-auto recover status --json` includes owner state/PIDs; doctor flags stale or unknown bindings.

Ubuntu: `sudo apt update && sudo apt install --only-upgrade cc-auto`, then `cc-auto-setup`.
Mac: `brew update && brew upgrade cuongnguyengit/tap/cc-auto`, then `cc-auto-setup`.

Resume in a new managed process after upgrading. Existing profiles and credentials are preserved. The legacy /proc reclamation applies to Linux; other platforms retain conservative protection for legacy folders without owner locks.
