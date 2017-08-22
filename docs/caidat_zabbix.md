## Cài đặt zabbix

## Yêu cầu cài đặt
- OS: CentOS 7.x 64 bit
- Zabbix 3.0

## Các bước triển khai


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
