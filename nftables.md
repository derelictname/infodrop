# nftables

Используя выразительный структурированный синтаксис позволяет обрабатывать трафик Ethernet, IP, TCP, соединения, определять наборы значений, учёт трафика, балансировку, ограничения.

Используется по умолчанию в Debian 10 и выше.

Структура

- таблицы (ip, ip6, inet, arp, bridge)
    - набор (ipv4_addr)
    - счётчики
    - ...
    - цепочки правил (filter, route, nat)
        - правила (statement, состоящий из критерия и действия)

<https://wiki.nftables.org/wiki-nftables/index.php/Main_Page>

Nftables менее удобен в редактировании через команды, чем iptables, но более удобен через файл. Уже присутствует сервис `nftables.service`, который загружает правила из `/etc/nftables.conf`.

При желании можно воспроизвести такую же структуру таблиц, цепочек и правил, как в iptables

```bash
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
    
    }
    chain forward {
    
    }
    chain output {
    
    }
}
```

Наполняем её правилами

```bash
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    set admins {
        typeipv4_addr
        elements = {
            22.33.44.55,
            33.44.55.66
        }
    }

    counter rsyslog {}

    # дополнительные цепочки должны идти до их применения
    chain rsyslog {
        udp dport 514
        tcp dport 2514 comment "relp" 
    }

    chain input {
        type filter hook input priority 0; policy drop;
        ct state invalid counter drop comment "Invalid packets"

        # { established, related } - пример анонимного набора
        ct state { established, related } accept comment "Traffic originated from us"
        iif "lo" accept
        iif != "lo" ip daddr 127.0.0.0/8 counter drop comment "Fake loopback"
        ip protocol icmp counter accept comment "Ping"

        # перемещение в дополнительную цепочку
        ip saddr 10.0.2.0/24 jump web

        # отдельный счётчик и перемещение в дополнительную цепочку
        udp dport 514 iifname vlan177 counter name rsyslog jump rsyslog

        # доступ к SSH с определённых IP
        tcp dport ssh ip saddr @admins accept
        
        # либо только защита от взломов
        tcp dport ssh ct state new limit rate 15/minute accept comment "Brute force"

    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }

    chain output {
        type filter hook output priority 0;
    }
}
```

[Справка по синтаксису](https://wiki.nftables.org/wiki-nftables/index.php/Quick_reference-nftables_in_10_minutes)

## Интерфейс командной строки

Чтобы добавлять какие-то правила нам нужна таблица

    nft add table inet my_table_1

В таблице создаём цепочку правил с обработкой пакетов

    nft add chain inet my_table_1 my_filters_1 '{ type filter hook input priority 0; }'

Добавляем правило в конец цепочки

    nft add rule inet my_table_1 my_filters_1 tcp dport 1234 drop

Просмотр всех таблиц, цепочек и правил

    nft list ruleset

Удаление всего

    nft flush ruleset
