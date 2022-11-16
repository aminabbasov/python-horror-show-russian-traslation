
> [!INFO] Этот текст является переводом [статьи](https://github.com/pablogsal/python-horror-show) о странностях в Python. Приятного чтения :)

Шоу ужасов Питона (Python's horror show)
====================

Здесь вы найдете коллекцию странных и необычных фрагментов кода Python, демонстрирующих очевидно странное поведение. Цель этих скриптов — **заморочить вашу голову**, но некоторые люди сообщают о приобретении странных новых **знаниях о Python** как о побочном эффекте.

За кулисами памяти (Hidden memory things)
--------------------

```python
>>> a = 5
>>> b = 5
>>> a is b
True

>>> a = -4
>>> b = -4
>>> a is b  
True

>>> a = 300
>>> b = 300
>>> a is b
False

>>> a = 300; b = 300
>>> a is b
True
```

Из ["Integer Objects"](https://docs.python.org/3/c-api/long.html#c.PyLong_FromLong) (Целочисленные объекты)
>Текущая реализация Python хранит массив численных объектов для всех целых чисел между `-5` и `256`, и когда вы создаете `int` в этом диапазоне, фактически вы просто возвращаете ссылку на уже существующий объект. Таким образом, должна быть возможность изменить значение 1. Я подозреваю, что поведение Python в этом случае не определено. :-)

Мы можем проверить это, используя оператор `id`:

```python
>>> a = 5
>>> b = 5
>>> id(a)
4531116864
>>> id(b)
4531116864

>>> a = 300
>>> b = 300
>>> id(a)
4537522896
>>> id(b)
4537523216

>>> a = 300; b = 300
>>> id(a)
4537523696
>>> id(b)
4537523696
```

Краткое объяснение состоит в том, что, так как в Python, все является объектом, каждый раз, когда вы используете число (целое число, дробное число...) **оно должно быть создано**, но это может быть очень неэффективно. И что делает Python, так это предварительно резервирует целые числа от `-5` до `256`, потому что они часто используются. Последний трюк (`a = 300; b = 300`) зависит от интерпретатора, но в базовом интерпретаторе Python (среди прочих), поскольку два присваивания происходят в одной строке, обе переменные будут ссылаться на один и тот же объект, чтобы не растрачивать место.

Индексы для чайников (Indexes for noobs)
--------------------

```python
>>> a = [1,2]
>>> a.index(1)
0
>>> a.index(2)
1
>>> a[a.index(1)]
1
>>> a[a.index(2)]
2

>>> a[a.index(1)], a[a.index(2)] = 2,1
>>> a
[1, 2]
```

Чтооо! Какого хрена здесь происходит? Легко: мы забываем, что все должно оцениваться последовательно. Давайте проверим это снова, но поочередно, одно утверждение за другим:

```python
>>> a = [1,2]

>>> a[a.index(1)] = 2
>>> a
[2, 2]
>>> a.index(2)
0
>>> a[a.index(2)] = 1
>>> a
[1, 2]
```

Ага! Итак, дело в том, что когда мы назначаем `a[a.index(1)] = 2`, поскольку `a.index(2)` даст нам **первый индекс, в котором появляется `2`**, он дает `1` и, следовательно, `a[a.index(2)] = 1` сбросит `a` к его начальному значению.

Слишком много сравнений (Too many equals)
---------------

Это одно из моих любимых:

```python
>>> a, b = a[b] = {},5
>>> a
{5: ({...}, 5)}
>>> b
5
```

Хмммммммм... Я думаю, проблема в том, что как только мы видим двойное равенство, мы думаем о формальной пропозициональной логике следствий утверждения. Но, как мы уже видели, вещи должны оцениваться последовательно. В данном случае есть два правила:

1. Слева на право
2. `a = b = c`  — синтаксический сахар для `a=c & b=c`

Итак, единственное, что нам нужно сделать, чтобы прекратить этот беспорядок — выполнить код, придерживающийся этих правил:

```python
>>> a, b = {},5
>>> a[b] = a,5  # Поскольку a - пустой словарь и b=5, это эквивалентно {}[5] = ({},5)
>>> a
{5: ({...}, 5)}
>>> b
5
```

Хэшируемые объекты (Hashable objects)
----------------

```python
>>> d = {1: 'a', True: 'b', 1.0: 'c'}
>>> d
{1: 'c'}
```

И... что здесь происходит? Ответ уже дан: вещи должны оцениваться последовательно.

```python
>>> d = dict()
>>> d[1] = ‘a’
>>> d[True] = ‘b’
>>> d[1.0] = ‘c’
>>> d
{1: 'c'}
```

Хорошо, но ¿почему True оценивается как 1?

```python
>>> 1 is True
False

>>> 1 == True
True
```

Причина: любые две строки, числа и т. д которые равны, будут иметь один и тот же хэш, что позволяет `dict` (реализованному как хэш-таблица) очень эффективно находить эти строки. **Ключи `dict` приравниваются по значению, а не по идентичности**.

На самом деле, это та же самая причина, по которой, как правило, **мы не можем использовать изменяемые объекты в качестве ключа в словаре**: хэш списка или другого изменяемого объекта основан не на его значении, а скорее на экземпляре списка и изменяется при изменении его содержимого.

Такой же подход справедлив и при создании множеств:

```python
>>> s = {1, True, 1.0}
>>> s
{1}
```

или используя другие значения как `0` и `False`:

```python
>>> d2 = {0: 'a', False: 'b'}
>>> d2
{0: 'b'}
```

Вход в пустоту (Enter the void)
---------------

```python
>>> all([])
True
>>> all([[]])
False
>>> all([[[]]])
True
```

При преобразовании в логический тип `[]` превращается в `False`, потому что он пустой, а `[[]]` становится `True`, поскольку он не пустой. Следовательно, `all([[]])` эквивалентно `all([False])`, а `all([[[]]])` равно `all([True])`. Как и в `all([])` если нет `False`, тогда тривиально `True`.

Поглощенный методом `iter` (Consumed by the `iter` method)
---------------------------

```python
>>> a = 2, 1, 3
>>> sorted(a) == sorted(a)
True
>>> reversed(a) == reversed(a)
False
```

В отличие от `sorted`, возвращающего список, `reversed` возвращает итератор. Итераторы будут равны при сравнении самих с собой, но не с другими итераторами, содержащими те же значения.

```python
>>> b = reversed(a)
>>> sorted(b) == sorted(b)
False
```

Итератор `b` используется при первом вызове `sorted`. Вызов второго `sorted(b)` после использования `b` просто возвращает `[]`.

Ложь — новая правда (`False` is the new `True`)
---------------------------
```python
>>> False == False in [False]
True
```

Ни `==`, ни `in` не выполняются первыми. Они оба являются [операторами сравнения](https://docs.python.org/3.5/reference/expressions.html#comparisons) с одинаковым приоритетом, поэтому они объединены в цепочку. Строка эквивалентна `False == False and False in [False]`, что равно `True`.

Возвращение в детство (Return to childhood)
---------------------------

```python
>>> x = (1 << 53) + 1
>>> x + 1.0 < x
True
```

Значение `x` может быть точно представлено значением `int` в Python, но не значением Python `float`, которое имеет точность в 52 бита. Когда `x` преобразуется из `int` в `float`, его необходимо округлить до ближайшего значения. Согласно правилам округления, ближайшее значение равно `x - 1`, которое _может_ быть представлено в виде `float`. 

Когда вычисляется `x + 1.0`,  `x` сначала преобразуется в `float` для выполнения сложения. Это делает его значение `x - 1`. Затем добавляется «1.0». Это возвращает значение обратно к x, но поскольку результат является числом с плавающей запятой, оно снова округляется до  `x - 1`.  

Далее происходит сравнение. В этом Python отличается от многих других языков. В C, например, если `double` сравнивается с `int`, то `int` сначала преобразуется в `double`. В данном случае это будет означать, что правая часть также будет округлена до `x - 1`, обе стороны будут равны, а сравнение `<` будет ложным. Python, однако, имеет специальную логику для обработки сравнения между `float` и `int`, и он способен правильно определить, что `float` со значением `x - 1` меньше, чем `int` с значением `х`.