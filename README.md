# OTUS ДЗ 17 SELinux

## Домашнее задание

Практика с SELinux
Цель: Тренируем умение работать с SELinux: диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется.
1. Запустить nginx на нестандартном порту 3-мя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.
К сдаче:
- README с описанием каждого решения (скриншоты и демонстрация приветствуются).

2. Обеспечить работоспособность приложения при включенном selinux.
- Развернуть приложенный стенд
https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems
- Выяснить причину неработоспособности механизма обновления зоны (см. README);
- Предложить решение (или решения) для данной проблемы;
- Выбрать одно из решений для реализации, предварительно обосновав выбор;
- Реализовать выбранное решение и продемонстрировать его работоспособность.
К сдаче:
- README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них;
- Исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.



Прежде чем приступить к выполнению ДЗ установим semanage с помощью команды ```yum install policycoreutils-python -y```. Далее включим анализ логов с помощью команды -```audit2why < /var/log/audit/audit.log```.<br>Так же установим nginx, поскольку в стандартных репо его нет выполним команду ```yum install epel-release```, а затем ```yum install nginx```.<br> Дополнительно нам потребуется утилита из набора инструментов ```yum install net-tools``` и мой любимый текстовый редактор ```yum install nano```



## 1. Запустить nginx на нестандартный порту 3-мя разными способами.

### 1.1 Переключатели setsebool

Пакет nginx уже установлен с Vagrantfile-a предназначенного для данного ДЗ, то сразу зайдем в ``/etc/nginx/nginx.conf`` и поменяем стандартный 80-й порт на 11988.

    systemctl restart nginx

Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.

    journalctl -u nginx -n 20

-- Logs begin at Ср 2020-09-02 08:07:08 UTC, end at Ср 2020-09-02 08:23:26 UTC.
сен 02 08:21:35 otus systemd[1]: Unit nginx.service cannot be reloaded because i
сен 02 08:23:25 otus systemd[1]: Starting The nginx HTTP and reverse proxy serve
сен 02 08:23:26 otus nginx[17271]: nginx: the configuration file /etc/nginx/ngin
сен 02 08:23:26 otus nginx[17271]: nginx: [emerg] bind() to 0.0.0.0:11988 failed
сен 02 08:23:26 otus nginx[17271]: nginx: configuration file /etc/nginx/nginx.co
сен 02 08:23:26 otus systemd[1]: nginx.service: control process exited, code=exi
сен 02 08:23:26 otus systemd[1]: Failed to start The nginx HTTP and reverse prox
сен 02 08:23:26 otus systemd[1]: Unit nginx.service entered failed state.
сен 02 08:23:26 otus systemd[1]: nginx.service failed.


Как описано по выводам команд сверху, selinux не позваляет nginx помен стандартный порт службы на 11988, для новых причин опять же воспользуемся audit2why, команда - ``audit2why < /var/log/audit/audit.log``.

Утилита audit2why рекомендует выполнить команду - ``setsebool -P nis_enabled 1``. После выполнения данного логического значения по рекомендациям audit2whynginx успешно запускается.

type=AVC msg=audit(1599035006.004:1208): avc:  denied  { name_bind } for  pid=17271 comm="nginx" src=11988 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket

	Was caused by:
	The boolean nis_enabled was set incorrectly.
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1


    [root@otus vagrant]# setsebool -P nis_enabled 1



    [root@otus vagrant]# systemctl restart nginx



    [root@otus vagrant]# systemctl status nginx


    nginx.service - The nginx HTTP and reverse proxy server
       Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
       Active: active (running) since Ср 2020-09-02 08:25:46 UTC; 8s ago
      Process: 17297 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
      Process: 17294 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
      Process: 17293 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
     Main PID: 17299 (nginx)
       CGroup: /system.slice/nginx.service
               ├─17299 nginx: master process /usr/sbin/nginx
               └─17300 nginx: worker process

    сен 02 08:25:46 otus systemd[1]: Starting The nginx HTTP and reverse proxy server...
    сен 02 08:25:46 otus nginx[17294]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    сен 02 08:25:46 otus nginx[17294]: nginx: configuration file /etc/nginx/nginx.conf test is successful
    сен 02 08:25:46 otus systemd[1]: Failed to parse PID from file /run/nginx.pid: Invalid argument
    сен 02 08:25:46 otus systemd[1]: Started The nginx HTTP and reverse proxy server.



    [root@otus vagrant]# netstat -tunap



    Active Internet connections (servers and established)
    Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
    tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      6308/rpcbind        
    tcp        0      0 0.0.0.0:11988           0.0.0.0:*               LISTEN      17299/nginx: master

