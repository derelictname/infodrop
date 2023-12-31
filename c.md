# Си

```c
#include <stdio.h>
int main(int argc, char **argv, char **envp)
{
        for (int i = 0; envp[i] != NULL; i++) {
                printf("we know %s\n", envp[i]);
        }
        for (int i = 1; i < argc; i++) {
                printf("spice %d = %s\n", i, argv[i]);
        }
}
```

Сохраняем в `prog1.c` и собираем программу

    gcc -v -fverbose-asm -O0 --save-temps -fno-asynchronous-unwind-tables prog1.c -o prog1

Запуск

    ./prog1 pepper salt cinnamon

Шаги сборки

1. Препроцессор выполняет макрокоманды `#include`, `#define` и прочие для изменения исходного кода. Макрокоманды выполняют вставку других файлов, подмену текста, условное использование участка кода.
2. Создание кода на ассемблере.
3. Перевод ассемблера в ELF файл перемещаемого типа с машинными командами (кодами операций).
4. Различные действия с ELF: добавление других ELF перемещаемого типа, замена меток на адреса, назначение динамического компоновщика и зависимостей от библиотек (ELF совместно используемого типа) и ассемблерных меток из них.

В начало программы, например, добавляется [C Run Time](https://en.wikipedia.org/wiki/Crt0) для разных подготовок, назначается динамический компоновщик и добавляется динамическая связь с libc. На выходе получаем ELF исполняемого, перемещаемого или совместно используемого типа.

Все эти действия по отдельности

    cpp prog1.c > prog1.i
    gcc -S prog1.i -o prog1.s
    as prog1.s -o prog1.o
    ld prog1.o /usr/lib/x86_64-linux-gnu/crt1.o -lc --dynamic-linker=/lib64/ld-linux-x86-64.so.2 -o prog1

При наличии статического варианта библиотек возможно добавление зависимостей внутрь программы

    gcc -static file1.c -o file1

## Команды препроцессора

Команды, выполняемые отдельной программой - препроцессором Си, которая производит изменения кода программы.

```c
#include "myprogram.h" // вставка файла из текущей папки
#include <stdio.h> // вставка файла из из системного пути

#define NUMBER int // NUMBER будет заменён на int
#define COUNT 300
#define TEST
#define ADD_10(a) a + 10 // подстановка с параметрами
#define FUN1(a, b) (a) * (b) + (a)
#define FUN2(a, b)               \
        {                        \
                printf("%s", a); \
                printf("%s", b); \
        }

int main(void)
{
        NUMBER a;
        a = FUN1(1 + 2, 3); // заменится на (1 + 2) * (3) + (1 + 2)

#ifdef TEST // условное использование кода
        a = 1;
#endif
}
```

`#include` обычно используется для вставки файлов с общими для всей программы макросами, обьявлениями функций, структурами и перечислениями (примеры в `/usr/include/linux/`). Эти файлы называют **заголовочными**. Их имя оканчивается на `.h`, а не на `.c`, как у обычного кода.

## Типы переменных

**Переменная** - слово, связанное с каким-либо изменяемым значением. В готовой программе это может быть значением с меткой в области памяти text или data, либо значением в области памяти "стек" относительно той функции, в которой объявлены.

```c
unsigned char var1; // инициализация переменной с длиной в 1 байт

char var2;
/* Без явного указания все типы будут sighed, у которых первый бит
используется под знак + или - для обозначения отрицательности значения.
Поэтому в unsigned char диапазон возможных значений от 0 до 256,
а в signed char от -127 до 128. */

// TODO
unsigned long long int var3 = 18446744073709551615;
float var4;
double var5;
long double var6;
```

**Указатель** - переменная, которая содержит адрес ячейки в памяти (например, адрес другой переменной или функции). При создании указателя используется тип, чтобы компилятор знал размер значения, когда происходит обращение к значению по адресу в указателе. Указатели часто используют для работы с одними данными в разных функциях. Также при адресной арифметике - смещение адреса для получения значения в соседней ячейке.

```c
int *ptr1;
int val1 = 10;
ptr1 = &val1; // сохраняем в указатель адрес переменной
void fun1(int *someptr) // функция принимает указатель
{
        *someptr = 100; // изменяем переменную по указателю
}
fun1(ptr1);
printf("address: %p, value: %d\n", ptr1, *ptr1);

void *anything; // указатель void не ограничен размером данных

struct twovals {
        int x;
        int y;
};
struct twovals struct1;
struct twovals *ptr2 = &struct1;
ptr2->x = 200; // обращение к полям указателя на структуру
```

**Массив** - инициализатор указателя на первый элемент последовательно расположенных данных. Обращение к элементам вычисляет смещение адреса относительно начала массива. На этапе сборки известен размер массива с учётом типа данных.

```c
int array1[] = {1, 2, 3};
printf("%p %p %d\n", array1, &array1[0], *array1);
```

**Строка** - указатель на первый байт последовательности байт и нулевым байтом в конце. Создаётся через указатель или массив (который всё равно будет указателем).

```c
char *text1 = "some text"; // помещается в область памяти text (нельзя изменять)
char text2[] = "some text"; // помещается в область памяти data
char text3[] = {'s', 'o', 'm', 'e', ' ', 't', 'e', 'x', 't', '\0'};
text1[1] = 'a'; // ошибка сегментирования
text2[1] = 'b';
```

**Перечисление** - набор именованных целочисленных констант. Используется для определения возможных значений какой-либо переменной (компилятор замечает попытку присвоения из другого перечисления), либо для удобного определения нескольких констант подряд.

```c
enum tea { WHITE, YELLOW, GREEN }; // 0, 1, 2
enum tea tea1 = GREEN; // 2
enum hexword { DEAD = 0xdead, FOOD = 0xf00d };
enum hexword myhexword = FOOD;
printf("tea %d is my %x\n", tea1, myhexword);
enum { ONE, TWO, THREE }; // анонимная инициализация
enum { FALSE, TRUE } enabled; // анонимная инициализация с объявлением переменной
```

**Структура** - составная переменная, хранящая в себе несколько расположенных рядом переменных. Может происходить выравнивание их расположения для оптимизации под конкретную архитектуру, но некоторые компиляторы позволяют это отключить, что означает упаковку.

```c
struct snake {
        int x;
        int y;
        char dir;
        unsigned int len;
};
struct snake snake1 = { 50, 50, 'w', 3 }; // C89-style
struct snake snake2 = { .dir = 's', .len 4, .x 50, .y 55 };
snake1.len++;
printf("Current snake size = %d\n", snake1.len);

// Анонимная инициализация и с объявлением переменной
struct {
        int var1;
        int var2;
} object1; 
```

**Объединение** - использование одного и того же участка памяти в виде разных типов переменных. Размер участка определяется по наибольшему типу. Используется в основном для переносимости программы на разное оборудование, чтобы значение гарантированно помещалось в переменную.

```c
union my_cool_union {
        long a;
        int b;
};
union my_cool_union c;
c.b = 123;
printf("%ld\n", c.a);
```

**Структура с битовыми полями** - структура со полями произвольной длины в битах. Размер и расположение данных в памяти будет зависеть от типа полей и порядка их расположения. Например, несколько полей может уместиться в первом из них, если их длина в сумме меньше длины типа данных.

```c
// Для удобства объединим структуру с битовыми полями и unsigned char
union {
        struct {
                unsigned char field1 : 2;
                unsigned char field2 : 2;
                unsigned char field3 : 4;
                // Вся структура займёт 1 байт,
                // так как все поля умещаются в первый unsigned char
        } struct1;
        unsigned char char1;
} union1;

int main(void)
{
        union1.struct1.field1 = 0b00;
        union1.struct1.field2 = 0b11;
        union1.struct1.field3 = 0b1011;

        // Печатаем на экран значение структуры в виде 0 и 1 (длина unsigned char)
        for (int i = sizeof(union1.char1) * 8 - 1; i >= 0; i--)
                printf("%d", (union1.char1 & (1 << i)) >> i);
        printf("\n");
}
```

**Синоним типов** - создание нового имени для уже существующего типа данных. Используется для удобства и читаемости. Часто используется вместе с анонимной инициализацией типа данных.

```c
typedef int number;
number n1 = 123;

// Определение типа для анонимного перечисления
typedef enum { WOOD, METAL, STONE } block_type;
block_type door = STONE;

// Определение типа для анонимной структуры
typedef struct {
        int x;
        int y;
        int size;
        int shape;
} primitive;

primitive square1 = { 0, 0, 10, 4 };
printf("primitive size is %d\n", square1.size);
```

**Преобразование типа** - изменение типа переменной. Например, обрезать лишнюю длину в переменной. При действиях над разными типами переменных может происходить неявное преобразование к одному из типов по определённым правилам. Желательно делать его явно для ясности кода.

```c
int a = 15;
double b = 3.1, c;
c = a / b;          // неявно
c = (double)a / b;  // явно
printf("%d\n", (int)1.9);
```

## Спецификаторы

**Статическая переменная** - переменная с явным расположением в программе. Это делается с помощью метки в ассемблере. Хранится в течении всей работы программы, доступна в пределах объектного файла, а каждое обращение происходит в один и тот же участок памяти (поэтому за этими обращениями нужно особенно следить, либо стараться не использовать статические переменные).

```c
void f(void)
{
        static int n = 0;
        n++;
        printf("%d\n", n);
}
int main(void)
{
        f(); // 1
        f(); // 2
}
```

extern

inline

register

## Функция

Функции становятся подпрограммами в ассемблере с использованием стека для хранения переданных значений и локальных переменных.

Если вызвать функцию до её **определения** в коде, то компилятор выполнит **неявное определение**, что нежелательно по ряду причин. Поэтому необходимо делать заранее (например, в заголовочных файлах) хотя бы **объявление** функции.

```c
// Объявление функции
void fun1(void);
void fun2(void);

int main(void)
{
        fun1(); // переход к функции, хотя её определение ниже
}

// Определение функции
void fun1(void)
{
        int a = 10; // локальные переменные хранятся в стеке
        printf("fun1 executed\n");
        fun2();
}

void fun2(void)
{
        printf("fun2 executed\n");
        fun1();
}
```

Сборка с оптимизацией

    gcc prog1.c -O2 -o prog1

Взаимный вызов функций приведёт к бесконечной рекурсии. Так как вызов функции сохраняет информацию в стековую область памяти, то она в итоге переполнится. Если вызовы стоят в конце, то оптимизация сможет исправить это, заменив переход в подпрограмму `call` на переход к команде `jne`, который не использует стек.

**Переменное число аргументов**

```c
#include <stdarg.h>
#include <stdio.h>

void show(int, ...);

int main(void)
{
        show(3, 321, 54, 7000);
}

void show(int count, ...)
{
        va_list ap;
        va_start(ap, count);
        for (int i = 1; i <= count; i++) {
                printf("arg passed %d\n", va_arg(ap, int));
        }
        va_end(ap);
}
```

**Указатель на функцию** - адрес ячейки в памяти с началом функции.

```c
#include <stdio.h>

void eating_apples(int count)
{
        printf("you ate %d apples\n", count);
}

void eating_potatoes(int count)
{
        printf("you ate %d potatoes\n", count);
}

void eat(void (*eating_function)(int), int count)
{
        eating_function(count);
        printf("dont forget to wash plates!\n");
}

int main(void)
{
        void (*fun1)(int);
        fun1 = eating_apples; // или так fun1 = &eating_apples
        fun1(33); // или так (*fun1)(33)

        eat(fun1, 3); // передача ссылки на функцию в другую функцию
        eat(eating_potatoes, 5);
}
```

## Компиляторы


- [GCC](https://en.wikipedia.org/wiki/GNU_Compiler_Collection) - является фронт-ендом сразу к нескольким языкам, преобразует код в AST, мидл-енд выполняет оптимизации и статический анализ, бэк-енд создаёт машинный код
- clang - компилятор и фронт-енд к языкам семейства Си, создаёт промежуточный машинный код для LLVM, который может выполнять оптимизации во время выполнения



## Без динамического компоновщика

```c
void _start(void)
{
        printf("hello\n");

        asm("mov $1, %rax\n"
            "mov $11, %rbx\n"
            "int $0x80");
}
```

Сборка

    gcc -Wl,--no-dynamic-linker -nostartfiles prog1.c -o prog1

Опции

- `-Wl,--no-dynamic-linker` - не указывать динамический компоновщик в ELF файле
- `-nostartfiles` - не добавлять C Run Time, потому что в нём метка по умолчанию `_start` и она вызывает `main` в конце, а не завершает программу системным вызовом


## Стили кодирования

- [Linux](https://www.kernel.org/doc/html/latest/process/coding-style.html)
- [GNU](https://www.gnu.org/prep/standards/standards.html)


## Ссылки

- https://ru.wikipedia.org/wiki/Компилятор
- https://ru.wikipedia.org/wiki/Компоновщик
- https://ru.wikipedia.org/wiki/Исполняемый_файл
- https://ru.wikipedia.org/wiki/Executable_and_Linkable_Format
