# syslog

Попытка стандартизировать централизованный сбор логов по сети.

Сообщения имеют категорию severity (уровень важности) и facility.

## rsyslog

Маршрутизатор, фильтр и обработчик сообщений логов. Основной источник сообщений - сокет, а назначение - файлы. Но также можно получать сообщения и из сети по разным протоколам (в том числе в базы данных и т.д.) или получать и сохранять в файлы.

https://www.rsyslog.com/newbie-guide-to-rsyslog/

В Debian 12 убран rsyslog в пользу systemd-journald. The potential for corruption of the binary format has led to much heated debate.

Отправка сообщений

```
# Отдельный набор правил обработки с отдельной очередью
ruleset(name="send") {
        action(
                type="omfwd"
                target="11.22.33.44"
                port="514"
                protocol="udp"
                queue.filename="forwarding"
                queue.size="1000000"
                queue.type="LinkedList"
        )
}

# Определяем фильтр и вызываем набор правил
if ( $programname contains "myprogram" ) then call send
```

*Если отправлять сообщения на удалённый сервер по TCP, а удалённый сервер станет недоступен, то основная очередь обработки сообщений остановится, в том числе большинство логов в /var/log/. Поэтому необходимо в таких случаях делать отдельную очередь.*

Получение сообщений

```
# Определяем формат имени файла
template(name="remote" type="list") {
        constant(value="/var/log/rsyslog/")
        property(name="hostname")
        constant(value="/")
        property(name="timegenerated" dateFormat="year")
        constant(value="-")
        property(name="timegenerated" dateFormat="month")
        constant(value="-")
        property(name="timegenerated" dateFormat="day")
        constant(value="/")
        property(name="programname")
        constant(value=".log")
}

# Определяем шаблон сообщения, оставляя только само сообщение
template(name="tpl1" type="list") {
        property(name="msg")
        constant(value="\n")
}

# Определяем фильтр сообщений по источнику и применяем действия
if ( $inputname == "imudp" ) then {
        action(type="omfile" dynaFile="remote" template="tpl1")
        stop
}
```

Тестовая отправка в syslog

    logger test1

- [Шаблоны](https://www.rsyslog.com/doc/v8-stable/configuration/templates.html)
- [Фильтры](https://www.rsyslog.com/doc/v8-stable/configuration/filters.html)
- [Очереди](https://www.rsyslog.com/doc/v8-stable/concepts/queues.html)
- [Шифрование](https://www.rsyslog.com/doc/master/tutorials/tls.html)

## Старый синтаксис настроек

Отправка логов

```
# Отправка на удалённый сервер по UDP
*.* @11.22.33.44:514

# Отправка на удалённый сервер по TCP
*.* @@11.22.33.44:514

# Отправка определённого facility и level
local0.error @11.22.33.44:514
```

Получение логов

```
# Шаблон имени файла с сообщениями
$template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%programname%.log"

# Фильтр сообщений, полученных по UDP, и назначение шаблона
:inputname, isequal, "imudp" ?RemoteLogs

# Прекращение обработки, чтобы не сообщение не ушло в обычные логи
& ~
```


