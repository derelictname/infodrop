# firewalld

Надстройка над iptables, ebtables, ipset, NetworkManager, nft. Вводит новую абстракцию - зону. Их может быть несколько и они являются по факту отдельной цепочкой правил. Чтобы направить трафик в определённую зону, ей назначается IP/порт источника или сетевой интерфейс (также в контексте NetworkManager это может быть соединение). У каждой зоны есть действие по умолчанию и разрешённые порты, сервисы (названия для портов) и протоколы.

Стандартные действия

- drop - никак не отвечать на пакеты
- block - отвечать icmp-host-prohibited
- public, external, dmz, work, home, internal - разрешить только определённые сервисы/порты
- trusted - пропускать все сервисы/порты

Назначить доверенной зоне интерфейс внутренней сети

    firewall-cmd --zone=trusted --change-interface=em1

Также привязка к зоне указывается в настройках интерфейса /etc/sysconfig/network-scripts/ifcfg-em1

    ZONE=trusted

Назначить источник зоне (источники не могут повторяться в разных зонах)

    firewall-cmd --zone=trusted --add-source=11.22.33.44

Отобразить привязки к зонам

    firewall-cmd --get-active-zones

Если привязки к зоне нет, то по умолчанию используется зона public.

Разрешить порт для зоны

    firewall-cmd --zone=public --add-port=123456/tcp

Отобразить параметры зоны

    firewall-cmd --info-zone=public

Отобразить параметры всех зон

    firewall-cmd --list-all-zones

## Примеры

Добавим специальную зону для определённых источников

    firewall-cmd --new-zone=zabbix --permanent
    firewall-cmd --reload

Добавляем источники

    firewall-cmd --zone=zabbix --add-source=11.22.33.44

И разрешённые для них порты или сервисы

    firewall-cmd --zone=zabbix --add-service=zabbix-agent

Проверяем

    firewall-cmd --info-zone=zabbix

Сохраняем

    firewall-cmd --runtime-to-permanent




