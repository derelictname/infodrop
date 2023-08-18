# syslog

Попытка стандартизировать централизованный сбор логов по сети.

Сообщения имеют категорию severity (уровень важности) и facility.

## rsyslog

Маршрутизатор, фильтр и обработчик сообщений логов. Основной источник сообщений - сокет, а назначение - файлы. Но также можно получать сообщения и из сети по разным протоколам (в том числе в базы данных и т.д.) или получать и сохранять в файлы.

https://www.rsyslog.com/newbie-guide-to-rsyslog/

В Debian 12 убран rsyslog в пользу systemd-journald. The potential for corruption of the binary format has led to much heated debate.

Отправка сообщений

```
# Задаём имя отправителя для удобства
global(localHostname="example.com")

# Отдельный набор правил обработки с отдельной очередью
ruleset(name="send") {
        action(
                type="omfwd"
                target="11.22.33.44"
                port="514"
                protocol="tcp"
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
# Включение TCP и назначение набора правил
module(load="imtcp")
input(type="imtcp" port="514" ruleset="save")

# Шаблон имени файла
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

# Шаблон сообщения
template(name="message" type="list") {
        property(name="msg" position.from="2")
        constant(value="\n")
}

# Набор правил обработки
ruleset(name="save") {
        action(type="omfile" dynaFile="remote" template="message")
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

# Прекращение обработки, чтобы не сообщение не ушло в локальные логи
& ~
```

## Хранение и поиск в manticoresearch

Docker Compose

```yaml
services:
  manticore:
    image: manticoresearch/manticore
    restart: always
    ports:
      - 9306:9306
    volumes:
      - manticore:/var/lib/manticore

  grafana:
    image: grafana/grafana-oss
    restart: always
    ports:
      - 3000:3000
    volumes:
      - grafana:/var/lib/grafana


volumes:
  manticore:
  grafana:
```

Подключаемся к manticoresearch

    mysql -P 9306

Создаёт таблицу в manticoresearch

```sql
CREATE TABLE systemevents
(
    id int,
    customerid bigint,
    receivedat timestamp,
    devicereportedtime timestamp,
    facility int,
    priority int,
    fromhost string,
    message text,
    ntseverity int,
    importance int,
    eventsource string,
    eventuser string,
    eventcategory int,
    eventid int,
    eventbinarydata text,
    maxavailable int,
    currusage int,
    minusage int,
    maxusage int,
    infounitid int ,
    syslogtag string,
    eventlogtype string,
    genericfilename string,
    systemid int
);
```

Шаблон SQL сообщения в rsyslog для отправки в manticoresearch

```
template(name="sql" type="list" option.sql="on") {
    constant(value="INSERT INTO systemevents (message, facility, fromhost, priority, devicereportedtime, receivedat, infounitid, syslogtag)")
    constant(value=" VALUES ('")
    property(name="msg")
    constant(value="', ")
    property(name="syslogfacility")
    constant(value=", '")
    property(name="hostname")
    constant(value="', ")
    property(name="syslogpriority")
    constant(value=", '")
    property(name="timereported" dateFormat="unixtimestamp")
    constant(value="', '")
    property(name="timegenerated" dateFormat="unixtimestamp")
    constant(value="', ")
    property(name="iut")
    constant(value=", '")
    property(name="syslogtag")
    constant(value="')")
}
```

Перенаправление сообщений в manticoresearch

```
module(load="imtcp")
input(type="imtcp" port="514" ruleset="save")

module (load="ommysql")
ruleset(name="save") {
    action(type="ommysql" server="127.0.0.1" db="Manticore" serverport="9306" uid="root" pwd="" template="sql")
}
```

Подключаем в Grafana источник MySQL и создаём панель (Timeseries и/или Logs) с SQL запросом

```sql
SELECT
  $__unixEpochGroup(devicereportedtime, $__interval) AS time,
  CONCAT(fromhost, ' ', syslogtag) AS metric,
  message,
  priority
FROM
  Manticore.systemevents
WHERE
  $__unixEpochFilter(time)
  AND MATCH('$search')
GROUP BY
  time WITHIN GROUP
ORDER BY
  priority ASC
ORDER BY
  time DESC
LIMIT
  1000
```

В дашборде добавляем переменную search для поиска по логам.

Ссылки

- <https://manticoresearch.com/blog/manticoresearch-grafana-integration/>
- <https://grafana.com/docs/grafana/latest/datasources/mysql/>

