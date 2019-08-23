# Катастрофический возврат

Некоторые регулярные выражения, простые с виду, могут выполняться оооочень долго, и даже "подвешивать" интерпретатор JavaScript.

Рано или поздно с этим сталкивается любой разработчик, потому что нечаянно создать такое регулярное выражение –- проще простого.

Бывает, что регулярное выражение работает нормально, но иногда, с некоторыми строками, "подвешивает" интерпретатор и потребляет 100% процессора.

Как правило, веб-браузер при этом предлагает "убить" скрипт и перезагрузить зависшую страницу. Явно плохая ситуация.

Ну а для серверного JavaScript это может стать серьёзной уязвимостью, если регулярные выражения используются для обработки пользовательских данных.

## Пример

Допустим, у нас есть строка, и мы хотим проверить, что она состоит из слов `pattern:\w+`, после каждого слова может быть пробел `pattern:\s?`.

Используем регулярное выражение `pattern:^(\w+\s?)*$`, которое задаёт 0 или более таких слов.

С виду оно, вроде, работает, например:

```js run
let reg = /^(\w+\s?)*$/;
let str = "An example string";

alert( reg.test(str) );
```

...Но на некоторых строках оно выполняется очень долго. Так долго, что интерпретатор JavaScript "зависает" с потреблением 100% процессора.

Если вы запустите пример ниже, то, скорее всего, ничего не увидите, так как JavaScript подвиснет. В браузере, скорее всего, понадобится перезагрузить страницу, так что осторожно с этим:

```js run
let reg = /^(\w+\s?)*$/;
let str = "An input string that takes a long time or even makes this regex to hang!";

// этот поиск будет выполняться очень, очень долго
alert( reg.test(str) );
```

Впрочем, некоторые движки регулярных выражений могут справиться с таким поиском, но большинство из них -- нет.

## Упрощённый пример

В чём же дело? Почему регулярное выражение "зависает"?

Чтобы это понять, упростим пример: уберём из него пробелы `pattern:\s?`. Получится `pattern:^(\w+)*$`.

И, для большей наглядности, заменим `pattern:\w`, на `pattern:\d`. Получившееся регулярное выражение тоже будет "зависать", например:

<!-- let str = `AnInputStringThatMakesItHang!`; -->

```js run
let reg = /^(\d+)*$/;

let str = "012345678901234567890123456789!";

// этот поиск будет выполняться очень, очень долго
alert( reg.test(str) );
```

В чём же дело, что не так с регулярным выражением?

Внимательный читатель, посмотрев на него, наверняка удивится, ведь он "какой-то странный". Квантификатор `pattern:*` здесь выглядит лишним. Если хочется найти число, то с тем же успехом можно искать `pattern:\d+$`.

Действительно, это регулярное выражение носит искусственный характер, но, разобравшись с ним, мы поймём и практический пример, данный выше. Причина их медленной работы одинакова.

Что же происходит во время поиска `pattern:(\d+)*$` в строке `subject:123456789!`?

1. Первым делом, движок регулярных выражений пытается найти `pattern:\d+`. Плюс `pattern:+` является жадным по умолчанию, так что он хватает все цифры, какие может:

    ```
    \d+.......
    (123456789)!
    ```

    Затем движок пытается применить квантификатор `pattern:*`, но больше цифр нет, так что звёздочка ничего не даёт.

    Далее по шаблону ожидается конец строки `pattern:$`, а в тексте символ `subject:!`, так что соответствий нет:

    ```
               X
    \d+........$
    (123456789)!
    ```

2. Так как соответствие не найдено, то "жадный" квантификатор `pattern:+` уменьшает количество повторений, возвращается на один символ назад.

    Теперь `pattern:\d+` – это все цифры, за исключением последней:
    ```
    \d+.......
    (12345678)9!
    ```
3. Далее движок снова пытается найти совпадение, начиная уже с новой позиции (`9`).

    Звёздочка `pattern:(\d+)*` теперь может быть применена - она даёт второе число `match:9`:

    ```

    \d+.......\d+
    (12345678)(9)!
    ```

    Движок ожидает найти `pattern:$`, но это ему не удаётся, ведь строка оканчивается на `subject:!`:

    ```
                 X
    \d+.......\d+
    (12345678)(9)!
    ```

4. Так как совпадения нет, то поисковой движок продолжает отступать назад. Общее правило таково: последний жадный квантификатор уменьшает количество повторений до тех пор, пока это возможно. Затем понижаетcя предыдущий "жадный" квантификатор и т.д.

    Перебираются все возможные комбинации. Вот их примеры.

    Когда первое число `pattern:\d+` содержит 7 цифр, а дальше 2 цифры:

    ```
                 X
    \d+......\d+
    (1234567)(89)!
    ```

    Когда первое число содержит 7 цифр, а дальше два числа по 1 цифре:

    ```
                   X
    \d+......\d+\d+
    (1234567)(8)(9)!
    ```

    Когда первое число содержит 6 цифр, а дальше одно число из 3 цифр:

    ```
                 X
    \d+.......\d+
    (123456)(789)!
    ```

    Когда первое число содержит 6 цифр, а затем два числа:

    ```
                   X
    \d+.....\d+ \d+
    (123456)(78)(9)!
    ```

    ...И так далее.


Существует много способов как разбить на числа набор цифр `123456789`. Если быть точным, их <code>2<sup>n</sup>-1</code>, где `n` - длина набора.

В случае `n=20` их порядка миллиона, при `n=30` - ещё в тысячу раз больше. На их перебор и тратится время.

Что же делать?

Может нам стоит использовать "ленивый" режим?

