# Ansible

Позволяет выполнять однотипные операции на множестве серверов. При этом используется упрощённый формат заданий - playbook. Задания описывают не действия, а конечное состояние. При выполнении списка заданий Ansible подключается по SSH, проверяет текущее состояние каждого задания (например, наличие требуемого файла или программы) и при необходимости выполняет модули (требуют наличия Python). Таким образом повторное выполнение заданий не приводит к повторным изменениям.

Для удобства создаём папку `inventory` и в ней создаём список целей в файле `myservers.yml`

```yaml
all:
  vars:
    timezone: Europe/Moscow

ungrouped:
  hosts:
    domain1.company.com:
      ansible_host: 20.30.40.50
    domain2.company.com:
    12.34.56.78:
    23.45.67.89:
      timezone: Europe/Minsk

production:
  hosts:
    mycoolserver.net:
    test[1:10].company.com:

  vars:
    session.gc_maxlifetime: 1800
    max_connections: 500
    innodb_buffer_pool_size: 12G
```

Списоком целей может быть тектовый файл, папка с файлами, исполняемый файл. Если это скрипт, то он должен иметь права на исполнение и содержать shebang.

Переменные подставляются в заданиях в виде {{ session.gc_maxlifetime }} или в файлах, при копировании модулем template.

Сервера могут повторяться в разных группах. При этом **переменные будут учитываться из всех групп**.

Группы объединяют сервера по смыслу (назначение, расположение, статус при разработке) и включать только нужные в обработку (имя группы указывается в PATTERN).

Для удобства указываем путь к списоку целей в файле `ansible.cfg`

```ini
[defaults]
inventory = inventory
```

Проверяем список целей

    ansible-inventory --list -y

Создаём список заданий в файле `playbook1.yml`

```yaml
# playbook - группа целей hosts (также называется PATTERN) и их задания tasks
- name: Prepare {{ inventory_hostname }}
  hosts:
    - 11.22.33.44
    - test3.company.com

  # tasks содержит play - последовательность заданий
  tasks:
    # task - задание
    - name: Install packages
      ansible.builtin.apt:
        name:
          - atop
          - vim
        state: present

    - name: Run myprogram1 (it creates file)
      ansible.builtin.command:
        cmd: myprogram1
        creates: '/etc/program1/program1.conf'
      tags:
        - mytag1

# Ещё один playbook в этом же файле
- name: Second tasks
  hosts:
    - all

  tasks:
```

Проверяем [грамотность](https://ansible-lint.readthedocs.io/en/latest/default_rules.html) составления файла заданий

    ansible-lint playbook1.yml

Проверяем последовательность заданий

    ansible-playbook playbook1.yml --list-tasks

Имитируем выполнение (кроме заданий, зависящих от других заданий)

    ansible-playbook playbook1.yml --check

Выполняем

    ansible-playbook playbook1.yml

Дополнительные опции

- `-i` - указать файл целей или папку
- `-u` - указать пользователя SSH
- `-k` - вывести запрос пароля SSH
- `--become` - использовать sudo
- `-K` - вывести запрос пароля sudo, если не добавлен NOPASSWD в sudoers
- `--list-tasks` - отобразить список заданий
- `-i example.org,` - инвентарный файл в виде списка через запятую

Если в задачах указать метки, то можно запускать их выборочно

    ansible-playbook playbook1.yml --tags mytag1

Некоторые папки используемые рядом с playbook

- files - файлы, используемые в заданиях
- roles - группировка заданий, файлов и т.д.
- library - свои модули
- plugins/modules - плагины для работы самого Ansible

Просмотр изменённых настроек Ansible

    ansible-config dump --only-changed

## Одиночные задания

Выполнение одиночного задания через модуль (по умолчанию ansible.builtin.command)

    ansible -a uptime all
    ansible -m hostname -a name=badserver goodserver
    ansible -m ansible.builtin.ping all

Описание опций и аргументов

- `-a uptime` - аргументы модуля
- `-m ansible.builtin.ping` - использовать другой модуль
- `all` - PATTERN (значение hosts в списке заданий)

Сбор информации

    ansible -m ansible.builtin.setup server1

## Roles

Однородные по значению файлы, задания и переменные можно объединять в роль. Роли хранятся в отдельных папках в общей папке **roles**, рядом с которой происходит выполнение **playbook**.

## Import и Include

Чтобы разбить список заданий на несколько файлов используются модули

- `import_task`, `import_role` - подставляет задания или роли в начале выполнения основного задания
- `include_task`, `include_role` - подставляет задания или роли, когда обработка доходит до этого места (таким образом может использоваться в цикле)

[Подробнее про разницу](https://serverfault.com/questions/875247/whats-the-difference-between-include-tasks-and-import-tasks)

## Полезные модули

- `ansible.builtin.lineinfile` - добавить/изменить строку (можно в конкретный блок)
- `ansible.builtin.blockinfile` - добавить/изменить текст в файл между метками Ansible
- `ansible.builtin.replace` - замена через регулярное выражение
- `ansible.builtin.tempfile` - создать временный файл/директорию, доступные только создавшему
- `ansible.builtin.reboot` - перезагрузить и дождаться запуска для продолжения

Модули, доступные без наличия Python

- `ansible.builtin.raw` - отправляет команды по SSH (например, для установки Python)
- `ansible.builtin.script` - передаёт скрипт на удалённый сервер и выполняет

Модуль template похож на copy, но копирует текстовые файлы с приминением шаблонизатора jinja2, таким образом подставляя значение переменных из playbook.

## Свои модули

Свои модули создаются в виде исполняемых файлов в папке library относительно исполняемого playbook.

Минимальный модуль `test1`

```bash
#!/bin/bash

echo "{\"changed\": true, \"uptime\": \"$(uptime)\"}"
```

`changed` является одним из стандартных ключей ответа.

Использование

```yaml
- name: Testing modules
  hosts:
    - all

  tasks:
    - name: Run custom module
      test1:
      register: someresult
      
    - name: Show result
      ansible.builtin.debug:
        msg: "{{ someresult }}"
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

Использование

```yaml
- name: Testing modules
  hosts:
    - all

  tasks:
  - name: Get users list
    users_facts:

  - name: Show result
    debug:
      var: ansible_facts.users
```

[Подробнее](https://docs.ansible.com/ansible/latest/dev_guide/developing_modules_general.html)

## Callback

Способы реакции на результат команд

    ansible -m ansible.builtin.setup -i hosts -t /tmp/facts

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

Подробнее про работу SSH в Ansible

- [Построение команды](https://github.com/ansible/ansible/blob/2e62724a8a8f801af35943d266dd906e029e20d6/lib/ansible/plugins/connection/ssh.py#L648)
- [Отправка команды](https://github.com/ansible/ansible/blob/bd6feeb6e7b334d5da572cbb5add7594be7fc61e/lib/ansible/plugins/connection/ssh.py#L889)

