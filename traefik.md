# Traefik

Динамический программный маршрутизатор (HTTP/S, TCP, UDP, балансировка трафика, ограничение доступа), динамически считывающий настройки из разных источников. Например, запускается и останавливается множество контейнеров, и Traefik без перезагрузки получает настройки от Docker/Kubernetes и создаёт маршруты до сервисов в контейнерах и выпускает сертификаты, ограничивает доступ к ним по разным правилам.

Пример основных настроек

```yaml
# traefik.yml

providers:
  docker:

    # Traefik связывается с Docker через его сокет и
    # считывает раздел labels у сервисов.
    # Для доступа к сервису необходимо правило для внутреннего маршуртизатора.
    # Пропишем правило по умолчанию для всех сервисов.
    defaultRule: "Host(`{{ trimPrefix `/` .Name }}.example.com`)"

    # В данном примере для каждого сервиса будет создан маршрутизатор
    # с правилом service1-project1.example.com, где
    # service1-project1 - название сервиса в Docker.

    # Примечание: для удобства работы с произвольным число сервисов
    # можно создать wildcard DNS запись *.example.com.

# Точки входа
entryPoints:
  entry1:
    address: ":80"
  entry2:
    address: ":443"

# Обработчики SSL acme умеет выполускать Let's Encrypt при получении
# запроса к сервису по домену.
certificatesResolvers:
  letsencrypt1:
    acme:
      email: email@example.com
      # Здесь будут хранится выпущенные серификаты.
      # Необходимо настроить persistent volume или
      # примонтировать папку в сервисе с самим Traefik
      storage: /letsencrypt/acme.json
      httpChallenge:
        entryPoint: entry1

# Веб-интерфейс Traefik для визуальной проверки настроек (менять нельзя)
api:
  dashboard: true
```

Пример сервисов

```yaml
# docker-compose.yml

services:

  traefik:
    image: traefik
    ports:
      - 80:80
      - 443:443
    volumes:
      - $PWD/traefik.yml:/etc/traefik/traefik.yml
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - letsencrypt:/letsencrypt
    labels:
      # В данном случае api@internal это внутренний endpoint Traefik,
      # dashboard это придуманное имя для маршрутизатора,
      # restricted это придуманное имя для middleware (промежуточных обработчиков),
      # типа ipwhitelist, который ограничивает доступ по IP
      traefik.http.routers.dashboard.service: api@internal     
      traefik.http.routers.dashboard.middlewares: restricted
      traefik.http.middlewares.restricted.ipwhitelist.sourcerange: 11.22.33.44


  app1:
    build: ./app1-build
    labels:

      # Правило маршрутизации запросов к сервису
      traefik.http.routers.app1-router1.rule: Host(`example.com`) && PathPrefix(`/api`)

      # Обработка URL
      # В данном примере вырезается приставка /api и сервис получает запрос без неё
      traefik.http.routers.app1-router1.middlewares: app1-strip1
      traefik.http.middlewares.app1-strip1.stripprefix.prefixes: /api

      # Обработка SSL
      traefik.http.routers.app1-router1.tls: true
      traefik.http.routers.app1-router1.tls.certresolver: letsencrypt1

      # Перенаправление на HTTPS
      traefik.http.routers.app1-router2.entrypoints: entry1
      traefik.http.routers.app1-router2.middlewares: app1-redirect1
      traefik.http.middlewares.app1-redirect1.redirectscheme.scheme: entry2
      traefik.http.middlewares.app1-redirect1.redirectscheme.permanent: true
      traefik.http.routers.app1-router1.entrypoints: entry2

    # Раздел ports не указан, так как Traefik умеет подсмотреть их сам
    # Если нужен другой порт, то это настраивается через правило маршрутизатора

  mysql:
    image: mariadb
    environment:
      MARIADB_ROOT_PASSWORD: ed1g2hhtrfd
    volumes:
      - db:/var/lib/mysql

volumes:
  letsencrypt:
  db:
```

## dnsChallenge

Выпуск wildcard Let's Encrypt возможен только при подтверждении по DNS.
Механизм DNS в Traefik пытается добавить TXT запись через API поддерживаемых сервисов.
Для примера ниже представлен вариант через ручное добавление записи с просмотром записи в логах.

```yaml
certificatesResolvers:

  # ...

  letsencrypt2:
    acme:
      email: email@example.com
      storage: /letsencrypt/acme.json
      dnsChallenge:
        provider: manual

log:
  level: DEBUG
```

Запускаем интерактивно контейнер и смотрим TXT запись

    docker compose run -it traefik

Создаём DNS запись, ждём обновления и нажимаем Enter в терминале сервиса.

## Ссылки

- [Подробнее о работе с Docker](https://doc.traefik.io/traefik/providers/docker/)
- [Руководство для Let's Encrypt](https://doc.traefik.io/traefik/user-guides/docker-compose/acme-http/)