К сожалению, нет: если мы заменим `pattern:\d+` на `pattern:\d+?`, то регулярное выражение всё ещё будет "зависать". Поменяется только порядок перебора, но не общее количество комбинаций.

Некоторые движки регулярных выражений содержат хитрые проверки и конечные автоматы, которые позволяют избежать бесконечного перебора или кардинально ускорить его, но не все движки и не всегда.

## Назад к словам и строкам

В примере выше, когда мы ищем слова по шаблону `pattern:^(\w+\s?)*$` в строке вида `subject:An input that hangs!`, происходит то же самое.

Дело в том, что каждое слово может быть представлено как в виде одного `pattern:\w+`, так и нескольких:

```
(input)
(inpu)(t)
(inp)(u)(t)
(in)(p)(ut)
...
```

С учетом того, что пробелы необязательны, регулярное выражение совершенно аналогично пробует разные комбинации того как регулярное выражение `pattern:(\w+)*` может "захватить" каждое слово. Перебираются все возможные варианты применения квантификаторов `pattern:+` и `pattern:*`.

Более того - для каждой комбинации одного слова проверяются все комбинации других слов строки. В её конце стоит `!`, поэтому совпадение невозможно, но проверки требуют много времени.

## Как исправить?

Есть два основных подхода, как это исправить.

Первый - избежать возможности того, что регулярное выражение может по-разному захватывать строку.

Перепишем регулярное выражение так: `pattern:^(\w+\s)*\w*` - то есть, будем искать любое количество слов с пробелом `pattern:(\w+\s)*`, после которых идёт (но не обязательно) обычное слово `pattern:\w*`.

На этот раз всё работает:

```js run
let reg = /^(\w+\s)*\w*$/;
let str = "An input string that takes a long time or even makes this regex to hang!";

alert( reg.test(str) ); // false
```

Почему же проблема исчезла?

Теперь звёздочка `pattern:*` стоит после `pattern:\w+\s` вместо `pattern:\w+\s?`. Стало невозможно разбить одно слово на несколько разных `\w+`. Исчезли и потери времени на это.

## Запрет возврата

Переписывать регулярное выражение не всегда удобно.

Альтернативный подход заключается в том, чтобы запретить возврат для квантификатора.

Движок регулярных выражений проверяет множество вариантов, которые для человека являются очевидно ошибочными.

Например, в выражении `pattern:(\d+)*$` для человека очевидно, что в `pattern:(\d+)*` не нужно "откатывать" `pattern:+`. От того, что вместо одного` \d+` у нас будет два независимых` \d+\d+`, ничего не изменится:

```
\d+........
(123456789)!

\d+...\d+....
(1234)(56789)!
```

Если говорить об изначальном примере `pattern:^(\w+\s?)*$`, то хорошо бы исключить возврат для `pattern:\w+`. То есть, для `pattern:\w+` нужно искать только слово целиком, максимально возможной длины. Не нужно уменьшать количество повторений `pattern:\w+`, пробовать разбить слово на два `pattern:\w+\w+`, и т.п.

В современных регулярных выражениях для решения этой проблемы придумали сверхжадные ("possessive") квантификаторы, которые вообще не используют возврат.

Они очень просты, даже проще, чем "жадные" – берут максимальное количество символов и всё. Поиск продолжается дальше.

Также есть "атомарные скобочные группы" -- средство, запрещающее перебор внутри скобок.

К сожалению, в JavaScript они не поддерживаются, но есть другое средство.

### Предпросмотр в помощь!

Мы можем исключить возврат с помощью предпросмотра.

Шаблон, захватывающий максимальное количество повторений `pattern:\w` без возврата, выглядит так: `pattern:(?=(\w+))\1`.

Другими словами:
- Предпросмотр `pattern:?=` ищет максимальное количество `pattern:\w+`, доступных с текущей позиции.
- Скобки предпросмотра `(?=...)` не запоминаются движком, поэтому оборачиваем `\w+` внутри в дополнительные скобки, чтобы движок регулярных выражений запомнил их содержимое.
- ...И чтобы далее на него сослаться обратной ссылкой `pattern:\1`.

То есть, мы смотрим вперед - и если там есть слово `pattern:\w+`, то ищем его же `pattern:\1`.

Предпросмотр возвращаться не умеет. Что он нашёл, то и берётся, целиком. Поэтому мы реализовали, по сути, сверхжадный квантификатор `pattern:+`.

Вместо `\w` можно подставить и более сложное регулярное выражение, это рецепт общего вида.

```smart
Больше о связи между сверхжадных квантификаторов и предпросмотра вы можете найти в статьях [Regex: Emulate Atomic Grouping (and Possessive Quantifiers) with LookAhead](http://instanceof.me/post/52245507631/regex-emulate-atomic-grouping-with-lookahead) и [Mimicking Atomic Groups](http://blog.stevenlevithan.com/archives/mimic-atomic-groups).
```

В нашем случае:

```js run
let reg = /^((?=(\w+))\2\s?)*$/;

let str = "An input string that takes a long time or even makes this regex to hang!";

alert( reg.test(str) ); // false, работает и быстро

alert( reg.test("A correct string") ); // true, пример подходящей строки
```

Здесь внутри скобок используем `pattern:\2`, так как есть ещё внешние скобки. Чтобы избежать путаницы с номерами скобок, можно дать скобкам имя.

Вот тот же пример, но с именем для скобочной группы со словом:

```js run
// скобки названы ?<word>, ссылка на них \k<word>
let reg = /^((?=(?<word>\w+))\k<word>\s?)*$/;

let str = "An input string that takes a long time or even makes this regex to hang!";

alert( reg.test(str) ); // false

alert( reg.test("A correct string") ); // true
```