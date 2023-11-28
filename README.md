# otus-task12

# Практика с SELinux

1. Запустить nginx на нестандартном порту 3-мя разными способами:
	* переключатели setsebool;
	* добавление нестандартного порта в имеющийся тип;
	* формирование и установка модуля SELinux.

## Запустить nginx на нестандартном порту 3-мя разными способами

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









