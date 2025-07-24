
## Сборка RPM-пакета и создание репозитория
#### Задачи
1) Создать свой RPM пакет (можно взять свое приложение, либо собрать, например,
Apache с определенными опциями).
2) Создать свой репозиторий и разместить там ранее собранный RPM.

Реализовать это все либо в Vagrant, либо развернуть у себя через Nginx и дать ссылку на репозиторий.

## Создание RPM-пакета 

#### Установка необходимых для задания пакетов
```
yum install -y wget rpmdevtools rpm-build createrepo  yum-utils cmake gcc git nano
nano .ssh/authorized_keys
● Для примера возьмем пакет Nginx и соберем его с дополнительным модулем ngx_broli"
mkdir rpm && cd rpm
yumdownloader --source nginx
● При установке такого пакета в домашней директории создается дерево каталогов для сборки, далее поставим все зависимости для сборки пакета Nginx"
rpm -Uvh nginx*.src.rpm
yum-builddep nginx
```
#### ● Также нужно скачать исходный код модуля ngx_brotli — он потребуется при сборке:
```
cd 
git clone --recurse-submodules -j8 https://github.com/google/ngx_brotli
cd ngx_brotli/deps/brotli
mkdir out && cd out
```
### ● Собираем модуль ngx_brotli:"
```
cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=OFF -DCMAKE_C_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_CXX_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_INSTALL_PREFIX=./installed ..
```   
+ Вывод команды: -- Build files have been written to: /root/ngx_brotli/deps/brotli/out"
```
cmake --build . --config Release -j 2 --target brotlienc
```
● Нужно поправить сам spec файл, чтобы Nginx собирался с необходимыми нам опциями: находим секцию с параметрами configure (до условий if) и добавляем указание на модуль (обязательно указать завершающий обратный слэш):
--add-module=/root/ngx_brotli \
```
cd ~/rpmbuild/SPECS/
ls
nano nginx.spec
```
● Теперь можно приступить к сборке RPM пакета:"
```
rpmbuild -ba nginx.spec -D 'debug_package %{nil}'
...
Checking for unpackaged file(s): /usr/lib/rpm/check-files /root/rpmbuild/BUILDROOT/nginx-1.20.1-22.el9.3.alma.1.x86_64
Wrote: /root/rpmbuild/SRPMS/nginx-1.20.1-22.el9.3.alma.1.src.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/nginx-1.20.1-22.el9.3.alma.1.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/nginx-mod-stream-1.20.1-22.el9.3.alma.1.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/nginx-mod-mail-1.20.1-22.el9.3.alma.1.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/nginx-mod-http-image-filter-1.20.1-22.el9.3.alma.1.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/nginx-mod-http-perl-1.20.1-22.el9.3.alma.1.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/noarch/nginx-filesystem-1.20.1-22.el9.3.alma.1.noarch.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/nginx-mod-http-xslt-filter-1.20.1-22.el9.3.alma.1.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/noarch/nginx-all-modules-1.20.1-22.el9.3.alma.1.noarch.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/nginx-core-1.20.1-22.el9.3.alma.1.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/nginx-mod-devel-1.20.1-22.el9.3.alma.1.x86_64.rpm
Executing(%clean): /bin/sh -e /var/tmp/rpm-tmp.4WMuWJ
+ umask 022
+ cd /root/rpmbuild/BUILD
+ cd nginx-1.20.1
+ /usr/bin/rm -rf /root/rpmbuild/BUILDROOT/nginx-1.20.1-22.el9.3.alma.1.x86_64
+ RPM_EC=0
++ jobs -p
+ exit 0
[root@alma96srv07 SPECS]#
```
● Убедимся, что пакеты создались:
```
[root@alma96srv07 SPECS]# cd
[root@alma96srv07 ~]# ll rpmbuild/RPMS/x86_64
total 1988
-rw-r--r--. 1 root root   36977 Jul 24 16:25 nginx-1.20.1-22.el9.3.alma.1.x86_64.rpm
-rw-r--r--. 1 root root 1019683 Jul 24 16:25 nginx-core-1.20.1-22.el9.3.alma.1.x86_64.rpm
-rw-r--r--. 1 root root  761121 Jul 24 16:25 nginx-mod-devel-1.20.1-22.el9.3.alma.1.x86_64.rpm
-rw-r--r--. 1 root root   20150 Jul 24 16:25 nginx-mod-http-image-filter-1.20.1-22.el9.3.alma.1.x86_64.rpm
-rw-r--r--. 1 root root   31602 Jul 24 16:25 nginx-mod-http-perl-1.20.1-22.el9.3.alma.1.x86_64.rpm
-rw-r--r--. 1 root root   18883 Jul 24 16:25 nginx-mod-http-xslt-filter-1.20.1-22.el9.3.alma.1.x86_64.rpm
-rw-r--r--. 1 root root   54464 Jul 24 16:25 nginx-mod-mail-1.20.1-22.el9.3.alma.1.x86_64.rpm
-rw-r--r--. 1 root root   81043 Jul 24 16:25 nginx-mod-stream-1.20.1-22.el9.3.alma.1.x86_64.rpm
[root@alma96srv07 ~]# cp ~/rpmbuild/RPMS/noarch/* ~/rpmbuild/RPMS/x86_64/
[root@alma96srv07 ~]# cd ~/rpmbuild/RPMS/x86_64
[root@alma96srv07 x86_64]#
```
### ● Теперь можно установить наш пакет и убедиться, что nginx работает
```
[root@alma96srv07 x86_64]# yum localinstall *.rpm
Last metadata expiration check: 1:06:37 ago on Thu Jul 24 15:35:35 2025.
Dependencies resolved.
...
Installed:
  almalinux-logos-httpd-90.6-2.el9.noarch                                    nginx-2:1.20.1-22.el9.3.alma.1.x86_64
  nginx-all-modules-2:1.20.1-22.el9.3.alma.1.noarch                          nginx-core-2:1.20.1-22.el9.3.alma.1.x86_64
  nginx-filesystem-2:1.20.1-22.el9.3.alma.1.noarch                           nginx-mod-devel-2:1.20.1-22.el9.3.alma.1.x86_64
  nginx-mod-http-image-filter-2:1.20.1-22.el9.3.alma.1.x86_64                nginx-mod-http-perl-2:1.20.1-22.el9.3.alma.1.x86_64
  nginx-mod-http-xslt-filter-2:1.20.1-22.el9.3.alma.1.x86_64                 nginx-mod-mail-2:1.20.1-22.el9.3.alma.1.x86_64
  nginx-mod-stream-2:1.20.1-22.el9.3.alma.1.x86_64

Complete!
[root@alma96srv07 x86_64]#
```
#### ● Далее мы будем использовать nginx для доступа к своему репозиторию

