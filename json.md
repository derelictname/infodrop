# json

JavaScript Object Notation

- [Описание формата](https://www.json.org/json-ru.html)
- [Генератор JSON из Bash](https://github.com/jpmens/jo)

Пример синтаксиса

```json
"1 2 3", 1, 2, 3, {}, [], true, false, null 
"just a string" ["string inside array", 1, 2]
{"key": "value", "inside": "object"}
{"key": ["of", "list", "inside", "object"]}
[{"key": "of"}, {"object": "inside"}, {"list": "\u2764"}]
```

Правильность синтаксиса можно проверять в консоли браузера по F12

## jq

Парсер JSON

https://stedolan.github.io/jq/manual/

```bash
cat file.json | jq '.'                                  # фильтр без изменений
curl example.com/api | jq '.[]'                         # все элементы списка
smartctl -j -i -A -H /dev/sda | jq '.[0]'               # первый элемент списка
ss -tulpn | jc --ss -p  | jq '.[].netid'                # ключ в каждом элементе
jc -p ss -tulpn | jq '.[] | .netid + " " + .local_port' # форматированный вывод
cat file.json | jq '.[] | {test5: .test2}'              # замена ключа
ip -j a | jq '.[] | .ifname + " " + .addr_info[].local'
```

Цветной less

    curl example.com/api | jq -C . | less -R

## jc

Парсер вывода разных команд и файлов в формат JSON/YAML

https://kellyjonbrazil.github.io/jc/



## jo

Генератор JSON в Bash


