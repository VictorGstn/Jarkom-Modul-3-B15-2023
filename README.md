# Laporan Resmi Modul 3 Jarkom 2023

Kelompok **B15**

Anggota:

- [MH] Muhammad Hidayat (05111940000131)
- [VG] Victor Gustinova (5025211159)

## 0. Register domain

```bash
apt-get update
apt-get install bind9 -y

echo '
zone "riegel.canyon.b15.com" {
        type master;
        file "/etc/bind/riegel/riegel.canyon.b15.com";
};

zone "granz.channel.b15.com" {
        type master;
        file "/etc/bind/granz/granz.channel.b15.com";
};
' > /etc/bind/named.conf.local

mkdir -p /etc/bind/riegel

echo '
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     riegel.canyon.b15.com. root.riegel.canyon.b15.com. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      riegel.canyon.b15.com.
@       IN      A       10.16.2.3 ; IP Load Balancer
www     IN      CNAME   riegel.canyon.b15.com.  
' > /etc/bind/riegel/riegel.canyon.b15.com

mkdir -p /etc/bind/granz

echo '
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     granz.channel.b15.com. root.granz.channel.b15.com. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      granz.channel.b15.com.
@       IN      A       10.16.2.3 ; IP Load Balancer
www     IN      CNAME   granz.channel.b15.com.  
' > /etc/bind/granz/granz.channel.b15.com

service bind9 restart


```
## 1. Konfigurasi

Lakukan konfigurasi sesuai gambar yang telah diberikan


## 2-5. DHCP

Pada DHCP server dijalankan bash berikut.
```
apt-get update -y
apt-get install isc-dhcp-server -y

echo 'INTERFACESv4="eth0"' > /etc/default/isc-dhcp-server

echo '
authoritative;
subnet 10.16.1.0 netmask 255.255.255.0 {
}
subnet 10.16.2.0 netmask 255.255.255.0 {
}
subnet 10.16.3.0 netmask 255.255.255.0 {
    range 10.16.3.16 10.16.3.32
    range 10.16.3.64 10.16.3.80;
    option routers 10.16.3.1;
    option broadcast-address 10.16.3.255;
    option domain-name-servers 10.16.1.3;
    default-lease-time 180;
    max-lease-time 5760;

}
subnet 10.16.4.0 netmask 255.255.255.0 { 
    range 10.16.4.12 10.16.4.20
    range 10.16.4.160 10.16.4.168;
    option routers 10.16.4.1;
    option broadcast-address 10.16.4.255;
    option domain-name-servers 10.16.1.3;
    default-lease-time 720;
    max-lease-time 5760;
}
' > /etc/dhcp/dhcpd.conf

service isc-dhcp-server stop
service isc-dhcp-server start
```

Lalu setting DHCP Relay pada aura dengan script berikut.

```
apt-get update
apt-get install isc-dhcp-relay -y
service isc-dhcp-relay start

echo '
SERVERS="10.16.1.2"
INTERFACES="eth1 eth2 eth3 eth4"
OPTIONS=
' > etc/default/isc-dhcp-relay

echo 'net.ipv4.ip_forward=1' > /etc/sysctl.conf

service isc-dhcp-relay restart
```

Lalu pada client yang akan mendapatkan service DHCP akan dijalankan script berikut.

```
echo '
auto eth0
iface eth0 inet dhcp
' > /etc/network/interfaces
```

### 6.Load Balancer

Pada load balancer jalankan script berikut

```
apt-get update
apt-get install php7.3-mbstring php7.3-xml php7.3-cli php7.3-common php7.3-intl php7.3-opcache php7.3-readline php7.3-mysql php7.3-fpm php7.3-curl unzip wget nginx -y

echo '
upstream myweb  {
 	server 10.16.3.2; #IP Lawine
 	server 10.16.3.3; #IP Linie
	server 10.16.3.4; #IP Lugner
}

server {
 	listen 80;
 	server_name granz.channel.b15.com;

 	location / {
 	proxy_pass http://myweb;
 	}
}
' > etc/nginx/sites-available/lb_granz

ln -s /etc/nginx/sites-available/lb_granz /etc/nginx/sites-enabled
service nginx restart

apt-get install apache2-utils -y

echo 'nameserver 10.16.1.3' > etc/resolv.conf
```

