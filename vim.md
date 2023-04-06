# vim

- `u` - отмена
- `Ctrl + r` - повтор
- `y` - копировать
- `d` - вырезать
- `p` - вставить после
- `P` - вставить перед
- `c` - стереть
- `>>` - увеличить отступ
- `<<` - уменьшить отступ
- `.` - повторить действия в последнем Editing mode
- `hjkl` - влево, вниз, вверх, вправо

Визуальный режим используется для выделения текста и последующих операций над ним

- `v` - символьное выделение
- `Shift + v` - строчное выделение
- `Ctrl + v` - блочное выделение

Закомментировать несколько строк

    Ctrl + v
    j
    Shift + i
    #
    ESC
    ESC

Раскомментировать несколько строк

    Ctrl + v
    j
    x

Автоматически проставить все отступы

    gg=G

Стереть всё

    :%d

Сохранить файл и выполнить команду

    :w | !gcc test1.c -o test1 && ./test1

Автозапуск команды при сохранении

    :autocmd BufWritePost <buffer> make

Запуск vim из stdin с подсветкой

    postconf -n | vim -c ':set syntax=pfmain' -

Убрать переносы `^M`

    :g/Ctrl+v+m/d

Заменить текущие табы на пробелы

    :%s/\t/    /g

## vimrc

```
set tabstop=4      " ширина табуляции
set softtabstop=4  " ширина отступа при нажатии TAB
set shiftwidth=0   " ширина отступа при << или >>

" Если softtabstop=2 или shiftwidth=2 при tabstop=4, то три отступа вставят 6 пробелов, но каждые 4 пробела заменятся на табуляцию

set expandtab      " заменять табуляцию на пробелы
syntax on          " включить подсветку синтаксиса
colorscheme slate  " цветовая схема

" Настройки для определённого формата
autocmd FileType yaml setlocal ts=2 sts=2

" Настройка определения формата
autocmd BufRead,BufNewFile *.nginx set ft=nginx
autocmd BufRead,BufNewFile */etc/nginx/* set ft=nginx
autocmd BufRead,BufNewFile nginx.conf set ft=nginx
```

Настройки также можно включать непосредственно в файл. Называется modeline. Проверяется пять первых и пять последних строк.

Формат

    [other chars]<spaces>vim:<spaces>settings

Пример

    # vim: syntax=apache tabstop=4 softtabstop=4

Улучшенные правила подсветки Nginx

```
mkdir -p ~/.vim/syntax
wget https://raw.githubusercontent.com/nginx/nginx/master/contrib/vim/syntax/nginx.vim -O ~/.vim/syntax/nginx.vim
```

Файловый менеджер

    :E

## Вкладки

    :tabedit file1

- gt
- gT

## Сессии

    :mksession session1
    vim -S session1
    :mks!


## Ссылки

- http://www.viemu.com/a-why-vi-vim.html


