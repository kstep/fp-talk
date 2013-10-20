# Введение в функциональное программирование

## Вопросы

* Что это такое?
* В чём оно состоит?
* Для чего это нужно?
* Как это делать?

## Классификация языков

* Декларативное программирование vs. императивное программирование
    * Императивное
        * Как это сделать?
        * Каковы конкретные шаги для достижения цели?
        * Тактика

    * Декларативное
        * Что надо сделать?
        * Каковы взаимосвязи между сущностями?
        * Стратегия

## Особенности функционального программирования

* Функциональное программирование — подвид декларативного

* Функциональное vs. императивное
    * Lazy (ленивые) vs. Eager (энергичные) evaluation
        * Генераторы, итераторы, корекурсия
    * Неизменяемость vs. Изменяемость
    * Чистые функции vs. Side-effects
        * Глобальное состояние — зло

* Другие фичи ФП
    * Функция как объект первого рода
    * Функции более высокого порядка
    * Лямбда-функции
        * Каррирование, частичное применение
        * η-конверсия
        * Крайность: числа чёрча (см. вики, [Church encoding in Python][1] и [вообще][2])
    * Оптимизация хвостовых вызовов (TCO)/оптимизация хвостовой рекурсии (TCR)
        * Замена циклов рекурсией, защита стека от переполнения через TCO
        * Примеры: факториал, числа фибоначчи, quicksort
    * Мета-программирование (optional)

[1]: http://jtauber.com/blog/2008/11/26/church_encoding_in_python/
[2]: http://www.google.com/search?q=числа+чёрча

* Для минимальной реализации ФП достаточна поддержка передачи функции в
  качестве аргумента другой функции
    * Крайность: реализация ФП на C?

## Для чего это всё надо?

* Чистые функции = thread-safe code + testability + scalability
    * Знаменитый Map-Reduce
* Экстремальная кастомизация кода через функции высшего порядка
    * Часто более легковесный подход по сравнению с ООП
    * Упрощение кодирования через абстрагирование общих подходов (e.g. кастомная
      сортировка, асинхронное программирование через обещания и call/cc,
      мемоизация)
* Лёгкое описание событийной модели через описание связями (FRP)
* Следствие: улученный контроль сложности

### Ложка дёгтя
* Debug hell
* Расходы на вызов функции

## Приёмы

```python
import itertools as it
```

### Примеры из реальной жизни

#### Lettuce-тесты *(Lootsie)*
```python
def compose(*funcs):
    """
    Compose a number of functions into single function to make call sequence
    more obvious (no more "read backwards this chain of functions") and
    avoid a lot of nested parens (no more f(g(y(n(m(z(x))))))).

    Very useful when writing callbacks.

    Compare code before `compose()`:
        condition = re.sub(r'{([\w.]+)}', lambda m: bindings.append(str(world.eval(m.group(1)))) or "%s", condition)

    And after `compose()`:
        condition = re.sub(r'{([\w.]+)}', compose(lambda m: m.group(1), world.eval, str, bindings.append, valuefn("%s")), condition)
    """
    return lambda x: reduce(lambda x, f: f(x), funcs, x)

def valuefn(x):
    return lambda y: x

def expect_length_of_collection_by_condition(step, collection, table, condition, value):
    # ...blah-blah-blah...

    condition = re.sub(r'{([\w.]+)}', compose(
        lambda m: m.group(1),
        world.eval,
        str,
        bindings.append,
        valuefn("%s")
    ), condition)

    # ...blah-blah-blah...
```

#### Генерация дерева каталогов и файлов в HTML *(https://gist.github.com/kstep/964803)*
```python
def builder(root):
    return it.starmap(
		lambda isdir, path, name:
			'''<li class="dir"><a href="#%(path)s" name="%(path)s">%(name)s</a><ul>
                        %(content)s</ul></li>''' % dict(path=path, name=name, content=''.join(builder(path)))
                    if isdir
                    else '<li class="file">%s</li>' % name,

		# remove sorted to get native order, add "key" argument to sorted() to define custom sorting
		sorted(

		# retrieve additional info (costly stat() calls)
		it.imap(lambda names: (os.path.isdir(names[0]),) + names,

		# full file name
		it.imap(lambda name: (os.path.join(root, name), name),

		# edit or remove this filter to get only desired files (e.g. name.endswith('.mp3'))
		it.ifilter(
		lambda name: not name.startswith('.'),

		os.listdir(root)
		))),
                
                key=lambda triple: (not triple[0], triple[2])))
```

