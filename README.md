**SELinux**

**1. Описание задания**  

1. Запустить nginx на нестандартном порту 3-мя разными способами:
переключатели setsebool;
добавление нестандартного порта в имеющийся тип;
формирование и установка модуля SELinux.
К сдаче:
README с описанием каждого решения (скриншоты и демонстрация приветствуются). 

2. Обеспечить работоспособность приложения при включенном selinux.
развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems; 
выяснить причину неработоспособности механизма обновления зоны (см. README);
предложить решение (или решения) для данной проблемы;
выбрать одно из решений для реализации, предварительно обосновав выбор;
реализовать выбранное решение и продемонстрировать его работоспособность


 **2. Выполенение задания.**

**Запуск nginx на нестандартном порту 3-мя разными способами:**
 
Первый способ.  

1. Создад директория для проекта, инитилизировал вагрантфайл, отредактировал его как в примере из методички, запусти виртуальную машину.
2. При запуске виртуальной машины убедился, что nginx не запустился.
3. Подключился по ssh к виртуальной машине. Перешел в root, проверил, что файрвол отключен, проверил конфиг nginx, проверил режим работы SELinux.
Нашел информацию в лог файле о блокировке порта.

`[root@selinux vagrant]# grep '4881' /var/log/audit/audit.log
 type=AVC msg=audit(1687180205.055:1020): avc:  denied  { name_bind } for  pid=22243 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 class=tcp_socket permissive=0`  

 При помощи утилиты audit2why выяснил причину, почему порт блокируется.  
 
 '[root@selinux vagrant]# grep 1687180205.055:1020 /var/log/audit/audit.log | audit2why
  type=AVC msg=audit(1687180205.055:1020): avc:  denied  { name_bind } for  pid=22243 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0      tclass=tcp_socket permissive=0
  
	  Was caused by:
	  The boolean nis_enabled was set incorrectly. 
	  Description:
	  Allow nis to enabled
   
	  Allow access by executing:
	  # setsebool -P nis_enabled 1'
   
   Исходя из полученной информации сделал вывод, что нужно поменять параметр nis_enabled.  
  
 4. Разрешил работу nginx на порту TCP 4881 с помощью переключателя setsebool.

    '[root@selinux vagrant]# setsebool -P nis_enabled 1'
    
    '[root@selinux vagrant]# systemctl restart nginx'
    
'[root@selinux vagrant]# systemctl status nginx  
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Ср 2023-06-21 07:22:30 UTC; 10s ago
  Process: 9746 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 9742 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 9741 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 9748 (nginx)
   CGroup: /system.slice/nginx.service
           ├─9748 nginx: master process /usr/sbin/nginx...
           └─9750 nginx: worker process'

6. Проверил статус параметра`getsebool -a | grep nis_enabled`
   `nis_enabled --> on`

7. Вернул запрет работы nginx на порту 4881 `setsebool -P nis_enabled off`

Второй способ.  

1. Нашел тип для http трафика
`[root@selinux vagrant]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
`
2. Добавил порт в тип http_port_t: `semanage port -a -t http_port_t -p tcp 4881`
3. Перезапустил службу nginx и проверим работу: `systemctl restart nginx`
`systemctl status nginx`
5. Удалил нестандартный порт из имеющегося типа с помощью команды: `semanage port -d -t http_port_t -p tcp 4881`

Третий способ.

1. Попробовал запустить nginx: `systemctl start nginx`
2. Воспользовался утилитой audit2allow
`[root@selinux vagrant]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp`

3. `semodule -i nginx.pp`
4. Запустил nginx `systemctl start nginx`
5. Посмотрел списко всех модулей `semodule -l`
6. Удалил модуль комадой: `semodule -r nginx`

**Обеспечение работоспособности приложения при включенном SELinux**

1. Установил ansible по иструкции https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html на хост машину.
2. Установил git на хост машину. Выполнил клонирование репозитория. `git clone https://github.com/mbfx/otus-linux-adm.git`
`[root@selinux ansible]# git clone https://github.com/mbfx/otus-linux-adm.git
Cloning into 'otus-linux-adm'...
remote: Enumerating objects: 558, done.
remote: Counting objects: 100% (456/456), done.
remote: Compressing objects: 100% (303/303), done.
remote: Total 558 (delta 125), reused 396 (delta 74), pack-reused 102
Receiving objects: 100% (558/558), 1.38 MiB | 1.76 MiB/s, done.
Resolving deltas: 100% (140/140), done.`
3. Перешел в директорию `cd otus-linux-adm/selinux_dns_problems`
4. Запустил виртуальные машины: `vagrant up`
5. 
