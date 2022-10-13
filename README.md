# DES

Реализация симметричного блочного алгоритма шифрования DES. В академических целях. Работает как со строками, так и с небольшими (в силу низкой производительности) файлами (буферами).

## Таблицы:

* IP.js — таблица начальной перестановки IP
* E.js — таблица функции расширения E
* IP-1.js — таблица конечной перестановки IP<sup>-1</sup>
* S-Boxes.js — S—блоки
* P.js — P-блоки
* PC-1.js — перестановка битов ключа PC-1
* PC-2.js — перестановка битов ключа PC-2

## Конструктор:

```
constructor(key) {    
    this.key = DES.allocateKey(key)
    this.roundKeys = DES.generateRoundKeys(this.key)
    this.status = ['ALLOCATE KEY']
    this.data = null
    this.blocks = []
}
```

Создание экземпляра класса:

```
const des = new DES(key)
```

Ключ может быть 56-битной строкой или буфером.

При создании экземпляра класса аллоцируются ключ и 16 раундовых ключей статическими методами:

* DES.allocateKey(key)
* DES.generateRoundKeys(this.key)

Экзмепляр класса имеет следующие публичные методы и свойства:

```
des.encrypt(plaintext)  // шифрование
des.decrypt(ciphertext) // расшифрование

des.data                // данные в виде буфера или...
des.dataAsString        // в виде строки
```

Открытый текст и зашифрованный текст могут быть строкой или буфером.

## Пример использования:

В файле *./examples/index.js*:

```
const fs = require('node:fs')       // для шифрования файлов
const DES = require('../index.js')

// Ключ шифрования/расшифрования:
const key = 'SECRETK'

// Создание экземпляра:
const des = new DES(key)

// Шифрование:
const plaintext = 'hello world'
des.encrypt(plaintext)
console.log(des.data)           // buffer
console.log(des.dataAsString)   // string

// Расшифрование:
const ciphertext = des.data
des.decrypt(ciphertext)
console.log(des.data)           // buffer
console.log(des.dataAsString)   // string

// Для шифрования файлов:
const file = fs.readFileSync('./input.txt')
des.encrypt(file)
fs.writeFileSync('output', des.data)

// Для расшифрования файлов:
const file2 = fs.readFileSync('./output')
des.decrypt(file2)
fs.writeFileSync('output2.txt', des.data)
```

## Входные данные примеров:

```
const key = 'SECRETK'
const plaintext = 'hello world'
```

## Статические методы:

### DES.allocateKey(key)

Ключ может быть 56-битной строкой или буфером. В противном случае он будет зациклен или обрезан до 56 бит.

Ключ в виде буфера: 
```
<Buffer 53 45 43 52 45 54 4b>
```

Из функции возвращается ключ в двоичном представлении, дополненный битами чётности. Таким образом 56-битный ключ расширяется до 64 бит.

```
[
    0, 1, 0, 1, 0, 0, 1, = 1,
    1, 0, 1, 0, 0, 0, 1, = 1,
    0, 1, 0, 1, 0, 0, 0, = 0,
    0, 1, 1, 0, 1, 0, 1, = 0, 
    0, 0, 1, 0, 0, 1, 0, = 0, 
    0, 0, 1, 0, 1, 0, 1, = 1,
    0, 1, 0, 1, 0, 0, 0, = 0, 
    1, 0, 0, 1, 0, 1, 1, = 0
]
```
### DES.generateRoundKeys(this.key)

Схема генерации раундовых ключей:

* Permuted Choice 1: 64 bit -> 56 bit
* Взятие левой и правой частей ключа: 56 bit -> 2x 28 bit 
* Инициализация раундовых ключей: 16x 56 bit
* Permuted Choice 2: 16x 48bit

#### PC-1

Из 64-битного ключа отбросить биты чётности (7, 15, 23, 31, 39, 47, 55, 63) и пермешать их, согласно таблице PC-1 в файле *./tables/PC-1.js*. В результате ключ сжимается до 56 бит.

