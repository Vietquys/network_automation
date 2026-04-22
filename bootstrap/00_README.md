# BOOTSTRAP SCRIPTS - Manual Initialization (Phần 1)

## Mục đích
Cấu hình **TỐI THIỂU** qua cáp Console để Ansible Server có thể kết nối **SSH** đến tất cả thiết bị.

## Thứ tự triển khai (BẮT BUỘC)

| Bước | File                | Thiết bị | Lý do                                      |
|------|---------------------|----------|---------------------------------------------|
| 1    | `01_Router.ios`     | Router   | Gateway, cần bật trước để tạo path routing  |
| 2    | `02_CORE-1.ios`     | CORE-1   | Tạo VLAN 99, SVI, kết nối lên Router        |
| 3    | `03_CORE-2.ios`     | CORE-2   | Tạo VLAN 99, SVI, kết nối lên Router        |
| 4    | `04_AC-1.ios`       | AC-1     | SVI VLAN 99, trunk lên CORE                 |
| 5    | `05_AC-2.ios`       | AC-2     | SVI VLAN 99, trunk lên CORE                 |
| 6    | `06_AC-3.ios`       | AC-3     | SVI VLAN 99, trunk lên CORE                 |
| 7    | `07_AC-4.ios`       | AC-4     | SVI VLAN 99, trunk lên CORE                 |
| 8    | `08_AC-5.ios`       | AC-5     | SVI VLAN 99, trunk lên CORE                 |

## Cách sử dụng
1. Kết nối cáp Console vào thiết bị
2. Bật thiết bị → vào mode `enable` → `configure terminal`
3. Copy/paste toàn bộ nội dung file `.ios` tương ứng
4. Lưu cấu hình: `write memory`

## SSH credentials sau bootstrap
- Username: `Cisco123`
- Password: `Cisco123`
- Enable secret: `Cisco123`

> Các file bootstrap đã bao gồm `ip domain-name`, local user, RSA key và `transport input ssh`, nên không cần Telnet cho giai đoạn bootstrap.

## Lưu ý quan trọng

### Default Gateway cho Access Switches
- Bootstrap đặt `ip default-gateway 192.168.99.2` (IP thật của CORE-1)
- Sau khi Ansible cấu hình HSRP, sẽ đổi thành `192.168.99.1` (VIP)
- Lý do: HSRP VIP chưa tồn tại trong giai đoạn bootstrap

### Router-CORE Connectivity
- Bootstrap dùng **standalone interface** (e0/1 trên Router ↔ e0/0 trên CORE)
- Ansible sẽ nâng cấp thành **EtherChannel (LACP)** sau

### Static Route tạm thời
- Router: `ip route 192.168.99.0 255.255.255.0 10.0.0.2` (route đến VLAN 99 qua CORE-1)
- CORE-1/2: `ip route 0.0.0.0 0.0.0.0 10.0.0.x` (default route về Router)
- Ansible sẽ thay thế bằng OSPF

## Kiểm tra sau Bootstrap
```
! Từ Ansible Server:
ping 192.168.9.99    # Router
ping 192.168.99.2    # CORE-1
ping 192.168.99.3    # CORE-2
ping 192.168.99.11   # AC-1
ping 192.168.99.12   # AC-2
ping 192.168.99.13   # AC-3
ping 192.168.99.14   # AC-4
ping 192.168.99.15   # AC-5

# Kiểm tra cổng SSH
nc -zv 192.168.9.99 22
nc -zv 192.168.99.2 22
```

# cài đặt nc (nếu chưa có)
```bash
sudo apt update
sudo apt install netcat-openbsd -y
```
