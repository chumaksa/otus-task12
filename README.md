# otus-task12

# Практика с SELinux

1. Запустить nginx на нестандартном порту 3-мя разными способами:
	* переключатели setsebool;
	* добавление нестандартного порта в имеющийся тип;
	* формирование и установка модуля SELinux.

2. Обеспечить работоспособность приложения при включенном selinux.
	* развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
	* выяснить причину неработоспособности механизма обновления зоны;
	* предложить решение (или решения) для данной проблемы;
	* выбрать одно из решений для реализации, предварительно обосновав выбор;
	* реализовать выбранное решение и продемонстрировать его работоспособность


## 1. Запустить nginx на нестандартном порту 3-мя разными способами

### Решение

Сервером будет выступать vm с almalinux 9. Развернём сервер с помощью Vagrant файла следующего содержания:
```

Vagrant.configure("2") do |config|
  config.vm.box = "almalinux/9"
  config.vm.define "selinux"
  config.vm.hostname = "selinux"
  config.vm.network "private_network", ip: "192.168.56.60"
end
```

Выполним подготовку нашего тествого сервера для выполнения задания. \
Установим SELinux и Nginx. \
Подготовку серевера будем выполнять с помощью Ansible. \
Playbook и все требуемые файлы в репозитории. \
Запускаем playbook
```

ansible-playbook nginx.yml 
```

Вывод:
```

PLAY [Install and configure nginx] *************************************************************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************************************************************
ok: [selinux]

TASK [Ensure nginx is installed] ***************************************************************************************************************************************************************************
changed: [selinux]

TASK [Ensure policycoreutils-python-utils is installed] ****************************************************************************************************************************************************
changed: [selinux]

TASK [Create nginx config file from template] **************************************************************************************************************************************************************
changed: [selinux]

RUNNING HANDLER [Reload nginx] *****************************************************************************************************************************************************************************
fatal: [selinux]: FAILED! => {"changed": false, "msg": "Unable to start service nginx: Job for nginx.service failed because the control process exited with error code.\nSee \"systemctl status nginx.service\" and \"journalctl -xeu nginx.service\" for details.\n"}

PLAY RECAP *************************************************************************************************************************************************************************************************
selinux                    : ok=4    changed=3    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
```

Видим, что на шаге "RUNNING HANDLER [Reload nginx]" произошла ошибка. Это ошибка возникла из-за блокировки работы Nginx по нестандартному порту. \

Заходим на наш тестовый сервер с помощью команды - vagrant ssh. \
Проверим включен ли firewall, чтобы убедиться, что блокировка порта производится не им.
```

[vagrant@selinux ~]$ systemctl status firewalld
○ firewalld.service - firewalld - dynamic firewall daemon
     Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; preset: enabled)
     Active: inactive (dead)
       Docs: man:firewalld(1)

```

Firewall выключен. \
Проверим нет ли ошибок в конфигурационных файлах Nginx.
```

[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Видим, что с конфигурацие Nginx всё в порядке. \
В AlmaLinux SELinux включен по умолчанию. Проверим режим его работы.
```

[root@selinux ~]# getenforce 
Enforcing
```

Посмотрим логи на предмет информации по номеру нашего кастомного порта для Nginx.
```

[root@selinux ~]# cat /var/log/audit/audit.log | grep 4881
type=AVC msg=audit(1701193269.465:1352): avc:  denied  { name_bind } for  pid=8309 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```

Видим ошибку доступа и блокровку работы Nginx на этом порту. \
С помощью утилиты audit2why получим больше информации об этом событии.
```

[root@selinux ~]# grep 1701193269.465:1352 /var/log/audit/audit.log | audit2why 
type=AVC msg=audit(1701193269.465:1352): avc:  denied  { name_bind } for  pid=8309 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
```

Последуем совету утилиты. Изменим параметр nisnis_enabled с помощью переключателей setsebool.
```

[root@selinux ~]# setsebool -P nis_enabled 1
```

Перезапустим Nginx и проверим его статус.
```

