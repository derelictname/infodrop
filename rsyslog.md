# rsyslog

Маршрутизатор, фильтр и обработчик сообщений по стандарту syslog. Основной источник сообщений - сокет (`nc -u -U /dev/log`), а назначение - файлы, но также можно отправлять и получать сообщения по сети, по разным протоколам (в том числе в базы данных, RESTful API и т.д.).

https://www.rsyslog.com/newbie-guide-to-rsyslog/

В Debian 12 убран rsyslog в пользу systemd-journald. The potential for corruption of the binary format has led to much heated debate.

---

https://rainer.gerhards.net/2008/04/on-unreliability-of-plain-tcp-syslog.html

Если при отправке по TCP удалённый получатель внезапно станет недоступен (падение, обрыв связи), то ответный TCP ACK прийдёт только после восстановления доступа. До восстановления доступа TCP send() будет сохранять сообщения в буфер, в который может поместиться до 1600 сообщений. При этом отправитель не знает о том, были ли получены сообщения и когда. Поэтому TCP подходит при коротких проблемах.

Отправка по протоколу RELP также имеет недостаток, но уже в самом Rsyslog. Потеря сообщений возникает, если были отправлены сообщения, оборвалась связь до получения подтверждения о их приходе и в этот момент остановлен Rsyslog отправителя.

---

Отправка сообщений

```
# Задаём имя отправителя для удобства
global(localHostname="example.com")

# Отдельный набор правил обработки с отдельной очередью
ruleset(name="send") {
    action(
        type="omfwd"
        protocol="tcp"
        target="11.22.33.44"
        port="514"
        queue.filename="forwarding"
        queue.size="1000000"
        queue.type="linkedlist"
    )
}

# Определяем фильтр и вызываем набор правил
if $programname contains "myprogram" then call send
```

