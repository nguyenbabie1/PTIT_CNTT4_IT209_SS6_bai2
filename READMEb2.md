# Bài 2: Cấu hình phân quyền Nhóm và sudoers bằng visudo

## 2. Mục tiêu

Tạo nhóm `devops-admin` và tài khoản `deployer`; cấp cho thành viên nhóm quyền quản lý dịch vụ bằng các thao tác `systemctl start`, `stop`, `restart`, `status` mà không cần nhập mật khẩu. Không cấp quyền chạy mọi lệnh quản trị cho tài khoản này.

## 3. Quy trình thực hiện

### 3.1. Tạo nhóm và người dùng

Thực hiện bằng tài khoản `root` trên VPS Linux:

```bash
groupadd devops-admin
adduser deployer
usermod -aG devops-admin deployer
id deployer
```

Lệnh `adduser` tạo tài khoản và yêu cầu đặt mật khẩu. Lệnh `usermod -aG` bổ sung tài khoản vào nhóm mà không loại bỏ các nhóm hiện có. Không thêm `deployer` vào nhóm `sudo`.

Nếu nhóm hoặc tài khoản đã tồn tại, bỏ qua lệnh tạo tương ứng.

### 3.2. Cấu hình sudoers bằng visudo

Mở cấu hình với Nano thông qua `visudo`:

```bash
EDITOR=nano visudo
```

Thêm dòng sau vào cuối `/etc/sudoers`:

```sudoers
%devops-admin ALL=(ALL) NOPASSWD: /usr/bin/systemctl start *, /usr/bin/systemctl stop *, /usr/bin/systemctl restart *, /usr/bin/systemctl status *
```

| Thành phần | Ý nghĩa |
| --- | --- |
| `%devops-admin` | Áp dụng cho các thành viên nhóm devops-admin |
| `ALL` trước dấu `=` | Áp dụng trên mọi máy sử dụng cấu hình này |
| `(ALL)` | Cho phép chạy các lệnh được liệt kê dưới danh tính người dùng đích; mặc định là root |
| `NOPASSWD:` | Không yêu cầu mật khẩu cho các lệnh được cấp quyền |
| Bốn lệnh `systemctl` | Cấp quyền start, stop, restart và status |
| `*` | Khớp phần đối số theo quy tắc wildcard của sudoers |

Lưu bằng **Ctrl + O**, nhấn **Enter**, sau đó thoát bằng **Ctrl + X**. File `/etc/sudoers.tmp` xuất hiện trong trình soạn thảo là file tạm do visudo sử dụng.

Kiểm tra cú pháp:

```bash
visudo -c
```

Yêu cầu khi kiểm tra: các file cấu hình được đọc phải báo `parsed OK`.

### 3.3. Xử lý dịch vụ cron chưa được cài đặt

Trong quá trình kiểm tra ban đầu, hệ thống báo:

```text
Failed to restart cron.service: Unit cron.service not found.
```

Cách xử lý bằng tài khoản `root`:

```bash
apt update
apt install cron -y
systemctl enable --now cron
```

### 3.4. Kiểm tra bằng tài khoản deployer

Từ tài khoản root, chuyển sang deployer:

```bash
su - deployer
```

Kiểm tra quyền và khả năng restart không cần mật khẩu:

```bash
sudo -l
sudo -k
sudo -n systemctl restart cron
echo $?
systemctl is-active cron
```

- `sudo -l`: liệt kê quyền sudo của tài khoản hiện tại.
- `sudo -k`: xóa trạng thái xác thực sudo được lưu trước đó.
- `sudo -n`: không hỏi mật khẩu; nếu cần mật khẩu thì báo lỗi ngay.
- `echo $?`: lấy mã thoát của lệnh ngay trước đó, nên chạy ngay sau lệnh restart.
- `systemctl is-active cron`: kiểm tra trạng thái hoạt động của cron.

## 4. Minh chứng kết quả thực tế

Đoạn log dưới đây được sao chép từ phiên kiểm tra bằng tài khoản `deployer` trên máy `NguyenbabieVM`:

```text
deployer@NguyenbabieVM:~$ sudo -l
Matching Defaults entries for deployer on NguyenbabieVM:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User deployer may run the following commands on NguyenbabieVM:
    (ALL) NOPASSWD: /usr/bin/systemctl start *, /usr/bin/systemctl stop *,
        /usr/bin/systemctl restart *, /usr/bin/systemctl status *
deployer@NguyenbabieVM:~$ sudo -k
deployer@NguyenbabieVM:~$ sudo -n systemctl restart cron
deployer@NguyenbabieVM:~$ echo $?
0
deployer@NguyenbabieVM:~$ systemctl is-active cron
active
```

## 5. Đánh giá kết quả

| Nội dung kiểm tra | Kết quả thực tế | Đánh giá |
| --- | --- | --- |
| Quyền sudo của deployer | Có đủ start, stop, restart, status với NOPASSWD | Đạt |
| Restart cron sau khi xóa trạng thái xác thực | Không yêu cầu mật khẩu, không báo lỗi | Đạt |
| Mã thoát của lệnh restart | `0` | Thành công |
| Trạng thái dịch vụ cron | `active` | Đang hoạt động |

`sudo -l` hiển thị quyền đã được áp dụng cho tài khoản, không nhất thiết in tên nhóm nguồn `%devops-admin`. Dòng sudoers tại mục 3.2 thể hiện cấu hình cấp quyền theo nhóm.

## 6. Kết luận

Kết quả thực tế xác nhận tài khoản `deployer` được phép thực hiện bốn thao tác quản lý dịch vụ đã cấu hình với `NOPASSWD`. Lệnh restart cron chạy thành công sau khi xóa trạng thái xác thực sudo, trả về mã `0`, và dịch vụ ở trạng thái `active`.

Quy tắc dùng wildcard trong bài thực hành cho phép quản lý nhiều dịch vụ. Khi áp dụng vào hệ thống thực tế, cần giới hạn rõ tên dịch vụ và đối số được phép theo nhu cầu vận hành.