## Создать свой репозиторий и разместить там ранее собранный RPM
● Теперь приступим к созданию своего репозитория. Директория для статики у Nginx по умолчанию /usr/share/nginx/html. Создадим там каталог repo
```
[root@alma96srv07 x86_64]#  mkdir /usr/share/nginx/html/repo
```
● Копируем туда наши собранные RPM-пакеты
```
[root@alma96srv07 x86_64]#cp ~/rpmbuild/RPMS/x86_64/*.rpm /usr/share/nginx/html/repo/
```
● Инициализируем репозиторий командой: [root@packages ~]# createrepo /usr/share/nginx/html/repo/
```
[root@alma96srv07 x86_64]# createrepo /usr/share/nginx/html/repo/
Directory walk started
*Directory walk done - 10 packages	\# Видим в репозитории 10 пакетов
Temporary output repo path: /usr/share/nginx/html/repo/.repodata/
*Preparing sqlite DBs			\# Используется sqlite
Pool started (with 5 workers)
Pool finished
[root@alma96srv07 x86_64]#
```
##### ● Для прозрачности настроим в NGINX доступ к листингу каталога. В файле /etc/nginx/nginx.conf в блоке server добавим следующие директивы:
```
    server {
        listen       80;
        listen       [::]:80;
        server_name  _;
        root         /usr/share/nginx/html;
        # Добавленные директивы по ДЗ-07
        index index.html index.htm;
        autoindex on;
        # Конец вставки

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;
```
● Проверяем синтаксис и перезапускаем NGINX
```
[root@alma96srv07 x86_64]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@alma96srv07 x86_64]#
```
● Nginx, оказывается, не был запущен
```
[root@alma96srv07 x86_64]# nginx -s reload
nginx: [error] invalid PID number "" in "/run/nginx.pid"
[root@alma96srv07 x86_64]# service nginx status
Redirecting to /bin/systemctl status nginx.service
○ nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: inactive (dead)

Jul 24 16:43:24 alma96srv07 systemd[1]: nginx.service: Unit cannot be reloaded because it is inactive.
Jul 24 16:43:24 alma96srv07 systemd[1]: nginx.service: Unit cannot be reloaded because it is inactive.
Jul 24 16:43:25 alma96srv07 systemd[1]: nginx.service: Unit cannot be reloaded because it is inactive.
Jul 24 16:43:25 alma96srv07 systemd[1]: nginx.service: Unit cannot be reloaded because it is inactive.
Jul 24 16:43:25 alma96srv07 systemd[1]: nginx.service: Unit cannot be reloaded because it is inactive.
[root@alma96srv07 x86_64]# service nginx start
Redirecting to /bin/systemctl start nginx.service
[root@alma96srv07 x86_64]# nginx -s reload
[root@alma96srv07 x86_64]#
```
+ Проверяем с помощью curl
```
[root@alma96srv07 x86_64]# curl -a http://localhost/repo/
<html>
<head><title>Index of /repo/</title></head>
<body>
<h1>Index of /repo/</h1><hr><pre><a href="../">../</a>
<a href="repodata/">repodata/</a>                                          24-Jul-2025 13:47                   -
<a href="nginx-1.20.1-22.el9.3.alma.1.x86_64.rpm">nginx-1.20.1-22.el9.3.alma.1.x86_64.rpm</a>            24-Jul-2025 13:46               36977
<a href="nginx-all-modules-1.20.1-22.el9.3.alma.1.noarch.rpm">nginx-all-modules-1.20.1-22.el9.3.alma.1.noarch..&gt;</a> 24-Jul-2025 13:46                8089
<a href="nginx-core-1.20.1-22.el9.3.alma.1.x86_64.rpm">nginx-core-1.20.1-22.el9.3.alma.1.x86_64.rpm</a>       24-Jul-2025 13:46             1019683
<a href="nginx-filesystem-1.20.1-22.el9.3.alma.1.noarch.rpm">nginx-filesystem-1.20.1-22.el9.3.alma.1.noarch.rpm</a> 24-Jul-2025 13:46                9692
<a href="nginx-mod-devel-1.20.1-22.el9.3.alma.1.x86_64.rpm">nginx-mod-devel-1.20.1-22.el9.3.alma.1.x86_64.rpm</a>  24-Jul-2025 13:46              761121
<a href="nginx-mod-http-image-filter-1.20.1-22.el9.3.alma.1.x86_64.rpm">nginx-mod-http-image-filter-1.20.1-22.el9.3.alm..&gt;</a> 24-Jul-2025 13:46               20150
<a href="nginx-mod-http-perl-1.20.1-22.el9.3.alma.1.x86_64.rpm">nginx-mod-http-perl-1.20.1-22.el9.3.alma.1.x86_..&gt;</a> 24-Jul-2025 13:46               31602
<a href="nginx-mod-http-xslt-filter-1.20.1-22.el9.3.alma.1.x86_64.rpm">nginx-mod-http-xslt-filter-1.20.1-22.el9.3.alma..&gt;</a> 24-Jul-2025 13:46               18883
<a href="nginx-mod-mail-1.20.1-22.el9.3.alma.1.x86_64.rpm">nginx-mod-mail-1.20.1-22.el9.3.alma.1.x86_64.rpm</a>   24-Jul-2025 13:46               54464
<a href="nginx-mod-stream-1.20.1-22.el9.3.alma.1.x86_64.rpm">nginx-mod-stream-1.20.1-22.el9.3.alma.1.x86_64.rpm</a> 24-Jul-2025 13:46               81043
</pre><hr></body>
</html>
[root@alma96srv07 x86_64]# lynx http://localhost/repo/
```
● Все готово для того, чтобы протестировать репозиторий.
● Добавим его в /etc/yum.repos.d:
```
[root@alma96srv07 x86_64]# cat >> /etc/yum.repos.d/otus.repo << EOF
[otus]
name=otus-linux
baseurl=http://localhost/repo
gpgcheck=0
enabled=1
EOF
[root@alma96srv07 x86_64]# cat /etc/yum.repos.d/otus.repo
[otus]
name=otus-linux
baseurl=http://localhost/repo
gpgcheck=0
enabled=1
[root@alma96srv07 x86_64]#
```
● Убедимся, что репозиторий подключился и посмотрим, что в нем есть:
```
[root@alma96srv07 x86_64]# yum repolist enabled | grep otus
otus                             otus-linux
[root@alma96srv07 x86_64]#
[root@alma96srv07 x86_64]# cd /usr/share/nginx/html/repo/
[root@alma96srv07 repo]# wget https://repo.percona.com/yum/percona-release-latest.noarch.rpm
--2025-07-24 17:06:41--  https://repo.percona.com/yum/percona-release-latest.noarch.rpm
Resolving repo.percona.com (repo.percona.com)... 49.12.125.205, 2a01:4f8:242:5792::2
Connecting to repo.percona.com (repo.percona.com)|49.12.125.205|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 28404 (28K) [application/x-redhat-package-manager]
Saving to: ‘percona-release-latest.noarch.rpm’

percona-release-latest.noarch.rpm   100%[===================================================================>]  27.74K  --.-KB/s    in 0s

2025-07-24 17:06:42 (61.9 MB/s) - ‘percona-release-latest.noarch.rpm’ saved [28404/28404]

[root@alma96srv07 repo]#
```
● Обновим список пакетов в репозитории
```
[root@alma96srv07 repo]# createrepo /usr/share/nginx/html/repo/
Directory walk started
Directory walk done - 11 packages
Temporary output repo path: /usr/share/nginx/html/repo/.repodata/
Preparing sqlite DBs
Pool started (with 5 workers)
Pool finished
[root@alma96srv07 repo]#  yum makecache
AlmaLinux 9 - AppStream                                                                                         3.5 kB/s | 4.2 kB     00:01
AlmaLinux 9 - BaseOS                                                                                            5.5 kB/s | 3.8 kB     00:00
AlmaLinux 9 - Extras                                                                                            5.6 kB/s | 3.8 kB     00:00
otus-linux                                                                                                      595 kB/s | 7.2 kB     00:00
Metadata cache created.
[root@alma96srv07 repo]# yum list | grep otus
percona-release.noarch                               1.0-31                              otus
[root@alma96srv07 repo]#
```
● Так как Nginx у нас уже стоит, установим репозиторий percona-releas
```
[root@alma96srv07 repo]# yum install -y percona-release.noarch
Last metadata expiration check: 0:00:58 ago on Thu Jul 24 17:07:47 2025.
Dependencies resolved.
...
Installed:
  percona-release-1.0-31.noarch

Complete!
[root@alma96srv07 repo]#
```
● Все прошло успешно. В случае, если потребуется обновить репозиторий (это
делается при каждом добавлении файлов) снова, нужно выполнить команду
createrepo /usr/share/nginx/html/repo/.


