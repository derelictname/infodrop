- `0b11111111`, `0xff` - один байт, восемь бит или символ в ASCII
- `0xffff` -  два байта, 18 бит или символ в Unicode
- `0xffffffff` - 4 байта или 32-битный адрес
- `0xffffffffffffffff` - 8 байт или 64-битный адрес

Символ в двухбайтовой кодировке

    ф = d184 = 11010001 10000100 = 2 байта

Особенность float - чисел с плавающей точкой (в том числе double - двойной точности)

    0.1 + 0.2 = 0.30000000000000004



- `x / 2^20` - байты в мегабайты
- `x / 2^30` - байты в гигабайты
- `x * 2^10` - гигабайты в мегабайты

Скорость передачи данных

    ( X * 8 ) / ( Y * 60 ) = Z

Обозначение

- X - объём данных, Гбайт
- Y - полоса пропускания, Гбит
- Z - время передачи, минут

Сокращение

- Mbps - megabits per second
- MBps - megabytes per second
