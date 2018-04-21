# LNMP 搭建
系统 :CentOS 7.4 64位
-
### 1.nginx 安装

安装需求环境编译软件 更新yum
```
yum install -y openssl-devel pcre-devel
yum install -y gcc
yum install -y gcc-c++
yum update
```
给nginx创建运行用户
```
//添加个组www
groupadd www
//添加个用户www 加入 www
useradd -g www www;
```
切换到用户家目录 下载nginx 1.12.2稳定版并解压
```
cd ~

wget http://nginx.org/download/nginx-1.12.2.tar.gz

tar -zxvf nginx-1.12.2.tar.gz
```

编译前配置   可跳过
```
cd nginx-1.12.2.tar.gz

vim src/core/nginx.h

//修改nginx名称和版本不让别人看的

//修改前
#define NGINX_VERSION       "1.12.2"
#define NGINX_VER           "nginx/" NGINX_VERSION

//修改后
#define NGINX_VERSION       "自定义" //例如 :6666
#define NGINX_VER           "自定义/" NGINX_VERSION //例如 :OKOKOK/

vim src/http/ngx_http_header_filter_module.c

:49

//修改前
static u_char ngx_http_server_string[] = "Server : nginx" CRLF;
//修改后
static u_char ngx_http_server_string[] = "Server : 自定义" CRLF; //例如 :OKOKOK

//程序出现错误 会显示nginx 和 版本号 在此隐藏

vim src/http/ngx_http_special_response.c

//修改前
static u_char ngx_http_error_tail[] =
"<hr><center>nginx</center>" CRLF

//修改后
static u_char ngx_http_error_tail[] =
"<hr><center>自定义</center>" CRLF //例如 :OKOKOK
```
开始编译安装
```
\\设置配置
./configure --prefix=/usr/local/nginx --with-http_dav_module --with-http_stub_status_module  --with-http_addition_module --with-http_sub_module  --with-http_flv_module --with-http_mp4_module --with-pcre --with-http_ssl_module --with-http_gzip_static_module  --user=www  --group=www

\\配置介绍
--with-http_dav_module  #增加PUT,DELETE,MKCOL：创建集合，COPY和MOVE方法
--with-http_stub_status_module  #获取Nginx的状态统计信息
--with-http_addition_module   #作为一个输出过滤器，支持不完全缓冲，分部分相应请求
--with-http_sub_module     #允许一些其他文本替换Nginx相应中的一些文本
--with-http_flv_module     #提供支持flv视频文件支持
--with-http_mp4_module  #提供支持mp4视频文件支持，提供伪流媒体服务端支持
--with-http_ssl_module         #启用ngx_http_ssl_module
--user=www             #上面添加的用户由此来执行
--group=www             #上面添加的组由此来执行

\\编译 安装
make && make install
```

等待编译安装完成无报错继续向下

