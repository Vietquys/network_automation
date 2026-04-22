# Dự án Tự động hóa Mạng Cisco với Ansible (Cisco Network Automation)

![Sơ đồ mạng](./config-all/Sơ%20Đồ.png)

## Tổng quan
Dự án triển khai tự động hóa cấu hình hạ tầng mạng Cisco (gồm 1 Router, 2 Core Switches L3, 5 Access Switches L2) từ trạng thái **thiết bị trắng** (blank config) cho đến khi **hoàn thiện 100%**, được chia thành 2 giai đoạn thực thi:
1. **Phần 1 - Bootstrap**: Cấu hình cơ sở thủ công (thông qua Console) để thiết lập kết nối IP và SSH, làm tiền đề cho hệ thống điều khiển.
2. **Phần 2 - Ansible Automation**: Sử dụng Ansible từ máy chủ điều khiển (Ubuntu) đẩy cấu hình toàn diện (VTP, VLAN, STP, EtherChannel, HSRP, OSPF, DHCP, SNMP...) theo dạng module hóa.

---

## Cấu trúc thư mục dự án

```text
network_automation/
├── config-all/                           # File cấu hình tĩnh cuối cùng để tham khảo/đối chiếu
├── bootstrap/                            # [Giai đoạn 1] Scripts Bootstrap
│   ├── 00_README.md
│   ├── 01_Router.ios
│   ├── 02_CORE-1.ios ~ 03_CORE-2.ios
│   └── 04_AC-1.ios ~ 08_AC-5.ios
└── ansible-network/                      # [Giai đoạn 2] Ansible Project (Chạy chính)
    ├── ansible.cfg                       # Cấu hình runtime mặc định cho Ansible
    ├── inventory.ini                     # Danh sách các thiết bị và nhóm (routers, core, access)
    ├── site.yml                          # Master playbook - Chạy tuần tự tất cả cấu hình
    ├── group_vars/                       # Biến dùng chung theo nhóm (routers, core, access, all)
    ├── host_vars/                        # Biến riêng rẽ cho từng host cụ thể
    ├── roles/                            # Mã nguồn Ansible được chia thành các tác vụ (base, vtp, vlan, ospf...)
    └── playbooks/                        # Các playbook cho phép chạy lẻ từng tính năng khi cần
```

---

## PHẦN 1: Bootstrap Scripts (`bootstrap/`)

### Mục đích
Áp dụng lượng cấu hình **tối thiểu nhất** qua đường Console để biến thiết bị trắng thành một Node có thể kết nối được qua **SSH** từ Ansible Server.

### Các thành phần chính được cấu hình
- Tên thiết bị (Hostname).
- Tài khoản quản trị (Username, Password, Enable Secret).
- Kích hoạt cổng Line VTY cho phép truy cập SSH (hoặc Telnet).
- IP Management tĩnh trên cổng SVI (VLAN 99) hoặc cổng vật lý.
- Cấu hình Trunk/Access tạm hoặc Static Route để định tuyến đường đi từ Ansible Server chạm tới được IP quản lý của thiết bị.
- *Lưu ý*: Các liên kết L2/L3 giữa các Switch giai đoạn này đều để dạng **Standalone** đơn lẻ, Ansible sẽ là người gom chúng lại thành LACP EtherChannel ở giai đoạn sau để tránh đứt gãy kết nối mạng.

---

## PHẦN 2: Cấu trúc Logic Ansible (`ansible-network/`)

### Luồng thực thi Master Playbook (`site.yml`)
`site.yml` tự động gọi các role tuần tự theo mức độ ưu tiên phụ thuộc về giao thức mạng (12 Plays):

1. **Base**: Đẩy cấu hình Hostname, thông tin đăng nhập, bảo mật.
2. **VTP+VLAN**: Áp dụng VTP Mode (Server trên Core, Client trên Access). Khởi tạo dải VLAN 10/20/99/100 trên Core.
3. **EtherChannel**: Gộp các cổng standalone thành Port-Channel bằng giao thức LACP.
4. **Trunk**: Phân phối Allowed VLANs chuẩn lên các cổng Trunk.
5. **STP**: Áp dụng Rapid-PVST+, set priority cho CORE-1 (Root) và CORE-2 (Secondary).
6. **SVI+HSRP**: Định tuyến Interface Vlan trên Core, chạy dự phòng HSRP v2. Set IP helper-address.
7. **SVI+GW Access**: Cấu hình SVI quản lý và IP Default-gateway cho dải Access.
8. **OSPF**: Chạy định tuyến động OSPF Area 0 giữa Core và Router, tự động gỡ bỏ các Static Route tạm của pha Bootstrap.
9. **DHCP**: Dựng Router làm DHCP Server cấp IP cho dải người dùng.
10. **Access Ports**: Phân bổ cổng downlink cho VLAN access, cấu hình PortFast + BPDU Guard.
11. **SNMP**: Kết nối các thiết bị với hệ thống giám sát Zabbix qua cấu hình SNMP Community/Traps.

