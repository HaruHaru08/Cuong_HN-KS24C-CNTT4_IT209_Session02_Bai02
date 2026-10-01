# **Bài 2: Khởi tạo User thường và thiết lập đặc quyền quản trị (Sudoers Configuration)**

## **Mục tiêu**

* Tạo tài khoản người dùng thường `devops` trên Ubuntu Server (Droplet) để không phải thao tác trực tiếp bằng `root`, theo nguyên tắc **Least Privilege** (đặc quyền tối thiểu).  
* Cấp quyền quản trị an toàn qua nhóm `sudo`.  
* Sao chép SSH Key từ `root` sang `devops` để đăng nhập bằng key.

## **Môi trường**

| Thông tin | Giá trị |
| ----- | ----- |
| Hệ điều hành | Ubuntu Server (DigitalOcean Droplet) |
| IP Droplet | `221.121.3.144` |
| User ban đầu | `root` |
| User mới | `devops` |

---

## **Phần A. Tạo user và cấu hình sudo**

### **1\. Đăng nhập Droplet bằng root và kiểm tra thông tin ban đầu**

bash

* ssh \-i /path/to/private\_key root@203.0.113.25  
* whoami  
* hostname  
  cat /etc/os-release | head \-n 2

Log terminal:

\$ ssh \-i \~/.ssh/id\_ed25519 root@203.0.113.25

root@droplet:\~\# whoami

root

root@droplet:\~\# hostname

droplet

root@droplet:\~\# cat /etc/os-release | head \-n 2

PRETTY\_NAME="Ubuntu 24.04 LTS"

NAME="Ubuntu"

### **2\. Tạo user `devops`**

bash

adduser devops

Log terminal:

root@droplet:\~\# adduser devops