编写启动脚本 设置开机启动
```
vim /etc/init.d/nginx

//输入以下内容
//start
#!/bin/sh
#
# nginx - this script starts and stops the nginx daemon
#
# chkconfig: - 85 15
# description: Nginx is an HTTP(S) server, HTTP(S) reverse \
# proxy and IMAP/POP3 proxy server
# processname: nginx
# config: /etc/nginx/nginx.conf
# config: /usr/local/nginx/conf/nginx.conf
# pidfile: /usr/local/nginx/logs/nginx.pid
 
# Source function library.
. /etc/rc.d/init.d/functions
 
# Source networking configuration.
. /etc/sysconfig/network
 
# Check that networking is up.
[ "$NETWORKING" = "no" ] && exit 0
nginx="/usr/local/nginx/sbin/nginx"
prog=$(basename $nginx)
NGINX_CONF_FILE="/usr/local/nginx/conf/nginx.conf"
[ -f /etc/sysconfig/nginx ] && . /etc/sysconfig/nginx
lockfile=/var/lock/subsys/nginx
 
make_dirs() {
    # make required directories
    user=`$nginx -V 2>&1 | grep "configure arguments:" | sed 's/[^*]*--user=\([^ ]*\).*/\1/g' -`
    if [ -z "`grep $user /etc/passwd`" ]; then
    useradd -M -s /bin/nologin $user
    fi
    options=`$nginx -V 2>&1 | grep 'configure arguments:'`
    for opt in $options; do
    if [ `echo $opt | grep '.*-temp-path'` ]; then
    value=`echo $opt | cut -d "=" -f 2`
    if [ ! -d "$value" ]; then
    # echo "creating" $value
    mkdir -p $value && chown -R $user $value
    fi
    fi
    done
}
 
start() {
    [ -x $nginx ] || exit 5
    [ -f $NGINX_CONF_FILE ] || exit 6
    make_dirs
    echo -n $"Starting $prog: "
    daemon $nginx -c $NGINX_CONF_FILE
    retval=$?
    echo
    [ $retval -eq 0 ] && touch $lockfile
    return $retval
}
 
stop() {
    echo -n $"Stopping $prog: "
    killproc $prog -QUIT
    retval=$?
    echo
    [ $retval -eq 0 ] && rm -f $lockfile
    return $retval
}
 
restart() {
    #configtest || return $?
    stop
    sleep 1
    start
}
 
reload() {
    #configtest || return $?
    echo -n $"Reloading $prog: "
    killproc $nginx -HUP
    RETVAL=$?
    echo
}
 
force_reload() {
    restart
}
 
configtest() {
    $nginx -t -c $NGINX_CONF_FILE
}
 
rh_status() {
    status $prog
}
 
rh_status_q() {
    rh_status >/dev/null 2>&1
}
 
case "$1" in
start)
rh_status_q && exit 0
$1
;;
stop)
 
rh_status_q || exit 0
$1
;;
restart|configtest)
$1
;;
reload)
rh_status_q || exit 7
$1
;;
force-reload)
force_reload
;;
status)
rh_status
;;
condrestart|try-restart)
rh_status_q || exit 0
;;
*)
echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
exit 2
esac

//end

//保存并推出
:x 

//设置文件的执行权限
chmod a+x /etc/init.d/nginx

//使用chkconfig进行管理
chkconfig --add /etc/init.d/nginx

//设置开机启动
chkconfig nginx on

//以后就可以用
service nginx start //开启nginx
service nginx stop  //关闭nginx
service nginx restart //重启nginx
service nginx reload //不关闭nginx重新加载配置
```

上面那一步不设置可以用
```
/usr/local/nginx/sbin/nginx //开启
/usr/local/nginx/sbin/nginx -s stop //关闭
```

配置优化
```
vim /usr/local/nginx/conf/nginx.conf
//Nginx运行CPU亲和力，比如四核：


worker_processes  4;
worker_cpu_affinity 0001 0010 0100 1000

//八核如下：

worker_processes 8;

worker_cpu_affinity 00000001 00000010 00000100 00001000 00010000 00100000 01000000 1000000


//worker_processes最多开启8个，8个以上性能提升不会再提升了，而且稳定性变得更低，所以8个进程够用了。

//设置Nginx最多打开的文件数 直接加在worker_processes *下面

worker_rlimit_nofile 65535;

//Nginx事件处理模型
events {
use epoll;
worker_connections 65535;
multi_accept on;
}


nginx采用epoll事件模型，处理效率高
work_connections是单个worker进程允许客户端最大连接数，这个数值一般根据服务器性能和内存来制定，实际最大值就是worker进程数乘以work_connections
实际我们填入一个65535，足够了，这些都算并发值，一个网站的并发达到这么大的数量，也算一个大站了！
multi_accept 告诉nginx收到一个新连接通知后接受尽可能多的连接

```

查看进程以及分析
```
[root@yankerp nginx-1.12.2]# ps -ef | grep nginx
root      42347      1  0 14:26 ?        00:00:00 nginx: master process nginx
www     42348  42347  0 14:26 ?        00:00:00 nginx: worker process

在这里我们还可以看到在查看的时候，work进程是www程序用户，但是master进程还是root，其中，master是监控进程，也叫主进程，work是工作进程，部分还有cache相关进程
```
查看运行状态
```
netstat -anput | grep nginx 
```
---
### 1.MariaDB 安装
---
卸载系统自带mariadb-libs

