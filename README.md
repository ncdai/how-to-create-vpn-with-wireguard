# Tự xây dựng VPN cho riêng mình với WireGuard

## Chuẩn bị

```bash
apt-get update && apt-get upgrade
```

Tiếp theo, chúng ta cần kích hoạt **chuyển tiếp IP**.
> Chuyển tiếp IP là khả năng hệ điều hành chấp nhận các gói mạng đến trên một giao diện, nhận biết rằng nó không dành cho bản thân hệ thống mà nó phải được truyền cho một mạng khác.

```bash
nano /etc/sysctl.conf
```

Tìm dòng `net.ipv4.ip_forward=1` và bỏ dấu `#`

Kích hoạt các thay đổi

```bash
sysctl -p
```

## Cài đặt WireGuard Server

```bash
apt install wireguard
cd /etc/wireguard
```

## Tạo Public và Private Key

```bash
KEY_NAME="client"
wg genkey | tee ${KEY_NAME}-private.key | wg pubkey > ${KEY_NAME}-public.key

KEY_NAME="server"
wg genkey | tee ${KEY_NAME}-private.key | wg pubkey > ${KEY_NAME}-public.key
```

Chạy lần lượt các lệnh sau (ghi chú lại các key)

```bash
cat server-private.key
cat server-public.key
cat client-private.key
cat client-public.key
```

Lấy tên card mạng mặc định
> Ubuntu thì card mạng mặc định là `eth0`

```bash
ip route show | grep default
# default via 165.22.48.1 dev eth0 proto static
```

## Config

Tạo file config `wg0.conf`

```bash
nano /etc/wireguard/wg0.conf
```

Nội dung file

> Thay thế `eth0` nếu tên card mạng của bạn khác nhé!

> AllowedIPs phải tăng số theo số lượng client mà bạn có. Ví dụ 10.8.77.3, hoặc 10.8.77.4 và cứ tăng dần nhé. Tối đa là 10.8.77.254

```bash
[Interface]
PrivateKey = server_private_key
Address = 10.8.77.1/24
SaveConfig = true
ListenPort = 51820
PostUp = /usr/sbin/iptables -A FORWARD -i wg0 -j ACCEPT; /usr/sbin/iptables -t nat -A POSTROUTING -s 10.8.77.0/24 -o eth0 -j MASQUERADE
PostDown = /usr/sbin/iptables -D FORWARD -i wg0 -j ACCEPT; /usr/sbin/iptables -t nat -D POSTROUTING -s 10.8.77.0/24 -o eth0 -j MASQUERADE

[Peer]
PublicKey = client_public_key
AllowedIPs = 10.8.77.2 
```

Tạo file `client.conf`
> Dùng cho máy ngoài kết nối tới WireGuard Server

```bash
nano /etc/wireguard/client.conf
```

Nội dung file

>  Thay đổi Endpoint : IP máy chủ của bạn

```bash
[Interface]
PrivateKey = client_private_key
Address = 10.8.77.2/24
DNS = 1.1.1.1, 8.8.8.8

[Peer]
PublicKey = server_public_key
AllowedIPs = 0.0.0.0/0
Endpoint = 128.199.70.5:51820
PersistentKeepalive = 24
```

```bash
# Kích hoạt config
wg-quick up wg0

# Kích hoạt WireGuard tự hoạt động khi khởi động
systemctl enable wg-quick@wg0

# Kiểm tra xem WireGuard Server đã chạy ngon lành chưa 
wg
```

## QR Code

```bash
apt install qrencode
qrencode -r client.conf -o wireguard-client-qrcode.png
python3 -m http.server 7777

# Tải QR Code
# http://IP_SERVER:7777/wireguard-client-qrcode.png
```

## Kết nối VPN

**iOS, Android**

- https://apps.apple.com/vn/app/wireguard/id1441195209?l=vi
- https://play.google.com/store/apps/details?id=com.wireguard.android&hl=vi&gl=US

Thêm cấu hình VPN bằng cách quét QR Code

> Android : Hãy đảm bảo bật Exclude Private IPs nếu không bạn sẽ không thể truy cập các máy chủ mạng LAN

**macOS, Linux, Windows**

- https://www.wireguard.com/install/

Lấy nội dung cấu hình cho máy tính

```bash
cat /etc/wireguard/client.conf
```
