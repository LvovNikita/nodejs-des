# DES

Реализация блочного алгоритма шифрования DES. В академических целях. В разработке

# Todo

* Получать 56-битный ключ и расширять его до 64 битами чётности

---

## Таблицы:

* IP.js — таблица начальной перестановки IP
* E.js — таблица функции расширения E
* IP-1.js — таблица конечной перестановки IP<sup>-1</sup>
* S-Boxes.js — S—блоки
* P.js — P-блоки
* PC-1.js — перестановка битов ключа PC-1
* PC-2.js — перестановка битов ключа PC-2

---

## static DES.getBlocks(file_path)

Статический метод. Принимает в качестве аргумента путь к файлу (открытому тексту). Возвращает Promise (массив блоков открытого текста, представленных как 8-байтовые Buffer'ы).

Пример блока (фрагмент строки "hello wo"):

```
block: Buffer(8) [Uint8Array] [104, 101, 108, 108, 111, 32, 119, 111] // (*1)
```

---

## new DES(block, key)

Конструктор. Принимает в качестве аргументов 64-битный блок (Buffer) и 64-битный ключ (строка или Buffer'а). В случае, если длина ключа не равна 64 битам, он обрезается или, наоброт, дополняется (циклически) до 64 бит (*см. DES.allocateKey(key)*). 

Пример ключа (строка "KEY", дополненная до "KEYKEYKE"):

```
key: Buffer(8) [Uint8Array] [75, 69, 89, 75, 69, 89, 75, 69] // (*4)
```

Устанавливает в качестве статуса состояния (доступен через свойство .status экземпляра класса): **INIT & ALLOCATE_KEY**.

---

## Схема шифрования

Для простоты отладки методы объединены в цепочку:

```
encrypt() = this
    .#byteBlockToBinary()
    .#ip()
    .#getBlockHalves()
    .#generateRoundKeys()
    .#f()
    .#fp()
    .#binaryBlockToBytes()
```

* .byteBlockToBinary() — блок данных, представляющий из себя 8-байтовый Buffer представить в виде двоичного массива;
* .ip() — начальная перестановка бит блока IP;
* .getBlockHalves() — разделить блок на две части L и R по 32 бита каждая;
* .generateRoundKeys() — сгенерировать раундовые подключи;
* .f() — раундовая функция F
* .fp() — конечная перестановка бит блока IP-1 (FP)
* .binaryBlockToBytes() — ...

---

## Схема генерации раундовых ключей

Для простоты отладки методы объединены в цепочку:

```
generateRoundKeys = this
    .#byteKeyToBinary()
    .#pc1()
    .#getKeyHalves()
    .#initRoundKeys()
    .#pc2()
```

* .byteKeyToBinary() — ...
* .pc1() — ...
* .getKeyHalves() — ...
* .initRoundKeys() — ...
* .pc2() — ...

---

## Методы класса DES:

### private DES.byteBlockToBinary()

Приватный метод. Преобразует 64-битный блок открытого текста из Buffer'а в двоичный массив нулей и единиц. 

Пример преобразования блока (*1):

```
block: [
    // "h" = 104 = 0b01101000
    '0', '1', '1', '0', '1', '0', '0', '0',
    // "e" = 101 = 0b01100101
    '0', '1', '1', '0', '0', '1', '0', '1',
    // ... 
    '0', '1', '1', '0', '1', '1', '0', '0', 
    '0', '1', '1', '0', '1', '1', '0', '0',
    '0', '1', '1', '0', '1', '1', '1', '1',
    '0', '0', '1', '0', '0', '0', '0', '0', 
    '0', '1', '1', '1', '0', '1', '1', '1', 
    '0', '1', '1', '0', '1', '1', '1', '1'
] // (*2)
```

Устанавливает в качестве статуса состояния: **CONVERT_BYTE_BLOCK_TO_BINARY**.

### private DES.ip()

Приватный метод. Реализует начальную перестановку бит в блоке IP, согласно таблице в файле *./tables/IP.js*. 57-ой бит становится нулевым, 49-ый — первым и т. д.

![IP](https://upload.wikimedia.org/wikipedia/commons/thumb/8/8f/DES-ip-1.svg/400px-DES-ip-1.svg.png) 
Источник изображения: Wikipedia

Пример перестановки бит блока (*2):

```
block: [
    // 57, 49, 41, 33, 25, 17,  9, 1, — номера бит из (*2) 
    '1', '1', '0', '1', '1', '1', '1', '1',
    // ...
    '0', '1', '0', '0', '0', '0', '0', '0',
    '1', '1', '0', '1', '1', '1', '1', '0',
    '1', '1', '0', '1', '0', '0', '1', '0',
    '0', '0', '0', '0', '0', '0', '0', '0',
    '1', '1', '1', '1', '1', '1', '1', '1',
    '1', '0', '0', '1', '1', '1', '0', '1',
    '1', '1', '0', '1', '0', '0', '0', '0'
] // (*3)
```

Устанавливает в качестве статуса состояния: **INITIAL_PERMUTATION**.

### private DES.getBlockHalves()

Приватный метод. Позволяет получить левую L и правую R части блока (32 бита каждая).

Пример левой части блока (*3):

```
L: [
    '1', '1', '0', '1', '1', '1', '1', '1', 
    '0', '1', '0', '0', '0', '0', '0', '0', 
    '1', '1', '0', '1', '1', '1', '1', '0', 
    '1', '1', '0', '1', '0', '0', '1', '0'
]
```

Пример правой части блока (*3):

```
R: [
    '0', '0', '0', '0', '0', '0', '0', '0', 
    '1', '1', '1', '1', '1', '1', '1', '1', 
    '1', '0', '0', '1', '1', '1', '0', '1',
    '1', '1', '0', '1', '0', '0', '0', '0'
]
```

Устанавливает в качестве статуса состояния: **GET_BLOCK_HALVES_L_R**.

### private DES.byteKeyToBinary()

Приватный метод. Преобразует 64-битный ключ из Buffer'а в двоичный массив нулей и единиц. 

Пример преобразования ключа (*4):

```
key: [
    // 75 = 0b01001011
    '0', '1', '0', '0', '1', '0', '1', '1', 
    // 69 = 0b01000101
    '0', '1', '0', '0', '0', '1', '0', '1', 
    // ...
    '0', '1', '0', '1', '1', '0', '0', '1',
    '0', '1', '0', '0', '1', '0', '1', '1',
    '0', '1', '0', '0', '0', '1', '0', '1',
    '0', '1', '0', '1', '1', '0', '0', '1',
    '0', '1', '0', '0', '1', '0', '1', '1',
    '0', '1', '0', '0', '0', '1', '0', '1'
] // (*5)
```

Устанавливает в качестве статуса состояния: **KEYS: CONVERT_BYTE_KEY_TO_BINARY**.

### private DES.pc1()

Приватный метод. Из 64-битного ключа отбросить биты чётности (7, 15, 23, 31, 39, 47, 55, 63) и пермешать их, согласно таблице PC-1 в файле *./tables/PC-1.js*. В результате ключ сжимается до 56 бит.

![PC-1](https://upload.wikimedia.org/wikipedia/commons/thumb/3/38/DES-pc1.svg/400px-DES-pc1.svg.png)
Источник изображения: Wikipedia

Пример ключа (*5) после PC-1:

```
key: [
    '0', '0', '0', '0', '0', '0', '0', '0',
    '1', '1', '1', '1', '1', '1', '1', '1',
    '0', '0', '0', '0', '0', '0', '0', '0',
    '0', '0', '1', '0', '0', '1', '0', '0',
    '1', '0', '0', '1', '1', '0', '0', '1',
    '0', '0', '1', '0', '0', '1', '1', '0',
    '1', '1', '0', '1', '0', '1', '0', '0'
] // (*6)
```
Устанавливает в качестве статуса состояния: **KEYS: PC-1_KEY_BITS_PERMUTATION**.

### private DES.getKeyHalves()

Приватный метод. Получить 28-битные части ключа C<sub>0</sub> и D<sub>0</sub>.

Пример левой части ключа C<sub>0</sub> (*6):

```
Ci[0]: [
    '0', '0', '0', '0', '0', '0', '0', '0', 
    '1', '1', '1', '1', '1', '1', '1', '1',
    '0', '0', '0', '0', '0', '0', '0', '0',
    '0', '0', '1', '0'
] // (*7)
```

Пример правой части ключа D<sub>0</sub> (*6):

```
Di[0]: [
                        '0', '1', '0', '0', 
    '1', '0', '0', '1', '1', '0', '0', '1',
    '0', '0', '1', '0', '0', '1', '1', '0',
    '1', '1', '0', '1', '0', '1', '0', '0'
] // (*7)
```

Устанавливает в качестве статуса состояния: **KEYS: GET_KEY_HALVES_C0_D0**.

### private DES.initRoundKeys()

Приватный метод. Инициализирует 16 раундовых ключей, в первом раунде получаемых из C<sub>0</sub> и D<sub>0</sub>, а в последующих из C<sub>i-1</sub> и C<sub>i-1</sub> (где i — номер раунда) путём циклического сдвига на 1 (1, 2, 9, 16 раунды) или 2 бита влево.

Результат — двумерный массив из 16 раундовых ключей длиной 56 битов.

Пример ключа на первый раунд, полученного из C0 и D0 (*7) циклическим сдвигом на 1 бит:

```
roundKeys[0]: [
    // C0 после циклического сдига влево на 1 бит
    '0', '0', '0', '0', '0', '0', '0', '1',
    '1', '1', '1', '1', '1', '1', '1', '0',
    '0', '0', '0', '0', '0', '0', '0', '0',
    '0', '1', '0', '0',
    // D0 после циклического сдига влево на 1 бит
                        '1', '0', '0', '1',
    '0', '0', '1', '1', '0', '0', '1', '0',
    '0', '1', '0', '0', '1', '1', '0', '1',
    '1', '0', '1', '0', '1', '0', '0', '0'
] // (*8)
```

Устанавливает в качестве статуса состояния: **KEYS: INIT_ROUND_KEYS**.

### private DES.pc2()

Приватный метод. Сжимает 56-битные раундовые ключи до 48 бит (игнорируются биты: 8, 17, 21, 24, 34, 37, 42, 53) и перемешивает биты, согласно таблице PC-2 в файле *./tables/PC-2.js*.

![PC-2](https://upload.wikimedia.org/wikipedia/commons/thumb/5/52/DES-pc2.svg/400px-DES-pc2.svg.png) 
Источник изображения: Wikipedia

Пример ключа на первый раунд после применения к нему (*8) PC-2:

```
roundKeys[0]: [
    // 13, 16, 10, 23,  0,  4,  2, 27 — номера бит из (*8)
    '1', '0', '1', '0', '0', '0', '0', '0',
    // ..
    '1', '0', '0', '1', '0', '0', '1', '0',
    '1', '1', '0', '0', '0', '0', '1', '0',
    '0', '0', '0', '0', '0', '0', '0', '0',
    '1', '1', '0', '1', '0', '1', '1', '0',
    '0', '1', '1', '1', '0', '1', '1', '1'
] // (*9)
```

### private DES.f()

Приватный метод. Раундовая функция. Выполняется 16 раз.

Для простоты отладки методы соединены в цепочку. На каждой итерации выполняется:

```
f() = this
    .#getNewL()
    .#e()
    .#RXorRoundKey(round)
    .#splitR(round)
    .#s(round)
    .#p()
    .#getNewR()
```

#### Описание шагов, входящих в раундовую функцию:

* .#getNewL() — левая часть L<sub>i+1</sub> становится правой R<sub>i</sub> (*3 — после первого раунда)
* .#e() — правая часть R, длиной 32 бита, расширяется до 48 бит, согласно таблице расширения E в файле *./tables/E.js* (часть бит дублируется). R (*3) в результате расширения в первом раунде:

```
R: [
    // 31,  0,  1,  2,  3,  4, 3,  4 — номера бит из (*3)
    '0', '0', '0', '0', '0', '0', '0', '0',
    '0', '0', '0', '1', '0', '1', '1', '1',
    '1', '1', '1', '1', '1', '1', '1', '1',
    '1', '1', '0', '0', '1', '1', '1', '1',
    '1', '0', '1', '1', '1', '1', '1', '0',
    '1', '0', '1', '0', '0', '0', '0', '0'
] // (*10)
```
![E](https://upload.wikimedia.org/wikipedia/commons/thumb/4/46/DES-ee.svg/400px-DES-ee.svg.png)
Источник изображения: Wikipedia

* .#RXorRoundKey(round) — 48-битная правая часть (*10), побитово складывается по модулю два с соответсвующим 48-битным раундовым ключом (*9 в первом раунде). PATCH: исправлена ошибка версии 0.1.0, когда сложение фактически не проиходило

Пример R (*10) после сложения с ключом первого раунда (*9): 

```
R: [
    // 1 ^ 0 = 1, 0 ^ 0 = 0, ...
    '1', '0', '1', '0', '0', '0', '0', '0',   
    '1', '0', '0', '0', '0', '1', '0', '1',
    '0', '0', '1', '1', '1', '1', '0', '1', 
    '1', '1', '0', '0', '1', '1', '1', '1', 
    '0', '1', '1', '0', '1', '0', '0', '0',
    '1', '1', '0', '1', '0', '1', '1', '1'
] // (*11)
```

* .#splitR(round) — разделить правую часть (*11) на 8 частей по 6 бит

Результат:

```
R_chunks: [
    [ '1', '0', '1', '0', '0', '0' ],
    [ '0', '0', '1', '0', '0', '0' ],
    [ '0', '1', '0', '1', '0', '0' ],
    [ '1', '1', '1', '1', '0', '1' ],
    [ '1', '1', '0', '0', '1', '1' ],
    [ '1', '1', '0', '1', '1', '0' ],
    [ '1', '0', '0', '0', '1', '1' ],
    [ '0', '1', '0', '1', '1', '1' ]
] // (*12)
```

* .#s(round) — S-блоки (блоки замены). Каждая из восьми частей R (*12) поступает на вход одного из восьми блоков замены, определённых в файле *./tables/S-Boxes.js*. Первый и последний биты части блока определяют номер строки (0-3), а оставшиеся четыре бита номер столбца (0-15) в блоке замены, указывающих на ...

* .#p()
* .#getNewR()