- 查询
```
rpm -qa|grep mariadb-libs
```

- 卸载
```
rpm -e 查询的输出结果 --nodeps
```
安装相关包
```
yum -y install libaio 
yum -y install libaio-devel 
yum -y install bison 
yum -y install bison-devel 
yum -y install zlib-devel 
yum -y install openssl
yum -y install openssl-devel 
yum -y install ncurses 
yum -y install ncurses-devel
yum -y install libcurl-devel
yum -y install libarchive-devel 
yum -y install boost 
yum -y install boost-devel 
yum -y install lsof 
yum -y install wget
yum -y install gcc 
yum -y install gcc-c++
yum -y install make
yum -y install cmake
yum -y install perl
yum -y install kernel-headers
yum -y install kernel-devel 
yum -y install pcre-devel

//全部复制进xshell 执行停止后还有一行没有执行 直接回车 
//一条一条复制执行也可以
```

下载源码包 并解压
```
cd ~

wget https://downloads.mariadb.org/interstitial/mariadb-10.2.14/source/mariadb-10.2.14.tar.gz

tar -zxvf mariadb-10.2.14.tar.gz

cd mariadb-10.2.14
```
创建mysql系统用户组 并 创建mysql系统用户加入mysql系统用户组
```
groupadd -r mysql

useradd -r -g mysql -s /sbin/nologin -d /usr/local/mysql -M mysql
```

创建maria安装目录 创建数据库存放目录 改变所属为mysql
```
mkdir -p /usr/local/mysql

mkdir -p /data/mysql

chown -R mysql:mysql /data/mysql
```

编译配置 然后编译安装 
```
cmake . -DCMAKE_INSTALL_PREFIX=/usr/local/mysql \
 -DMYSQL_DATADIR=/data/mysql \
 -DSYSCONFDIR=/etc \
 -DWITHOUT_TOKUDB=1 \
 -DWITH_INNOBASE_STORAGE_ENGINE=1 \
 -DWITH_ARCHIVE_STPRAGE_ENGINE=1 \
 -DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
 -DWIYH_READLINE=1 \
 -DWIYH_SSL=system \
 -DVITH_ZLIB=system \
 -DWITH_LOBWRAP=0 \
 -DMYSQL_UNIX_ADDR=/tmp/mysql.sock \
 -DDEFAULT_CHARSET=utf8 \
 -DDEFAULT_COLLATION=utf8_general_ci
 
 //编译安装
 make && make install
```

配置MariaDB
```
cd /usr/local/mysql/

scripts/mysql_install_db --user=mysql --datadir=/data/mysql

//以上命令直接执行
```

复制MariaDB配置文件到/etc目录
```
cp support-files/my-large.cnf /etc/my.cnf
```
创建启动脚本 并设置开机自动启动
```
cp support-files/mysql.server /etc/rc.d/init.d/mysql

chkconfig --add /etc/init.d/mysql

chkconfig mysql on
```
启动服务 && 停止
```
service mysql start
service mysql stop
```

配置环境变量, 以便在任何目录下输入mysql
```
//新建一个文件
vim /etc/profile.d/mysql.sh
//输入以下内容

export PATH=$PATH:/usr/local/mysql/bin/

//保存并退出
:x
//为脚本赋于可执行权限
chmod 0777 /etc/profile.d/mysql.sh

//进行mysql.sh脚本所在目录, 并执行脚本, 以立即生效环境变量
source /etc/profile.d/mysql.sh
```