![PC-1](https://upload.wikimedia.org/wikipedia/commons/thumb/3/38/DES-pc1.svg/400px-DES-pc1.svg.png)

Источник изображения: Wikipedia

Пример ключа (*5) после PC-1. В первой строке биты 56, 48, 40, 32, 24, 16, 8, 0 ключа до перестановки:

```
[
    1, 0, 0, 0, 0, 0, 1, 0,
    0, 1, 0, 0, 1, 1, 0, 1,
    0, 0, 1, 1, 1, 0, 1, 0,
    1, 1, 0, 0, 1, 0, 1, 0,
    1, 0, 1, 1, 1, 0, 0, 1,
    0, 0, 0, 0, 0, 0, 1, 0,
    1, 0, 0, 0, 0, 1, 0, 1
]
```

#### Взятие левой и правой частей ключа

56-битный ключ разделяется на две части по 28 бита:

```
C0 = [
    1, 0, 0, 0, 0, 0, 1, 0,
    0, 1, 0, 0, 1, 1, 0, 1,
    0, 0, 1, 1, 1, 0, 1, 0,
    1, 1, 0, 0
]

D0 = [
                1, 0, 1, 0, 
    1, 0, 1, 1, 1, 0, 0, 1,
    0, 0, 0, 0, 0, 0, 1, 0,
    1, 0, 0, 0, 0, 1, 0, 1
]
```

#### Инициализация раундовых ключей

Инициализируетcя 16 раундовых ключей, в первом раунде получаемых из C<sub>0</sub> и D<sub>0</sub>, а в последующих из C<sub>i-1</sub> и D<sub>i-1</sub> (где i — номер раунда) путём циклического сдвига на 1 (1, 2, 9, 16 раунды) или 2 бита влево.

Результат — двумерный массив из 16 раундовых ключей длиной 56 битов.

Пример ключей на первые три раунда, полученных из C0 и D0 циклическим сдвигом:

```
[
    [
        0, 0, 0, 0, 0, 1, 0, 0,
        1, 0, 0, 1, 1, 0, 1, 0, 
        0, 1, 1, 1, 0, 1, 0, 1,
        1, 0, 0, 1, 
                    0, 1, 0, 1, 
        0, 1, 1, 1, 0, 0, 1, 0, 
        0, 0, 0, 0, 0, 1, 0, 1,
        0, 0, 0, 0, 1, 0, 1, 1
    ],
    [
        0, 0, 0, 0, 1, 0, 0, 1,
        0, 0, 1, 1, 0, 1, 0, 0,
        1, 1, 1, 0, 1, 0, 1, 1,
        0, 0, 1, 0, 
                    1, 0, 1, 0,
        1, 1, 1, 0, 0, 1, 0, 0,
        0, 0, 0, 0, 1, 0, 1, 0,
        0, 0, 0, 1, 0, 1, 1, 0
    ],
    [
        0, 0, 1, 0, 0, 1, 0, 0,
        1, 1, 0, 1, 0, 0, 1, 1,
        1, 0, 1, 0, 1, 1, 0, 0,
        1, 0, 0, 0, 
                    1, 0, 1, 1,
        1, 0, 0, 1, 0, 0, 0, 0,
        0, 0, 1, 0, 1, 0, 0, 0,
        0, 1, 0, 1, 1, 0, 1, 0
    ],
    ...
]
```

#### Permuted Choice 2

56-битные раундовые сжимаются до 48 бит (игнорируются биты: 8, 17, 21, 24, 34, 37, 42, 53), согласно таблице PC-2 в файле *./tables/PC-2.js*.

![PC-2](https://upload.wikimedia.org/wikipedia/commons/thumb/5/52/DES-pc2.svg/400px-DES-pc2.svg.png) 

Источник изображения: Wikipedia

Примеры ключей на первые три раунда после применения к ним PC-2. В первой строке — биты 13, 16, 10, 23, 0, 4, 2, 27 ключей до сжатия:

```
[
    [
        0, 0, 0, 1, 0, 0, 0, 1,
        1, 1, 0, 0, 0, 1, 1, 0,
        0, 0, 0, 0, 0, 1, 1, 0,
        0, 0, 0, 0, 0, 1, 1, 0,
        0, 0, 0, 1, 0, 0, 1, 1,
        1, 1, 1, 0, 0, 1, 0, 1
    ],
    [
        1, 1, 1, 1, 0, 1, 0, 0,
        0, 0, 1, 0, 1, 1, 1, 0,
        0, 1, 0, 0, 1, 0, 0, 0,
        0, 1, 1, 0, 1, 1, 0, 0,
        0, 1, 1, 0, 0, 0, 0, 0,
        1, 0, 0, 0, 0, 0, 1, 0
    ],
    [
        0, 1, 0, 0, 0, 0, 1, 0,
        1, 1, 1, 1, 0, 1, 1, 0,
        0, 0, 1, 0, 0, 0, 0, 0,
        0, 1, 1, 0, 0, 1, 0, 0,
        0, 1, 1, 0, 0, 0, 0, 0,
        0, 1, 0, 0, 1, 1, 1, 1
    ],
    ...
]
```

## Публичные методы экземпляра:

### des.encrypt(plaintext)

Метод принимает буфер или строку в качестве аргумента.

Открытый текст в виде буфера:

```
<Buffer 68 65 6c 6c 6f 20 77 6f 72 6c 64>
```

Схема шифрования (см. приватные методы):

* #allocateBlocks()
* #blocksToBinary() — блоки данных, представленные 64-битными буферами представить в виде двоичного массива
* #ip() — начальная перестановка IP бит блоков 
* #getBlocksHalves() — разделить блоки на две части L и R по 32 бита каждая
* #f() — раундовая функция F
* #fp() — конечная перестановка бит блоков IP<sup>-1</sup> (FP)
* #blocksToBuffer() — блоки данных, представленные 64-битными двоичными массивами представить в виде буферов

### des.decrypt(buffer)

Схема расшифрования полностью соответствует схеме расшифрования, но с обратным порядком раундовых поключей!

## Приватные методы экземпляра:

### allocateBlocks()

Открытый текст дополняюется слева нулями до кратного восьми количества байт. 

```
<Buffer 00 00 00 00 00 68 65 6c 6c 6f 20 77 6f 72 6c 64>
```

После чего открытый текст делится на блоки по 64 бита (8 байт):

```
blocks: [
    <Buffer 00 00 00 00 00 68 65 6c>,
    <Buffer 6c 6f 20 77 6f 72 6c 64>
]
```

### blocksToBinary()

Блоки, представленные 64-битными буферами, становятся двоичными массивами:

```
blocks: [
    [
        0, 0, 0, 0, 0, 0, 0, 0, = 0x00
        0, 0, 0, 0, 0, 0, 0, 0, = 0x00
        0, 0, 0, 0, 0, 0, 0, 0, = 0x00
        0, 0, 0, 0, 0, 0, 0, 0, = 0x00
        0, 0, 0, 0, 0, 0, 0, 0, = 0x00
        0, 1, 1, 0, 1, 0, 0, 0, = 0x68
        0, 1, 1, 0, 0, 1, 0, 1, = 0x65
        0, 1, 1, 0, 1, 1, 0, 0  = 0x6c
    ],
    [
        0, 1, 1, 0, 1, 1, 0, 0, = 0x6c
        0, 1, 1, 0, 1, 1, 1, 1, = 0x6f
        0, 0, 1, 0, 0, 0, 0, 0, = 0x20
        0, 1, 1, 1, 0, 1, 1, 1, = 0x77
        0, 1, 1, 0, 1, 1, 1, 1, = 0x6f
        0, 1, 1, 1, 0, 0, 1, 0, = 0x72
        0, 1, 1, 0, 1, 1, 0, 0, = 0x6c
        0, 1, 1, 0, 0, 1, 0, 0  = 0x64
    ]
]
```

### ip()

Реализует начальную перестановку IP бит в блоке, согласно таблице в файле *./tables/IP.js*.

![IP](https://upload.wikimedia.org/wikipedia/commons/thumb/8/8f/DES-ip-1.svg/400px-DES-ip-1.svg.png) 

Источник изображения: Wikipedia

Пример перестановки бит в блоках. В первой строке — биты 57, 49, 41, 33, 25, 17, 9, 1 блока до перестановки:

```
blocks: [
    [
        1, 1, 1, 0, 0, 0, 0, 0, 
        0, 0, 0, 0, 0, 0, 0, 0,
        1, 1, 0, 0, 0, 0, 0, 0,
        0, 1, 0, 0, 0, 0, 0, 0,
        0, 0, 0, 0, 0, 0, 0, 0,
        1, 1, 1, 0, 0, 0, 0, 0,
        1, 0, 1, 0, 0, 0, 0, 0,
        0, 0, 0, 0, 0, 0, 0, 0
    ],
    [
        1, 1, 1, 1, 1, 0, 1, 1, 
        0, 0, 1, 0, 1, 0, 0, 0,
        1, 1, 0, 1, 1, 0, 1, 1,
        0, 0, 0, 1, 1, 0, 1, 0,
        0, 0, 0, 0, 0, 0, 0, 0, 
        1, 1, 1, 1, 1, 1, 1, 1,
        0, 1, 0, 1, 0, 0, 1, 1, 
        0, 0, 1, 1, 1, 0, 1, 0
    ]
]
```

### getBlocksHalves()

Блоки разделяются на две части по 32 бита:

```
blocks: [
    {
        L: [
            1, 1, 1, 0, 0, 0, 0, 0, 
            0, 0, 0, 0, 0, 0, 0, 0,
            1, 1, 0, 0, 0, 0, 0, 0,
            0, 1, 0, 0, 0, 0, 0, 0
        ],
        R: [
            0, 0, 0, 0, 0, 0, 0, 0,
            1, 1, 1, 0, 0, 0, 0, 0,
            1, 0, 1, 0, 0, 0, 0, 0, 
            0, 0, 0, 0, 0, 0, 0, 0
        ]
    },
    {
        L: [
            1, 1, 1, 1, 1, 0, 1, 1, 
            0, 0, 1, 0, 1, 0, 0, 0,
            1, 1, 0, 1, 1, 0, 1, 1, 
            0, 0, 0, 1, 1, 0, 1, 0
        ],
        R: [
            0, 0, 0, 0, 0, 0, 0, 0,
            1, 1, 1, 1, 1, 1, 1, 1,
            0, 1, 0, 1, 0, 0, 1, 1,
            0, 0, 1, 1, 1, 0, 1, 0
        ]
    }
]
```

### f()

Раундовая функция. Выполняется 16 раз.

Cхема одной итерации функции:

![Сеть Фейстеля](https://upload.wikimedia.org/wikipedia/commons/2/2c/%D0%A1%D0%B5%D1%82%D1%8C%D0%A41.PNG)

Источник изображения: Wikipedia

* Запоминается правая часть блока R (32 бита)
* R расширяется, согласно таблице E в файле *./tables/E.js* до 48 бит (часть бит дублируются)
* R складывается по модулю 2 с раундовым ключом (48 бит)
* R разделяется на 8 блоков по 6 бит
* S-блоки: см. ниже (32 бита на выходе) 
* P-блоки: см. ниже (32 бита на выходе)
* L на новый раунд = запомненное значение R из первого пункта 
* R на следующий раунд = R складывается по модулю 2 с L
* Финальная перестановка после 16 раундов

#### Expansion Function (E)

![E](https://upload.wikimedia.org/wikipedia/commons/thumb/4/46/DES-ee.svg/400px-DES-ee.svg.png)

Источник изображения: Wikipedia

Результат расширения правых частей блоков в первом раунде. В первых строках биты 31, 0, 1, 2, 3, 4, 3, 4 правых частей до расширения:

```
blocks: [
    {
        L: [...],
        R: [
            0, 0, 0, 0, 0, 0, 0, 0,
            0, 0, 0, 1, 0, 1, 1, 1,
            0, 0, 0, 0, 0, 0, 0, 1,
            0, 1, 0, 1, 0, 0, 0, 0,
            0, 0, 0, 0, 0, 0, 0, 0,
            0, 0, 0, 0, 0, 0, 0, 0
        ]
    },
    {
        L: [...],
        R: [
            0, 0, 0, 0, 0, 0, 0, 0,
            0, 0, 0, 1, 0, 1, 1, 1,
            1, 1, 1, 1, 1, 1, 1, 0,
            1, 0, 1, 0, 1, 0, 1, 0,
            0, 1, 1, 0, 1, 0, 0, 1,
            1, 1, 1, 1, 0, 1, 0, 0
        ]
    }
]
```

#### R xor RoundKey

48-битная правая часть, побитово складывается по модулю два с соответсвующим 48-битным раундовым ключом.

Результат сложения по модулю 2 правых частей блоков с ключом первого раунда в первом раунде:

```
blocks: [
    {
        L: [...],
        R: [
            0, 0, 0, 1, 0, 0, 0, 1,
            1, 1, 0, 1, 0, 0, 0, 1,
            0, 0, 0, 0, 0, 1, 1, 1,
            0, 1, 0, 1, 0, 1, 1, 0,
            0, 0, 0, 1, 0, 0, 1, 1,
            1, 1, 1, 0, 0, 1, 0, 1
        ]
    },
    {
        L: [...],
        R: [
            0, 0, 0, 1, 0, 0, 0, 1,
            1, 1, 0, 1, 0, 0, 0, 1,
            1, 1, 1, 1, 1, 0, 0, 0,
            1, 0, 1, 0, 1, 1, 0, 0,
            0, 1, 1, 1, 1, 0, 1, 0,
            0, 0, 0, 1, 0, 0, 0, 1
        ]
    }
]
```

#### Деление R на 8 блоков по 6 бит

Результат деления R на 8 блоков по 6 бит:

```
[
    [ 0, 0, 0, 1, 0, 0 ],
    [ 0, 1, 1, 1, 0, 1 ],
    [ 0, 0, 0, 1, 0, 0 ],
    [ 0, 0, 0, 1, 1, 1 ],
    [ 0, 1, 0, 1, 0, 1 ],
    [ 1, 0, 0, 0, 0, 1 ],
    [ 0, 0, 1, 1, 1, 1 ],
    [ 1, 0, 0, 1, 0, 1 ]
]
[
    [ 0, 0, 0, 1, 0, 0 ],
    [ 0, 1, 1, 1, 0, 1 ],
    [ 0, 0, 0, 1, 1, 1 ],
    [ 1, 1, 1, 0, 0, 0 ],
    [ 1, 0, 1, 0, 1, 1 ],
    [ 0, 0, 0, 1, 1, 1 ],
    [ 1, 0, 1, 0, 0, 0 ],
    [ 0, 1, 0, 0, 0, 1 ]
]
```

#### S-блоки

S-блоки (блоки замены). Каждая из восьми 6-битных частей R поступает на вход одного из восьми блоков замены, определённых в файле *./tables/S-Boxes.js*. Первый и последний биты части блока R определяют номер строки (0-3), а оставшиеся четыре бита номер столбца (0-15) в блоке замены. На выходе из S-блока 6-битная последовательность сжимается до 4 бит. Таким образом 48-битная правая часть блока R сжимается до 32 бит. Иллюстрация:

![S](/docs//S-box.png)

Источник изображения: ResearchGate.net

![S](https://upload.wikimedia.org/wikipedia/commons/thumb/a/a3/DES-f-function.png/623px-DES-f-function.png)

Источник изображения: Wikipedia

То есть:

```
[
    [ 0, 0, 0, 1, 0, 0 ], = 0 строка, 2 столбец в 1 S-блоке = 13 = 1101
    [ 0, 1, 1, 1, 0, 1 ], = 1 строка, 14 столбец во 2 S-блоке = 11 = 1011
    [ 0, 0, 0, 1, 0, 0 ], = 0 строка, 2 столбец в 3 S-блоке = 9 = 1001
    [ 0, 0, 0, 1, 1, 1 ], = 1 строка, 3 столбец в 4 S-блоке = 5 = 0101
    [ 0, 1, 0, 1, 0, 1 ], = 1 строка, 10 столбец в 5 S-блоке = 15 = 111
    [ 1, 0, 0, 0, 0, 1 ], = 3 строка, 0 столбец в 6 S-блоке = 4 = 0100
    [ 0, 0, 1, 1, 1, 1 ], = 1 строка, 7 столбец в 7 S-блоке = 10 = 1010
    [ 1, 0, 0, 1, 0, 1 ]  = 3 строка, 2 столбец в 8 S-блоке = 14 = 1110
]
[
    [ 0, 0, 0, 1, 0, 0 ], =  0 строка, 2 столбец в 1 S-блоке
    [ 0, 1, 1, 1, 0, 1 ], =  1 строка, 14 столбец во 2 S-блоке
    [ 0, 0, 0, 1, 1, 1 ], =  1 строка, 3 столбец в 3 S-блоке
    [ 1, 1, 1, 0, 0, 0 ], =  2 строка, 12 столбец в 4 S-блоке
    [ 1, 0, 1, 0, 1, 1 ], =  3 строка, 5 столбец в 5 S-блоке
    [ 0, 0, 0, 1, 1, 1 ], =  1 строка, 3 столбец в 6 S-блоке
    [ 1, 0, 1, 0, 0, 0 ], =  2 строка, 4 столбец в 7 S-блоке
    [ 0, 1, 0, 0, 0, 1 ]  =  1 строка, 8 столбец в 8 S-блоке
]
```

Правые части после замены в первом раунде:

```
[
    1, 1, 0, 1,     1, 0, 1, 1,
    1, 0, 0, 1,     0, 1, 0, 1, 
    1, 1, 1, 1,     1, 0, 0, 1, 
    1, 0, 1, 0,     0, 1, 0, 0
]
[
    1, 1, 0, 1,     1, 0, 1, 1, 
    1, 0, 0, 1,     0, 0, 0, 1,
    1, 1, 0, 1,     0, 0, 1, 0,
    0, 1, 0, 0,     1, 1, 0, 0
]
```

#### P-блоки

P-блок (блок перестановки). Правая часть R поступает на вход блока перестановки, определённого в файле *./tables/P.js*.

![P](https://upload.wikimedia.org/wikipedia/commons/thumb/c/c4/DES-pp.svg/1920px-DES-pp.svg.png)

Источник изображения: Wikipedia

Правые части после перестановки в первом раунде. В первых строках биты 15, 6, 19, 20, 28, 11, 27, 16 правых частей до расширения:

```
[
    1, 1, 1, 1, 0, 1, 0, 1, 
    1, 0, 0, 0, 1, 1, 0, 0, 
    1, 1, 1, 1, 0, 1, 0, 1,
    1, 0, 1, 0, 0, 0, 1, 1
]
[
    1, 1, 1, 0, 1, 1, 0, 1,
    1, 0, 1, 1, 1, 1, 0, 0,
    1, 1, 0, 0, 0, 0, 0, 1,
    0, 0, 1, 0, 0, 0, 1, 0
]
```

#### Блок на следующий раунд

Получаем новую правую часть R на следующий раунд, путём сложения по модулю 2 бит левой части L и правой части R. 

Новая левая часть L на следующий раунд равна запомненному значению R в начале работы функции (см. схему).

```
blocks: [
    {
        L: [
            0, 0, 0, 0, 0, 0, 0, 0,
            1, 1, 1, 0, 0, 0, 0, 0,
            1, 0, 1, 0, 0, 0, 0, 0,
            0, 0, 0, 0, 0, 0, 0, 0
        ],
        R: [
            0, 0, 0, 1, 0, 1, 0, 1,
            1, 0, 0, 0, 1, 1, 0, 0,
            0, 0, 1, 1, 0, 1, 0, 1,
            1, 1, 1, 0, 0, 0, 1, 1
        ]
    },
    {
        L: [
            0, 0, 0, 0, 0, 0, 0, 0,
            1, 1, 1, 1, 1, 1, 1, 1,
            0, 1, 0, 1, 0, 0, 1, 1, 
            0, 0, 1, 1, 1, 0, 1, 0
        ],
        R: [
            0, 0, 0, 1, 0, 1, 1, 0, 
            1, 0, 0, 1, 0, 1, 0, 0, 
            0, 0, 0, 1, 1, 0, 1, 0,
            0, 0, 1, 1, 1, 0, 0, 0
        ]
    }
  ]
```

#### Финальная перестановка

После 16 раундов полученные части склеиваются.

Результат

```
blocks: [
    [
        1, 0, 1, 0, 0, 0, 1, 1, 
        1, 1, 1, 1, 1, 0, 0, 1,
        1, 0, 0, 0, 1, 0, 0, 1,
        1, 1, 0, 1, 1, 0, 0, 1,
        0, 0, 1, 0, 1, 0, 0, 0,
        1, 1, 0, 1, 1, 0, 1, 1,
        0, 0, 0, 1, 1, 1, 0, 0, 
        0, 0, 0, 1, 1, 0, 1, 0
    ],
    [
        1, 0, 0, 0, 0, 1, 1, 0, 
        1, 1, 1, 1, 0, 0, 0, 0, 
        1, 0, 0, 1, 1, 0, 1, 1,
        1, 1, 1, 1, 1, 1, 1, 0, 
        1, 0, 1, 0, 1, 0, 1, 1,
        1, 0, 1, 1, 1, 1, 1, 1,
        0, 1, 1, 0, 0, 1, 1, 0,
        0, 0, 0, 1, 1, 0, 0, 0
    ]
]
```

### fp()

Конечная перестановка бит блоков IP<sup>-1</sup>

![IP-1](https://upload.wikimedia.org/wikipedia/commons/thumb/6/65/Permutation_initiale.svg/1024px-Permutation_initiale.svg.png)

Результат перестановки. В первых строках биты 39, 7, 47, 15, 55, 23, 63, 31 блоков до перестановки:

```
blocks: [
    [
        0, 1, 1, 1, 0, 1, 0, 1, = 0x75
        0, 1, 1, 0, 0, 0, 1, 0, = 0x62
        0, 0, 0, 0, 1, 0, 0, 0, = 0x08
        1, 0, 1, 1, 1, 1, 1, 1, = 0xbf
        0, 0, 1, 1, 1, 0, 1, 1, = 0x3b
        1, 1, 0, 1, 0, 0, 0, 0, = 0xd0
        0, 0, 1, 1, 0, 0, 0, 1, = 0x31
        0, 1, 1, 1, 0, 1, 0, 1  = 0x75
    ],
    [
        1, 0, 1, 0, 0, 1, 0, 0, = 0xa4
        1, 1, 1, 0, 1, 1, 0, 1, = 0xed
        0, 1, 1, 0, 1, 0, 0, 1, = 0x69
        1, 0, 1, 0, 0, 1, 1, 1, = 0xa7
        0, 0, 1, 1, 0, 1, 1, 1, = 0x37
        1, 0, 1, 1, 1, 0, 0, 1, = 0xb9
        0, 0, 0, 1, 1, 0, 0, 1, = 0x19
        1, 1, 1, 1, 0, 1, 0, 1  = 0xf5
    ]
]
```

### blocksToBuffer()

Представление блоков в виде буфера:

```
<Buffer 75 62 08 bf 3b d0 31 75 a4 ed 69 a7 37 b9 19 f5>
```

В виде строки:

```
u�;�1u��i�7�
```