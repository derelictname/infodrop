# Ansible

Инструмент автоматизации настройки сервисов и системы на серверах. Задания пишут в декларативном стиле, описывая таким образом целевое состояние системы. То есть сначала выполняется проверка, что конечный результат задания уже достигнут, в таком случае оно не выполняется и сообщается об этом. Для неспосредственного выполнения действий Ansible подключается по SSH, используя модуль subprocess, и использует интерпретатор Python на удалённом сервере.

Сначала необходимо составить список всех целей, над которыми будем выполнять действия

```yaml
all:
  hosts:
    example.com:
    server01.org:
    11.22.33.44:
    123.123.123.123:
```

Также это может быть папка с несколькими такими файлами (для удобства разбиения) и/или скрипт (например, Python, который должен быть исполняемым файлом и иметь shebang), отдающий нужный формат.

В другом файле составляется перечень заданий

```yaml
# playbook - файл с множеством групп заданий,
# назначенных указанным серверам

- name: Набор взаимосвязанных по смыслу действий на сервере
  hosts:
    - example.com

  # play - группа заданий
  tasks:
    # task - задание
    - name: Установка программ
      apt:
        name:
          - atop
          - vim
        state: present

    - name: Перезапуск MySQL
      systemd:
        name: mariadb
        state: restarted
      # используется для возможности выполнять task отдельно
      tags:
        - mytag1

- name: Перечень ещё каких-то манипуляций
  hosts:
    - server01.org
    - 11.22.33.44

  tasks:
    - name: Запуск какой-то программы, создающей файл
      command:
        cmd: myprogram1
        creates: '/home/user1/file.txt'
```

За выполнение каждой задачи отвечает модуль (apt, systemd, command и т.д.).

Перед запуском рекомендуется вывести задачи, которые будут выполнены

    ansible-playbook -i hosts.yml -u root -k --list-tasks playbook1.yml

- `-i` - инвентарный файл/папка
- `-u` - пользователь удалённой системы для входа по SSH и выполнения заданий
- `-k` - принудительный запрос пароля SSH (если не настроена авторизация по ключу)
- `--list-tasks` - вывести список задач, но не выполнять
- `playbook1.yml` - имя файла с заданиями

Запуск из playbook только задач с меткой `mysql`

    ansible-playbook -i hosts.yml -u root -k --tags mytag1 playbook1.yml

- `--tags` - выполнить только задания с определённой меткой

Другие опции

- `--check` - имитировать выполнение плейбука
- `--become` - повысить привелегии с помощью sudo
- `-K` - запросить ввод пароля для sudo