Lalu pada worker

```
apt-get update
apt-get install php7.3-mbstring php7.3-xml php7.3-cli php7.3-common php7.3-intl php7.3-opcache php7.3-readline php7.3-mysql php7.3-fpm php7.3-curl unzip wget nginx -y
mkdir /var/www
wget --no-check-certificate -O 'filephp.zip' 'https://drive.google.com/uc?id=1ViSkRq7SmwZgdK64eRbr5Fm1EGCTPrU1' 
unzip 'filephp.zip'
mv modul-3 /var/www
mv /var/www/modul-3 /var/www/granz

echo '
server {

 	listen 80;

 	root /var/www/granz;

 	index index.php index.html index.htm;
 	server_name _;

 	location / {
 			try_files $uri $uri/ /index.php?$query_string;
 	}

 	# pass PHP scripts to FastCGI server
 	location ~ \.php$ {
 	include snippets/fastcgi-php.conf;
 	fastcgi_pass unix:/var/run/php/php7.3-fpm.sock;
 	}

location ~ /\.ht {
 			deny all;
 	}

 	error_log /var/log/nginx/granz_error.log;
 	access_log /var/log/nginx/granz_access.log;
}
' > /etc/nginx/sites-available/granz

ln -s /etc/nginx/sites-available/granz /etc/nginx/sites-enabled

service php7.3-fpm start
service php7.3-fpm restart
service nginx restart
```

### 7.Test

Lakukan test dengan 
```
ab -n 1000 -c 100 http://granz.channel.b15.com/
```

Grimoire yang didapatkan dapat dilihat pada docs berikut.

https://docs.google.com/document/d/1V9ykNFu6z0oLfzMwr31MjQoSDTazuWKf_ZUFia4YJgY/edit?usp=sharing

### 8.Test algoritma

dilakukan 4 testing algoritma selain round robin. Berikut merupakan script yang dipakai.

IP HASH

```
echo '
upstream myweb {
  ip_hash;
  server 10.16.3.2; # IP Lawine
  server 10.16.3.3; # IP Linie
  server 10.16.3.4; # IP Lugner
}

server {
  listen 80;
  server_name granz.channel.b15.com;

  location / {
    proxy_pass http://myweb;
  }
}
' > etc/nginx/sites-available/lb_granz

service nginx restart
```

Least Connection

```
echo '
upstream myweb {
  least_conn;
  server 10.16.3.2; # IP Lawine
  server 10.16.3.3; # IP Linie
  server 10.16.3.4; # IP Lugner
}

server {
  listen 80;
  server_name granz.channel.b15.com;

  location / {
    proxy_pass http://myweb;
  }
}
' > etc/nginx/sites-available/lb_granz

service nginx restart

```

Generic Hash

```
echo '
upstream myweb  {
	hash $remote_addr consistent;
 	server 10.16.3.2; #IP Lawine
 	server 10.16.3.3; #IP Linie
	server 10.16.3.4; #IP Lugner
}

server {
 	listen 80;
 	server_name granz.channel.b15.com;

 	location / {
 	proxy_pass http://myweb;
 	}
}
' > etc/nginx/sites-available/lb_granz

service nginx restart
```

Grimoire hasil didapatkan terdapat pada link docs sebelumnya.

### 9. Test Worker

Pada test ini digunakan script round robin hanya saja jumlah worker akan disesuaikan dengan cara menghapus line pada upstream. Hasil terdapat pada google docs.

### 10. Auth

Jalankan script berikut pada load balancer.