- [Фильтры](https://www.rsyslog.com/doc/master/configuration/filters.html)
- [Очереди](https://www.rsyslog.com/doc/master/rainerscript/queue_parameters.html)

*При отправке сообщений по протоколу TCP единстванная очередь обработки сообщений заблокируется, если удалённый сервер станет недоступен. Поэтому необходимо настраивать отдельную очередь обработки. В приведённых примерах это учтено.*

Получение сообщений

```
# Включение приёма по TCP и назначение набора правил
module(load="imtcp")
input(type="imtcp" port="514" ruleset="save")

# Модуль удаления пробела в начале сообщения
module(load="mmrm1stspace")

# Настройки сохранения сообщений в файлы
module(
    load="builtin:omfile"
    fileowner="root"
    filegroup="adm"
    dirowner="root"
    dirgroup="root"
    filecreatemode="0640"
    dircreatemode="0755"
)

# Шаблон имени файла
template(name="filename" type="list") {
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

# Набор правил обработки
ruleset(name="save") {
    action(type="mmrm1stspace")
    action(type="omfile" dynafile="filename")
}
```

- [Шаблоны](https://www.rsyslog.com/doc/master/configuration/templates.html)
- [Параметры шаблонов](https://www.rsyslog.com/doc/master/configuration/properties.html)

## TLS

    apt install rsyslog-gnutls

https://www.rsyslog.com/doc/master/tutorials/tls.html

## Elasticsearch и Kibana

Добавляем поддержку Elasticsearch

    apt install rsyslog-elasticsearch

Поднимаем сервисы через Docker Compose

```yaml
services:

  elasticsearch:
    image: elasticsearch:8.8.1
    ports:
      - 127.0.0.1:9200:9200
    volumes:
      - elastic:/usr/share/elasticsearch/data
    environment:
      discovery.type: single-node
      xpack.security.enabled: false

  kibana:
    image: kibana:8.8.1
    ports:
      - 5601:5601
    volumes:
      - kibana:/usr/share/kibana/data

volumes:
  elastic:
  kibana:
```

Создаём шаблон индекса в Elasticsearch для хранения сообщений

```bash
curl -X PUT "localhost:9200/_index_template/rsyslog" -H 'Content-Type: application/json' -d'
{
  "template": {
    "mappings": {
      "dynamic_templates": [],
      "properties": {
        "severity": {
          "type": "keyword"
        },
        "syslogtag": {
          "type": "keyword"
        },
        "host": {
          "type": "keyword"
        },
        "message": {
          "type": "text"
        },
        "facility": {
          "type": "keyword"
        }
      }
    }
  },
  "index_patterns": [
    "rsyslog"
  ],
  "data_stream": {}
}
'
```

Настройки Rsyslog для отправки сообщений в Elasticsearch

```
template(name="es-document-template" type="list" option.jsonf="on") {
    property(outname="@timestamp" name="timereported" dateFormat="rfc3339" format="jsonf")
    property(outname="host" name="hostname" format="jsonf" format="jsonf")
    property(outname="severity" name="syslogseverity-text" caseConversion="upper" format="jsonf")
    property(outname="facility" name="syslogfacility-text" format="jsonf")
    property(outname="syslogtag" name="syslogtag" format="jsonf")
    property(outname="message" name="msg" format="jsonf")
}

# Требуется для действия create в Elasticsearch
template(name="es-bulkid-template" type="list") {
    property(name="$!es_record_id")
}

ruleset(name="save") {
    set $!es_record_id = $uuid;          # Создаём уникальный uuid
    action(
        type="omelasticsearch"
        server="127.0.0.1"
        template="es-document-template"
        searchindex="rsyslog"
        searchtype=""                    # В Elasticsearch 8 отменили это значение
        bulkmode="on"                    # Отправка нескольких документов разом
        bulkid="es-bulkid-template"
        dynbulkid="on"
        writeoperation="create"
        queue.type="linkedlist"
        queue.size="5000"
        queue.dequeuebatchsize="300"
        action.resumeretrycount="-1"
    )
}
```

Теперь в Kibana в Data View можно добавить новый обзор для индекса `rsyslog` и смотреть данные в Discovery и настроить интеграцию Logs.

Также можно сохранять сообщения напрямую в индексы. Для этого в шаблоне индекса отключается создание Data stream, к имени индексов добавляется `-*` и настроить в Rsyslog обычное индексирование документа с динамическим именем индекса

```
template(name="es-index-template" type="list") {
    constant(value="syslog-")
    property(name="timereported" dateformat="year")
    constant(value="-")
    property(name="timereported" dateformat="month")
    constant(value="-")
    property(name="timereported" dateformat="day")
}

ruleset(name="save") {
    action(
        type="omelasticsearch"
        template="es-document-template"
        searchindex="es-index-template"
        searchtype=""
        dynsearchindex="on"
        bulkmode="on"
        queue.type="linkedlist"
        queue.size="5000"
        queue.dequeuebatchsize="300"
        action.resumeretrycount="-1"
    )
}
```

## Manticoresearch и Grafana

Добавляем поддержку MySQL в Rsyslog

    apt install rsyslog-mysql

Поднимаем сервисы через Docker Compose

```yaml
services:
  manticore:
    image: manticoresearch/manticore
    restart: always
    ports:
      - 127.0.0.1:9306:9306
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

Подключается к Manticoresearch

    mysql -P 9306

Создаём таблицу

```sql
CREATE TABLE systemevents
(
    time timestamp,
    host string,
    severity string,
    facility string,
    syslogtag string,
    message text
) engine='columnar';
```

Шаблон SQL запроса для Rsyslog

```
template(name="ms-record-template" type="list" option.sql="on") {
    constant(value="INSERT INTO systemevents (time, host, severity, facility, syslogtag, message)")
    constant(value=" VALUES ('")
    property(name="timereported" dateFormat="unixtimestamp")
    constant(value="', ")
    property(name="hostname")
    constant(value=", '")
    property(name="syslogseverity-text")
    constant(value="', ")
    property(name="syslogfacility-text")
    constant(value="', ")
    property(name="syslogtag")
    constant(value=", '")
    property(name="msg")
    constant(value="')")
}
```

Перенаправление приходящих на сервер сообщений в Manticoresearch

```
module(load="imtcp")
input(type="imtcp" port="514" ruleset="save")

module (load="ommysql")
ruleset(name="save") {
    action(
        type="ommysql"
        server="127.0.0.1"
        db="Manticore"
        serverport="9306"
        uid="root"
        pwd=""
        template="ms-record-template"
        queue.type="linkedlist"
        queue.size="5000"
        queue.dequeuebatchsize="300"  # Пакетный INSERT
        action.resumeretrycount="-1"
    )
}
```

Пример SQL запроса для панели в Grafana

```sql
SELECT
  INTEGER(time) DIV ($__interval_ms DIV 1000) * ($__interval_ms DIV 1000) AS time,
  COUNT(*) AS value,
  $breakdown AS label
FROM
  Manticore.systemevents
WHERE
  $__unixEpochFilter(time)
  AND syslogtag IN ($syslogtag)
  AND severity IN ($severity)
  AND host IN ($host)
  AND MATCH('$search')
GROUP BY
  $breakdown,
  time
ORDER BY
  time ASC
LIMIT
  400
```

Создать переменные `syslogtag`, `severity` `host` и `breakdown` с типом Query и указать запрос для получения их значений

    SELECT groupby() FROM Manticore.systemevents GROUP BY syslogtag

Создать переменную `search` с типом Custom.

Прочая информация

- <https://manticoresearch.com/blog/manticoresearch-grafana-integration/>
- <https://grafana.com/docs/grafana/latest/datasources/mysql/>

## Loki и Grafana

https://grafana.com/docs/loki/latest/api/#push-log-entries-to-loki

## Старый синтаксис

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
