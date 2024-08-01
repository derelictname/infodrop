# Traefik

Веб-сервер, выступающий в роли обратного прокси, умеющий изменять свою конфигурацию в реальном времени, получая информацию через различные источники. В основном источниками выступают оркестраторы контейнеров.

Из примечательных возможностей

- ограничение доступа
- балансировка запросов
- выпуск сертификатов

Запросы в Traefik проходят через цепочку логических объектов

1. Точка входа
2. Маршрутизатор
3. Сервис

При работе с Docker Traefik связывается с ним через его сокет и следит за состоянием контейнеров и меняет маршрутизацию. Большая часть настроек маршрутизации определяется через раздел labels в описании сервисов Docker Compose, которая считывается при создании контейнера.

Пример настроек

```yaml
# traefik.yml

global:
  checkNewVersion: false
  sendAnonymousUsage: false

providers:

  # https://doc.traefik.io/traefik/routing/providers/docker/
  docker:
    exposedByDefault: false

    # При желании можно установить правило маршрутизации по умолчанию.
    # Например, сервис service1 в проекте project1 будет доступен
    # по домену service1-project1.example.com.
    defaultRule: Host(`{{ trimPrefix `/` .Name }}.example.com`)

# Точки входа, на которых будет слушать запросы Traefik.
entryPoints:
  entry1:
    address: :80
    http:
      # По умолчанию перенаправлять на entry2 и схему HTTPS.
      redirections:
        entryPoint:
          to: entry2
          scheme: https
  entry2:
    address: :443
    http:
      # Обработчик зашифрованного соединения.
      tls:
        certResolver: resolver1

# Для выпуска сертификатов используются обработчики SSL acme.
# Traefik выпускает Let's Encrypt для домена при получении запроса.
certificatesResolvers:
  resolver1:
    acme:
      # Почта для регистрации в провайдере SSL и его уведомлений.
      email: email@example.com

      # Желательно настроить хранение выпущенных сертификатов.
      # Например, persistent volume или пробросить папку.
      storage: /letsencrypt/acme.json
      httpChallenge:
        entryPoint: entry1

# Веб-интерфейс Traefik для просмотра состояния.
api:
  dashboard: true

# Для обнаружения проблем в настройках через вывод контейнера Traefik.
log:
  level: WARN
```

Пример сервиса

```yaml
# docker-compose.yml

services:
  # Запускаем Traefik в виде сервиса Docker Compose.
  traefik:
    image: traefik
    ports:
      - 80:80
      - 443:443
    volumes:
      # Пробрасываем файл с настройками Traefik.
      - $PWD/traefik.yml:/etc/traefik/traefik.yml

      # Пробрасываем сокет Docker для чтения настроек контейнеров.
      - /var/run/docker.sock:/var/run/docker.sock:ro

      # Папка для хранения выпущенных сертификатов.
      # Примечание: проброс файлов в Docker работает неожиданным образом, так как,
      # если в создаваемом контейнере файла ещё нет, то вместо него будет создана папка.
      - letsencrypt:/letsencrypt

    # Настройки можно задавать через traefik.yml и/или labels.
    labels:
      traefik.enable: true

      # Создаём маршрутизатор с произвольным именем и назначаем на него
      # внутренний enedpoint Traefik.
      traefik.http.routers.dashboard.service: api@internal

      # Приоритет правил маршрутизаторов зависит от длины правила.
      traefik.http.routers.dashboard.rule: Host(`server.org`) && PathPrefix(`/api`, `/dashboard`)

      # Создаём обработчик с именем restricted и назначаем его на маршрутизатор.
      traefik.http.routers.dashboard.middlewares: restricted

      # Назначаем на обработчик фильтр по IP адресу.
      traefik.http.middlewares.restricted.ipwhitelist.sourcerange: 11.22.33.44

  # Какой-то сайт или сервис.
  app1:
    build: ./app1-build

    labels:
      traefik.enable: true

      # Создаём маршрутизатор для HTTP.
      traefik.http.routers.app1.rule: Host(`example.com`) && PathPrefix(`/api`)

      # Создаём обработчик, обрезающий часть URL в запросе.
      traefik.http.middlewares.strip1.stripprefix.prefixes: /api

      # Добавляем обработчик запросов в маршрутизатор запросов
      traefik.http.routers.app1.middlewares: strip1

    # Явно указывать порты сервиса необязательно, так как Traefik
    # определяет их из метаданных контейнера. Явное указание порта
    # настраивается через правило маршрутизатора.

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

## wildcard сертификаты

Выпуск wildcard сертификатов доступен только через подтверждение с помощью DNS записи. Traefik умеет работать с разными DNS провайдерами и сам добавяет нужную TXT запись для подтверждения владения доменом.

Пример ручного провайдера

```yaml
certificatesResolvers:
  resolver2:
    acme:
      email: email@example.com
      storage: /letsencrypt/acme.json
      dnsChallenge:
        provider: manual

# Обязательно
log:
  level: DEBUG
```

После запуска контейнера Traefik в его логах появится TXT запись, которую нужно добавить в зоне нашего домена

    docker compose run -it traefik

После создания ждём обнаружения записи и нажимаем Enter в терминале контейнера.

При продлении сертификата создаётся новая TXT запись, поэтому ручной способ нельзя автоматизировать.

## Ссылки

- [Подробнее о работе с Docker](https://doc.traefik.io/traefik/providers/docker/)
- [Руководство для Let's Encrypt](https://doc.traefik.io/traefik/user-guides/docker-compose/acme-http/)
- [Список labels](https://doc.traefik.io/traefik/reference/dynamic-configuration/docker/)