```
htpasswd -c -b /etc/nginx/rahasiakita netics ajkyyy

echo '
upstream myweb  {
 	server 10.16.3.2; #IP Lawine
 	server 10.16.3.3; #IP Linie
	server 10.16.3.4; #IP Lugner
}

server {
 	listen 80;
 	server_name granz.channel.b15.com;

 	location / {
 	proxy_pass http://myweb;
	auth_basic "Restricted Content";
        auth_basic_user_file /etc/nginx/rahasiakita;
 	}
}
' > etc/nginx/sites-available/lb_granz
```

### 11. Proxy Pass

Tambahkan location /its dari script no 10 sebagai berikut

```
echo '
upstream myweb  {
 	server 10.16.3.2; #IP Lawine
 	server 10.16.3.3; #IP Linie
	server 10.16.3.4; #IP Lugner
}

server {
 	listen 80;
 	server_name granz.channel.b15.com;

 	location / {
 	proxy_pass http://myweb;
	auth_basic "Restricted Content";
        auth_basic_user_file /etc/nginx/rahasiakita;
 	}

	location ~ /its {
	proxy_pass https://www.its.ac.id;
	}
}
' > etc/nginx/sites-available/lb_granz

service nginx restart
```

### 12. Address Restriction

Untuk menjalankan restriction maka tambahkan IP yanf di allow lalu deny all.

```
echo '
upstream myweb  {
 	server 10.16.3.2; #IP Lawine
 	server 10.16.3.3; #IP Linie
	server 10.16.3.4; #IP Lugner
}

server {
 	listen 80;
 	server_name granz.channel.b15.com;

 	location / {
	allow 10.16.3.69;  
    	allow 10.16.3.70;
	allow 10.16.4.167;  
    	allow 10.16.4.168;
    	deny all;
 	proxy_pass http://myweb;
	auth_basic "Restricted Content";
        auth_basic_user_file /etc/nginx/rahasiakita;
 	}

	location /its/ {
	proxy_pass https://www.its.ac.id;
	}
}
' > etc/nginx/sites-available/lb_granz


service nginx restart
```
### 13. Database

berikut merupakana script yang perlu dijalankan pada databae.

```
apt-get update
apt-get install mariadb-server -y
service mysql start

echo '# This group is read both by the client and the server
# use it for options that affect everything
[client-server]

# Import all .cnf files from configuration directory
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mariadb.conf.d/

# Options affecting the MySQL server (mysqld)
[mysqld]
skip-networking=0
skip-bind-address
' > /etc/mysql/my.cnf


echo'
[server]


[mysqld]

user                    = mysql
pid-file                = /run/mysqld/mysqld.pid
socket                  = /run/mysqld/mysqld.sock
#port                   = 3306
basedir                 = /usr
datadir                 = /var/lib/mysql
tmpdir                  = /tmp
lc-messages-dir         = /usr/share/mysql

bind-address            = 0.0.0.0

query_cache_size        = 16M

log_error = /var/log/mysql/error.log
expire_logs_days        = 10

character-set-server  = utf8mb4
collation-server      = utf8mb4_general_ci

[embedded]

[mariadb]

[mariadb-10.3]
' > /etc/mysql/mariadb.conf.d/50-server.cnf

service mysql restart
```



Lalu untuk membuat user maka jalankan perintah berikut.

```
mysql -u root -p
Enter password: 

CREATE USER 'kelompokb15'@'%' IDENTIFIED BY 'passwordb15';
CREATE USER 'kelompokb15'@'localhost' IDENTIFIED BY 'passwordb15';
CREATE DATABASE dbkelompoka09;
GRANT ALL PRIVILEGES ON *.* TO 'kelompokb15'@'%';
GRANT ALL PRIVILEGES ON *.* TO 'kelompokb15'@'localhost';
FLUSH PRIVILEGES;
```

Pada worker, install maria-db
```
apt-get update
apt-get install mariadb-client -y
```

### 14. Worker Laravel
### 15. Register
### 16. Login
### 17. GET
### 18. LB Laravel
### 19. Konfigurasi
### 20. Laravel Least Connection
