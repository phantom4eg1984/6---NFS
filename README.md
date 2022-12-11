=============================Настройка сервера===================================
Необходимо доустановить утилиты для отладки NFS сервера:
[root@nfss ~]# yum install nfs-utils

Запускаем firewall и убеждаем что сервис был запущен:
[root@nfss ~]# systemctl enable firewalld --now
[root@nfss ~]# systemctl status firewalld.service 

Разрешаем доступ к сервисам которые использует NFS:
[root@nfss ~]# firewall-cmd --add-service="nfs3" --add-service="rpc-bind" --add-service="mountd" --permanent

Настройки конфигурации сервера NFS находятся в файле /etc/nfs.conf. По умолчанию для запуска NFSv3 файл править нет необходимости.

Запускаем сервис NFS и проверяем что порты 2049/udp, 2049/tcp, 20048/udp, 20048/tcp, 111/udp, 111/tcp прослушиваются на сетевом интерфейсе.

root@nfss ~]# systemctl enable nfs --now

[root@nfss ~]# ss -tnplu | grep 2049
udp    UNCONN     0      0         *:2049                  *:*                  
udp    UNCONN     0      0      [::]:2049               [::]:*                  
tcp    LISTEN     0      64        *:2049                  *:*                  
tcp    LISTEN     0      64     [::]:2049               [::]:*                  
[root@nfss ~]# ss -tnplu | grep 20048
udp    UNCONN     0      0         *:20048                 *:*                   users:(("rpc.mountd",pid=2647,fd=7))
udp    UNCONN     0      0      [::]:20048              [::]:*                   users:(("rpc.mountd",pid=2647,fd=9))
tcp    LISTEN     0      128       *:20048                 *:*                   users:(("rpc.mountd",pid=2647,fd=8))
tcp    LISTEN     0      128    [::]:20048              [::]:*                   users:(("rpc.mountd",pid=2647,fd=10))
[root@nfss ~]# ss -tnplu | grep 111
udp    UNCONN     0      0         *:111                   *:*                   users:(("rpcbind",pid=386,fd=6))
udp    UNCONN     0      0      [::]:111                [::]:*                   users:(("rpcbind",pid=386,fd=9))
tcp    LISTEN     0      128       *:111                   *:*                   users:(("rpcbind",pid=386,fd=8))
tcp    LISTEN     0      128    [::]:111                [::]:*                   users:(("rpcbind",pid=386,fd=11))

Создаем каталог который будем экспортировать, настраиваем права доступа.
[root@nfss ~]# mkdir -p /srv/share/upload
[root@nfss ~]# chown -R nfsnobody:nfsnobody /srv/share/
[root@nfss ~]# chmod 0777 /srv/share/upload/

[root@nfss ~]# ls -la /srv/share/
total 0
drwxr-xr-x. 3 nfsnobody nfsnobody 20 Dec  8 15:31 .
drwxr-xr-x. 3 root      root      19 Dec  8 15:31 ..
drwxrwxrwx. 2 nfsnobody nfsnobody  6 Dec  8 15:31 upload

В файле /etc/exports добавляем строку для экспорта нашего каталога:
[root@nfss ~]# cat /etc/exports
/srv/share 192.168.100.92(rw,sync,root_squash)

Экспортируем каталог и проверяем результат того что каталог был экспортировоан:
[root@nfss ~]# exportfs -r
[root@nfss ~]# exportfs -s
/srv/share  192.168.100.92(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)

=============================Настройка клиента===================================
Устанавливаем nfs-utils
[root@nfsc ~]# yum install -y nfs-utils

Включаем firewall и проверяем его статус
[root@nfsc ~]# systemctl enable firewalld --now
[root@nfsc ~]# systemctl status firewalld

Добавляем в /etc/fstab строку для монтирования каталога по NFS. Перезапускаем сервисы, после этого при первом обращении к каталогу /mnt будет монтироваться
каталог upload по NFS.
[root@nfsc ~]# echo "192.168.100.91:/srv/share/ /mnt nfs vers=3,proto=udp,noauto,x-systemd.automount 0 0" >> /etc/fstab
[root@nfsc ~]# systemctl daemon-reload 
[root@nfsc ~]# systemctl restart remote-fs.target

[root@nfsc ~]# mount | grep mnt
systemd-1 on /mnt type autofs (rw,relatime,fd=28,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=10972)
192.168.100.91:/srv/share on /mnt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.100.91,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.100.91)

=============================Проверка. Диагностика.===================================
Создаем тестовые файлы, проверяем права доступа на сервере и клиенте:
[root@nfss ~]# touch /srv/share/upload/testfile
[root@nfss ~]# echo "Test" >> /srv/share/upload/testfile
[root@nfss ~]# ls -la /srv/share/upload/testfile 
-rw-r--r--. 1 root root 5 Dec 10 08:24 /srv/share/upload/testfile
[root@nfsc ~]# ls -la /mnt/upload/testfile 
-rw-r--r--. 1 root root 5 Dec 10 08:24 /mnt/upload/testfile
[root@nfsc ~]# cat /mnt/upload/testfile 
Test

[root@nfsc ~]# cd /mnt/upload/
[root@nfsc upload]# touch test1
[root@nfsc upload]# echo "Test1" >> /mnt/upload/test1
[root@nfsc upload]# cat /mnt/upload/test1
Test1
[root@nfsc upload]# ls -la /mnt/upload/
total 8
drwxrwxrwx. 2 nfsnobody nfsnobody 35 Dec 10 08:39 .
drwxr-xr-x. 3 nfsnobody nfsnobody 20 Dec  8 15:31 ..
-rw-r--r--. 1 nfsnobody nfsnobody  6 Dec 10 08:40 test1
-rw-r--r--. 1 root      root       5 Dec 10 08:24 testfile


Производим различные проверки, собираем диагностическиую информацию на сервере:
[root@nfss ~]# systemctl status nfs
● nfs-server.service - NFS server and services
   Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled; vendor preset: disabled)
  Drop-In: /run/systemd/generator/nfs-server.service.d

           └─order-with-mounts.conf
   Active: active (exited) since Sat 2022-12-10 08:15:00 UTC; 16min ago

[root@nfss ~]# systemctl status firewalld.service 
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2022-12-10 08:14:55 UTC; 23min ago

[root@nfss ~]# exportfs -s
/srv/share  192.168.100.92/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)

[root@nfss ~]# showmount -a 192.168.100.91
All mount points on 192.168.100.91:
192.168.100.92:/srv/share

На клиенте:
[root@nfsc upload]# showmount -a 192.168.100.91
All mount points on 192.168.100.91:
192.168.100.92:/srv/share

Все проверки были пройдены. NFS сервер был создан, каталог примонтирован на клиенте.