#### Функциональный парсер *(https://github.com/kstep/greencss)*
```python
def seq(*parsers):
    '''
    Sequence of parsers
    '''
    @wrap_parser('seq', *parsers)
    def wrapper(inp):
        tokens = []
        rest = inp
        for p in parsers:
            token, rest = p(rest)
            if token is None:
                rest = inp
                tokens = None
                break
            tokens.extend(token)
        return tokens, rest
    wrapper.__name__ = 'seq(%s)' % ', '.join(map(lambda p: p.__name__, parsers))
    return wrapper

def alter(*parsers):
    '''
    One of parser
    '''
    @wrap_parser('alter', *parsers)
    def wrapper(inp):
        rest = inp
        token = None
        for p in parsers:
            token, rest = p(inp)
            if token is not None:
                break
        return token, rest
    return wrapper
```

```python
spaces = space * inf
delims = delim * inf
identifier = (alpha - alnum * inf) / join
flag = ('!' - _('not-').opt - alnum * (1, inf)) / join
flags = flag - (spaces/0 - flag) * inf

uinteger = digit * (1, inf) / join
integer = (_('-').opt - uinteger) / join
number = (integer - ('.' - uinteger).opt) / join
```

```python
'''
Пример эта-конверсии: описание рекурсивной синтаксической структуры:

margin->
    top: $b * 2 / $a

border->
    color->
        top: $topcolor
        left: $leftcolor
    width->
        left: $leftwidth
        right: 2 * $leftwidth
'''

_cproperty = _(lambda inp: cproperty(inp))

cproperty = (
        identifier - (':' - spaces)/0 - values - spaces/0 - (flags.opt >> Flags) - EOL >> Property |
        (identifier - _('>')/0 - EOL -
            _cproperty.indent) / ComplexProperty |
        cmacrocall
        )
```

#### Построение SQL запроса на основе данных из таблицы *(StyleStalk)*
```php
// $cleanup_result is the result of other SELECT query,
// no need to escape its data.
// $cleanup_result is an array if dicts (database rows)
$cleanup_condition = '('.implode(') OR (', array_map(
    function ($item) {
        return implode(' AND ', array_map(
            function ($key) use (&$item) {
                return "`$key` = '$item[$key]'";
            }, array_keys($item)));
    }, $cleanup_result)).')';
$cleanup_query = "DELETE FROM `$source_table` WHERE $cleanup_condition";
```

### Самые часто используемые функции

* map/imap, reduce (foldr/foldl)

```haskell
map :: (a -> b) -> [a] -> [b]
map f (x:xs) = f x : map f xs
map _ [] = []

foldr :: (a -> b -> b) -> b -> [a] -> b
foldr f z (x:xs) = (f z x) : foldr f z xs
foldr _ z [] = z

foldl :: (a -> b -> a) -> a -> [b] -> a
foldl f z (x:xs) = foldl f (f z x) xs
foldl _ z [] = z
```

* zip/izip

```haskell
zip :: [a] -> [b] -> [(a, b)]
zip (x:xs) (y:ys) = (x, y) : zip xs ys
zip [] _ = []
zip _ [] = []
```

* takewhile/dropwhile

```haskell
takeWhile :: (a -> Bool) -> [a] -> [a]
takeWhile _ [] = []
takeWhile p (x:xs) =
    case (p x) of
        True -> x : takeWhile p xs
        False -> []

dropWhile :: (a -> Bool) -> [a] -> [a]
dropWhile _ [] = []
dropWhile p (x:xs) =
    case (p x) of
        True -> dropWhile p xs
        False -> x:xs
```

* groupby

```haskell
groupBy :: Eq b => [a] -> (a -> b) -> [(b, [a])]
groupBy [] _ = []
groupBy (x:xs) f =
    let fx = f x
        group = \x -> fx == f x
    in
        (fx, x:(takeWhile group xs)) : (groupBy (dropWhile group xs) f)
```

* filter/ifilter

```haskell
filter :: (a -> Bool) -> [a] -> [a]
filter _ [] = []
filter p (x:xs) = if p x then x:(filter p xs) else filter p xs
```

* repeat, tee, cycle, islice, chain, ...

```haskell
repeat :: a -> [a]
repeat x = x : repeat x

tee :: [a] -> ([a], [a])
tee xs = (xs, xs)

cycle :: [a] -> [a]
cycle [] = []
cycle xs = xs ++ cycle xs

step :: Int -> [a] -> [a]
step _ [] = []
step 1 xs = xs
step j (x:xs) | j > 1 = x : (step j . drop (j - 1)) xs
slice :: [a] -> Int -> Int -> Int -> [a]
slice [] _ _ _ = []
slice xs s e j = step j . take (e - s) . drop s $ xs

chain :: [a] -> [a] -> [a]
chain xs ys = xs ++ ys
```
