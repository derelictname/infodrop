# Dockerfile

Специальный файл со сценарием для сборки образов.

```bash
# Вначале нужно взять образ в качестве основы
FROM debian

# Команды, запускаемые в контейнере при сборке
RUN apt-get update && apt-get install -y --no-install-recommends vsftpd openssl
RUN useradd -m -p "$(openssl passwd -1 '/7aR0/\b')" user1

# Копирование файлов из текущей папки внутрь образа
COPY vsftpd.conf /etc/vsftpd.conf

RUN mkdir /empty

# Команда по умолчанию при запуске контейнера из готового образа
CMD /usr/sbin/vsftpd /etc/vsftpd.conf
```

## ENTRYPOINT

Команда, которая приминает в качестве аргумента команду контейнера по умолчанию или указанную явно при запуске. По умолчанию ENTRYPOINT является `/bin/sh -c`.

Например, мы запустили контейнер так

    docker run debian date

В контейнере при этом выполнится `/bin/sh -c date`.

Менять ENTRYPOINT имеет смысл, например, для изменения поведения контейнера в зависимости от запускаемой команды.

Пример Dockerfile

```bash
# Копируем скрипт
COPY start.sh /start.sh

# Делаем его исполняемым
RUN chmod +x /start.sh

# Устанавливаем в качестве точки входа
ENTRYPOINT ["/start.sh"]

# Команда по умолчанию при запуске контейнера
CMD ["myprogram", "--option1", "--option2", "value"]
```

Пример скрипта

```bash
#!/usr/bin/env sh

set -e

# Проверяем, что контейнер запущен с командой по умолчанию.
if [ "$1" = 'myprogram' ]
then
    # Создаём пользователя с паролем, который берётся из переменной окружения.
    # Таким образом мы можем указать пароль при запуске контейнера.
    echo -e "$MYPASSWORD\n$MYPASSWORD" | adduser $MYUSER
    
    # Другие команды.
    touch /1

    # Запуск команды из аргументов, переданных скрипту.
    exec "$@"
fi

# В остальных случаях (например, запуск bash).
exec "$@"
```

- [Справка по всем директивам](https://docs.docker.com/engine/reference/builder/)
- [Рекомендации по написанию](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Пример приложения](https://docs.docker.com/get-started/02_our_app/)
- [Как не надо](https://codefresh.io/containers/docker-anti-patterns/)
