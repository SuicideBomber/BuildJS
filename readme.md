# BuildJS

## Что это и зачем это

Случай 1. При создании скрипта на JavaScript всего лишь в несколько сотен строк довольно быстро возникает желание разбить
код на несколько файлов. При тысячах строк это будет уже неизбежно. Как правило, в таких случаях создают несколько
тегов `script` в документе, однако, итоговый скрипт, отдаваемый пользователю, должен находится в одном файле.

Случай 2. У вас есть файл с десятками полезных функций, но в одном из проектов вам из этого файла требуются только две из них.
Поэтому вы стоите перед выбором: или подключать весь файл с десятками килобайт ненужного кода, или выдирать из файла
только нужные функции. А если эти функции зависят от других функций, всё становится ещё хуже.

Для решения этих и подобных проблем предназначен BuildJS. Он занимается сборкой js-файлов или их частей в один файл.

## Использование сборщика на NodeJS

В дальнейшем тут появится более подробное описание. Пока суть. Для сборки файла test.js нужно выполнить команду

    node nodejs/build.js test.js

Можно передавать флаги

    node nodejs/build.js test.js -foo -bar

## Синтаксис

В JavaScript файлах расставляются специальные комментарии, по которым BuildJS понимает, какие файлы нужно подключить
и какие части этих файлов. Комментарии начинаются на `//#`, например, `//#include foo.js`, `//#if debug`,
`//#endif`.

### Подключение кода из внешних файлов

Для подключения кода из другого файла используется директива `//#include`.

    //#include foo.js
    //#include bar.js

    // Используем функции из foo.js и bar.js

По сути, директива `//#include foo.js` заменяется содержимым файла foo.js. Это важно, т.к. если написать данную
директиву в середине файла, то и код из подключаемого файла вставится в середину.

    (function() {
        // Создаём локальную область видимости и подключаем внутрь неё код из других файлов
        //#include foo.js
        // Теперь функции и переменные из foo.js не видны снаружи и не засоряют глобальную область видимости.
    })();

Любой файл подключается единожды в том месте, где встречено первое его упоминание. Например, файл foo.js:

    //#include bar.js
    alert('foo.js');

Файл bar.js:

    alert('bar.js');

Файл index.js:

    alert(1);
    //#include foo.js
    alert(2);
    //#include bar.js
    alert(3);

Результатом обработки будет

    alert(1);
    alert('bar.js');
    alert('foo.js');
    alert(2);
    alert(3);

### Подключение части кода из внешних файлов

Для того, чтобы дать возможность подключать только часть файла, нужно в этом файле отметить области, которые
не включать в результирующий код. Например, создание класса `MyClass`:

    var MyClass = function() {};

    MyClass.prototype.method1 = function() {};

    //#label method2
    // Фича, без которой можно и обойтись.
    MyClass.prototype.method2 = function() {};
    //#endlabel

    //#label method3
    MyClass.prototype.method3 = function() {};
    //#endlabel

    //#label method4
    MyClass.prototype.method4 = function() {};
    //#endlabel

Теперь, если нам нужна только базовая функциональность класса `MyClass`, мы можем подключить только код, который
находится вне меток.

    //#include MyClass.js::base

`base` -- это специальная метка, означающая "подключить неразмеченный код". В результате мы получим класс только
с методом `method1`, без которого функциональность класса невозможна.

Если нам дополнительно нужен метод `method2`, можно его подключить следующим образом:

    //#include MyClass.js::method2

Если методы `method2` и `method4`

    //#include MyClass.js::method2::method4

Вообще, всегда рекомендуется подключать файлы с указанием областей, которые вам нужны. Даже если в файле нет тегов
`label`, следует подключать область `base`. Даже если вам нужен весь файл, следует перечислить все области этого
файла. Т.к. в последующем в этот файл может добавиться дополнительная функциональность, вовсе вам ненужная.

### Подключение областей текущего файла

Т.к. программист, использующий ваш файл, не должен знать зависимостей между различными областями, то нужно
указывать зависимости явно. Например, всё тот же MyClass.js.

    var MyClass = function() {};

    MyClass.prototype.method1 = function() {};

    //#label method2
    // Фича, без которой можно и обойтись.
    MyClass.prototype.method2 = function() {};
    //#endlabel

    //#label method3
    //#include ::method2
    MyClass.prototype.method3 = function() {
        this.method2();
        // ...
    };
    //#endlabel

    //#label method4
    MyClass.prototype.method4 = function() {};
    //#endlabel

В конечном коде нам нужен только метод `method3`, про то, что он использует `method2` мы ничего не знаем, и знать
нам это не надо. Мы просто пишем

    //#include MyClass.js::method3

И `method2` подключится автоматически.

### Подключение внешних файлов в середину скрипта

Очень аккуратно подключайте внешние файлы в середину скрипта. Перепишем MyClass.js в другом виде.

    var MyClass = function() {};

    MyClass.prototype = {
        //#label method2
        // Фича, без которой можно и обойтись.
        method2: function() {},
        //#endlabel

        //#label method3
        //#include ::method2"
        method3: function() {
            this.method2();
            // ...
        },
        //#endlabel

        //#label method4
        //#include String.js::trim
        method4: function() {},
        //#endlabel

        method1: function() {}
    };

В этом примере потенциальная ошибка. Область `method4` требует область `trim` файла String.js, подключение
которой происходит тут же. В результате в сгенерированном коде будет синтаксическая ошибка, т.к. содержимое
файла String.js появится в середине описания объекта. А потенциальной ошибка является потому, что если любая
область файла String.js подключается где-то ранее, то и область `trim` будет подключена там же.

В данном случае в начале файла MyClass.js следует написать

    //#label method4
    //#include String.js::trim
    //#endlabel

### Флаги сборщика

Внутри сборщика могут быть определены флаги, имеющие одно из двух значений: `true` или `false`. При обращении к
несуществующему флагу считается, что он имеет значение false. Флаги могут быть определены как снаружи, при запуске
сборщика, так и изнутри, тегами `//#set` и //#unset. Используются флаги в теге `//#if`.

    //#if debug
    console.log('debug');
    //#endif

    //#set foo

    //#if not foo
    console.log('foo is not true');
    //#endif

## По всем вопроса обращаться

[kolyaj@yandex.ru](mailto:kolyaj@yandex.ru)