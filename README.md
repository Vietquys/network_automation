# Hướng dẫn triển khai project Network Automation trên Ubuntu Server (Ansible Server)
# Network Automation với Ansible (Ubuntu Server)

Tài liệu này tập trung đúng mục tiêu: **đưa đúng file vào đúng chỗ trên Ubuntu Server** và **chạy Ansible để apply cấu hình lên thiết bị mạng** theo cấu trúc repo đã có.
Tài liệu này hướng dẫn triển khai thực tế repo `network_automation` để cấu hình thiết bị Cisco IOS bằng Ansible theo đúng luồng:

> **Giả định:** bạn đã cài Ansible + collection Cisco IOS (`cisco.ios`) trên Ubuntu Server, và các thiết bị đã bootstrap để SSH/Telnet từ Ansible Server.
**bootstrap ban đầu → kiểm tra kết nối → dry-run → apply → verify**.

---

## Phần 1: Chuẩn bị trên Ubuntu Server
## 1) Tổng quan cấu trúc repo

### 1.1 Tạo thư mục triển khai chuẩn
```text
network_automation/
├── ansible-network/
│   ├── ansible.cfg
│   ├── inventory.ini
│   ├── site.yml
│   ├── group_vars/
│   ├── host_vars/
│   ├── roles/
│   └── playbooks/
├── bootstrap/
│   ├── 00_README.md
│   └── *.ios
├── walkthrough.md
└── README.md
```

Chọn 1 thư mục làm workspace, ví dụ `/opt/automation`:
### Ý nghĩa từng thư mục chính

```bash
sudo mkdir -p /opt/automation
sudo chown -R $USER:$USER /opt/automation
```
- **`ansible-network/`**: thư mục chạy Ansible chính.
  - `ansible.cfg`: cấu hình Ansible, dùng inventory tương đối `inventory.ini`.
  - `inventory.ini`: danh sách thiết bị và nhóm (`routers`, `core`, `access`, `network`).
  - `group_vars/`, `host_vars/`: biến theo nhóm/thiết bị.
  - `roles/`: các role theo từng mảng cấu hình (base, vtp, vlan, ospf, dhcp, snmp...).
  - `site.yml`: playbook tổng để triển khai toàn bộ theo đúng thứ tự phụ thuộc.
  - `playbooks/`: playbook riêng cho từng mảng (VTP, STP, OSPF, DHCP, ...).

**Vì sao đặt ở đây:**
- Dễ quản trị trên server (tách biệt với thư mục hệ thống).
- Thuận tiện backup/đồng bộ toàn bộ project.
- **`bootstrap/`**: cấu hình khởi tạo thủ công (console) cho thiết bị trước khi tự động hóa.

**Kết quả mong đợi:** có thư mục `/opt/automation` và user hiện tại có quyền ghi.
- **`walkthrough.md`**: tài liệu diễn giải luồng triển khai lab.

---

## Phần 2: Đưa file vào đúng vị trí
## 2) Giải thích riêng thư mục `bootstrap`

### 2.1 Mục tiêu cấu trúc trên Ubuntu Server
`bootstrap/` là bước **initial config** cho thiết bị mới/blank để đạt điều kiện tối thiểu:

Sau khi copy xong, nên có cấu trúc:
- Có IP quản trị.
- Có quyền truy cập từ Ansible Server (SSH/Telnet tùy lab).
- Có route/kết nối cơ bản để Ansible chạm được vào thiết bị.

```text
/opt/automation/
├── config-all/
├── bootstrap/
└── ansible-network/
    ├── ansible.cfg
    ├── inventory.ini
    ├── site.yml
    ├── group_vars/
    ├── host_vars/
    ├── roles/
    └── playbooks/
```
### Khi nào dùng?

### 2.2 Copy thư mục `config-all`
- Dùng **trước** Ansible, khi thiết bị chưa thể remote.
- Sau khi bootstrap xong và ping/SSH được, mới chuyển sang chạy trong `ansible-network/`.

