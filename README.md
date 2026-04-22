# Hướng dẫn triển khai project Network Automation trên Ubuntu Server (Ansible Server)

Tài liệu này tập trung đúng mục tiêu: **đưa đúng file vào đúng chỗ trên Ubuntu Server** và **chạy Ansible để apply cấu hình lên thiết bị mạng** theo cấu trúc repo đã có.

> **Giả định:** bạn đã cài Ansible + collection Cisco IOS (`cisco.ios`) trên Ubuntu Server, và các thiết bị đã bootstrap để SSH/Telnet từ Ansible Server.

---

## Phần 1: Chuẩn bị trên Ubuntu Server

### 1.1 Tạo thư mục triển khai chuẩn

Chọn 1 thư mục làm workspace, ví dụ `/opt/automation`:

```bash
sudo mkdir -p /opt/automation
sudo chown -R $USER:$USER /opt/automation
```

**Vì sao đặt ở đây:**
- Dễ quản trị trên server (tách biệt với thư mục hệ thống).
- Thuận tiện backup/đồng bộ toàn bộ project.

**Kết quả mong đợi:** có thư mục `/opt/automation` và user hiện tại có quyền ghi.

---

## Phần 2: Đưa file vào đúng vị trí

### 2.1 Mục tiêu cấu trúc trên Ubuntu Server

Sau khi copy xong, nên có cấu trúc:

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

### 2.2 Copy thư mục `config-all`

```bash
cp -a /duong-dan-nguon/automation/config-all /opt/automation/
```

**Đặt ở đâu:** `/opt/automation/config-all`  
**Vì sao:** đây là bộ cấu hình đích để đối chiếu/so sánh sau triển khai, không phải thư mục Ansible runtime.

**Kết quả mong đợi:**
- Có thư mục `/opt/automation/config-all`.
- Chứa file tổng hợp cấu hình mong muốn (theo thiết kế của team).

> **Lưu ý:** trong bản repo hiện tại bạn gửi, thư mục `config-all` không có sẵn trong working tree. Bạn vẫn cần copy từ nguồn lưu trữ nội bộ nếu team đang giữ riêng.

### 2.3 Copy thư mục `bootstrap`

```bash
cp -a /duong-dan-nguon/automation/bootstrap /opt/automation/
```

**Đặt ở đâu:** `/opt/automation/bootstrap`  
**Vì sao:** lưu script bootstrap để khởi tạo thiết bị trắng (console), phục vụ pre-stage trước khi Ansible apply full.

**Kết quả mong đợi:** có các file `.ios` và `00_README.md` trong `bootstrap/`.

### 2.4 Copy thư mục `ansible-network`

```bash
cp -a /duong-dan-nguon/automation/ansible-network /opt/automation/
```

**Đặt ở đâu:** `/opt/automation/ansible-network`  
**Vì sao:** đây là root Ansible project; `ansible.cfg` đang trỏ inventory theo đường dẫn tương đối (`inventory = inventory.ini`), nên chạy lệnh từ đúng thư mục này sẽ đơn giản và đúng ngữ cảnh.

**Kết quả mong đợi:** đủ các thành phần `ansible.cfg`, `inventory.ini`, `site.yml`, `group_vars/`, `host_vars/`, `roles/`, `playbooks/`.

---

## Phần 3: Kiểm tra cấu trúc

Di chuyển vào project và kiểm tra nhanh:

```bash
cd /opt/automation/ansible-network
pwd
ls -la
find group_vars host_vars roles playbooks -maxdepth 2 -type f
```

### 3.1 Kiểm tra `inventory.ini`

```bash
sed -n '1,200p' inventory.ini
```

**Cần thấy:** các nhóm `routers`, `core`, `access`, `network:children`, `switches:children` với IP đúng lab thực tế.

### 3.2 Kiểm tra `group_vars/`

```bash
ls -la group_vars
```

**Cần thấy file chính:**
- `all.yml`
- `routers.yml`
- `core.yml`
- `access.yml`

**Ý nghĩa thực thi:** tham số dùng chung theo nhóm thiết bị.

### 3.3 Kiểm tra `host_vars/`

```bash
ls -la host_vars
```

**Cần thấy:** file theo từng host (ví dụ `Router.yml`, `CORE-1.yml`, `AC-1.yml`...).

**Ý nghĩa thực thi:** biến riêng theo từng thiết bị.

### 3.4 Kiểm tra `roles/`

```bash
find roles -maxdepth 3 -type f | sort
```

**Cần thấy:** mỗi role có `tasks/main.yml`.

### 3.5 Kiểm tra `playbooks/`

```bash
ls -la playbooks
```

**Cần thấy:** các playbook riêng (`pb_vtp.yml`, `pb_ospf.yml`, `pb_dhcp.yml`, ...).

### 3.6 Validate nhanh inventory bằng Ansible

```bash
ansible-inventory -i inventory.ini --list > /tmp/inventory.json
ansible-inventory -i inventory.ini --graph
```

**Kết quả mong đợi:** inventory parse thành công, hiện đúng cây nhóm/host.

