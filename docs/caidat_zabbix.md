## Cài đặt zabbix

## 1. Yêu cầu cài đặt
- OS: CentOS 7.x 64 bit
	- IP addresses: 192.168.20.39
	- Subnet mask: 255.255.255.0
	- Gateway: 192.168.20.254
- Zabbix 3.0

## 2. Các bước triển khai

### 2.1. Khai báo IP, hostname 
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

### 2.2. Thực hiện cài đặt zabbix 
- Đăng nhập vào máy chủ zabbix với quyền root và thực hiện các bước dưới.
- Cài các gói bổ trợ: http, php, mariadb-server
	```sh
	yum -y install httpd
	yum -y install php php-mbstring php-pear
	yum -y install mariadb-server
	```

#### 2.2.1. Cấu hình mariadb 
 - Sửa file `/etc/my.cnf` bằng cách add thêm dòng dưới 
		```sh
		[mysqld]
		character-set-server=utf8
		```
	- Khởi động mariadb
		```sh
		systemctl start mariadb
		systemctl enable mariadb
		```
	- Kiểm tra trạng thái của service mariadb bằng lệnh
		```sh
		systemctl status mariadb
		```
	- Kết quả như dưới là ok, nếu chưa ổn thì kiểm tra lại các bước trên. 
		```sh
		[root@zabbixserver ~]# systemctl status mariadb
		● mariadb.service - MariaDB database server
			 Loaded: loaded (/usr/lib/systemd/system/mariadb.service; enabled; vendor preset: disabled)
			 Active: active (running) since Tue 2017-08-22 16:47:55 +07; 13s ago
		 Main PID: 1298 (mysqld_safe)
			 CGroup: /system.slice/mariadb.service
							 ├─1298 /bin/sh /usr/bin/mysqld_safe --basedir=/usr
							 └─1457 /usr/libexec/mysqld --basedir=/usr --datadir=/var/lib/mysql --plugin-dir=/usr/lib64/mysql/plugin --log-error=/var/log/mariadb/mariadb.log --pid-...

		Aug 22 16:47:53 zabbixserver mariadb-prepare-db-dir[1220]: Alternatively you can run:
		Aug 22 16:47:53 zabbixserver mariadb-prepare-db-dir[1220]: '/usr/bin/mysql_secure_installation'
		Aug 22 16:47:53 zabbixserver mariadb-prepare-db-dir[1220]: which will also give you the option of removing the test
		Aug 22 16:47:53 zabbixserver mariadb-prepare-db-dir[1220]: databases and anonymous user created by default.  This is
		Aug 22 16:47:53 zabbixserver mariadb-prepare-db-dir[1220]: strongly recommended for production servers.
		Aug 22 16:47:53 zabbixserver mariadb-prepare-db-dir[1220]: See the MariaDB Knowledgebase at http://mariadb.com/kb or the
		Aug 22 16:47:53 zabbixserver mariadb-prepare-db-dir[1220]: MySQL manual for more instructions.
		Aug 22 16:47:53 zabbixserver mysqld_safe[1298]: 170822 16:47:53 mysqld_safe Logging to '/var/log/mariadb/mariadb.log'.
		Aug 22 16:47:53 zabbixserver mysqld_safe[1298]: 170822 16:47:53 mysqld_safe Starting mysqld daemon with databases from /var/lib/mysql
		Aug 22 16:47:55 zabbixserver systemd[1]: Started MariaDB database server.
		```	
	
- Đăng nhập vào mariadb với quyền `root` của database và thực hiện phân quyền truy nhập vào database.
	```sh
	mysql -u root -p
	```
	- Ấn Enter vì database chưa được khai báo mật khẩu. Màn hình sẽ chuyển sang chế độ thao tác với database
	
- Khai báo mật khẩu cho tài khoản `root` của database. Lưu ý đây không phải là tài khoản root để ssh vào máy chủ. Mật khẩu sử dụng để truy cập database là `Welcome123`
	```sh
	GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'Welcome123' WITH GRANT OPTION;
	GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' IDENTIFIED BY 'Welcome123' WITH GRANT OPTION;
	GRANT ALL PRIVILEGES ON *.* TO 'root'@'127.0.0.1' IDENTIFIED BY 'Welcome123' WITH GRANT OPTION;
	GRANT ALL PRIVILEGES ON *.* TO 'root'@'192.168.20.39' IDENTIFIED BY 'Welcome123' WITH GRANT OPTION;
	DROP USER 'root'@'zabbixserver';
	DROP USER ''@'localhost';
	DROP USER 'root'@'::1';
	FLUSH PRIVILEGES;
	```

- Để thoát khỏi cửa sổ đăng nhập của mariadb thực hiện lệnh exit
	```sh
	MariaDB [(none)]> exit
	```

#### 2.2.2. Tải và cài đặt zabbix	
- Tải gói chứa repos của zabbix 3.0
	```sh
	yum -y install php-mysql php-gd php-xml php-bcmath
	yum -y install http://repo.zabbix.com/zabbix/3.0/rhel/7/x86_64/zabbix-release-3.0-1.el7.noarch.rpm
	```
	
- Cài đặt zabbix server 
	```sh
	yum -y install zabbix-get zabbix-server-mysql zabbix-web-mysql zabbix-agent 
	```

- Đăng nhập vào mysql và tạo database cho zabbix server. Lúc này cần nhập mật khẩu của tài khoản root của database đã được phân quyền ở bước trên. 
	```sh
	mysql -u root -p 
	```
	
- Tạo database tên tên là `zabbixdb`, tài khoản truy cập là `zabbixuser` và mật khẩu là `Welcome123`
	```sh	
	create database zabbixdb; 
	grant all privileges on zabbixdb.* to zabbixuser@'localhost' identified by 'Welcome123'; 
	grant all privileges on zabbixdb.* to zabbixuser@'%' identified by 'Welcome123'; 
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
		
	- Sửa dòng `DBName` (tên của Database cho zabbix đã tạo ở trên. 
		```sh
		DBName=zabbixdb
		```
		
	- Sửa dòng `DBUser` là tên của user 
		```sh
		DBUser=zabbixuser
		```


### 2.3. Cấu hình để khởi động zabbix server và giám sát chính nó.

- Kích hoạt zabbix server 
	```sh
	systemctl start zabbix-server 
	systemctl enable zabbix-server 
	```

- Khai báo cấu hình cho zabbix trong file cấu hình của httpd 
	```sh
	sed -i -e 's/# php_value date.timezone Europe\/Riga/php_value date.timezone Asia\/Ho_Chi_Minh/g' /etc/httpd/conf.d/zabbix.conf
	```

- Khởi động htttp
	```sh
	systemctl restart httpd 
	```

- Sửa các dòng dưới trong file `/etc/zabbix/zabbix_agentd.conf` bằng lệnh dưới
	```sh
	sed -i -e 's/Hostname=Zabbix server/Hostname=ZabbixServer/g' /etc/zabbix/zabbix_agentd.conf
	```
	
- Kích hoạt zabbix-agent để zabbix server giám sát chính nó.
	```sh
	systemctl start zabbix-agent 
	systemctl enable zabbix-agent 
	```

	
## 3. Thiếp lập cấu hình ban đầu cho zabbix server

- Truy cập vào máy chủ zabbix bằng ip và khai báo các thiết lập cho zabbix

## 4. Add host zabbix-server để giám sát.