### 1.2 Добавление нестандартного порта в имеющийся тип

  Для того чтобы пролистать все порты входящие в тип - http_port_t, достаточно выполнить команду - ``semanage port -l | grep http``.



Для добавления нестандартного порта внужный нам тип же воспользуемся утилитой semanage.

    [root@otus vagrant]# semanage port -a -t http_port_t -p tcp 11988
    [root@otus vagrant]# semanage port -l | grep http
    http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
    http_cache_port_t              udp      3130
    http_port_t                    tcp      11988, 80, 81, 443, 488, 8008, 8009, 8443, 9000
    pegasus_http_port_t            tcp      5988
    pegasus_https_port_t           tcp      5989



### 1.3 Формирование и установка модуля SELinux.

Опять же поменяем порт 80 на 11988 в /etc/nginx/nginx.conf.
Далее скомпилируем модуль на основе логики /var/log/audit/audit.log, в котором содержится информация о запретах и ​​установим созданный модуль.

    [root@otus vagrant]# audit2allow -M httpd_new --debug < /var/log/audit/audit.log
    ******************** IMPORTANT ***********************
    To make this policy package active, execute:

    semodule -i httpd_new.pp

    [root@otus vagrant]# ll
    итого 8
    -rw-r--r--. 1 root root 964 сен  2 08:31 httpd_new.pp
    -rw-r--r--. 1 root root 247 сен  2 08:31 httpd_new.te

Проверим работу Nginx по настроенному порту

    [root@otus vagrant]# netstat -tunap
    Active Internet connections (servers and established)
    Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
    tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      6308/rpcbind        
    tcp        0      0 0.0.0.0:11988           0.0.0.0:*               LISTEN      17299/nginx: master


## 2. Обеспечить работоспособность приложения при включенном selinux.
Скачиваем стенд из репозитория https://github.com/mbfx/otus-linux-adm , и поднимаем стенд с помощью команды ``vagrant up`` из под папок selinux_dns_problems /
После подключения клиентской машине попробуем выполнить к след. команду, предназначенную для новой записи, зоны и добавить в нее записи, после чего получим ошибку

    [root@client vagrant]# nsupdate -k /etc/named.zonetransfer.key


    > server 192.168.50.10
    > zone ddns.lab
    > update add www.ddns.lab. 60 A 192.168.50.15
    > send
    update failed: SERVFAIL
    >


Выполним команду ``audit2why < /var/log/audit/audit.log`` для того, чтобы выяснить, какая именно причина нам не обновить нашу зону и какой модль ему для этого необходим.

    [root@ns01 vagrant]# audit2why < /var/log/audit/audit.log


    type=AVC msg=audit(1598303918.278:786): avc:  denied  { write } for  pid=2953 comm="isc-worker0000" path="/etc/named/dynamic/named.ddns.lab.view1.jnl" dev="sda1" ino=12185 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

    	Was caused by:
    		Missing type enforcement (TE) allow rule.

    		You can use audit2allow to generate a loadable module to allow this access.





    [root@ns01 vagrant]# audit2allow -a




    #============= named_t ==============

    #!!!! WARNING: 'etc_t' is a base type.
    allow named_t etc_t:file create;


Как видим нужно дать необходимое разрешение для named_t
Для этого воспользуемся утилитой audit2allow и выполним след команды, после сразу перезапустим нашу службу dns


    [root@ns01 vagrant]# audit2allow -a -M named_t




    ******************** IMPORTANT ***********************
    To make this policy package active, execute:

    semodule -i named_t.pp



    [root@ns01 vagrant]# semodule -i named_t.pp


    [root@ns01 vagrant]# systemctl restart named