---

## Phần 4: Chạy thử Ansible

> Chạy tất cả lệnh trong thư mục `/opt/automation/ansible-network`.

### 4.1 Test kết nối mức Ansible

```bash
ansible network -i inventory.ini -m ansible.builtin.ping
```

**Kết quả mong đợi:** host trả về `pong`.

### 4.2 Test lệnh thật tới Cisco IOS

```bash
ansible network -i inventory.ini -m cisco.ios.ios_command -a "commands=['show version']"
```

**Kết quả mong đợi:** trả output `show version` từ thiết bị.

### 4.3 Dry-run trước khi apply thật

```bash
ansible-playbook -i inventory.ini site.yml --check --diff
```

**Kết quả mong đợi:** play chạy được theo đúng thứ tự, hiển thị thay đổi dự kiến mà chưa ghi cấu hình.

---

## Phần 5: Apply cấu hình thật

### 5.1 Chạy playbook tổng (khuyến nghị)

```bash
ansible-playbook -i inventory.ini site.yml
```

**Vì sao:** `site.yml` đã sắp thứ tự role theo dependency (base → vtp/vlan → etherchannel → trunk → stp → hsrp → ospf → dhcp → access_ports → snmp).

**Kết quả mong đợi:**
- Không có task failed.
- Thiết bị được cấu hình đầy đủ theo luồng chuẩn của project.

### 5.2 Chạy playbook riêng (khi cần chỉnh từng mảng)

Ví dụ:

```bash
ansible-playbook -i inventory.ini playbooks/pb_vtp.yml
ansible-playbook -i inventory.ini playbooks/pb_etherchannel.yml
ansible-playbook -i inventory.ini playbooks/pb_hsrp.yml
ansible-playbook -i inventory.ini playbooks/pb_ospf.yml
ansible-playbook -i inventory.ini playbooks/pb_dhcp.yml
ansible-playbook -i inventory.ini playbooks/pb_snmp.yml
ansible-playbook -i inventory.ini playbooks/pb_stp.yml
```

**Khi nào dùng:**
- Rollout theo phase.
- Re-apply một phần sau khi thay đổi nhỏ.
- Debug phạm vi hẹp.

### 5.3 Verify sau apply

```bash
ansible core -i inventory.ini -m cisco.ios.ios_command -a "commands=['show vtp status']"
ansible routers:core -i inventory.ini -m cisco.ios.ios_command -a "commands=['show etherchannel summary']"
ansible core -i inventory.ini -m cisco.ios.ios_command -a "commands=['show standby brief']"
ansible routers:core -i inventory.ini -m cisco.ios.ios_command -a "commands=['show ip ospf neighbor']"
ansible routers -i inventory.ini -m cisco.ios.ios_command -a "commands=['show ip dhcp binding']"
ansible network -i inventory.ini -m cisco.ios.ios_command -a "commands=['show snmp host']"
```

**Kết quả mong đợi:** thông số vận hành khớp thiết kế mạng của bạn.

---

## Phần 6: Troubleshooting

### 6.1 Lỗi inventory/biến

Dấu hiệu:
- `ERROR! ... inventory`
- `VARIABLE IS NOT DEFINED`

Cách xử lý:
```bash
ansible-inventory -i inventory.ini --graph
ansible-inventory -i inventory.ini --host CORE-1
```
- Soát tên host trong `inventory.ini` phải khớp tên file trong `host_vars/`.
- Soát biến nhóm trong `group_vars/`.

### 6.2 Lỗi kết nối thiết bị

Dấu hiệu:
- `UNREACHABLE!`
- timeout SSH

Cách xử lý:
```bash
ping -c 3 192.168.99.2
nc -zv 192.168.99.2 22
ansible network -i inventory.ini -m cisco.ios.ios_command -a "commands=['show clock']" -vvv
```
- Kiểm tra route từ Ubuntu Server đến management subnet.
- Kiểm tra bootstrap đã bật VTY/SSH đúng chưa.

### 6.3 Lỗi module/collection

Dấu hiệu:
- `couldn't resolve module/action 'cisco.ios.ios_command'`

Cách xử lý:
```bash
ansible-galaxy collection list | grep cisco.ios
ansible-galaxy collection install cisco.ios
```

### 6.4 Debug chi tiết playbook

```bash
ansible-playbook -i inventory.ini site.yml -vvv
ansible-playbook -i inventory.ini site.yml --start-at-task "<ten task>"
ansible-playbook -i inventory.ini playbooks/pb_ospf.yml --limit CORE-1
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

Nếu làm đúng theo các bước trên, bạn sẽ có một môi trường triển khai rõ ràng trên Ubuntu Server:
- File được đặt đúng chỗ, đúng vai trò.
- Inventory/vars/roles/playbooks được kiểm tra trước khi chạy.
- Có quy trình test kết nối, dry-run, apply thật, và debug khi lỗi.

Mấu chốt để vận hành ổn định: **luôn chạy từ `ansible-network/`, kiểm tra inventory trước, dry-run trước khi apply thật**.

---

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
