## Храните деньги в ...

Попытаемся в это посте понять как лучше хранить деньги в базах данных и какой тип использовать для работы с ними в коде.

> А в чём вообще проблем? Что не так с числами с плавающей точкой?

Если кратко, то с ними все хорошо, но только они вообще не про деньги. Приведу несколько примеров, которые сбивают с толку:

```python
# Python 3.9.7 (default, Sep 10 2021, 14:59:43)
# [GCC 11.2.0] on linux
>>> 0.1+0.11
0.21000000000000002  # почему?
>>> 1.3+1.6
2.9000000000000004  # что за ...?
```

Если думаете, что проблема в языке Python, то ошибаетесь, вот код на Go

```go
// go version go1.18 linux/amd64
fmt.Printf("%.6f", float32(1.3+1.6))  // 2.9000000
fmt.Printf("%.9f", float32(1.3+1.6))  // 2.900000095
fmt.Printf("%.15f", float32(1.3+1.6)) // 2.900000095367432
fmt.Printf("%.25f", float32(1.3+1.6)) // 2.9000000953674316406250000

// ещё немного "магии"
fmt.Printf("%f", float32(16777217.0)) // 16777216.000000
```

И аналогичные примеры на Си:

```c
// gcc 11.2.0
float a;
a = 3.2;
printf("a=%.15f\n", a); // a=3.200000047683716
a = 0.1;
printf("a=%.9f\n", a);  // a=0.100000001
```

Очевидно, что с такими погрешностями в вычислениях нормально обрабатывать денежные транзакции не получится.

## Немного истории

Как это не странно, когда-то компьютеры ничего не знали о типе Float. Вычисления были целочисленными или каждый производитель ЭВМ использовал свой подход.

