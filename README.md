# otus-task12

# Практика с SELinux

1. Запустить nginx на нестандартном порту 3-мя разными способами:
	* переключатели setsebool;
	* добавление нестандартного порта в имеющийся тип;
	* формирование и установка модуля SELinux.

## Запустить nginx на нестандартном порту 3-мя разными способами

### Решение

Сервером будет выступать vm с debian 12. Развернём сервер с помощью Vagrant файла следующего содержания:
```

Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"
  config.vm.define "selinux"
  config.vm.hostname = "selinux"
  config.vm.network "private_network", ip: "192.168.56.60"
end
```

Выполним подготовку нашего тествого сервера для выполнения задания. \
Установим SELinux и Nginx. \
Подготовку серевера будем выполнять с помощью Ansible. 	
