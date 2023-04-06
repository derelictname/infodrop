# ipsec

С помощью специальной программы на 500 UDP порту два участника обмениваюся настройками и далее все IP пакеты между ними становится подписанным и, при желании, шифрованным.

В начале формируется общий ключ сессии по алгоритму Диффи-Хеллмана, потом стороны обмениваются открытыми ключами для расшифровки пакетов. Защищённый таким образом канал передачи данных называется Security Association и каждый из них имеет свои настройки. Информация о всех каналах хранится общей базе данных модуля в ядре операционной системы, который производит необходимые манипуляции над входящими и исходящими пакетами.

Режимы работы
- транспортный - шифрует и подписывает только данные IP пакета (например, для туннелей поверх защищённого канала)
- туннельный - шифрует весь IP пакет и вставляет в поле данных нового пакета (получается VPN)

В работе также участвуют протоколы

- AH (Authentication Header) - целостность данных IP пакета, подтверждение источника
- ESP (Encapsulating Security Payload) - шифрование полезной нагрузки

## Установка

Установка пакета

    apt install strongswan || apt install libreswan

Пример настройки подключения IKEv1

```
# /etc/ipsec.d/conn1.conf

conn conn1
    auto=start
    type=transport
    authby=secret
    left=%defaultroute
    right=11.22.33.44
    rightid=%any
    rightprotoport=udp/l2tp
    keyingtries=%forever
    ike=aes256-sha2_256-modp2048,aes256-sha2_256-modp1536,aes256-sha2_256-modp1024,aes256-sha1-modp2048,aes256-sha1-modp1536,aes256-sha1-modp1024,aes256-sha1-ecp384,aes128-sha1-modp1024,aes128-sha1-ecp256,3des-sha1-modp2048,3des-sha1-modp1024!
    esp=aes256-sha1,aes128-sha1,3des-sha1!
    keyexchange=ikev1
```

Pre-Shared Key

```
# /etc/ipsec.secrets

: PSK "test1"
```

Запуск службы

    systemctl restart strongswan-starter

Проверка настроек

    ipsec status
    ip xfrm

## l2tp

Протокол сеансового уровня для создания VPN с помощью туннелей PPP канального уровня (другой пример канального уровня - Ethernet). Так как канал передачи данных не защищён, то необходимо сначала настроить IPsec.

В cloud ядрах облачных образов отключена поддержка PPP протокола. Необходимо устанавливать стандартное ядро.

Установка

    apt install xl2tpd

Настройки l2tp

```
# /etc/xl2tpd/xl2tpd.conf

[lac vpn1]
lns = 11.22.33.44
autodial = yes
pppoptfile = /etc/ppp/options.vpn1
```

Настройки PPP

```
# /etc/ppp/options.vpn1

noauth
name superman
password PaSsWoRd1!
```

Запуск

    systemctl restart xl2tpd

network-manager-l2tp-gnome - упрощённый вариант настройки L2TP + IPsec





