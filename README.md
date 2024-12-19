# Systemd

Инициализация системы. Systemd.

Домашнее задание
Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/default).
Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта (https://gist.github.com/cea2k/1318020).
Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно.


1. Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова

Для начала создаю файл с конфигурацией для сервиса в директории /etc/default - из неё сервис будет брать необходимые переменные.

root@vagrant:~# nano /etc/default/watchlog

root@vagrant:~# cat /etc/default/watchlogog
# Configuration file for my watchlog service
# Place it to /etc/default

# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log

Затем создал /var/log/watchlog.log
root@vagrant:~# nano /var/log/watchlog.log

Создал скрипт:
root@vagrant:~# cat > /opt/watchlog.sh
#!/bin/bash

WORD=$1
LOG=$2
DATE=`date`

if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word, Master!"
else
exit 0
fi

Команда logger отправляет лог в системный журнал.
Добавил права на запуск файла:
root@vagrant:~# chmod +x /opt/watchlog.sh
root@vagrant:~# ll -a /opt/watchlog.sh
-rwxr-xr-x 1 root root 131 Dec 18 13:11 /opt/watchlog.sh*

Создал юнит для сервиса:
root@vagrant:~# cat > /etc/systemd/system/watchlog.service
[Unit]
Description=My watchlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/default/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG

Создал юнит для таймера:
root@vagrant:~# cat > /etc/systemd/system/watchlog.timer
[Unit]
Description=Run watchlog script every 30 second

[Timer]
# Run every 30 second
OnUnitActiveSec=30
Unit=watchlog.service

[Install]
WantedBy=multi-user.target

Далее запустил service и timer:

root@vagrant:~# systemctl enable watchlog.service

root@vagrant:~# systemctl start watchlog.service

root@vagrant:~# systemctl enable watchlog.timer

root@vagrant:~# systemctl start watchlog.timer

И убедился в результате:
root@vagrant:~# tail -n 1000 /var/log/syslog  | grep word
Dec 19 08:09:08 vagrant root: Thu Dec 19 08:09:08 AM UTC 2024: I found word, Master!
Dec 19 08:09:52 vagrant root: Thu Dec 19 08:09:52 AM UTC 2024: I found word, Master!
Dec 19 08:10:29 vagrant root: Thu Dec 19 08:10:29 AM UTC 2024: I found word, Master!
Dec 19 08:11:00 vagrant root: Thu Dec 19 08:11:00 AM UTC 2024: I found word, Master!



2. Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта
Устанавливаю spawn-fcgi и необходимые для него пакеты:

root@vagrant:~# apt install spawn-fcgi php php-cgi php-cli \
 apache2 libapache2-mod-fcgid -y
........
Created symlink /etc/systemd/system/multi-user.target.wants/apache2.service > /lib/systemd/system/apache2.service.
Created symlink /etc/systemd/system/multi-user.target.wants/apache-htcacheclean.service > /lib/systemd/system/apache-htcacheclean.service.
Processing triggers for ufw (0.36.1-4ubuntu0.1) ...
Processing triggers for man-db (2.10.2-1) ...
Processing triggers for libc-bin (2.35-0ubuntu3.8) ...
Processing triggers for php8.1-cli (8.1.2-1ubuntu2.20) ...
Processing triggers for php8.1-cgi (8.1.2-1ubuntu2.20) ...
Processing triggers for libapache2-mod-php8.1 (8.1.2-1ubuntu2.20) ...

Init скрипт взял отсюда: https://gist.github.com/cea2k/1318020, его буду переписывать:

Но перед этим создам файл с настройками для будущего сервиса в файле /etc/spawn-fcgi/fcgi.conf.
Выглядит он так:

root@vagrant:~# cat /etc/spawn-fcgi/fcgi.conf
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u www-data -g www-data -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"

И юнит-файл:

root@vagrant:~# cat /etc/systemd/system/spawn-fcgi.service
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/spawn-fcgi/fcgi.conf
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target

Убеделся, что все успешно работает:

root@vagrant:~# systemctl enable spawn-fcgi
Created symlink /etc/systemd/system/multi-user.target.wants/spawn-fcgi.service > /etc/systemd/system/spawn-fcgi.service.
root@vagrant:~# systemctl start spawn-fcgi

root@vagrant:~# systemctl status spawn-fcgi
? spawn-fcgi.service - Spawn-fcgi startup service by Otus
     Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2024-12-19 10:43:32 UTC; 3min 42s ago
   Main PID: 58351 (php-cgi)
      Tasks: 33 (limit: 2218)
     Memory: 14.0M
        CPU: 35ms
     CGroup: /system.slice/spawn-fcgi.service
             +-58351 /usr/bin/php-cgi
             +-58352 /usr/bin/php-cgi
             +-58353 /usr/bin/php-cgi
             +-58354 /usr/bin/php-cgi
             +-58355 /usr/bin/php-cgi
             +-58356 /usr/bin/php-cgi
             +-58357 /usr/bin/php-cgi
             +-58358 /usr/bin/php-cgi
             +-58359 /usr/bin/php-cgi
             +-58360 /usr/bin/php-cgi
             +-58361 /usr/bin/php-cgi
             +-58362 /usr/bin/php-cgi
             +-58363 /usr/bin/php-cgi
             +-58364 /usr/bin/php-cgi
             +-58365 /usr/bin/php-cgi
             +-58366 /usr/bin/php-cgi
             +-58367 /usr/bin/php-cgi
             +-58368 /usr/bin/php-cgi
             +-58369 /usr/bin/php-cgi
             +-58370 /usr/bin/php-cgi
             +-58371 /usr/bin/php-cgi
             +-58372 /usr/bin/php-cgi
             +-58373 /usr/bin/php-cgi
             +-58374 /usr/bin/php-cgi
             +-58375 /usr/bin/php-cgi
             +-58376 /usr/bin/php-cgi
             +-58377 /usr/bin/php-cgi
             +-58378 /usr/bin/php-cgi
             +-58379 /usr/bin/php-cgi
             +-58380 /usr/bin/php-cgi
             +-58381 /usr/bin/php-cgi
             +-58382 /usr/bin/php-cgi
             L-58383 /usr/bin/php-cgi

Dec 19 10:43:32 vagrant systemd[1]: Started Spawn-fcgi startup service by Otus.

3. Теперь по условию ДЗ нужно доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно

Установливаю Nginx из стандартного репозитория:

root@vagrant:~# apt install nginx -y

Для запуска нескольких экземпляров сервиса модифицирую исходный service для использования различной конфигурации, 
а также PID-файлов. Для этого создам новый Unit для работы с шаблонами (/etc/systemd/system/nginx@.service):

root@vagrant:~# cat /etc/systemd/system/nginx@.service
# =======================
#
# ExecStop sends SIGSTOP (graceful stop) to the nginx process.
# If, after 5s (--retry QUIT/5) nginx is still running, systemd takes control
# and sends SIGTERM (fast shutdown) to the main process.
# After another 5s (TimeoutStopSec=5), and if nginx is alive, systemd sends
# SIGKILL to all the remaining processes in the process group (KillMode=mixed).
#
# nginx signals reference doc:
# http://nginx.org/en/docs/control.html
#
[Unit]
Description=A high performance web server and a reverse proxy server
Documentation=man:nginx(8)
After=network.target nss-lookup.target

[Service]
Type=forking
PIDFile=/run/nginx-%I.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-%I.conf -q -g 'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;'
ExecReload=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx-%I.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target

Далее создаю два файла конфигурации (/etc/nginx/nginx-first.conf, /etc/nginx/nginx-second.conf). 
Их можно сформировать из стандартного конфига /etc/nginx/nginx.conf, с модификацией путей до PID-файлов и разделением по портам и сразу проверить их работу:

root@vagrant:~# systemctl start nginx@first
root@vagrant:~# systemctl status nginx@first
? nginx@first.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/etc/systemd/system/nginx@.service; disabled; vendor preset: enabled)
     Active: active (running) since Thu 2024-12-19 12:56:41 UTC; 1min 53s ago
       Docs: man:nginx(8)
    Process: 1764 ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-first.conf -q -g daemon on; master_process on; (c>
    Process: 1765 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-first.conf -g daemon on; master_process on; (code=exite>
   Main PID: 1766 (nginx)
      Tasks: 3 (limit: 2275)
     Memory: 3.2M
        CPU: 33ms
     CGroup: /system.slice/system-nginx.slice/nginx@first.service
             +-1766 "nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-first.conf -g daemon on; master_process >
             +-1767 "nginx: worker process" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" >
             L-1768 "nginx: worker process" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" >

Dec 19 12:56:41 vagrant systemd[1]: Starting A high performance web server and a reverse proxy server...
Dec 19 12:56:41 vagrant systemd[1]: Started A high performance web server and a reverse proxy server.

root@vagrant:~# systemctl start nginx@second
root@vagrant:~# systemctl status nginx@second
? nginx@second.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/etc/systemd/system/nginx@.service; disabled; vendor preset: enabled)
     Active: active (running) since Thu 2024-12-19 12:56:49 UTC; 2min 37s ago
       Docs: man:nginx(8)
    Process: 1771 ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-second.conf -q -g daemon on; master_process on; (>
    Process: 1772 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_process on; (code=exit>
   Main PID: 1773 (nginx)
      Tasks: 3 (limit: 2275)
     Memory: 3.3M
        CPU: 34ms
     CGroup: /system.slice/system-nginx.slice/nginx@second.service
             +-1773 "nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_process>
             +-1774 "nginx: worker process" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" >
             L-1775 "nginx: worker process" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" >

Dec 19 12:56:49 vagrant systemd[1]: Starting A high performance web server and a reverse proxy server...
Dec 19 12:56:49 vagrant systemd[1]: Started A high performance web server and a reverse proxy server.

или:

root@vagrant:~# ss -tnulp | grep nginx
tcp   LISTEN 0      511           0.0.0.0:9001      0.0.0.0:*    users:(("nginx",pid=1768,fd=6),("nginx",pid=1767,fd=6),("nginx",pid=1766,fd=6))
tcp   LISTEN 0      511           0.0.0.0:9002      0.0.0.0:*    users:(("nginx",pid=1775,fd=6),("nginx",pid=1774,fd=6),("nginx",pid=1773,fd=6))
tcp   LISTEN 0      511           0.0.0.0:80        0.0.0.0:*    users:(("nginx",pid=764,fd=6),("nginx",pid=763,fd=6),(
nginx",pid=761,fd=6))
tcp   LISTEN 0      511              [::]:80           [::]:*    users:(("nginx",pid=764,fd=7),("nginx",pid=763,fd=7),(
nginx",pid=761,fd=7))

Задание выполнено.
__________________
end

