# tc

Инструмент настройки подсистемы ядра Linux, контролирующей сетевой трафик.

Сетевым интерфейсам присваивается правило очереди (qdisc - queuing discipline), имеющая двухбайтовое шестнадцатеричное наименование.

Добавление очереди с применением планировщика Token Bucket Filter

    tc qdisc add dev eth0 root tbf rate 100mbit latency 50ms burst 1540

- qdisc - изменяем планировщик
- add - добавить новое правило
- root - планировщик исходящего
- tbf rate 100mbit - степень замедления
- latency 50ms - задержка замедления
- burst 1540 - объём корзины

Отобразить текущие очереди

    tc qdisc show
    tc qdisc show dev eth0

Отобразить счётчики очередей

    tc -s qdisc show
    tc -s qdisc show dev eth0

- sent - отправлено вне очереди
- dropped - отбросы пакетов из-за переполнения всех очередей
- overlimits - the number of times the configured link capacity is filled

Каждая очередь имеет планировщик - механизм контроля трафика. Например, алгоритм mq разбивает трафик  на классы для каждого потока при многопоточной системе.

    man tc-cbq
    man tc-sfq
    man tc-fs_codel

Системный планировщик по умолчанию

    sysctl -a | grep qdisc

Порядок обработки пакетов основной очереди может разбиваться на классы. В таком случае к основной очереди добавляется номер

    tc qdisc add dev eth0 root handle 1: cbq avpkt 1000 bandwidth 1000mbit

- `handle 1:` - номер очереди

Класс привязывается по индексу

    tc class add dev eth0 parent 1: classid 1:1 cbq rate 100mbit allot 1500 prio 5 bounded isolated

- `parent 1:` - родительская очередь
- `classid 1:1` - номер класса

Распределение в очереди может явно определяться через фильтры

    tc filter add dev eth0 parent 1: protocol ip prio 16 u32 match ip src 11.22.33.44 flowid 1:1

- `flowid 1:1` - очередь назначения

Удаление очереди

    tc qdisc del dev eth0 root






https://www.techrepublic.com/article/how-to-limit-bandwidth-on-linux-to-better-test-your-applications/

https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/linux-traffic-control_configuring-and-managing-networking

https://joshrosso.com/c/tc/
