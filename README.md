# ****Практика с SELinux.**** #

### Описание домашннего задания ###

1. Запустить nginx на нестандартном порту 3-мя разными способами:
* переключатели setsebool;
* добавление нестандартного порта в имеющийся тип;
* формирование и установка модуля SELinux.
К сдаче:
* README с описанием каждого решения (скриншоты и демонстрация приветствуются). 

2. Обеспечить работоспособность приложения при включенном selinux.
* развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems; 
* выяснить причину неработоспособности механизма обновления зоны (см. README);
* предложить решение (или решения) для данной проблемы;
* выбрать одно из решений для реализации, предварительно обосновав выбор;
* реализовать выбранное решение и продемонстрировать его работоспособность

### Выполнение ###

Задание выполняется на рабочей станции с ОС Ubuntu 22.04.4 LTS с заранее установленными Vagrant 2.4.1 и VirtualBox 7.0, а также Ansible 2.16.5 и Python 3.10.12. Перед выполнением предварительно подготовлен репозиторий https://github.com/ConstantaNF/selinux.git

### **Подготовка окружения** ###

Для развёртывания управляемой ВМ посредством Vagrant создаю Vagrantfile в заранее подготовленном каталоге `/home/adminkonstantin/selinux` со следующим содержимым:

```
# -*- mode: ruby -*-
# vim: set ft=ruby :


MACHINES = {
  :selinux => {
        :box_name => "centos/7",
        :box_version => "2004.01",
        #:provision => "test.sh",       
  },
}


Vagrant.configure("2") do |config|


  MACHINES.each do |boxname, boxconfig|


      config.vm.define boxname do |box|


        box.vm.box = boxconfig[:box_name]
        box.vm.box_version = boxconfig[:box_version]


        box.vm.host_name = "selinux"
        box.vm.network "forwarded_port", guest: 4881, host: 4881


        box.vm.provider :virtualbox do |vb|
              vb.customize ["modifyvm", :id, "--memory", "1024"]
              needsController = false
        end


        box.vm.provision "shell", inline: <<-SHELL
          yum install -y policycoreutils-python
          yum install -y nano
          #install epel-release
          yum install -y epel-release
          #install nginx
          yum install -y nginx
          #change nginx port
          sed -ie 's/:80/:4881/g' /etc/nginx/nginx.conf
          sed -i 's/listen       80;/listen       4881;/' /etc/nginx/nginx.conf
          #disable SELinux
          #setenforce 0
          #start nginx
          systemctl start nginx
          systemctl status nginx
          #check nginx port
          ss -tlpn | grep 4881
        SHELL
    end
  end
end
```

Стартую ВМ:

```
adminkonstantin@2OSUbuntu:~/selinux$ vagrant up
```

Результатом выполнения команды является созданная виртуальная машина с установленным nginx, который работает на порту TCP 4881. Порт TCP 4881 уже проброшен до хоста. SELinux включен. Во время развёртывания стенда попытка запустить nginx завершается с ошибкой,
вызванной тем, что SELinux блокирует работу nginx на нестандартном порту.

Подключаюсь к созданной ВМ по ssh:

```
adminkonstantin@2OSUbuntu:~/selinux$ vagrant ssh
```

Использую УЗ root:

```
[vagrant@selinux ~]$ sudo -i
```

### **1. Запуск nginx на нестандартном порту 3-мя разными способами** ###

Для начала проверим, что в ОС отключен файервол и конфигурация nginx настроена без ошибок:

```
[root@selinux ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```

```
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Далее проверим режим работы SELinux:

```
[root@selinux ~]# getenforce
Enforcing
```

### **Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool** ###

Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта:

```
[root@selinux ~]# cat /var/log/audit/audit.log | grep 4881
type=AVC msg=audit(1718692554.285:883): avc:  denied  { name_bind } for  pid=3051 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```

Копируем время, в которое был записан этот лог и с помощью утилиты audit2why смотрим причину блокировки:

```
[root@selinux ~]# cat /var/log/audit/audit.log | grep 1718692554.285:883 | audit2why
type=AVC msg=audit(1718692554.285:883): avc:  denied  { name_bind } for  pid=3051 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
```

Исходя из вывода утилиты, мы видим, что нам нужно поменять параметр nis_enabled. 
Включим параметр nis_enabled и перезапустим nginx:

```
[root@selinux ~]# setsebool -P nis_enabled on
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2024-06-18 08:34:28 UTC; 17s ago
  Process: 2077 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2075 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 2073 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2079 (nginx)
   CGroup: /system.slice/nginx.service
           ├─2079 nginx: master process /usr/sbin/nginx
           └─2081 nginx: worker process

Jun 18 08:34:28 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jun 18 08:34:28 selinux nginx[2075]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 18 08:34:28 selinux nginx[2075]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jun 18 08:34:28 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

Также можно проверить работу nginx из браузера. Заходим в любой браузер на хосте и переходим по адресу http://127.0.0.1:4881

