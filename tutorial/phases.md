# Теория компиляторов для неофитов
## Устройство компилятора

[Введение <](introduction.md)
[Содержание](content.md)
[> Лексический анализатор](lexer.md)

-------

Чтобы понять устройство компилятора, нужно обратиться к задачам, которые он
решает.
Мы выше дали такое определение компилятора - это программа, преобразующая
программный кода на языке высокого уровня в эквивалентный низкоуровневый код.
Представим себя на месте компилятора и попробуем откомпилировать несложную
программу.

Входными данными для нас будет некий алгоритм, возьмем для примера вычисление
суммы первых `n` членов арифметической прогрессии с первым членом `a` и
разностью `d`.
Первая сложность, с которой мы сталкиваемся, это то, что на вход нам дан не
алгоритм как таковой, а некоторое его представление, обычно текстовое.
Например, это может быть код на Matlab:

```Matlab
sum(a+d*(0:n-1))
```

код на Haskell:

```Haskell
foldl (+) 0 [ a+d*(j-1) | j <- [1..n] ]
```

функция на C:

```C
double sum=0; for(int j=0; j<n; j++) sum+=a+j*d;
```

и т.д.
Как мы видим, исходный код выглядит совершенно различно, хотя реализует один и
тот же алгоритм (сгенерированный код может отличаться порядком операций,
объемом используемой памяти, представлениями чисел и т.п.):

```
sum <- 0
для каждого j из 1 .. n:
  sum <- sum + a + d*(j-1)
```