```bash
cp -a /duong-dan-nguon/automation/config-all /opt/automation/
```
> Luồng chuẩn: **bootstrap → ansible**.

**Đặt ở đâu:** `/opt/automation/config-all`  
**Vì sao:** đây là bộ cấu hình đích để đối chiếu/so sánh sau triển khai, không phải thư mục Ansible runtime.
---

**Kết quả mong đợi:**
- Có thư mục `/opt/automation/config-all`.
- Chứa file tổng hợp cấu hình mong muốn (theo thiết kế của team).
## 3) Cài môi trường trên Ubuntu Server (copy-paste)

> **Lưu ý:** trong bản repo hiện tại bạn gửi, thư mục `config-all` không có sẵn trong working tree. Bạn vẫn cần copy từ nguồn lưu trữ nội bộ nếu team đang giữ riêng.
> Khuyến nghị Ubuntu 22.04/24.04, chạy bằng user có quyền `sudo`.

### 2.3 Copy thư mục `bootstrap`
### Bước 1: Cài Git + Python + pipx

```bash
cp -a /duong-dan-nguon/automation/bootstrap /opt/automation/
sudo apt update
sudo apt install -y git python3 python3-pip pipx sshpass
pipx ensurepath
exec $SHELL -l
```

**Đặt ở đâu:** `/opt/automation/bootstrap`  
**Vì sao:** lưu script bootstrap để khởi tạo thiết bị trắng (console), phục vụ pre-stage trước khi Ansible apply full.

**Kết quả mong đợi:** có các file `.ios` và `00_README.md` trong `bootstrap/`.

### 2.4 Copy thư mục `ansible-network`
### Bước 2: Cài Ansible bằng pipx

```bash
cp -a /duong-dan-nguon/automation/ansible-network /opt/automation/
pipx install --include-deps ansible
```

**Đặt ở đâu:** `/opt/automation/ansible-network`  
**Vì sao:** đây là root Ansible project; `ansible.cfg` đang trỏ inventory theo đường dẫn tương đối (`inventory = inventory.ini`), nên chạy lệnh từ đúng thư mục này sẽ đơn giản và đúng ngữ cảnh.

**Kết quả mong đợi:** đủ các thành phần `ansible.cfg`, `inventory.ini`, `site.yml`, `group_vars/`, `host_vars/`, `roles/`, `playbooks/`.

---

## Phần 3: Kiểm tra cấu trúc
> Nếu không dùng pipx, có thể dùng pip:
>
> ```bash
> python3 -m pip install --user ansible
> ```

Di chuyển vào project và kiểm tra nhanh:
### Bước 3: Kiểm tra version

```bash
cd /opt/automation/ansible-network
pwd
ls -la
find group_vars host_vars roles playbooks -maxdepth 2 -type f
git --version
python3 --version
ansible --version
ansible-playbook --version
ansible-galaxy --version
```

### 3.1 Kiểm tra `inventory.ini`
### Bước 4: Cài collection cần thiết

```bash
sed -n '1,200p' inventory.ini
ansible-galaxy collection install cisco.ios ansible.netcommon
ansible-galaxy collection list | egrep 'cisco\.ios|ansible\.netcommon'
```

**Cần thấy:** các nhóm `routers`, `core`, `access`, `network:children`, `switches:children` với IP đúng lab thực tế.
---

### 3.2 Kiểm tra `group_vars/`
## 4) Clone repo và giữ đúng cấu trúc

### Cách clone (khuyến nghị `/opt/automation`)

```bash
ls -la group_vars
sudo mkdir -p /opt/automation
sudo chown -R $USER:$USER /opt/automation
cd /opt/automation
git clone https://github.com/Vietquys/network_automation.git
cd network_automation
```

**Cần thấy file chính:**
- `all.yml`
- `routers.yml`
- `core.yml`
- `access.yml`

**Ý nghĩa thực thi:** tham số dùng chung theo nhóm thiết bị.