![изображение](https://github.com/ConstantaNF/selinux/assets/162187256/18c60e49-0ceb-4d13-a7a5-e46f66576bc2)

Проверим статус включенного параметра nis_enabled:

```
[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> on
```

Вернём запрет работы nginx на порту 4881 обратно. Для этого отключим nis_enabled, после чего служба nginx снова не запустится:

```
[root@selinux ~]# setsebool -P nis_enabled off
[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> off
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Tue 2024-06-18 08:46:17 UTC; 16s ago
  Process: 2128 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 2126 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2079 (code=exited, status=0/SUCCESS)

Jun 18 08:46:17 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Jun 18 08:46:17 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jun 18 08:46:17 selinux nginx[2128]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 18 08:46:17 selinux nginx[2128]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Jun 18 08:46:17 selinux nginx[2128]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jun 18 08:46:17 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Jun 18 08:46:17 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Jun 18 08:46:17 selinux systemd[1]: Unit nginx.service entered failed state.
Jun 18 08:46:17 selinux systemd[1]: nginx.service failed.
```

### **Разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип** ###

Поиск имеющегося типа, для http трафика:

```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

Добавим порт в тип http_port_t:

```
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep  http_port_t
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```

Теперь перезапустим службу nginx и проверим её работу:

```
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2024-06-18 09:21:29 UTC; 11s ago
  Process: 2170 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2167 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 2166 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2172 (nginx)
   CGroup: /system.slice/nginx.service
           ├─2172 nginx: master process /usr/sbin/nginx
           └─2174 nginx: worker process

Jun 18 09:21:29 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jun 18 09:21:29 selinux nginx[2167]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 18 09:21:29 selinux nginx[2167]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jun 18 09:21:29 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

Проверим работу nginx из браузера. Заходим в любой браузер на хосте и переходим по адресу http://127.0.0.1:4881

![изображение](https://github.com/ConstantaNF/selinux/assets/162187256/18c60e49-0ceb-4d13-a7a5-e46f66576bc2)

Удалим нестандартный порт из имеющегося типа:

```
[root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep  http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Tue 2024-06-18 09:27:20 UTC; 11s ago
  Process: 2170 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2194 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 2193 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2172 (code=exited, status=0/SUCCESS)

Jun 18 09:27:20 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Jun 18 09:27:20 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jun 18 09:27:20 selinux nginx[2194]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 18 09:27:20 selinux nginx[2194]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Jun 18 09:27:20 selinux nginx[2194]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jun 18 09:27:20 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Jun 18 09:27:20 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Jun 18 09:27:20 selinux systemd[1]: Unit nginx.service entered failed state.
Jun 18 09:27:20 selinux systemd[1]: nginx.service failed.
```

### **Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux** ###

Посмотрим логи SELinux, которые относятся к nginx:

```
[root@selinux ~]# grep nginx /var/log/audit/audit.log
type=SOFTWARE_UPDATE msg=audit(1718692553.898:881): pid=2876 uid=0 auid=1000 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='sw="nginx-filesystem-1:1.20.1-10.el7.noarch" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=SOFTWARE_UPDATE msg=audit(1718692554.083:882): pid=2876 uid=0 auid=1000 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='sw="nginx-1:1.20.1-10.el7.x86_64" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=AVC msg=audit(1718692554.285:883): avc:  denied  { name_bind } for  pid=3051 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1718692554.285:883): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=55c1417328b8 a2=10 a3=7fff2c8a3c60 items=0 ppid=1 pid=3051 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1718692554.285:884): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=SERVICE_START msg=audit(1718699668.363:460): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=SERVICE_START msg=audit(1718700377.881:468): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=SERVICE_STOP msg=audit(1718700377.881:469): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=AVC msg=audit(1718700377.898:470): avc:  denied  { name_bind } for  pid=2128 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1718700377.898:470): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=55d6002c48b8 a2=10 a3=7ffc01647ba0 items=0 ppid=1 pid=2128 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1718700377.898:471): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=SERVICE_START msg=audit(1718702489.409:482): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=SERVICE_STOP msg=audit(1718702840.091:486): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=AVC msg=audit(1718702840.115:487): avc:  denied  { name_bind } for  pid=2194 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1718702840.115:487): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=5633f68de8b8 a2=10 a3=7fff00608d50 items=0 ppid=1 pid=2194 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1718702840.115:488): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
```

Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту: 

```
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp

[root@selinux ~]# semodule -i nginx.pp
```

Попробуем снова запустить nginx:

```
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2024-06-18 09:34:44 UTC; 6s ago
  Process: 2225 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2223 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 2222 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2227 (nginx)
   CGroup: /system.slice/nginx.service
           ├─2227 nginx: master process /usr/sbin/nginx
           └─2229 nginx: worker process

Jun 18 09:34:44 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jun 18 09:34:44 selinux nginx[2223]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 18 09:34:44 selinux nginx[2223]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jun 18 09:34:44 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

Задание выполнено.























































































































































