Вернемся на клиентскую ВМ и заново попробовал хостовую запись на наш dns-сервер


    [vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key



    > server 192.168.50.10
    > zone ddns.lab
    > update add www.ddns.lab. 60 A 192.168.50.15
    > send
    >

Как видим, никаких проблем на этом не возникло, а это значит, что внесенные изменения дали нужный результат, чтобы окончательно в этом проверим состояние службы, где увидим, что наша запись добавлена.


    [root@ns01 vagrant]# systemctl status named
    ● named.service - Berkeley Internet Name Domain (DNS)
       Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
       Active: active (running) since Mon 2020-09-02 12:02:42 UTC; 1min 40s ago
      Process: 3004 ExecStop=/bin/sh -c /usr/sbin/rndc stop > /dev/null 2>&1 || /bin/kill -TERM $MAINPID (code=exited, status=0/SUCCESS)
      Process: 3017 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
      Process: 3015 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
     Main PID: 3019 (named)
       CGroup: /system.slice/named.service
               └─3019 /usr/sbin/named -u named -c /etc/named.conf

    сен 02 12:02:42 ns01 named[3019]: network unreachable resolving './DNSKEY/IN': 2001:500:a8::e#53
    сен 02 12:02:42 ns01 named[3019]: network unreachable resolving './NS/IN': 2001:500:a8::e#53
    сен 02 12:02:42 ns01 named[3019]: network unreachable resolving './DNSKEY/IN': 2001:503:c27::2:30#53
    сен 02 12:02:42 ns01 named[3019]: network unreachable resolving './NS/IN': 2001:503:c27::2:30#53
    сен 02 12:02:42 ns01 named[3019]: managed-keys-zone/view1: Unable to fetch DNSKEY set '.': timed out
    сен 02 12:02:42 ns01 named[3019]: resolver priming query complete
    сен 02 12:02:42 ns01 named[3019]: managed-keys-zone/default: Unable to fetch DNSKEY set '.': timed out
    сен 02 12:02:42 ns01 named[3019]: resolver priming query complete
    сен 02 12:04:22 ns01 named[3019]: client @0x7f0b1803c3e0 192.168.50.15#48036/key zonetransfer.key: view view1: signer "zonetransfer.key" approved
    сен 02 12:04:22 ns01 named[3019]: client @0x7f0b1803c3e0 192.168.50.15#48036/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': adding an RR at 'www.ddns.lab' A 192.168.50.15


Более оптимальным и менее опасным решением проблемы будет анализ лога через утилиту audit2why - ``audit2why < /var/log/audit/audit.log``

    type=AVC msg=audit(1598624073.018:951): avc:  denied  { rename } for  pid=3469 comm="isc-worker0000" name="tmp-bLASqNVp6V" dev="sda1" ino=486352 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

    	Was caused by:
    		Unknown - would be allowed by active policy
    		Possible mismatch between this policy and the one under which the audit message was generated.

    		Possible mismatch between current in-memory boolean settings vs. permanent ones.


Как в выводе, для решения проблемы воспользуемся утилитой audit2allow, который добавит разрешение на содержании файла - /var/log/audit/audit.log
Выполним команды ``audit2allow -M named-selinux --debug < /var/log/audit/audit.log``и ``semodule -i named-selinux.pp``. После чего наша зона будет обновляться и новые записи будут добавлены в нужный файл.


Можно коротко описать еще три варианта решения проблемы:

1й способ
    Отключение selinux (худший вариант);
    Выполним команду ``cat /var/log/messages | grep ausearch`` для показа остальных двух способов

    сен 02 12:10:33 ns01 python: SELinux is preventing isc-worker0000 from unlink access on the file named.ddns.lab.view1.#012#012*****  Plugin catchall_labels (83.8 confidence) suggests   *******************#012#012If you want to allow isc-worker0000 to have unlink access on the named.ddns.lab.view1 file#012Then you need to change the label on named.ddns.lab.view1#012Do#012# semanage fcontext -a -t FILE_TYPE 'named.ddns.lab.view1'#012where FILE_TYPE is one of the following: dnssec_trigger_var_run_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t.#012Then execute:#012restorecon -v 'named.ddns.lab.view1'#012#012#012*****  Plugin catchall (17.1 confidence) suggests   **************************#012#012If you believe that isc-worker0000 should be allowed unlink access on the named.ddns.lab.view1 file by default.#012Then you should report this as a bug.#012You can generate a local policy module to allow this access.#012Do#012allow this access for now by executing:#012# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000#012# semodule -i my-iscworker0000.pp#012

2й способ
    Команда ``ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000 | semodule -i my-iscworker0000.pp``;

3й способ
    Присвоение файлу named.ddns.lab.view1один из вышеперечисленных (в выводе из лог-файла) типов dnssec_trigger_var_run_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t, с помощью утилиты semanage. Команда - semanage fcontext -a -t FILE_TYPE 'named.ddns.lab.view1'.
