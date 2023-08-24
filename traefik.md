# Traefik

Динамический обратный прокси, который умеет получать настройки в реальном времени из оркестраторов контейнеров.

Например, Traefik связывается с Docker через сокет и наблюдает состояние контейнеров и раздел labels в описании сервисов Docker Compose. При запуске или остановке контейнеров Traefik изменит маршрутизацию, предоставив доступ к контейнеру или перенаправив запросы на другой.

Также Traefik может ограничивать доступ, балансировать запросы и выпускать сертификаты.

Запросы в Traefik проходят через цепочку логических объектов

1. Точка входа
2. Маршрутизатор
3. Обработчики
4. Сервис

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

# Точки входа, на которые сам Traefik принимает запросы.
entryPoints:
  entry1:
    address: :80
    http:
      redirections:
        entryPoint:
          to: entry2
          scheme: https
  entry2:
    address: :443
    http:
      tls:
        certResolver: resolver1

# Для выпуска сертификатов используются обработчики SSL acme.
# Если Traefik видит запрос, то выпускает Let's Encrypt для домена.
certificatesResolvers:
  resolver1:
    acme:
      email: email@example.com

      # Желательно настроить хранение выпущенных сертификатов.
      # Например, persistent volume или примонтировать папку.
      storage: /letsencrypt/acme.json
      httpChallenge:
        entryPoint: entry1

# Веб-интерфейс Traefik для просмотра состояния.
api:
  dashboard: true

# Для обнаружения проблем в настройках через вывод контейнера Traefik.
log:
  level: DEBUG
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
    
      # Монтируем настройки Traefik.
      - $PWD/traefik.yml:/etc/traefik/traefik.yml

      # Монтируем сокет Docker для чтения настроек контейнеров.
      - /var/run/docker.sock:/var/run/docker.sock:ro

      # Постоянное хранилище сертификатов.
      - letsencrypt:/letsencrypt

    labels:
      traefik.enable: true

      # Создаём маршрутизатор с произвольным именем и назначаем на него
      # внутренний enedpoint Traefik.
      traefik.http.routers.dashboard.service: api@internal     
      
      # Приоритет правил маршрутизаторов зависит от длины правила.
      traefik.http.routers.dashboard.rule: Host(`example.com`) && PathPrefix(`/api`, `/dashboard`)

      # Создаём обработчик с именем restricted и назначаем его на маршрутизатор.
      traefik.http.routers.dashboard.middlewares: restricted

      # Назначаем на обработчик фильтр по IP адресу.
      traefik.http.middlewares.restricted.ipwhitelist.sourcerange: 11.22.33.44

  # Произвольный сервис.
  app1:
    build: ./app1-build

    labels:
      traefik.enable: true

      # Создаём маршрутизатор для HTTP.
      traefik.http.routers.app1.rule: Host(`example.com`) && PathPrefix(`/api`)

      # Создаём обработчик, обрезающий часть URL в запросе.
      traefik.http.routers.app1.middlewares: app1-strip1
      traefik.http.middlewares.strip1.stripprefix.prefixes: /api

    # Явно указывать порты сервиса необязательно, так как Traefik сам
    # проверяет их в описании контейнера.
    # Изменение порта настраивается через правило маршрутизатора.

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

Выпуск wildcard сертификатов доступен только через подтверждение с помощью DNS записи.
Traefik умеет работать с множеством DNS провайдеров и сам добавяет TXT запись.

Добавим обработчик TLS через ручное добавление TXT записи

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

TXT запись будет в логах контейнера, поэтому запускаем его сразу с выводом в терминал

    docker compose run -it traefik

Создаём DNS запись, ждём обнаружения записи и нажимаем Enter в терминале.

При обновлении сертификата сгенерируется новая TXT запись, поэтому ручной способ нельзя автоматизировать.

## Ссылки

- [Подробнее о работе с Docker](https://doc.traefik.io/traefik/providers/docker/)
- [Руководство для Let's Encrypt](https://doc.traefik.io/traefik/user-guides/docker-compose/acme-http/)
- [Перечень labels](https://doc.traefik.io/traefik/reference/dynamic-configuration/docker/)