Этот псевдокод является еще одним представлением алгоритма, другим
представлением могла бы быть математическая формула.
Но чем же является сам алгоритм?
Мы понимаем под алгоритмом последовательность действий, которую необходимо
выполнить для достижения желаемого результата (про философские аспекты
определения алгоритма можно прочитать в [Википедии](https://ru.wikipedia.org/wiki/%D0%90%D0%BB%D0%B3%D0%BE%D1%80%D0%B8%D1%82%D0%BC)).
Одну и туже последовательность действий можно описать различными способами,
поэтому на первом этапе компиляции выполняется преобразование исходного кода в
абстрактное синтаксическое дерево (AST), которое является более удобным
средством для анализа и преобразования алгоритма, чем текстовое представление.
Глядя на наш алгоритм мы видим, что он представляет собой последовательность из
двух последовательно выполняемых операций: присваивания переменной `sum`
начального значения и выполнения цикла.
Это значит, что в корне AST будет операция последовательного выполнения частей алгоритма (назовем ее `затем`), а операндами для нее будут две части нашего алгоритма.

```
AST = затем(блок1, блок2)
```

Первый подалгоритм `блок1` состоит одного оператора присваивания (назовем его
`присвоить`), первым аргументом которого будет имя переменной (являющееся
листом AST), а вторым присваиваемое значение, которое в рассматриваемом случае
просто число, потому также является листом.

```
блок1 = присвоить(sum, 0)
```

Вторая часть `блок2` нашего алгоритма имеет корнем оператор выполнения цикла
(назовем его `цикл`), у которого первым аргументом (порядок нумерации
произволен) будет блок кода, выполняемый внутри цикла, в остальными параметрами
будут имя изменяемой в цикле переменной, начальное значение и конечное значение:

```
блок2 = цикл(блок3, j, 1, n)
```

Выполняемое в цикле выражение состоит из знакомого нам оператора присваивания,
однако присваиваемое значения в этот раз нетривиально, обозначим его через
`выражение1`:

```
блок3 = присвоить(sum, выражение1)
```

Присваеваемое выражение является композицией арифметических операций (им
соответствуют узлы, обозначаемые символом операции) и операций подстановки
значения переменной (назовем ее `значениеПеременной`), у которой один
аргумент - имя переменной:

```
выражение1 = +(значениеПеременной(sum), выражение2)
выражение2 = +(значениеПеременной(a), выражение3)
выражение3 = *(значениеПеременной(d), выражение4)
выражение4 = -(значениеПеременной(j), 1)
```

Полученное древовидное представление алгоритма тяжелее воспринимается человеком,
однако оно лучше передает структуру алгоритма, явно выделяет все выполняемые
операции, явно указывает операнды.
Для выполнения записанного в виде AST алгоритма достаточно указать для каждого
типа узла примитивное действие, и последовательно применить эти действия для
каждого узла (порядок обхода узлов важен).

Для построения AST обычно используется промежуточное представление программы,
называемое деревом разбора.
Дерево разбора однозначно соответствует текстовой записи программы и строится
на основе формальной грамматики языка исходного кода.
Формальная грамматика задается набором правил подстановки подстрок вместо
других подстрок:

```EBNF
что заменяется = на что заменяется
```

В теории формальных грамматик рассматривают строки, включающие в себя не только
символы, из которых построен исходной текст программы, но и так называемые
*нетерминалы*, которые нужны только для задания порядка применения правил.
При конструировании строки порядок применения правил ничем не ограничен,
применение правил также не обязательно (вы не можете запретить применять
правила, или требовать применять их только в указанном порядке).
Как только в конструируемой строке не остается нетерминалов, строка считается
строкой из языка, на котором написан исходный код.

При построении компиляторов чаще всего используются *контекстно свободные
грамматики*, которые содержат только правила, в левой части которых содержится
только один символ и он нетерминал.
Этот нетерминал приблизительно соответствует узлу AST.
Например, мы можем определить синтаксис, соответствующих предложенному
псевдокоду" для всех узлов AST из примера следующим образом (у предлагаемого
варианта масса недостатков, на следующем уроке мы узнаем, как сделать
хорошую грамматику):

```EBNF
затем = блок блок
блок = затем | присвоить | цикл
присвоить = переменная "<-" выражение
цикл = "для каждого" переменная "из" выражение ".." выражение: блок
выражение = число | переменная | ( выражение ) | выражение операция выражение
операция = "+" | "*" | "-" | "/"
```

Как мы видим, между деревом разбора и AST нет прямого соответствия,
например, дерево разбора содержит правило для выражения в скобках, но в AST
скобки никак не отражены.
Это связано с тем, что скобки в текстовой записи нужны только для группировки,
(для указания порядка вычислений), в AST же порядок вычислений зафиксирован
структурой дерева.
Одно и тоже AST может соответствовать нескольким текстовым представлениям,
и соответственно нескольким деревьям разбора,
например, `a+b+c` и `a+(b+c)` имеют одинаковое AST, но дерево разбора
для второго выражения содержит дополнительный узел со скобками.

Этап преобразования текста программы в однозначно ему соответствующее
дерево разбора называется синтаксическим анализом (включающим в себя лексический
анализ, как мы увидим в следующем уроке) или парсингом.
Часть компилятора, выполняющая это преобразование, называется синтаксическим
анализатором или парсером.
Часто парсер не создает дерево разбора явно, а сразу строит AST для экономии
памяти.

Построенное по исходному коду AST не всегда имеет смысл, так как код может
содержать семантические ошибки.
Например, в нашем примере код мог ссылаться на несуществующую переменную.
Компилятор перед выполнением дальнейших преобразований AST должен проверить
его корректность.

Подойдя к этому шагу компилятор не обязательно извлек всю семантическую
информацию из исходного кода, так как часть AST до сих пор не обработана.
Например, имена переменных не нужны для генерации кода, они нужны только
для облегчения чтения кода человеком, поэтому компилятор может заменить
все упоминания переменные в AST на ссылку на объект, хранящий информацию о
переменной или на порядковый номер переменной.
На данном этапе можно учесть области видимости переменных, сгенерировать
таблицы переменных, разобрать пользовательские типы данных и т.п.

На следующем шаге AST преобразуется к эквивалентному, но предпочтительному
представлению.
Рассмотрим основные задачи такого преобразования.
Языки программирования обычно содержат *синтаксический сахар*,
т.о. команды, операторы, конструкции языка, которые могут выражены через
другие конструкции языка, но наличие которых желательно, так как они сокращают
код или улучшают его читаемость.
Например, к синтаксическому сахару можно отнести циклы, лямбда функции,
абстракцию списков и т.п.
Для упрощения последующих стадий компиляции на этом шаге имеет смысл
развернуть синтаксический сахар в некий базовый набор инструкций.
Например, в рассматриваемом примере можно заменить конструкцию цикла на
условный переход:

```
sum <- 0
j <- 1
метка:
sum <- sum + a + d*(j-1)
j <- j + 1
если j<=n, то перейти на метку
```

В текстовом представлении такое преобразование нетривиально, однако в виде AST
достаточно заменить один узел `цикл(блок,переменная,от,до)` на эквивалентное
дерево:

```
затем(присвоить(переменная
               ,от)
     ,затем(блок
           ,затем(присвоить(переменная
                           ,+(значениеПеременной(переменная)
                             ,1
                             )
           ,если(небольше(значениеПеременной(j)
                         ,значениеПеременной(до))
                ,перейтивыше(2))))))
```

Здесь мы ввели несколько вспомогательных типов узлов AST, которые обычно
присутствуют в императивных языках управления:
`небольше` для сравнения чисел, `если` для условного выполнения блока,
`перейтивыше` для продолжения выполнения с предка на указанной число уровней
выше.

Другой важной задачей, для решения которой требуется преобразование AST,
является оптимизация кода.
Например, компилятор может найти значения выражений, которые можно вычислить
на этапе компиляции, упростить выражения вроде `0*a` и `a-a`, убрать мертвый код
и неиспользуемые переменные и т.п.

Построенное к этому моменту AST имеет узлы, выражающие операции, естественные
для языка исходного кода, однако операции, допустимые в исходном коде, могут
быть весьма далеки от реализуемых на аппаратном уровне инструкций, например,
если исходный текст написан на функциональном языке программирования, а код
генерируется для x86 совместимых процессоров.
В это случае имеет смысл произвести замену AST для исходного языка на AST для
генерируемого кода.
Такого рода преобразование может, как и в случае устранения синтаксического
сахара, сводится к замене услов AST на эквивалентные поддеревья с другим
набором типов узлов.

Следующей фазой компиляции является кодогенерация.
На этом этапе AST преобразуется в промежуточный код (IR),
содержащий инструкции очень близкие к реализованным аппаратно.
На этом этапе решаются вопросы размещения переменный в регистрах и памяти,
узлы AST заменяются на эквивалентные последовательности инструкций,
инструкции объединяются в блоки, находятся переходы между блоками.
На данном этапе часто используется
[SSA](https://ru.wikipedia.org/wiki/SSA)
представление кода, в рамках которого каждой переменной значение
в рамках одного блока можно присвоить только один раз.
SSA представление позволяет эффективно проводить некоторые виды оптимизации,
например, легко находить мертвый код, так как он соответствует вычисляемым,
но не используемым переменным.
Вернемся к нашему примеру и преобразуем AST в псевдокод в SSA форме:

```LLVM
start:
%n = ?
%a = ?
%d = ?
%sum0 = 0
%j0 = 1
loop:
%sum = phi [%sum0, label %start] [%sum2, label %loop]
%j = phi [%j0, label %start] [%j3, label %loop]
%floatone = 1.0
%j1 = float(%j)
%j2 = %j1 - %floatone
%prod = %d * %j2
%sum1 = %sum + %a
%sum2 = %sum1 + %prod
%integerone = 1
%j3 = %j + %integerone
%b = %j3 <= %n
br %b, label %loop
```

Нам пришлось ввести обозначения для регистров, хранящих промежуточные значения,
что необходимо в SSA кода.
Имена регистров и метки мы начинали с символа `%` для улучшения читаемости кода.
Набор инструкций процессора обычно очень ограничен, для примера мы предположили,
что арифметические операторы работают только с регистрами, т.е. константа не
может быть аргументом арифметической операции, поэтом нам пришлось сначала
загрузить числовые значения в регистры.
Регистры могут хранить разные типы данных, над которыми возможны разные
операции.
Преобразование типа операция, которая должна выполняться явно, в нашем случае
мы ввели инструкцию `float` для преобразования целого в число с плавающей
запятой.
Инструкция `br` на последней строке выполняет условный переход, если `%b` имеет
значение истины.
Использование условного перехода потребовали использовать `phi` узел, который
позволяет сохранить в регистр `%sum` значение регистра `%sum0`, если мы вошли в
блок `loop` из блока `start`, и значение `%sum2`, если мы сделали переход из
блока `loop`.
Без использования узла `phi` мы бы не смогли изменить значение переменной цикла.

Код в SSA представлении может использовать неограниченное число регистров,
однако реально существующие процессоры имеют конечное число регистров, поэтому
при кодогенерации компилятор должен отобразить регистры SSA на регистры
процессора.
Блок `loop` в нашем SSA коде использует 10 регистров.
Если процессор содержит меньшее число регистров, то компилятор должен добавить
инструкции, сохраняющие значения регистров в память и восстанавливающие их при
необходимости.
Однако использовать 10 регистров явно избыточно, так как например, переменная
`%floatone` используется только один раз, после чего ее значение можно в
регистре не хранить.
Аналогично можно подсчитать число одновременно используемых регистров,
причем нужно считать отдельно целочисленные регистры и регистры для чисел с
плавающей запятой:

```LLVM
; используется регистров целочисленных ; с плавающей запятой
start:
%n = ? ; 1 ; 0
%a = ? ; 1 ; 1
%d = ? ; 1 ; 2
%sum0 = 0 ; 1 ; 3
%j0 = 1 ; 2 ; 3
loop:
%sum = phi [%sum0, label %start] [%sum2, label %loop] ; 2 ; 3
%j = phi [%j0, label %start] [%j3, label %loop] ; 2 ; 3
%floatone = 1.0 ; 2 ; 4
%j1 = float(%j) ; 2 ; 5
%j2 = %j1 - %floatone ; 2 ; 4
%prod = %d * %j2 ; 2 ; 4
%sum1 = %sum + %a ; 2 ; 4
%sum2 = %sum1 + %prod ; 2 ; 3
%integerone = 1 ; 3 ; 3
%j3 = %j + %integerone ; 2 ; 3
%b = %j3 <= %n ; 3 ; 3
br %b, label %loop ; 2 ; 3
```

Таким образом, максимально используется 3 целочисленных регистра и 5 регистров
с плавающей запятой.
Заметим, что число используемых регистров зависит от порядка выполнения
инструкций, т.e. компилятор должен пытаться переупорядочивать инструкции, чтобы
уменьшить использование медленной памяти для хранения значений, не помещающихся
в регистры.
На этом этапе можно провести много типов оптимизации, например, современные
процессоры имеют несколько арифметико-логических устройств, поэтому они могут
выполнять несколько арифметических операций одновременно, и задачей компилятора
является поиск перестановки инструкций, дающей максимальное распараллеливание
вычислений.
Мы ограничимся преобразованием программы из SSA формы в машинный инструкции,
переиспользующие целочисленные регистры `%in` и регистры для чисел с плавающей
запятой `%fn`. Заметим, что при этом преобразовании `phi` узлы пропадают:

```LLVM
start:
%i1 = ? ; %n
%f1 = ? ; %a
%f2 = ? ; %d
%f3 = 0 ; %sum0 = %sum2 = %sum
%i2 = 1 ; %j0 = %j = %j3
loop:
%f4 = 1.0 ; %floatone
%f5 = float(%i2) ; %j1
%f4 = %f5 - %f4 ; %j2
%f4 = %f2 * %f4 ; %prod
%f3 = %f3 + %f1 ; %sum1
%f3 = %f3 + %f4 ; %sum2
%i3 = 1 ; %integerone
%i2 = %i2 + %i3 ; %j3
%i3 = %i2 <= %1 ; %b
br %i3, label %loop
```

Мы выполнили много шагов компиляции и на данный момент имеем почти готовый
машинный код, в котором мы однако сохранили метки, не выяснили, где в памяти
будут хранится инструкции и не оформили код в виде исполняемого модуля,
который может запустить операционная система.
Эти шаги делаются на последнем этапе, которые часто отделяют от компиляции
и называют компоновкой.
Компиляция, особенно с оптимизацией, является весьма ресурсоемкой операцией,
поэтому обычно ее выполняют не для всех программы целиком, а для отдельных
ее частей: модулей или классов.
Компоновка же является относительно простой операцией (если не выполняется
межмодульная оптимизация), поэтому ее можно эффективно провести сразу над
всеми модулями.
Результат компиляции в этом случае называют объектным файлом (или модулем,
или кодом), а результат компоновки - исполняемым файлом.
Отделение этапа компоновки от компиляции также позволяет распространять
программные библиотеки не распространяя исходный код.
Для этого создается библиотечный файл (dll, so и др.),
содержащий объектный код и названия функций.
При компоновки кода ссылки в пользовательском коде на имена функций заменяются
на адреса этих функций в памяти.

Приведем окончательный результат компиляции нашей программы,
предполагая для простоты, что каждая инструкция занимает ровно один байт,
и что инструкции располагаются с нулевого адреса:

```LLVM
0000: %i1 = ? ; start
0001: %f1 = ?
0002: %f2 = ?
0003: %f3 = 0
0004: %i2 = 1
0005: %f4 = 1.0 ; loop
0006: %f5 = float(%i2)
0007: %f4 = %f5 - %f4
0008: %f4 = %f2 * %f4
0009: %f3 = %f3 + %f1
000a: %f3 = %f3 + %f4
000b: %i3 = 1
000c: %i2 = %i2 + %i3
000d: %i3 = %i2 <= %1
000e: br %i3, 0005
```

Разобрав выполняемых компилятором последовательность действий, мы пришли
к следующим фазам компиляции:

* Синтаксический анализ и генерация AST.
* Проверка корректности программы, генерация вспомогательных структур.
* Преобразование AST, оптимизация алгоритма.
* Кодогенерация, оптимизация кода.
* Компоновка.

Перечисленные шаги являются ориентировочными, их число и порядок может
отличаться в разных компиляторах.
Фазы компиляции могут выполняться отдельными программами,
причем части компилятора могут переиспользоваться разными компиляторами.
Например, разные компилятора языка С могут использовать один синтаксический
анализатор.
Компиляторы кода процессора x86 с разных языков могут использовать
один кодогенератор.
Компоновцик может собирать объектный код, сгенерированный компиляторами
разных языков.
Многие виды оптимизации могут использоваться компиляторами многих языков
под многие архитектуры.
Примером такого модульного компилятора является LLVM-CLang.

Поздравляю, вы добрались до конца урока и имеете общее представление
о работе компилятора.
Если что-то вам показалось непонятным или было отмечено только мельком,
не переживайте, мы рассмотрим каждый этап компиляции подробно в следующих
уроках.

### Вопросы для закрепления

* Что такое абстрактное синтаксическое дерево?
* Зачем нужна формальная грамматика для языка программирования?
* Чем отличается дерево разбора и абстрактное синтаксическое дерево?
* Зачем нужен промежуточный код? Чем удобно SSA представление?
* Что такое объектный код? Зачем нужна компоновка?

### Задания

* Найдите минимальный набор узлов AST, необходимых для реализации калькулятора,
  поддерживающего арифметику, тригонометрические функции и ветвления.
* Найдите десять эквивалентные преобразования AST для калькулятора.
* Предложите набор инструкций в SSA представлении, который реализует узлы AST
  калькулятора.

### Список литературы

* Альфред В. Ахо, Моника С. Лам, Рави Сети, Джеффри Д. Ульман. Компиляторы: принципы, технологии и инструментарий = Compilers: Principles, Techniques, and Tools. — 2 изд. — М.: Вильямс, 2008.

* Робин Хантер. Основные концепции компиляторов = The Essence of Compilers. — М.: Вильямс, 2002. — С. 256.

* [LLVM Language Reference Manual](http://llvm.org/docs/LangRef.html)


-------

[Введение <](introduction.md)
[Содержание](content.md)
[> Лексический анализатор](lexer.md)