Adding user \`devops' ...

Adding new group \`devops' (1000) ...

Adding new user \`devops' (1000) with group \`devops' ...

Creating home directory \`/home/devops' ...

Copying files from \`/etc/skel' ...

New password: \*\*\*\*\*\*\*\*

Retype new password: \*\*\*\*\*\*\*\*

passwd: password updated successfully

Changing the user information for devops

Enter the new value, or press ENTER for the default

        Full Name \[\]:

        Room Number \[\]:

        Work Phone \[\]:

        Home Phone \[\]:

        Other \[\]:

Is the information correct? \[Y/n\] Y

> Phải đặt mật khẩu cho `devops` vì `sudo` sẽ yêu cầu nhập mật khẩu này. Nếu cần đổi lại sau này: `sudo passwd devops`.

### **3\. Thêm `devops` vào nhóm `sudo`**

bash

* usermod \-aG sudo devops  
  id devops

Log terminal:

root@droplet:\~\# usermod \-aG sudo devops

root@droplet:\~\# id devops

uid=1000(devops) gid=1000(devops) groups=1000(devops),27(sudo)

Giải thích:

* `-a` (append) đi cùng `-G` để **thêm** vào nhóm mà không xóa các nhóm hiện có.  
* Nhóm `sudo` trên Ubuntu mặc định được phép chạy mọi lệnh với quyền root (dòng `%sudo ALL=(ALL:ALL) ALL` trong `/etc/sudoers`).

  ### **4\. Sao chép SSH Key từ `root` sang `devops`**

  bash  
  rsync \--archive \--chown=devops:devops \~/.ssh /home/devops/

Log terminal:

root@droplet:\~\# rsync \--archive \--chown=devops:devops \~/.ssh /home/devops/

root@droplet:\~\# ls \-lah /home/devops/

total 28K

drwxr-x--- 4 devops devops 4.0K Oct  1 10:00 .

drwxr-xr-x 4 root   root   4.0K Oct  1 09:58 ..

\-rw-r--r-- 1 devops devops  220 Jan  6  2022 .bash\_logout

\-rw-r--r-- 1 devops devops 3.7K Jan  6  2022 .bashrc

\-rw-r--r-- 1 devops devops  807 Jan  6  2022 .profile

drwx------ 2 devops devops 4.0K Oct  1 09:45 .ssh

Giải thích:

* `--archive` giữ nguyên quyền, thời gian và cấu trúc thư mục.  
* `--chown=devops:devops` đổi chủ sở hữu sang `devops` ngay khi sao chép.

  ### **5\. Phân quyền `.ssh` và `authorized_keys`**

Dù `rsync --archive` đã giữ quyền gốc, vẫn thiết lập lại tường minh để đảm bảo chính xác:

bash

* chmod 700 /home/devops/.ssh  
* chmod 600 /home/devops/.ssh/authorized\_keys  
* chown \-R devops:devops /home/devops/.ssh  
  ls \-lah /home/devops/.ssh

Log terminal:

root@droplet:\~\# ls \-lah /home/devops/.ssh

total 12K

drwx------ 2 devops devops 4.0K Oct  1 09:45 .

drwxr-x--- 4 devops devops 4.0K Oct  1 10:00 ..

\-rw------- 1 devops devops  569 Oct  1 09:45 authorized\_keys

Giải thích:

* `700` (`drwx------`) trên `.ssh`: chỉ chủ sở hữu được đọc/ghi/truy cập.  
* `600` (`-rw-------`) trên `authorized_keys`: chỉ chủ sở hữu được đọc/ghi.  
* OpenSSH có cơ chế `StrictModes`: nếu quyền quá rộng, SSH sẽ **từ chối** key và đăng nhập thất bại. Không dùng `chmod 777` vì vi phạm nguyên tắc cấp quyền tối thiểu.

  ### **6\. Kiểm tra đăng nhập và sudo**

Mở terminal mới trên máy cá nhân (giữ nguyên phiên root cũ để phòng khi có lỗi):

bash

* ssh \-i /path/to/private\_key devops@203.0.113.25  
* whoami  
  sudo whoami

Log terminal:

\$ ssh \-i \~/.ssh/id\_ed25519 devops@203.0.113.25

Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.8.0 x86\_64)

...

devops@droplet:\~\$ whoami

devops

devops@droplet:\~\$ sudo whoami

\[sudo\] password for devops: \*\*\*\*\*\*\*\*

root

Kết quả: đăng nhập bằng SSH Key thành công; `whoami` trả về `devops` (user thường), còn `sudo whoami` yêu cầu mật khẩu và trả về `root` nên `devops` đã có đặc quyền quản trị.

---

## **Phần B. Dùng sudo để cài Nginx và cấu hình tường lửa**

Phần này kiểm chứng `devops` thực hiện được công việc quản trị hằng ngày mà không cần đăng nhập `root`. Mọi lệnh đều chạy bằng user `devops`.

### **7\. Cài đặt Nginx**

bash

* sudo apt update && sudo apt \-y upgrade  
  sudo apt install nginx \-y

> Lưu ý: với user thường, **mỗi** lệnh `apt` phải có `sudo` riêng. Viết `sudo apt update && apt -y upgrade` sẽ lỗi `Permission denied` ở vế sau vì `sudo` chỉ áp dụng cho lệnh đứng ngay sau nó.

Log terminal (rút gọn):

text

devops@droplet:\~\$ sudo apt update && sudo apt \-y upgrade

Hit:1 http\://mirrors.digitalocean.com/ubuntu noble InRelease

...

Reading package lists... Done

devops@droplet:\~\$ sudo apt install nginx \-y

Reading package lists... Done

The following NEW packages will be installed:

  nginx nginx-common

...

Setting up nginx (1.24.0-2ubuntu7) ...

### **8\. Quản lý dịch vụ Nginx bằng systemctl**

bash

* sudo systemctl status nginx  
* sudo systemctl enable nginx  
* sudo systemctl restart nginx  
  sudo systemctl reload nginx

Giải thích các lệnh:

| Lệnh | Công dụng |
| ----- | ----- |
| `sudo systemctl status nginx` | Xem trạng thái dịch vụ |
| `sudo systemctl start nginx` | Khởi động dịch vụ |
| `sudo systemctl stop nginx` | Dừng dịch vụ |
| `sudo systemctl restart nginx` | Khởi động lại (ngắt kết nối cũ, nạp lại cấu hình) |
| `sudo systemctl reload nginx` | Nạp lại cấu hình mà không gián đoạn kết nối hiện tại |
| `sudo systemctl enable nginx` | Tự khởi động cùng hệ điều hành |

Log terminal:

devops@droplet:\~\$ sudo systemctl status nginx

nginx.service \- A high performance web server and a reverse proxy server

     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)

     Active: active (running) since Thu 2026-10-01 10:12:40 UTC; 5s ago

       Docs: man:nginx(8)

   Main PID: 2471 (nginx)

      Tasks: 2 (limit: 1100\)

     Memory: 2.4M

text

### **9\. Cấu hình tường lửa UFW cho HTTP/HTTPS**

Phải cho phép SSH **trước khi bật UFW** để tránh tự khóa mất quyền truy cập từ xa:

bash

* sudo ufw allow OpenSSH  
* sudo ufw app list  
* sudo ufw allow 'Nginx Full'  
* sudo ufw enable  
  sudo ufw status

Log terminal:

text

devops@droplet:\~\$ sudo ufw allow OpenSSH

Rules updated

Rules updated (v6)

devops@droplet:\~\$ sudo ufw app list

Available applications:

  Nginx Full

  Nginx HTTP

  Nginx HTTPS

  OpenSSH

devops@droplet:\~\$ sudo ufw allow 'Nginx Full'

Rules updated

Rules updated (v6)

devops@droplet:\~\$ sudo ufw enable

Command may disrupt existing ssh connections. Proceed with operation (y|n)? y

Firewall is active and enabled on system startup

devops@droplet:\~\$ sudo ufw status

Status: active

To                         Action      From

\--                         \------      \----

OpenSSH                    ALLOW       Anywhere

Nginx Full                 ALLOW       Anywhere

OpenSSH (v6)               ALLOW       Anywhere (v6)

Nginx Full (v6)            ALLOW       Anywhere (v6)

Giải thích: profile `Nginx Full` mở cả cổng 80 (HTTP) và 443 (HTTPS).

### **10\. Kiểm tra Nginx hoạt động**

bash

* curl \-I http\://localhost  
  sudo ss \-lntp | grep :80

Log terminal:

text

* devops@droplet:\~\$ curl \-I http\://localhost  
* HTTP/1.1 200 OK  
* Server: nginx/1.24.0 (Ubuntu)  
* Content-Type: text/html  
* devops@droplet:\~\$ sudo ss \-lntp | grep :80  
  LISTEN 0 511 0.0.0.0:80 0.0.0.0:\* users:(("nginx",pid=2471,fd=5))  
  ---

  ## **Kết quả**

| Yêu cầu | Trạng thái |
| ----- | ----- |
| Tạo user `devops` | Hoàn thành |
| Thêm `devops` vào nhóm `sudo` | Hoàn thành |
| Sao chép SSH Key từ root sang devops | Hoàn thành |
| Phân quyền `.ssh` (700) và `authorized_keys` (600) | Hoàn thành |
| SSH bằng `devops` thành công | Hoàn thành |
| `sudo whoami` trả về `root` | Hoàn thành |
| Cài Nginx, quản lý bằng `systemctl`, mở UFW bằng `devops` | Hoàn thành (mở rộng) |

