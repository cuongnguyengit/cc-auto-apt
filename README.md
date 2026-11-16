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