В 1975 Intel начал разработку сопроцессора для вычислений с плавающей точкой для своих микропроцессоров i8086/8 и i432. Там была достаточная интересная история про развитие идей и конкуренцию с DEC, подробнее можно прочитать [тут](https://web.archive.org/web/20160304114859/http://www.intel.com/content/dam/www/public/us/en/documents/case-studies/floating-point-case-study.pdf), но суть в том, что в 1980 уже вышел первый черновик стандарта, а в 1985 и его финальная версия **IEEE 754-1985**.

**IEEE 754** определяется промышленный стандарт представления чисел с плавающей точкой в компьютерах. За прошедшие годы выходила еще пара его версий [IEEE 754-2008](https://ieeexplore.ieee.org/document/4610935) и [IEEE 754-2019](https://ieeexplore.ieee.org/document/8766229).

Если очень кратно, то суть IEEE 754 в том, чтобы решить несколько проблем:

- в каком формате хранить числа с плавающей точкой в компьютерах (двоичное представление таких чисел)
- какая должна быть у них точность
- как обеспечить скорость вычисления при работе с ними 
- как много памяти они должны занимать
- не должно быть неоднозначности в представлении таких чисел

Попробуйте подумать над этими вопросами и поймете, что это не так уж и просто. А точнее **невозможно отобразить бесконечное множество вещественных чисел на конечных набор байт**, с которыми работает компьютер. За все приходится платить и потеря точности это как раз та самая цена.

Главная идея IEEE 754 в представлении чисел в научной нотации. Вы наверное уже видели такие записи `1234.5=1.2345e3` или `0.0054321=5.4321e-3`? Это короткая запись выражений:

$$
1234.5 = 1.2345 \cdot 10^3
$$

$$
0.0054321 = 5.4321 \cdot 10^{-3}
$$

С числами с плавающей точкой все тоже самое, только за основу взята не десятичная система исчисления, а двоичная:

$$  
1234.5 \approx 1.2 \cdot 2^{10}
$$

$$
0.0054321 \approx 1.4 \cdot 2^{-8}
$$

Можно записать и более точно, тогда будет понятнее

$$
4 = 1 \cdot 2^{2}
$$

$$
16 = 1 \cdot 2^{4}
$$

$$
0.5 = 1 \cdot 2^{-1}
$$

$$
0.75 = 1.5 \cdot 2^{-1}
$$

$$
0.25 = 1 \cdot 2^{-2}
$$

$$
38.4 = 1.2 \cdot 2^5
$$

$$
0.0375 = 1.2 \cdot 2^{-5}
$$

Каждое число представляется в виде

$$
(-1)^{sign} \cdot (1 + fraction) \cdot 2^{(exponent - bias)}
$$

Тут несколько констант и 3 переменные, определяющие запись

1. <span style="color:#c5fcff">**sign**</span> - 1 бит под знак числа, чтобы понимать положительное оно или отрицательное. Удобно потом брать значение и использовать его в выражении `(-1)^sign` для определения знака.
2. <span style="color:#ffaead">**fraction**</span> - мантисса, вещественна часть числа от 0 до 1, в примере про `0.75 = 1.5 * 2^(-1)`, мантисса будет `0.5 = (1.5 - 1)`. Единицу **отбрасывают** для экономии памяти, так как она все равно всегда есть в нормализованной записи, то зачем ее хранить, если можно помнить о том, что её просто нужно всегда в конце добавить к результату.
3. <span style="color:#9fffad">**exponent**</span> - порядок, та самая степень двойки как в примерах выше, но со сдвигом на *bias*. Последний решает проблему знака, но не у самого числа, а у порядка. Например для чисел с одинарной точностью (Float32), `bias=127`, тогда экспонента -2 будет хранится как 125, а +2 как 129. Это позволяет использовать всего 8 бит и диапазон значений от 1 до 254, чтобы работать с вариантам степеней двойки от -126 до +127.

Еще IEEE 754 оговаривает такие необычные комбинации как плюс/минус бесконечности (Inf) и неопределенное число (NaN), например полученное в результате деления на ноль.

## Приведение к машинному виду

Теперь попробуем конвертировать числа с плавающей точкой в машинный вид и обратно. Для простоты возьмем единичную точность, которая обычно в языках программирования представлена как тип *Float32*, где

- первые 23 бита определяют мантиссу (<span style="color:#ffaead">fraction</span>)
- следующие 8 бит это порядок (<span style="color:#9fffad">exponent</span>)
- и последний бит - знак числа (<span style="color:#c5fcff">sign</span>)

![float32.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1649966393306/5k3wNhAbO.png)

Используем формулы выше, но часть переменных нам уже известна sign=0, так как число 0.15625 положительное, а bias=127 это константа для Float32:

$$
(-1)^0 \cdot (1 + fraction) \cdot 2^{(exponent - 127)}
$$

Оценим число, чтобы найти остальные элементы. Тут все как с десятичной системой, нужно делить до тех пор пока не получим нормальную форму, когда результат станет от 1 до 2.

$$
0.15625 / 2^{-1} = 0.3125
$$

$$
0.15625 / 2^{-2} = 0.625
$$

$$
0.15625 / 2^{-3} = 1.25
$$

Получилось, что `(1 + fraction) = 1.25`, а `(exponent - 127) = -3`, тогда окончательный вид такой. 

$$
(-1)^0 \cdot (1 + 0.25) \cdot 2^{(124 - 127)}
$$

- 130 в бинарном представлении как раз `1111100`
- а 0.25 - `01000000000000000000000`

Представление 0.25 в бинарном виде в этом примере слишком очевидное, это 1/4. Другой вариант - это итерационное умножение на 2, когда берем целую часть, пока дробная не равна 0.

```python 
0.25 * 2 = 0.5 # берем 0, а дальше используем число 0.5 полностью
0.5  * 2 = 1.0 # берем 1, а дальше используем десятичную часть
0.0  * 2 = 0.0 # всё, больше уже ничего не измениться, дальше все 0
# ...
```

Но если взять другой, не такой "гладкий" пример, то есть вероятность, что получение мантиссы никогда не сойдется. Например, возьмем число `43.52`

$$
43.52 = (-1)^0 \cdot (1 + 0.36) \cdot 2^{(132 - 127)}
$$

```python
0.36 * 2 = 0.72 # 0
0.72 * 2 = 1.44 # 1
0.44 * 2 = 0.88 # 0
0.88 * 2 = 1.76 # 1
0.76 * 2 = 1.52 # 1
0.52 * 2 = 1.04 # 1
0.04 * 2 = 0.08 # 0
0.08 * 2 = 0.16 # 0
0.16 * 2 = 0.32 # 0
0.32 * 2 = 0.64 # 0
0.64 * 2 = 1.28 # 1
0.28 * 2 = 0.56 # 0
0.56 * 2 = 1.12 # 1
0.12 * 2 = 0.24 # 0
0.24 * 2 = 0.48 # 0
0.48 * 2 = 0.96 # 0
0.96 * 2 = 1.92 # 1
0.92 * 2 = 1.84 # 1
0.84 * 2 = 1.68 # 1
0.68 * 2 = 1.36 # 1
0.36 * 2 = 0.72 # 0 - упс, это уже где-то было
# начинали с такого числа 0.36
# нашли 21 из 23 нужных бит, но следующие 2 это повторения
0.72 * 2 = 1.44 # 1
0.44 * 2 = 0.88 # 0
# точно знаем, что последовательность периодическая и не сходится,
# но записать больше 23 бит физически не можем
# это похоже на бесконечные рациональные дроби вида
# 5/27 = 0.1851851851851851...
# только у нашей мантиссы 20 бит будут повторяться бесконечно
```

Получили запись `0 10000100 01011100001010001111011`,  это лишь приближение, а значит, что точного значения для числа `43.52` мы сохранить в памяти компьютера не можем, в каком-то вычислении ошибка все равно проявит себя:

```python
# python
>>> print('{:.15f}'.format(43.52))
43.520000000000003
```

Кстати, если заметили, то последний бит у мантиссы `01011100001010001111011` был не 0 как мы посчитали, а 1. Это из-за округления, примерно так же как если бы мы пытались записать десятичное число с точностью до 2-х знаков `round(0.005) = 0.01`.

## Конвертирование из машинного вида

Теперь возьмем любое число с плавающей точкой в бинарном виде и найдем его значение в десятичных дробях. Например `1 10000101 11101101110100101111001`:

- sign=-1, значит число отрицательное
- (exponent - 127) = 10000101 (bin) = 133 (dec), значит exponent=6
- `11101101110100101111001` раскладываем по степеням двойки

$$
1 \cdot 2^{-1} + 1 \cdot 2^{-2} + 1 \cdot 2^{-3} + 0 \cdot 2^{-4} + ... = 0.9290000200271606
$$

Получилось `-1.9290000200271606 * 2^6 = -123.45600128173828`. Кстати эту сумму я считал на компьютере, складывая float значения, поэтому тут уже есть дополнительная ошибка. Но начальное число для преобразования в бинарный формат я выбрал `-123.456`, ясно видим накопленные погрешности.

> А что не так с числом `16777217` из примеров в начале статьи? Почему ошибка такая большая, на 1 в целой части?

```go
// go
fmt.Printf("%f", float32(16777217.0)) // 16777216.000000
```

Проблема в том, что машинном виде по стандарту **IEEE 754** число будет представлено как `0 10010111 00000000000000000000000`, а это в точности такой же вид как и у числа `16777216`. У `16777217` мантисса полностью нулевая, а exponent=24, просто не хватило точности 23 бит для сохранения числа `0.0000000596046448`, единичный бит появляется только с 24-й позиции.

$$
16777217 = 1.0000000596046448 \cdot 2^{24}
$$

$$
16777216 = 1.0 \cdot 2^{24}
$$

## Способы работы с Float в языках программирования

Это все были лишь теоретические выкладки, но есть способы посмотреть на приведение чисел с плавающей точкой к бинарному виду в явном виде.

В C/C++ можно посмотреть на код ассемблера и найти там нужную переменную как long.

```sh
gcc -S -O0 source_file.c -o-
```

В языке Go в пакете [math](https://pkg.go.dev/math#Float32bits) есть методы Float32bits/Float32frombits, которые позволяют делать конвертацию из float в uint и обратно.

```go
var (
	isFloat float32
	isUint uint32
)
isFloat = 43.52
isUint = math.Float32bits(isFloat)
// float=43.520000 -> binary=1000010001011100001010001111011
fmt.Printf("float=%f -> binary=%b\n", isFloat, isUint)

isFloat = math.Float32frombits(isUint)
// binary=1000010001011100001010001111011 -> float=43.520000
fmt.Printf("binary=%b -> float=%f\n", isUint, isFloat)
```

Для Python есть функция перевод массива байт (бинарного представления) во встроенный тип float [тут](https://ru.wikipedia.org/wiki/%D0%A7%D0%B8%D1%81%D0%BB%D0%BE_%D0%BE%D0%B4%D0%B8%D0%BD%D0%B0%D1%80%D0%BD%D0%BE%D0%B9_%D1%82%D0%BE%D1%87%D0%BD%D0%BE%D1%81%D1%82%D0%B8#Python).

## Как же хранить деньги

После всего вышесказанного возможно у вас возникли вопросы.

> Если у float такие ошибки, зачем его вообще использовать? И есть ли вообще способ считать точно?

1. Числа плавающей точкой можно смело использовать, но только не для финансовых расчетов, где важна каждая копейка. Существуют много разных областей, где такая ювелирная точность не нужна. Другой важный момент это скорость работы с ними. В современных компьютерах вычисления с числами с плавающей точкой поддерживаются аппаратным сопроцессором (FPU - floating point unit), а это существенное ускорение в работе, то на что когда-то и ставил Intel.
2. Да есть способ считать более точно, исключая погрешности до приемлемых границ. Далее рассмотрим тип Decimal.

Существуют языки, где поддержаны рациональные числа. Вот пример на [scheme](https://groups.csail.mit.edu/mac/projects/scheme/) диалекте Lisp.

```lisp
(+ (/ 6 50) (/ 12 25))  

;Value: 3/5
```

Это эквивалентно

$$
\frac{6}{50} + \frac{12}{25} = 
\frac{3}{25} + \frac{12}{25} =
\frac{15}{25} = \frac{3}{5}
$$

Иногда это может быть очень удобно, например для числа `1/3` и понятно как это хранить, нужно структура с двумя полями. Но к сожалению возникают и проблемы. Задача нахождения общего знаменателя имеет сложность `О(N+M)`, а еще дроби могут становиться очень громоздкими.

В том же стандарте IEEE 754 есть раздел про Decimal формат, когда представление числа кодируется не степенью двойки, а десятки. Это уже совсем похоже на научную нотацию с буквами `e/E`. Поэтому основная формула очень похожа на реализацию для Float:

$$
(-1)^{sign} \cdot coefficient \cdot 10^{exponent}
$$

- **sign** - знак числа
- **coefficient** - целый коэффициент, иногда сразу включает в себя *sign*
- **exponent** - экспонента для степени 10, тоже целое число

Хранят такое число уже не одним 32-битным куском, а составной структурой из эти 2-х или 3-х полей. Приведем примеры и будет понятнее:

```
1       => {coefficient: 1, exponent: 0}
2.0     => {coefficient: 20, exponent: -1}
-1.2    => {coefficient: -12, exponent: -1}
123.45  => {coefficient: 12345, exponent: -2}
0.5     => {coefficient: 5, exponent: -1}
1e7     => {coefficient: 1, exponent: 7}
-0.0345 => {coefficient: -345, exponent: -5}
```

Арифметика с такими структурами тоже предельно ясная, аккуратно по отдельности работаем с целыми частями и экспонентами, формируем новые значения. Можно посмотреть как это сделано в популярных языках, по коду всегда всё понятнее:

- в Python реализация в файле [_pydecimal.py](https://github.com/python/cpython/blob/main/Lib/_pydecimal.py)
- в Go нет встроенного типа Decimal, но есть сторонние библиотеки, например [github.com/shopspring/decimal](https://github.com/shopspring/decimal/blob/master/decimal.go)

Кстати в Python предложение о добавлении Decimal было сделано в 2003 году в [PEP 327](https://peps.python.org/pep-0327/) и его тоже интересно почитать, чтобы понять основные мотивы для включения такой конструкции в язык.

Надеюсь, с языками программирования все стало чуть-чуть понятнее, с базами данных все точно так же. Многие СУБД уже имеют не только встроенный Float, но и Decimal, его и надо использовать для хранения денег. Если такого типа нет, то можно брать целочисленный с аналогии с парой `coefficient / exponent`, где 2-я часть фиксированная. Например договариваемся всё хранить до миллионной доли одной единицы валюты, тогда число `123.45` в БД сохраняем как `123450000` (то есть `exponent -6`). Языки программирования в основном дают возможность создавать Decimal из строк:

```python
# python
from decimal import Decimal

>>> exponent = -6
>>> coefficient = 123450000
>>> Decimal(f'{coefficient}e{exponent}')
Decimal('123.450000')
```

## Типичные ошибки при работе с деньгами

Во-первых, как уже написано ранее не использовать совсем или очень аккуратно работать с типом Float.

Помнить про НДС (налог на добавленную стоимость), он же VAT (value-added tax). Он обычно задается в процентах и определяет какая часть пойдет на оплату услуг непосредственно, а какая является налогом. Например при НДС 20%, сумма в 600 делится на 500 и 100 соответственно. Но что, если клиент платит 100 при НДС=18%, какая величина налога?

$$
tax = \frac{amount \cdot rate}{1 + rate} = 
\frac{100 \cdot \frac{18}{100}}{1 + \frac{18}{100}} = 
\frac{50 \cdot 18}{59} = 
15\frac{15}{59}
$$

Не самое ровное число. В десятичной записи это бесконечная дробь с периодом в 58 знаков. Поэтому, во-вторых нужно выбрать тип округления и всегда его придерживаться во всех расчетах. Если считаете НДС через [round half up](https://en.wikipedia.org/wiki/Rounding#Round_half_up), то делайте этого везде.

И отсюда же вытекает следующая типичная ошибка. НДС группы товаров не равен НДС одного товара на такую же сумму. Возьмём цену 100 и снова НДС 18%, при округление [round half up](https://en.wikipedia.org/wiki/Rounding#Round_half_up) значение для налога будет `round(15.2542) = 15.25`, то есть для 3 таких товаров `45.75`. А теперь посчитаем НДС для одной цены в 300 и это уже `round(45.7627) = 45.76`. Получили лишнюю копейку, которая для 3-х отдельных оплат не относилась бы к налогу.

Следующая проблема это обновление записей с базах данных. Даже если вы храните значения как Decimal то следующий запрос на самом деле может дать неожиданный результат:

```sql
UPDATE accounts 
SET balance=balance+1.23 
WHERE id=100500;
```

Лучше явно приводить значение к нужному типу, а не оставлять это на усмотрение базы данных, которая может воспринять `1.23` как Float.

```sql
UPDATE accounts 
SET balance=balance+CAST('1.23' AS Decimal(28, 8)) 
WHERE id=100500;
```

Ну и последняя, совсем очевидная, но от этого не более редкая ошибка это подсчет "итого". Если вы формируете какой-то отчет, куда выводите уже округленные денежные значения и в нем есть секция для финального суммирования, то нельзя ее оставлять на самостоятельное заполнение. Сумма по округленным значениям в общем случае может отличаться от округленной суммы всех значений.

## Выводы

Главный вывод про стандарт **IEEE 754** и его систему хранения данных в том, что точность числа не зависит от количество десятичных знаков в человекочитаемом представлении, так как все равно происходит приведение к нормальной форме. То есть большие ошибки могут быть даже на очень коротких и простых числах. Или очень сложные на вид переменные хорошо попадают в степень двойки и хранятся без погрешностей.

Нужно помнить, что есть еще более точные числа с плавающей точкой (double, float64). Да, в них тоже есть погрешности, но они уже значительно меньше.

Числа с плавающей запятой вполне могут использоваться для быстрых расчетов различных метрик, но только не финансовых.

Для точных расчетов денег используйте или целочисленные вычисления или тип Decimal.

Никогда не забывайте про округление, выберете и придерживайтесь одного его типа.

Явно контролируйте тип финансовых данные, не оставляйте его на автоматическое приведение базам или языкам программирования.

## Ссылки

1. [754-2019 - IEEE Standard for Floating-Point Arithmetic](https://ieeexplore.ieee.org/document/8766229)
2. [The IEEE 754 Format](http://mathcenter.oxford.emory.edu/site/cs170/ieee754/)
3. Wikipedia [Single-precision floating-point format](https://en.wikipedia.org/wiki/Single-precision_floating-point_format)
4. [Что нужно знать про арифметику с плавающей запятой](https://habr.com/ru/post/112953/)
5. Журал "Хакер" статья ["Всё, точка, приплыли! Учимся работать с числами с плавающей точкой и разрабатываем альтернативу с фиксированной точностью десятичной дроби"](https://xakep.ru/2015/01/01/vsyo-tochka-priplyli/). [Версия](https://habr.com/ru/company/xakep/blog/257897/) для Хабра.
6. Генри С. Уоррен ["Алгоритмические трюки для программистов"](http://www.williamspublishing.com/Books/978-5-8459-1838-3.html), 2-е издание (Henry Warren ["Hacker's Delight, 2nd Edition"](https://www.amazon.com/Hackers-Delight-2nd-Henry-Warren/dp/0321842685))
7. YouTube ["Как работают числа с плавающей точкой"](https://www.youtube.com/watch?v=U0U8Ddx4TgE)
8. [github.com/shopspring/decimal](https://github.com/shopspring/decimal)
9. [PEP 327 – Decimal Data Type](https://peps.python.org/pep-0327/)