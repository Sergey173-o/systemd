# systemd
понимать различие систем инициализации; использовать основные утилиты systemd; изучить состав и синтаксис systemd unit; создание unit-файла

## Мониторинг лога

```bash

cd /etc/default/
cat > files/watchlog.default << 'EOF'
# Configuration file for my watchlog service
# Place it to /etc/default
WORD="ALERT"
LOG=/var/log/watchlog.log
EOF
root@nfsc:~/systemd-hw# cat > files/watchlog.sh << 'EOF'
#!/bin/bash
WORD=$1
LOG=$2
DATE=$(date)
if grep "$WORD" "$LOG" &> /dev/null
then
    logger "$DATE: I found word, Master!"
else
    exit 0
fi
EOF
chmod +x files/watchlog.sh
root@nfsc:~/systemd-hw#

```

### Unit сервиса 

```bash
root@nfsc:~/systemd-hw# cat > files/watchlog.service << 'EOF'
[Unit]
Description=My watchlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/default/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
EOF
root@nfsc:~/systemd-hw# cat > files/watchlog.timer << 'EOF'
[Unit]
Description=Run watchlog script every 30 second

[Timer]
OnUnitActiveSec=30
Unit=watchlog.service

[Install]
WantedBy=multi-user.target
EOF
root@nfsc:~/systemd-hw#

```


```bash
root@nfsc:~/systemd-hw# sudo systemctl daemon-reload
sudo systemctl enable --now watchlog.timer
echo "test line with ALERT here" | sudo tee -a /var/log/watchlog.log
sleep 35
sudo tail -n 20 /var/log/syslog | grep "found word"
Created symlink /etc/systemd/system/multi-user.target.wants/watchlog.timer → /etc/systemd/system/watchlog.timer.
test line with ALERT here
root@nfsc:~/systemd-hw#

```

---

##  spawn-fcgi 

### Установка 

```bash
sudo apt-get update
sudo apt-get install -y spawn-fcgi php php-cgi php-cli apache2 libapache2-mod-fcgid
```

### Конфиг `/etc/spawn-fcgi/fcgi.conf`  `files/fcgi.conf`

```bash
sudo mkdir -p /etc/spawn-fcgi
cat > files/fcgi.conf << 'EOF'
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u www-data -g www-data -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"
EOF
```

### Unit spawn-fcgi.service

```bash
cat > files/spawn-fcgi.service << 'EOF'
[Unit]
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
EOF
```

Проверка:

```bash
root@nfsc:~/systemd-hw# sudo systemctl daemon-reload
sudo systemctl start spawn-fcgi
sudo systemctl status spawn-fcgi
● spawn-fcgi.service - Spawn-fcgi startup service by Otus
     Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; preset: enabled)
     Active: active (running) since Wed 2026-08-26 17:28:41 UTC; 169ms ago
   Main PID: 36706 (php-cgi)
      Tasks: 33 (limit: 4600)
     Memory: 14.8M (peak: 15.1M)
        CPU: 67ms
     CGroup: /system.slice/spawn-fcgi.service
             ├─36706 /usr/bin/php-cgi
             ├─36711 /usr/bin/php-cgi
             ├─36712 /usr/bin/php-cgi
             ├─36713 /usr/bin/php-cgi
             ├─36714 /usr/bin/php-cgi
             ├─36715 /usr/bin/php-cgi
             ├─36716 /usr/bin/php-cgi
             ├─36717 /usr/bin/php-cgi
             ├─36718 /usr/bin/php-cgi
             ├─36719 /usr/bin/php-cgi
             ├─36720 /usr/bin/php-cgi
             ├─36721 /usr/bin/php-cgi
             ├─36722 /usr/bin/php-cgi
             ├─36723 /usr/bin/php-cgi
             ├─36724 /usr/bin/php-cgi
             ├─36725 /usr/bin/php-cgi
             ├─36726 /usr/bin/php-cgi
             ├─36727 /usr/bin/php-cgi
             ├─36728 /usr/bin/php-cgi
             ├─36729 /usr/bin/php-cgi
             ├─36730 /usr/bin/php-cgi
             ├─36731 /usr/bin/php-cgi
             ├─36732 /usr/bin/php-cgi
             ├─36733 /usr/bin/php-cgi
             ├─36734 /usr/bin/php-cgi
             ├─36735 /usr/bin/php-cgi
             ├─36736 /usr/bin/php-cgi
             ├─36737 /usr/bin/php-cgi
             ├─36738 /usr/bin/php-cgi
             ├─36739 /usr/bin/php-cgi
             ├─36740 /usr/bin/php-cgi
             ├─36741 /usr/bin/php-cgi
             └─36742 /usr/bin/php-cgi

Aug 26 17:28:41 nfsc systemd[1]: Started spawn-fcgi.service - Spawn-fcgi startup service by Otus.
root@nfsc:~/systemd-hw#
```

---

##  Несколько инстансов nginx

### Установка

```bash
sudo apt-get install -y nginx
```

### Шаблонный unit 

```bash
cat > files/nginx@.service << 'EOF'
# Stop dance for nginx
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
EOF
```

### Конфиги инстансов 

```bash
sudo cp /etc/nginx/nginx.conf /tmp/nginx-first.conf
sudo sed -i \
  -e 's|^pid .*;|pid /run/nginx-first.pid;|' \
  /tmp/nginx-first.conf

vim /tmp/nginx-first.conf

server {
    listen 9001;

```

```bash
sudo cp /etc/nginx/nginx.conf /tmp/nginx-second.conf
sudo sed -i \
  -e 's|^pid .*;|pid /run/nginx-second.pid;|' \
  /tmp/nginx-second.conf

vim /tmp/nginx-second.conf
  http {
     server {
    listen 9002;
     }

```


### Проверка

```bash
sudo systemctl daemon-reload
sudo systemctl start nginx@first
sudo systemctl start nginx@second

ss -tnulp | grep nginx
ps afx | grep nginx

root@nfsc:~/systemd-hw# ss -tnulp | grep nginx
ps afx | grep nginx
tcp   LISTEN 0      511                 0.0.0.0:9001       0.0.0.0:*    users:(("nginx",pid=37333,fd=5),("nginx",pid=37332,fd=5),("nginx",pid=37331,fd=5),("nginx",pid=37330,fd=5),("nginx",pid=37329,fd=5))
tcp   LISTEN 0      511                 0.0.0.0:9002       0.0.0.0:*    users:(("nginx",pid=37316,fd=5),("nginx",pid=37315,fd=5),("nginx",pid=37314,fd=5),("nginx",pid=37313,fd=5),("nginx",pid=37312,fd=5))
  37342 pts/1    S+     0:00                              \_ grep --color=auto nginx
  37312 ?        Ss     0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_process on;
  37313 ?        S      0:00  \_ nginx: worker process
  37314 ?        S      0:00  \_ nginx: worker process
  37315 ?        S      0:00  \_ nginx: worker process
  37316 ?        S      0:00  \_ nginx: worker process
  37329 ?        Ss     0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-first.conf -g daemon on; master_process on;
  37330 ?        S      0:00  \_ nginx: worker process
  37331 ?        S      0:00  \_ nginx: worker process
  37332 ?        S      0:00  \_ nginx: worker process
  37333 ?        S      0:00  \_ nginx: worker process


```