初始化MariaDB
```
cd /usr/local/mysql/

./bin/mysql_secure_installation

Enter current password for root (enter for none):     输入当前root密码(没有输入)

Set root password? [Y/n]     设置root密码?(是/否)

New password:    输入新root密码

Re-enter new password:        确认输入root密码

Password updated successfully!         密码更新成功

By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

默认情况下,MariaDB安装有一个匿名用户,
允许任何人登录MariaDB而他们无需创建用户帐户。
这个目的是只用于测试,安装去更平缓一些。
你应该进入前删除它们生产环境。

Remove anonymous users? [Y/n]         删除匿名用户?(是/否)

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

通常情况下，root只应允许从localhost连接。
这确保其他用户无法从网络猜测root密码。

Disallow root login remotely? [Y/n]     不允许root登录远程?(是/否)

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

默认情况下，MariaDB提供了一个名为“测试”的数据库，任何人都可以访问。
这也只用于测试，在进入生产环境之前应该被删除。

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

重新加载权限表将确保所有到目前为止所做的更改将立即生效。

Reload privilege tables now? [Y/n]      现在重新加载权限表(是/否)

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

```

以上就完成了安装 下一步是远程连接设置 不需要可以跳过
```
mysql -u root -p
你的密码
use mysql
update user set host="%" where user="root";
会报错 但是不用管他
ERROR 1062 (23000): Duplicate entry '%-root' for key 'PRIMARY'
ctrl+c
service mysql restart

再进行连接即可
```
### 1.PHP 安装

安装扩展包并更新系统内核：
```
yum install epel-release -y
yum update
```

安装php依赖组件（包含Nginx依赖）：
```
yum -y install wget vim pcre pcre-devel openssl openssl-devel libicu-devel gcc gcc-c++ autoconf libjpeg libjpeg-devel libpng libpng-devel freetype freetype-devel libxml2 libxml2-devel zlib zlib-devel glibc glibc-devel glib2 glib2-devel ncurses ncurses-devel curl curl-devel krb5-devel libidn libidn-devel openldap openldap-devel nss_ldap jemalloc-devel cmake boost-devel bison automake libevent libevent-devel gd gd-devel libtool* libmcrypt libmcrypt-devel mcrypt mhash libxslt libxslt-devel readline readline-devel gmp gmp-devel libcurl libcurl-devel openjpeg-devel
```
下载php安装包解压：
```
wget http://am1.php.net/distributions/php-7.2.4.tar.gz
tar -zxvf php-7.2.4.tar.gz
cd php-7.2.4
```
设置变量并开始源码编译：
```
cp -frp /usr/lib64/libldap* /usr/lib/

./configure --prefix=/usr/local/php \
--with-config-file-path=/usr/local/php/etc \
--enable-fpm \
--with-fpm-user=www \
--with-fpm-group=www \
--enable-mysqlnd \
--with-mysqli=mysqlnd \
--with-pdo-mysql=mysqlnd \
--enable-mysqlnd-compression-support \
--with-iconv-dir \
--with-freetype-dir \
--with-jpeg-dir \
--with-png-dir \
--with-zlib \
--with-libxml-dir \
--enable-xml \
--disable-rpath \
--enable-bcmath \
--enable-shmop \
--enable-sysvsem \
--enable-inline-optimization \
--with-curl \
--enable-mbregex \
--enable-mbstring \
--enable-intl \
--with-libmbfl \
--enable-ftp \
--with-gd \
--with-openssl \
--with-mhash \
--enable-pcntl \
--enable-sockets \
--with-xmlrpc \
--enable-zip \
--enable-soap \
--with-gettext \
--disable-fileinfo \
--enable-opcache \
--with-pear \
--enable-maintainer-zts \
--with-ldap=shared \
--without-gdbm 
```
开始安装
```
make -j 4 && make install
```
完成安装后配置php.ini文件：
```
cp php.ini-development /usr/local/php/etc/php.ini
cp /usr/local/php/etc/php-fpm.conf.default /usr/local/php/etc/php-fpm.conf
cp /usr/local/php/etc/php-fpm.d/www.conf.default /usr/local/php/etc/php-fpm.d/www.conf
```

项目上线后配置次配置文件 修改 php.ini 相关参数：