[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Tue 2023-11-28 17:47:52 UTC; 16s ago
    Process: 8398 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 8399 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 8400 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 8401 (nginx)
      Tasks: 2 (limit: 5508)
     Memory: 1.8M
        CPU: 115ms
     CGroup: /system.slice/nginx.service
             ├─8401 "nginx: master process /usr/sbin/nginx"
             └─8402 "nginx: worker process"

Nov 28 17:47:52 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Nov 28 17:47:52 selinux nginx[8399]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Nov 28 17:47:52 selinux nginx[8399]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Nov 28 17:47:52 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

С помощью curl или путём захода через браузер по http://192.168.56.60:4881/ проверяем работоспособность Nginx.

Для дальнейшего выполнения домашнего задания вернём обратно запрет работы Nginx на нестандартном порту.
```

[root@selinux ~]# setsebool -P nis_enabled 0
[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> off
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.
```

Теперь исправим проблему с помощью добавления нестандартного порта в имеющийся тип. \
Найдём тип для http трафика.
```

[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

Далее добавим наш порт 4881 в имеющийся тип http_port_t, перезапустим nginx и проверим его работоспособность.
```

[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep  http_port_t
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
     Active: active (running) since Wed 2023-11-29 15:41:59 UTC; 15s ago
    Process: 976 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 977 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 978 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 979 (nginx)
      Tasks: 2 (limit: 5508)
     Memory: 1.8M
        CPU: 113ms
     CGroup: /system.slice/nginx.service
             ├─979 "nginx: master process /usr/sbin/nginx"
             └─980 "nginx: worker process"

Nov 29 15:41:59 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Nov 29 15:41:59 selinux nginx[977]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Nov 29 15:41:59 selinux nginx[977]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Nov 29 15:41:59 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

Для дальнейшго выполнения задания возвращаем запрет обратно, удаляем добавленный порт.
```

[root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep  http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.
```

Nginx снова не работает. Исправим эту проблему с помощью формирования и установки модуля SELinux. \
Для формирования модуля воспользуемся утилитой audit2allow, которая создаёт этот модуль на основе логов.
```

[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```

Установим сформированный модуль и выполним запуск Nginx.
```

[root@selinux ~]# semodule -i nginx.pp
[root@selinux ~]# systemctl start nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
     Active: active (running) since Wed 2023-11-29 16:06:07 UTC; 8s ago
    Process: 1030 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 1031 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 1033 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 1034 (nginx)
      Tasks: 2 (limit: 5508)
     Memory: 1.8M
        CPU: 114ms
     CGroup: /system.slice/nginx.service
             ├─1034 "nginx: master process /usr/sbin/nginx"
             └─1035 "nginx: worker process"

Nov 29 16:06:06 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Nov 29 16:06:07 selinux nginx[1031]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Nov 29 16:06:07 selinux nginx[1031]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Nov 29 16:06:07 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

С помощью curl или путём захода через браузер по http://192.168.56.60:4881/ проверяем работоспособность Nginx.

## 2. Обеспечить работоспособность приложения при включенном selinux.

### Решение

Для выполенния этой задчи создадим подкаталог "selinux_dns_problems" и все дальнейшие работы будем проводить в нём. \
Скопируем в эту папку все файлы с https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems и запустим тестовый стенд.
```

chumaksa@debpc:~/otus/otus-task12/selinux_dns_problems$ ls -1
provisioning
README.md
Vagrantfile

chumaksa@debpc:~/otus/otus-task12/selinux_dns_problems$ vagrant up

chumaksa@debpc:~/otus/otus-task12/selinux_dns_problems$ vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```

Подключимся к client и попробуем внести изменения в DNS зону.
```

[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
```

Видим ошибку. Воспользуемся утилитой audit2why для получения дополнительных сведений о проблеме.
```

[vagrant@client ~]$ su -l
Password: 
[root@client ~]# cat /var/log/audit/audit.log | audit2why
[root@client ~]#
```

Видим, что на client нет ошибок. Подключимся к DNS серверу ns01 и посмотрим логи на нём.
```

chumaksa@debpc:~/otus/otus-task12/selinux_dns_problems$ vagrant ssh ns01
Last login: Thu Feb 29 06:55:58 2024 from 10.0.2.2
[vagrant@ns01 ~]$ su -l
Password: 
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1709190166.679:1982): avc:  denied  { create } for  pid=5324 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.


```

В логах мы видим, что ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t. \
Проверим данную проблему в каталоге /etc/named.
```

[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```

Видим, что контекст безопасности неправильный. \
Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге. \
Посмотреть в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux.
```

[root@ns01 ~]# semanage fcontext -l | grep named
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0 
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0 
/etc/unbound(/.*)?                                 all files          system_u:object_r:named_conf_t:s0 
/var/run/bind(/.*)?                                all files          system_u:object_r:named_var_run_t:s0 
/var/log/named.*                                   regular file       system_u:object_r:named_log_t:s0 
/var/run/named(/.*)?                               all files          system_u:object_r:named_var_run_t:s0 
/var/named/data(/.*)?                              all files          system_u:object_r:named_cache_t:s0 
/dev/xen/tapctrl.*                                 named pipe         system_u:object_r:xenctl_t:s0 
/var/run/unbound(/.*)?                             all files          system_u:object_r:named_var_run_t:s0 
/var/lib/softhsm(/.*)?                             all files          system_u:object_r:named_cache_t:s0 
/var/lib/unbound(/.*)?                             all files          system_u:object_r:named_cache_t:s0 
/var/named/slaves(/.*)?                            all files          system_u:object_r:named_cache_t:s0 
/var/named/chroot(/.*)?                            all files          system_u:object_r:named_conf_t:s0 
/etc/named\.rfc1912.zones                          regular file       system_u:object_r:named_conf_t:s0 
/var/named/dynamic(/.*)?                           all files          system_u:object_r:named_cache_t:s0 
/var/named/chroot/etc(/.*)?                        all files          system_u:object_r:etc_t:s0 
/var/named/chroot/lib(/.*)?                        all files          system_u:object_r:lib_t:s0 
/var/named/chroot/proc(/.*)?                       all files          <<None>>
/var/named/chroot/var/tmp(/.*)?                    all files          system_u:object_r:named_cache_t:s0 
/var/named/chroot/usr/lib(/.*)?                    all files          system_u:object_r:lib_t:s0 
/var/named/chroot/etc/pki(/.*)?                    all files          system_u:object_r:cert_t:s0 
/var/named/chroot/run/named.*                      all files          system_u:object_r:named_var_run_t:s0 
/var/named/chroot/var/named(/.*)?                  all files          system_u:object_r:named_zone_t:s0 
/usr/lib/systemd/system/named.*                    regular file       system_u:object_r:named_unit_file_t:s0 
/var/named/chroot/var/run/dbus(/.*)?               all files          system_u:object_r:system_dbusd_var_run_t:s0 
/usr/lib/systemd/system/unbound.*                  regular file       system_u:object_r:named_unit_file_t:s0 
/var/named/chroot/var/log/named.*                  regular file       system_u:object_r:named_log_t:s0 
/var/named/chroot/var/run/named.*                  all files          system_u:object_r:named_var_run_t:s0 
/var/named/chroot/var/named/data(/.*)?             all files          system_u:object_r:named_cache_t:s0 
/usr/lib/systemd/system/named-sdb.*                regular file       system_u:object_r:named_unit_file_t:s0 
/var/named/chroot/var/named/slaves(/.*)?           all files          system_u:object_r:named_cache_t:s0 
/var/named/chroot/etc/named\.rfc1912.zones         regular file       system_u:object_r:named_conf_t:s0 
/var/named/chroot/var/named/dynamic(/.*)?          all files          system_u:object_r:named_cache_t:s0 
/var/run/ndc                                       socket             system_u:object_r:named_var_run_t:s0 
/dev/gpmdata                                       named pipe         system_u:object_r:gpmctl_t:s0 
/dev/initctl                                       named pipe         system_u:object_r:initctl_t:s0 
/dev/xconsole                                      named pipe         system_u:object_r:xconsole_device_t:s0 
/usr/sbin/named                                    regular file       system_u:object_r:named_exec_t:s0 
/etc/named\.conf                                   regular file       system_u:object_r:named_conf_t:s0 
/usr/sbin/lwresd                                   regular file       system_u:object_r:named_exec_t:s0 
/var/run/initctl                                   named pipe         system_u:object_r:initctl_t:s0 
/usr/sbin/unbound                                  regular file       system_u:object_r:named_exec_t:s0 
/usr/sbin/named-sdb                                regular file       system_u:object_r:named_exec_t:s0 
/var/named/named\.ca                               regular file       system_u:object_r:named_conf_t:s0 
/etc/named\.root\.hints                            regular file       system_u:object_r:named_conf_t:s0 
/var/named/chroot/dev                              directory          system_u:object_r:device_t:s0 
/etc/rc\.d/init\.d/named                           regular file       system_u:object_r:named_initrc_exec_t:s0 
/usr/sbin/named-pkcs11                             regular file       system_u:object_r:named_exec_t:s0 
/etc/rc\.d/init\.d/unbound                         regular file       system_u:object_r:named_initrc_exec_t:s0 
/usr/sbin/unbound-anchor                           regular file       system_u:object_r:named_exec_t:s0 
/usr/sbin/named-checkconf                          regular file       system_u:object_r:named_checkconf_exec_t:s0 
/usr/sbin/unbound-control                          regular file       system_u:object_r:named_exec_t:s0 
/var/named/chroot_sdb/dev                          directory          system_u:object_r:device_t:s0 
/var/named/chroot/var/log                          directory          system_u:object_r:var_log_t:s0 
/var/named/chroot/dev/log                          socket             system_u:object_r:devlog_t:s0 
/etc/rc\.d/init\.d/named-sdb                       regular file       system_u:object_r:named_initrc_exec_t:s0 
/var/named/chroot/dev/null                         character device   system_u:object_r:null_device_t:s0 
/var/named/chroot/dev/zero                         character device   system_u:object_r:zero_device_t:s0 
/usr/sbin/unbound-checkconf                        regular file       system_u:object_r:named_exec_t:s0 
/var/named/chroot/dev/random                       character device   system_u:object_r:random_device_t:s0 
/var/run/systemd/initctl/fifo                      named pipe         system_u:object_r:initctl_t:s0 
/var/named/chroot/etc/rndc\.key                    regular file       system_u:object_r:dnssec_t:s0 
/usr/share/munin/plugins/named                     regular file       system_u:object_r:services_munin_plugin_exec_t:s0 
/var/named/chroot_sdb/dev/null                     character device   system_u:object_r:null_device_t:s0 
/var/named/chroot_sdb/dev/zero                     character device   system_u:object_r:zero_device_t:s0 
/var/named/chroot/etc/localtime                    regular file       system_u:object_r:locale_t:s0 
/var/named/chroot/etc/named\.conf                  regular file       system_u:object_r:named_conf_t:s0 
/var/named/chroot_sdb/dev/random                   character device   system_u:object_r:random_device_t:s0 
/etc/named\.caching-nameserver\.conf               regular file       system_u:object_r:named_conf_t:s0 
/usr/lib/systemd/systemd-hostnamed                 regular file       system_u:object_r:systemd_hostnamed_exec_t:s0 
/var/named/chroot/var/named/named\.ca              regular file       system_u:object_r:named_conf_t:s0 
/var/named/chroot/etc/named\.root\.hints           regular file       system_u:object_r:named_conf_t:s0 
/var/named/chroot/etc/named\.caching-nameserver\.conf regular file       system_u:object_r:named_conf_t:s0 
/var/named/chroot/lib64 = /usr/lib
/var/named/chroot/usr/lib64 = /usr/lib
```

Изменим тип контекста безопасности для каталога /etc/named.
```

[root@ns01 ~]# chcon -R -t named_zone_t /etc/named

[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```

Видим, что контекст безопасности успешно изменился. \
Далее зайдём на client и попробуем опять внести изменения.
```

[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit

[root@client ~]# dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51286
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; Query time: 0 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Thu Feb 29 07:18:39 UTC 2024
;; MSG SIZE  rcvd: 96
```

Видим, что изменения прошли без ошибок и успешно применились.