### 3.3 Kiểm tra `host_vars/`
### Hoặc clone tại home

```bash
ls -la host_vars
mkdir -p ~/automation
cd ~/automation
git clone https://github.com/Vietquys/network_automation.git
cd network_automation
```

**Cần thấy:** file theo từng host (ví dụ `Router.yml`, `CORE-1.yml`, `AC-1.yml`...).

**Ý nghĩa thực thi:** biến riêng theo từng thiết bị.
### Vì sao phải giữ nguyên cấu trúc?

### 3.4 Kiểm tra `roles/`
- `ansible-network/ansible.cfg` đang dùng đường dẫn tương đối `inventory = inventory.ini`.
- Nếu chạy sai thư mục hoặc di chuyển `ansible-network` lung tung, Ansible dễ đọc sai inventory/vars.
- Thực hành đúng: **không tách rời file trong `ansible-network/`**.

```bash
find roles -maxdepth 3 -type f | sort
```
---

**Cần thấy:** mỗi role có `tasks/main.yml`.
## 5) Cách sử dụng Ansible với repo này

### 3.5 Kiểm tra `playbooks/`
## Bước 1: vào đúng thư mục chạy

```bash
ls -la playbooks
cd /opt/automation/network_automation/ansible-network
# hoặc: cd ~/automation/network_automation/ansible-network
```

**Cần thấy:** các playbook riêng (`pb_vtp.yml`, `pb_ospf.yml`, `pb_dhcp.yml`, ...).

### 3.6 Validate nhanh inventory bằng Ansible
## Bước 2: kiểm tra inventory

```bash
ansible-inventory -i inventory.ini --list > /tmp/inventory.json
ansible-inventory -i inventory.ini --graph
ansible-inventory -i inventory.ini --list | head -n 40
```

**Kết quả mong đợi:** inventory parse thành công, hiện đúng cây nhóm/host.

---

## Phần 4: Chạy thử Ansible

> Chạy tất cả lệnh trong thư mục `/opt/automation/ansible-network`.
## Bước 3: test kết nối

### 4.1 Test kết nối mức Ansible
### Ping module

```bash
ansible network -i inventory.ini -m ansible.builtin.ping
```

**Kết quả mong đợi:** host trả về `pong`.

### 4.2 Test lệnh thật tới Cisco IOS
### Test lệnh IOS thật

```bash
ansible network -i inventory.ini -m cisco.ios.ios_command -a "commands=['show version']"
```

**Kết quả mong đợi:** trả output `show version` từ thiết bị.

### 4.3 Dry-run trước khi apply thật
## Bước 4: dry-run trước khi apply

```bash
ansible-playbook -i inventory.ini site.yml --check --diff
```

**Kết quả mong đợi:** play chạy được theo đúng thứ tự, hiển thị thay đổi dự kiến mà chưa ghi cấu hình.

---

## Phần 5: Apply cấu hình thật

### 5.1 Chạy playbook tổng (khuyến nghị)
## Bước 5: apply thật

```bash
ansible-playbook -i inventory.ini site.yml
```

**Vì sao:** `site.yml` đã sắp thứ tự role theo dependency (base → vtp/vlan → etherchannel → trunk → stp → hsrp → ospf → dhcp → access_ports → snmp).
---

## 6) Chạy playbook riêng (khi cần)

**Kết quả mong đợi:**
- Không có task failed.
- Thiết bị được cấu hình đầy đủ theo luồng chuẩn của project.
Repo có các playbook thành phần trong `ansible-network/playbooks/`:

### 5.2 Chạy playbook riêng (khi cần chỉnh từng mảng)
- `pb_vtp.yml`
- `pb_stp.yml`
- `pb_ospf.yml`
- `pb_dhcp.yml`
- `pb_etherchannel.yml`
- `pb_hsrp.yml`
- `pb_snmp.yml`

Ví dụ:
### Lệnh chạy mẫu