```
vim /usr/local/php/etc/php.ini

//修改参数如下
expose_php = Off
short_open_tag = ON
max_execution_time = 300
max_input_time = 300
memory_limit = 128M
post_max_size = 32M
date.timezone = Asia/Shanghai
mbstring.func_overload=2

//此条配置注意 先ls /usr/local/php/lib/php/extensions 输出结果填充下方
extension = "/usr/local/php/lib/php/extensions/输出结果/ldap.so"
```

次配置也是项目上线后配置即可
```
vim /usr/local/php/etc/php.ini
//修改前
disable_functions =

//选择1修改后、
disable_functions = passthru,exec,system,chroot,scandir,chgrp,chown,shell_exec,proc_open,proc_get_status,ini_alter,ini_alter,ini_restore,dl,openlog,syslog,readlink,symlink,popepassthru,stream_socket_server,escapeshellcmd,dll,popen,disk_free_space,checkdnsrr,checkdnsrr,getservbyname,getservbyport,disk_total_space,posix_ctermid,posix_get_last_error,posix_getcwd, posix_getegid,posix_geteuid,posix_getgid, posix_getgrgid,posix_getgrnam,posix_getgroups,posix_getlogin,posix_getpgid,posix_getpgrp,posix_getpid, posix_getppid,posix_getpwnam,posix_getpwuid, posix_getrlimit, posix_getsid,posix_getuid,posix_isatty, posix_kill,posix_mkfifo,posix_setegid,posix_seteuid,posix_setgid, posix_setpgid,posix_setsid,posix_setuid,posix_strerror,posix_times,posix_ttyname,posix_uname

//选择2
disable_functions = passthru,exec,system,chroot,chgrp,chown,shell_exec,proc_open,proc_get_status,popen,ini_alter,ini_restore,dl,openlog,syslog,readlink,symlink,popepassthru
```

配置www.conf
```
vim /usr/local/php/etc/php-fpm.d/www.conf

listen = 127.0.0.1:9000
listen.owner = www
listen.group = www
listen.mode = 0660
listen.allowed_clients = 127.0.0.1
pm = dynamic
listen.backlog = -1
pm.max_children = 180
pm.start_servers = 50
pm.min_spare_servers = 50
pm.max_spare_servers = 180
request_terminate_timeout = 120
request_slowlog_timeout = 50
slowlog = var/log/slow.log
```

配置php-fpm.conf
```
//取下以下注释并填写完整路径：

vim /usr/local/php/etc/php-fpm.conf

pid = /usr/local/php/var/run/php-fpm.pid
```

创建system系统单元文件php-fpm启动脚本
```
vim /usr/lib/systemd/system/php-fpm.service
//内容
[Unit]
Description=The PHP FastCGI Process Manager
After=syslog.target network.target

[Service]
Type=simple
PIDFile=/usr/local/php/var/run/php-fpm.pid
ExecStart=/usr/local/php/sbin/php-fpm --nodaemonize --fpm-config /usr/local/php/etc/php-fpm.conf
ExecReload=/bin/kill -USR2 $MAINPID

[Install]
WantedBy=multi-user.target
```
启动php-fpm服务并加入开机自启动：
```
systemctl enable php-fpm.service
systemctl restart php-fpm.service
```

让nginx服务器可以访问php脚本

```
//切换www用户
su www
//创建网站根目录
mkdir /home/www/localhost

cd /home/www/localhost

//创建index.php
vim index.php

<?php
    phpinfo();
    
:x
//保存退出

//切换回root
su root

vim /usr/local/nginx/conf/nginx.conf

//内容如下
//整体替换
user www www;
#user  nobody;
#worker_processes  1;
worker_processes  4;
worker_cpu_affinity 0001 0010 0100 1000;
worker_rlimit_nofile 65535;
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
use epoll;
worker_connections 65535;
multi_accept on;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;
	root	     /home/www/localhost;
        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            #root   html;
            index  index.php index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            #root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        location ~ \.php$ {
        #    root           html;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}

:x
//保存
reboot
记得一定要创建好index.php否则报错403
```