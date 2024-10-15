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

Списоком целей может быть тектовый файл, папка с файлами, скрипт (должен иметь права на исполнение и содержать shebang) или аргумент `-i` со списоком через запятую.

Переменные подставляются в заданиях в виде `{{ session.gc_maxlifetime }}` или в файлах, при копировании модулем template.

Сервера могут повторяться в разных группах. При этом **переменные будут учитываться из всех групп**.

Группы объединяют сервера по смыслу (назначение, расположение, статус при разработке) и включать только нужные в обработку (имя группы указывается в PATTERN).

Для удобства указываем путь к списоку целей в файле `ansible.cfg`

```ini
[defaults]
inventory = inventory
```

Просмотр изменённых настроек

    ansible-config dump --only-changed

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

Проверяем изменения (кроме заданий, зависящих от других заданий)

    ansible-playbook playbook1.yml --check

https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_checkmode.html

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

- https://docs.ansible.com/ansible/2.8/user_guide/playbooks_best_practices.html
- https://docs.ansible.com/ansible/latest/dev_guide/overview_architecture.html#ansible-search-path
- https://docs.ansible.com/ecosystem.html

## Ad-hoc

Выполнение одиночного задания через модуль

    ansible -o -a uptime all
    ansible server1 -m ansible.builtin.apt -a "name=socat state=present update_cache=true" --check -o
    ansible -m ansible.builtin.ping all

Описание опций и аргументов

- `-o` - однострочный вывод
- `-a uptime` - аргументы модуля
- `-m ansible.builtin.ping` - указать используемый модуль (модуль по умолчанию `command`)
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

## Module

- `ansible.builtin.lineinfile` - добавить/изменить строку (можно в конкретный блок)
- `ansible.builtin.blockinfile` - добавить/изменить текст в файле между метками Ansible
- `ansible.builtin.replace` - замена через регулярное выражение
- `ansible.builtin.tempfile` - создать временный файл/директорию, доступные только создавшему
- `ansible.builtin.reboot` - перезагрузить и дождаться запуска для продолжения
- `ansible.builtin.assert`

Модули, доступные без наличия Python

- `ansible.builtin.raw` - отправляет команды по SSH (например, для установки Python)
- `ansible.builtin.script` - передаёт скрипт на удалённый сервер и выполняет

Модуль template похож на copy, но копирует текстовые файлы с приминением шаблонизатора jinja2, таким образом подставляя значение переменных из playbook.

## Module (dev)

Модули являются также и плагинами.

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

## Plugins

Начиная с версии 2.11 свои плагины размещаются в коллекциях. Старое размещение доступно через  `~/.ansible/plugins/inventory` (имя папки зависит от типа плагина), либо указать свой путь через настройки

```ini
[defaults]
inventory_plugins = plugins/inventory
```

Также команда `ansible-playbooks` смотрит модули в папке `inventory_plugins` и подобных.

Файл с настройками плагина

```yaml
# myplugin.yml

plugin: myplugin  # этот параметр используется плагином auto
username: test1
password: test1
```

Пример плагина

```python
from ansible.inventory.data import InventoryData
from ansible.plugins.inventory import BaseInventoryPlugin
import psycopg


# Docs are required and used to test config file
DOCUMENTATION = r"""
name: db_source
plugin_type: inventory
short_description: Get hosts from database
description: Returns a dynamic host inventory from PostgreSQL database, filter machines that have IP address

options:
  username:
    description: describe this config option
    required: True
    type: string
  password:
    description: describe this config option
    required: True
    type: string
"""


class InventoryModule(BaseInventoryPlugin):
    NAME = "db_source"

    # used internally by Ansible, it should match the file name but not required
    def verify_file(self, path: str):
        # base class verifies that file exists and is readable by current user
        return super(InventoryModule, self).verify_file(path)

    def parse(self, inventory: InventoryData, loader, path, cache=True):

        # Call base class __init__ first
        super(InventoryModule, self).parse(inventory, loader, path, cache)

        # Reading config file
        self._read_config_data(path)

        # Assign values to attributes for convenience
        self.username = self.get_option("username")

        SQL_LIST = r"""
        SELECT hostname, ip_address
        FROM
            servers
        WHERE
            ip_addresses IS NOT NULL
        LIMIT 10;
        """

        with psycopg.connect(
            f"host=10.20.30.40 dbname=db1 user={self.username}"
        ) as connection:
            self.display.v("Connected to server")
            with connection.cursor() as cursor:
                cursor.execute(SQL_LIST)
                for title, hostname, host in cursor:
                    self.inventory.add_host(hostname)
                    self.inventory.set_variable(hostname, "ansible_host", str(ip_address))
```

Вывести список плагинов

    ansible-doc -t inventory -l

Документация

    ansible-doc -t inventory myplugin

Проверка

    ansible-inventory -i myplugin.yml --list -y -v

- https://docs.ansible.com/ansible/latest/dev_guide/developing_inventory.htm
- https://docs.ansible.com/ansible/latest/dev_guide/developing_plugins.html
- https://docs.ansible.com/ansible/latest/dev_guide/developing_locally.html
- https://www.redhat.com/sysadmin/ansible-plugin-inventory-files

## Collections

С версии 2.11 для организации отдельных плагинов, модулей, ролей, плейбуков теперь используются коллекции.

https://docs.ansible.com/ansible/latest/reference_appendices/config.html#collections-paths

Cоздаём папку `~/.ansible/collections/ansible_collections/`, либо в текущей папке создаём `collections/ansible_collections` и указываем на неё в настройках

```ini
[defaults]
collections_paths = collections
```

В папке `ansible_collections` выполняем

    ansible-galaxy collection init mynamespace.mycollection

`mynamespace` обычно имя компании или псевдоним, `mycollection` обозначает суть коллекции.

Смотрим список коллекций

    ansible-galaxy collection list

Документация

    ansible-doc -t inventory mynamespace.mycollection.myplugin

https://docs.ansible.com/ansible/latest/reference_appendices/faq.html#where-did-all-the-modules-go

## Callback

Способы реакции на результат команд

    ansible -m ansible.builtin.setup -i hosts -t /tmp/facts

Для более читаемого вывода ошибок при выполнении нужно добавить в `~/.ansible.cfg`

```ini
[defaults]
stdout_callback = yaml
```

## Ссылки

- [Руководство](https://docs.ansible.com/ansible/latest/user_guide/index.html#getting-started)
- [Пример структуры проекта](https://docs.ansible.com/ansible/latest/user_guide/sample_setup.html)
- [Групповые переменные](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#splitting-out-vars)
- [Перечисление хостов](https://docs.ansible.com/ansible/latest/user_guide/intro_patterns.html)
- [Интерфейс на Vue.js](https://www.ansible-semaphore.com)

Подробнее про работу SSH в Ansible

- [Построение команды](https://github.com/ansible/ansible/blob/2e62724a8a8f801af35943d266dd906e029e20d6/lib/ansible/plugins/connection/ssh.py#L648)
- [Отправка команды](https://github.com/ansible/ansible/blob/bd6feeb6e7b334d5da572cbb5add7594be7fc61e/lib/ansible/plugins/connection/ssh.py#L889)