```bash
ansible-playbook -i inventory.ini playbooks/pb_vtp.yml
ansible-playbook -i inventory.ini playbooks/pb_etherchannel.yml
ansible-playbook -i inventory.ini playbooks/pb_hsrp.yml
ansible-playbook -i inventory.ini playbooks/pb_stp.yml
ansible-playbook -i inventory.ini playbooks/pb_ospf.yml
ansible-playbook -i inventory.ini playbooks/pb_dhcp.yml
ansible-playbook -i inventory.ini playbooks/pb_etherchannel.yml
ansible-playbook -i inventory.ini playbooks/pb_hsrp.yml
ansible-playbook -i inventory.ini playbooks/pb_snmp.yml
ansible-playbook -i inventory.ini playbooks/pb_stp.yml
```

**Khi nào dùng:**
- Rollout theo phase.
- Re-apply một phần sau khi thay đổi nhỏ.
- Debug phạm vi hẹp.
### Khi nào chạy riêng, khi nào chạy `site.yml`?

- Chạy **`site.yml`** khi triển khai full lab hoặc rollout mới toàn bộ.
- Chạy **playbook riêng** khi:
  - Chỉ thay đổi 1 mảng (ví dụ OSPF/SNMP).
  - Cần sửa nhanh một chức năng.
  - Cần debug phạm vi hẹp.

---

## 7) Quy trình triển khai chuẩn

1. **Bootstrap thiết bị** qua console (`bootstrap/*.ios`).
2. **Kiểm tra kết nối** từ Ubuntu Server (ping + ansible test command).
3. **Dry-run** với `--check --diff`.
4. **Apply config thật** bằng `site.yml` (hoặc playbook riêng).
5. **Verify trạng thái** bằng lệnh show qua `ios_command`.

### 5.3 Verify sau apply
---

## 8) Verify sau triển khai (show commands)

```bash
# VTP
ansible core -i inventory.ini -m cisco.ios.ios_command -a "commands=['show vtp status']"

# STP
ansible core:access -i inventory.ini -m cisco.ios.ios_command -a "commands=['show spanning-tree summary']"

# EtherChannel
ansible routers:core -i inventory.ini -m cisco.ios.ios_command -a "commands=['show etherchannel summary']"

# HSRP
ansible core -i inventory.ini -m cisco.ios.ios_command -a "commands=['show standby brief']"

# OSPF
ansible routers:core -i inventory.ini -m cisco.ios.ios_command -a "commands=['show ip ospf neighbor']"

# DHCP
ansible routers -i inventory.ini -m cisco.ios.ios_command -a "commands=['show ip dhcp binding']"

# SNMP
ansible network -i inventory.ini -m cisco.ios.ios_command -a "commands=['show snmp host']"
```

**Kết quả mong đợi:** thông số vận hành khớp thiết kế mạng của bạn.

---

## Phần 6: Troubleshooting
## 9) Troubleshooting nhanh

### 6.1 Lỗi inventory/biến
### 9.1 Lỗi inventory

Dấu hiệu:
- `ERROR! ... inventory`
- `VARIABLE IS NOT DEFINED`
Triệu chứng: không parse inventory, sai group/host.

Cách xử lý:
```bash
ansible-inventory -i inventory.ini --graph
ansible-inventory -i inventory.ini --host CORE-1
```
- Soát tên host trong `inventory.ini` phải khớp tên file trong `host_vars/`.
- Soát biến nhóm trong `group_vars/`.

### 6.2 Lỗi kết nối thiết bị
Kiểm tra tên host trong `inventory.ini` phải khớp file trong `host_vars/`.

Dấu hiệu:
- `UNREACHABLE!`
- timeout SSH
### 9.2 Lỗi SSH / unreachable

Cách xử lý:
```bash
ping -c 3 192.168.99.2
nc -zv 192.168.99.2 22
ansible network -i inventory.ini -m cisco.ios.ios_command -a "commands=['show clock']" -vvv
```
- Kiểm tra route từ Ubuntu Server đến management subnet.
- Kiểm tra bootstrap đã bật VTY/SSH đúng chưa.