---

## HƯỚNG DẪN TRIỂN KHAI VÀ CHẠY DỰ ÁN (Walkthrough)

### Bước 1: Chuẩn bị máy chủ Ansible (Ubuntu Server)
*Đảm bảo bạn có kết nối Internet để tải các công cụ cần thiết.*

```bash
# 1. Cài đặt các gói hệ thống
sudo apt update
sudo apt install -y git python3 python3-pip pipx sshpass

# 2. Cài đặt Ansible qua pipx
pipx install --include-deps ansible
pipx ensurepath
exec $SHELL -l

# 3. Tải bộ module chuẩn của Cisco cho Ansible
ansible-galaxy collection install cisco.ios ansible.netcommon

# 4. Tạo virtual env (best practice cho Ansible)
sudo apt update
sudo apt install python3-paramiko -y
```

### Bước 2: Lấy dự án về (Clone Repository)
Tải mã nguồn tại thư mục người dùng:

```bash
git clone https://github.com/Vietquys/network_automation.git
```

### Bước 3: Bootstap cấu hình lên thiết bị thật
1. Sử dụng dây Console kết nối vào thiết bị phần cứng thật (hoặc EVE-NG/GNS3 Console).
2. Copy toàn bộ nội dung từng file trong `bootstrap/*.ios` tương ứng dán (paste) trực tiếp vào thiết bị ở chế độ Global Configuration (`conf t`).
3. Từ Ubuntu Server, đảm bảo có thể **ping** thành công đến IP của các node mạng.

### Bước 4: Kiểm tra trạng thái Inventory
> **QUAN TRỌNG:** Luôn phải di chuyển (cd) vào thư mục `ansible-network` trước khi gõ lệnh liên quan tới Ansible để Ansible đọc đúng file cấu hình `ansible.cfg` khu vực.

```bash
cd ~/network_automation/ansible-network

# 1. Kiểm tra Ansible đọc danh sách thiết bị thành công chưa
ansible-inventory -i inventory.ini --graph

# 2. Ping thông qua thư viện mạng của Ansible (đảm bảo SSH OK)
ansible network -i inventory.ini -m ansible.builtin.ping

# 3. Lệnh test lấy show version
ansible network -i inventory.ini -m cisco.ios.ios_command -a "commands='show version'"
```

### Bước 5: Đẩy tự động hóa toàn phần (Ansible Apply)
Bạn có thể cho chạy giả lập (Dry-run) để Ansible so sánh cấu hình hiện tại với mong muốn trước khi áp dụng:

```bash
# Kiểm thử sự thay đổi không lưu lại cấu hình (--check)
ansible-playbook -i inventory.ini site.yml --check --diff

# 🔥 ÁP DỤNG CẤU HÌNH THẬT
ansible-playbook -i inventory.ini site.yml
```

**Chạy riêng lẻ từng phần (khi muốn fix lỗi hoặc cấu hình bổ sung tính năng cục bộ):**
```bash
ansible-playbook -i inventory.ini playbooks/pb_vtp.yml
ansible-playbook -i inventory.ini playbooks/pb_etherchannel.yml
ansible-playbook -i inventory.ini playbooks/pb_ospf.yml
ansible-playbook -i inventory.ini playbooks/pb_dhcp.yml
# ... xem thêm các file khác tại thư mục playbooks/
```

---

## Xác minh kết quả hệ thống sau khi chạy (Verification)

Sử dụng trực tiếp Ansible để lấy lệnh show đồng loạt từ thiết bị thay vì truy cập thủ công:

```bash
# Kiểm tra VTP Status trên vùng Core
ansible core -i inventory.ini -m cisco.ios.ios_command -a "commands='show vtp status'"

# Kiểm tra Spanning-Tree tổng quan
ansible core:access -i inventory.ini -m cisco.ios.ios_command -a "commands='show spanning-tree summary'"

# Kiểm tra EtherChannel Trunking
ansible routers:core -i inventory.ini -m cisco.ios.ios_command -a "commands='show etherchannel summary'"

# Kiểm tra trạng thái HSRP (Active/Standby)
ansible core -i inventory.ini -m cisco.ios.ios_command -a "commands='show standby brief'"

# Kiểm tra Neighbor của OSPF
ansible routers:core -i inventory.ini -m cisco.ios.ios_command -a "commands='show ip ospf neighbor'"

# Kiểm tra trạng thái cấp DHCP
ansible routers -i inventory.ini -m cisco.ios.ios_command -a "commands='show ip dhcp binding'"

# Kiểm tra máy chủ SNMP Traps
ansible network -i inventory.ini -m cisco.ios.ios_command -a "commands='show snmp host'"
```