Проверка корректности составления по множеству [правил](https://ansible-lint.readthedocs.io/en/latest/default_rules.html)

    ansible-lint playbook1.yml

Пример более сложного инвентарного файла с группами и переменными

```yaml
all:
  # Сервера вне групп обработаются только при указании PATTERN - all
  hosts:
    11.22.33.44:
    server1:
      ansible_host: 22.33.44.55

  # Группы объединяют сервера по смыслу (назначение, расположение, статус при разработке)
  # и включать только нужные в обработку (имя группы указывается в PATTERN)
  children:
    php:
      hosts:
        33.44.55.66:
      # Переменные будут подставляться в playbook вместо "{{ session.gc_maxlifetime }}"
      # или при копировании файлов модулем template
      vars:
        session.gc_maxlifetime: 1800

    mysql:
      hosts:
        44.55.66.77:
        db1.example.org:
      vars:
        max_connections: 500
        innodb_buffer_pool_size: "12G"
```

Сервера могут повторяться в разных группах. При этом **переменные будут учитываться из всех групп** (проверять через `ansible-inventory`).

Выполнение одиночного task

    ansible localhost -m hostname -a name=test123
    ansible -i example.org, -a id -u john all
    ansible -i example.org, -m ping all

- `-i example.org,` - инвентарный файл в виде списка через запятую
- `-a id` - аргументы выполнения модуля
- `all` - PATTERN для выбора хостов

Сбор информации о сервере

    ansible -i hosts.yml example.com -m ansible.builtin.setup

Структура папок относительно выполняемого **playbook**

- files - конфиги, ключи, исполняемые файлы для копирования на сервер
- roles - см. ниже
- library - см. ниже
- plugins/modules

Модуль template похож на copy, но копирует текстовые файлы с приминением шаблонизатора jinja2, таким образом подставляя значение переменных из playbook.

## Roles

Однородные по значению файлы, задания и переменные можно объединять в роль. Роли хранятся в отдельных папках в общей папке **roles**, рядом с которой происходит выполнение **playbook**.

## Import или Include

Для организации заданий в отдельные файлы существуют модули

- `import_task`/`import_role` - импортирует роли и задания в начале выполнения
- `include_task`/`include_role` - импортирует роли и задания в процессе выполнения, когда обработка доходит до этого задания, таким образом может использоваться в цикле

[Подробнее про разницу](https://serverfault.com/questions/875247/whats-the-difference-between-include-tasks-and-import-tasks)

## Callback

Способы реакции на результат команд

    ansible -m setup -i hosts -t /tmp/facts

## Modules

- `ansible.builtin.lineinfile` - добавить/изменить строку (можно в конкретный блок)
- `ansible.builtin.blockinfile` - добавить/изменить текст в файл между метками Ansible
- `ansible.builtin.replace` - замена через регулярное выражение
- `ansible.builtin.tempfile` - создать временный файл/директорию, доступные только создавшему
- `ansible.builtin.reboot` - перезагрузить и дождаться запуска для продолжения

Модули, доступные без наличия Python

- `ansible.builtin.raw` - отправляет команды по SSH (например, для установки Python)
- `ansible.builtin.script` - передаёт скрипт на удалённый сервер и выполняет

## modules

В папке library относительно выполняемого playbook можно размещать свои модули.

Минимальный пример

```bash
#!/bin/bash

echo "{\"changed\": false, \"id\": \"$(id)\"}"
```

`changed` является одним из стандартных ключей ответа.

Применение

```yaml
- name: testing custom modules
  hosts:
    - all
  tasks:
    - name: my custom module
      test1:
      register: myresult
    - name: Output result
      debug:
        msg: "{{ result }}"
```


Если модуль только собирает информацию, то его имя должно оканчиваться на `_facts`, а сам он возвращать ключ `ansible_facts` с ключом внутри, который будет добавлен в общую информацию.

```bash
#!/bin/bash

printf "{\"ansible_facts\": {\"users\": ["

while IFS=: read -r username password uid gid comment home shell
do
    if [[ "$shell" == !("/sbin/nologin"|"/usr/sbin/nologin"|"/bin/false"|"/sbin/halt"|"/sbin/shutdown") ]]
        then printf "{\"username\": \"$username\", \"uid\": \"$uid\", \"comment\": \"$comment\", \"home\": \"$home\"}"
    fi
done < /etc/passwd | sed 's/}{/}, {/g'

printf "]}}\n"
```

Применение

```yaml
- hosts:
    - all

  tasks:
  - users_facts:

  - debug:
      var: ansible_facts.users
```

[Подробнее](https://docs.ansible.com/ansible/latest/dev_guide/developing_modules_general.html)

## Настройка

Просмотр изменённых настроек Ansible

    ansible-config dump --only-changed

Для более читаемого вывода ошибок при выполнении нужно добавить в `~/.ansible.cfg`

```ini
[defaults]
stdout_callback = yaml
```

## plugins

https://docs.ansible.com/ansible/latest/dev_guide/developing_plugins.html

## Ссылки

- [Руководство](https://docs.ansible.com/ansible/latest/user_guide/index.html#getting-started)
- [Пример структуры проекта](https://docs.ansible.com/ansible/latest/user_guide/sample_setup.html)
- [Групповые переменные](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#splitting-out-vars)
- [Перечисление хостов](https://docs.ansible.com/ansible/latest/user_guide/intro_patterns.html)
- [Интерфейс на Vue.js](https://www.ansible-semaphore.com)