### 6.3 Lỗi module/collection
Kiểm tra lại IP quản trị, route, ACL, user/password sau bootstrap.

Dấu hiệu:
- `couldn't resolve module/action 'cisco.ios.ios_command'`
### 9.3 Lỗi module Cisco

Triệu chứng: `couldn't resolve module/action 'cisco.ios.ios_command'`.

Cách xử lý:
```bash
ansible-galaxy collection list | grep cisco.ios
ansible-galaxy collection install cisco.ios
ansible-galaxy collection install cisco.ios ansible.netcommon
```

### 6.4 Debug chi tiết playbook
### 9.4 Debug chi tiết

```bash
ansible-playbook -i inventory.ini site.yml -vvv
ansible-playbook -i inventory.ini site.yml --start-at-task "<ten task>"
ansible-playbook -i inventory.ini playbooks/pb_ospf.yml --limit CORE-1
ansible-playbook -i inventory.ini site.yml --limit CORE-1
ansible-playbook -i inventory.ini site.yml --start-at-task "PLAY 9 | Cấu hình OSPF Area 0 trên Router + CORE"
```

- Dùng `-vvv` để xem chi tiết transport/command.
- Dùng `--limit` để khoanh vùng 1 host/1 group.
- Dùng `--start-at-task` để chạy lại từ task lỗi.

### 6.5 Đề xuất sắp xếp lại (không đổi logic gốc)

1. **Chuẩn hóa thư mục gốc**: giữ đúng `automation/` chứa 3 nhánh `config-all`, `bootstrap`, `ansible-network` để onboarding dễ hơn.
2. **Giữ mọi lệnh chạy từ `ansible-network/`**: tránh sai path tương đối của `ansible.cfg`.
3. **Bổ sung 1 file runbook ngắn ở root** (ví dụ `DEPLOY.md`) trỏ tới tài liệu này và thứ tự bootstrap → ansible apply.

---

## Phần 7: Kết luận
## 10) Kết luận

Nếu làm đúng theo các bước trên, bạn sẽ có một môi trường triển khai rõ ràng trên Ubuntu Server:
- File được đặt đúng chỗ, đúng vai trò.
- Inventory/vars/roles/playbooks được kiểm tra trước khi chạy.
- Có quy trình test kết nối, dry-run, apply thật, và debug khi lỗi.
Triển khai đúng và an toàn nhất với repo này là:

Mấu chốt để vận hành ổn định: **luôn chạy từ `ansible-network/`, kiểm tra inventory trước, dry-run trước khi apply thật**.

---
**Bootstrap trước để có kết nối quản trị → sau đó mới chạy Ansible trong `ansible-network/`.**

## Checklist triển khai nhanh

- [ ] Đã tạo `/opt/automation` và cấp quyền ghi.
- [ ] Đã copy đủ `config-all/`, `bootstrap/`, `ansible-network/` vào `/opt/automation`.
- [ ] Đã vào đúng thư mục `/opt/automation/ansible-network`.
- [ ] Đã kiểm tra `inventory.ini` đúng IP/hostname thực tế.
- [ ] Đã xác nhận đủ `group_vars/`, `host_vars/`, `roles/`, `playbooks/`.
- [ ] `ansible-inventory --graph` chạy OK.
- [ ] Test `ansible.builtin.ping` thành công tới group `network`.
- [ ] Test `cisco.ios.ios_command` (`show version`) thành công.
- [ ] Đã chạy `site.yml --check --diff` trước khi apply thật.
- [ ] Đã chạy `ansible-playbook site.yml` (hoặc playbook riêng theo phase).
- [ ] Đã verify các dịch vụ chính (VTP, EtherChannel, HSRP, OSPF, DHCP, SNMP).
Giữ nguyên cấu trúc thư mục, luôn dry-run trước khi apply thật để giảm rủi ro cấu hình.
