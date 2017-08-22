## Cài đặt zabbix

## Yêu cầu cài đặt
- OS: CentOS 7.x 64 bit
	- IP addresses: 192.168.20.39
	- Subnet mask: 255.255.255.0
	- Gateway: 192.168.20.254
- Zabbix 3.0

## Các bước triển khai

### Khai báo IP, hostname 
- Thiết lập hostname cho máy chủ zabbix
	```sh
	hostnamectl set-hostname zabbixserver
	```
	
- Thiết lập IP cho máy chủ, trong môi trường LAB này ta sẽ thiếp lập ip cho card `ens160`
	```sh
	nmcli con modify ens160 ipv4.addresses 192.168.20.39/24
	nmcli con modify ens160 ipv4.gateway 192.168.20.254
	nmcli con modify ens160 ipv4.dns 8.8.8.8
	nmcli con modify ens160 ipv4.method manual
	nmcli con modify ens160 connection.autoconnect yes
	```
	
- Thiết lập các chính sách về Iptables và công cụ quản lý card mạng của CentOS
	```sh
	sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
	sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
	sudo systemctl disable firewalld
	sudo systemctl stop firewalld
	sudo systemctl stop NetworkManager
	sudo systemctl disable NetworkManager
	sudo systemctl enable network
	sudo systemctl start network
	init 6
	```

### Thực hiện cài đặt zabbix 

- Cài các gói bổ trợ: http, php, mariadb-server
	```sh
	yum -y install httpd
	yum -y install php php-mbstring php-pear
	yum -y install mariadb-server
	```

- Tải gói chứa repos của zabbix 3.0
	```sh
	yum -y install php-mysql php-gd php-xml php-bcmath
	yum -y install http://repo.zabbix.com/zabbix/3.0/rhel/7/x86_64/zabbix-release-3.0-1.el7.noarch.rpm
	```
	
- Cài đặt zabbix server 
	```sh
	yum -y install zabbix-get zabbix-server-mysql zabbix-web-mysql zabbix-agent 
	```

- Đăng nhập vào mysql và tạo database cho zabbix server. Lúc này không cần nhập mật khẩu vì lúc cài đặt mariadb-server chúng ta không khai báo mật khẩu. 
	```sh
	mysql -u root -p 
	```
	
- Tạo database tên tên là `zabbix`, tài khoản truy cập là `zabbix` và mật khẩu là `Welcome123`
	```sh	
	create database zabbix; 
	grant all privileges on zabbix.* to zabbix@'localhost' identified by 'Welcome123'; 
	grant all privileges on zabbix.* to zabbix@'%' identified by 'Welcome123'; 
	flush privileges; 
	
	exit
	```

- Giải nén file zip database mẫu của zabbix và import để tạo các bảng trong database của zabbix 
	```sh
	cd /usr/share/doc/zabbix-server-mysql-*/ 

	gunzip create.sql.gz 

	mysql -u root -p zabbix < create.sql
	```
	- Cần nhập mật khẩu đã tạo cho zabbix ở trên lúc trước.

- Sửa các dòng dưới trong file `/etc/zabbix/zabbix_server.conf`

	- Sửa dòng `DBHost`
		```sh
		DBHost=localhost
		```
	- Sửa dòng `DBPassword`
		```sh
		DBPassword=Welcome123
		```

### Cấu hình để zabbix server giám sát chính nó.

- Sửa các dòng dưới trong file `/etc/zabbix/zabbix_agentd.conf`
	```sh
	Server=127.0.0.1
	ServerActive=127.0.0.1
	Hostname=zabbixserver
	```

- Kích hoạt zabbix server 
	```sh
	systemctl start zabbix-agent 
	systemctl enable zabbix-agent 
	```
	
- Khởi động htttp
	```sh
	systemctl restart httpd 
	```
	
### Thiếp lập cấu hình ban đầu cho zabbix server

- Truy cập vào máy chủ zabbix bằng ip và khai báo các cấu hình. 