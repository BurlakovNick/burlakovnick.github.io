---
title: Комбинаторы парсеров на Sprache
header:
  teaser: images/sprache/die_sprache.png
excerpt: Напишем вместе несколько парсеров без бойлерплейта
date: '2018-08-23 00:00:00'
---

Разработчики иногда сталкиваются с задачей разобрать текст (распарсить) в структурированном формате и извлечь из него полезную информацию. Примеры того, что можно парсить — JSON, логи приложения, исходный код на любом языке программирования.

Парсинг (особенно популярных форматов) — [не та задача, которую нужно программировать самостоятельно](https://blog.newrelic.com/engineering/7-things-never-code/). Поэтому каждый программист должен написать несколько парсеров — чтобы повеселиться и никогда так больше не делать, конечно же.

В этой статье расскажу, как можно легко написать свой парсер с помощью библиотеки [Sprache](https://github.com/sprache/Sprache).

## Наивный подход

В одном из домашних проектов мне потребовалось отфильтровать архив текстов, содержащих определенные слова. Фильтры оказалось удобно задавать в виде выражений, похожих на [Must/Should](https://www.elastic.co/guide/en/elasticsearch/guide/current/combining-filters.html) из Elastic Search. Фильтр `Must` требует, чтобы все слова встретились в тексте, `Should` — хотя бы одно. Из простых условий с помощью `Must` и `Should` можно комбинировать сложные.

Условия для фильтра задаются текстом в таком виде:

```
(
    [Греция, Салоники, Родос],
    [Лиссабон, Порту, Португалия],
    Дублин
)
```

Слова для фильтрации разделяются запятыми. Условие `Must` заключается в круглые скобки, `Should` — в квадратные.

Фильтр можно описать простой грамматикой (в [форме Бэкуса — Наура](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form)):

```xml
<word> := <letter> | <letter><word>
<list> := <word> | <word> "," <list>
<should> := "[" <list> "]"
<must> := "(" <list> ")"
<expr> := <word> | <must> | <should>
```

Проще простого! Давайте напишем наивную реализацию парсера. Для этого воспользуемся дедовским подходом, который выручал еще на парах по алгоритмам в университете:
- Запомним текущее слово. Будем дописывать к нему новый символ, если встретим его (и символ не будет скобкой или запятой);
- Заведем стек для запоминания последовательности открытых скобок;
- Вместе с открытой скобкой будем хранить список из построенных фильтров внутри этой скобки.

Посмотрим на реализацию парсера на [github](https://github.com/BurlakovNick/SpracheExamples/blob/master/Parsers/ParserExamples/Example1/NaiveFilterParser.cs).

Мерлинова борода! Получилось не так-то просто. Почти сто строк кода, множество условий, низкоуровневая работа со стеком — ужас!

В наивном подходе я вижу несколько проблем:
- Код парсера лапшеобразный, в него страшно добавлять новые фичи;
- Есть несколько крайних случаев, которые легко пропустить (модульными тестами я отловил 3 бага перед тем, как этот код заработал);
- И код совершенно не отображает грамматику, которую мы разбираем.

Но ведь грамматика языка простая! Не должно быть так сложно. Хочется получить парсер автоматически, просто скормив машине описание грамматики. Можно ли так сделать? Оказывается, можно!

## Монадические комбинаторы парсеров

Один из подходов к построению парсеров — представить простые парсерсы в виде функций, а затем научиться комбинировать их с помощью функций высшего порядка (комбинаторов).

Попытаюсь доступно изложить идею. За хардкором отсылаю к статье — [Hutton, Meijer, Monadic parser combinators](http://www.cs.nott.ac.uk/~pszgmh/monparsing.pdf).

Парсер — это функция, которая принимает строку, пытается распарсить какое-то выражение и говорит, какую часть входной строки осталось разобрать:

```csharp
public delegate IResult<T> Parser<out T>(IInput input);

public interface IResult<T>
{
    bool Success { get; }
    IInput Remainder { get; }
    T ParsedValue { get; }
}

public interface IInput
{
    char Current { get; } 
    IInput Advance();
}
```

`IInput` — итератор по входным данным парсера, помогает получить текущий символ (`Current`) и двинуться дальше (`Advance`).

Простой пример парсера — парсер единственного символа:

```csharp
public static Parser<string> Char(char ch)
{
    return (Parser<string>)(input => {
        if (input.Current == ch)
            return Result.Success<string>(
                input.Current.ToString(),
                input.Advance());
        else
            return Result.Failure<string>(input);
    });   
}
```

Парсер для одиночного символа прост — мы либо встретили нужный символ (и тогда двигаем итератор `input` дальше), либо распарсить символ не получилось. Пользоваться готовым парсером так же легко, как и любым делегатом — нужно лишь вызвать готовый парсер и передать ему на вход какой-то текст, например:

```csharp
var result = Char('a')("abacaba");

//Результат - 'a'
Console.WriteLine(result.Value);
//Результат - "bacaba"
Console.WriteLine(result.Remainder);
```

Из простых парсеров можно собирать сложные с помощью комбинаторов — функций, которые принимают на вход другие парсеры и создают из них нечто большее.

Самый простой комбинатор — `Or`. Например, распарсим букву `a` или букву&nbsp;`b`:

```csharp
public static Parser<T> Or<T>(
    this Parser<T> left,
    Parser<T> right)
{
    return (Parser<string>)(input => {
        var result = left(input);
        return result.Success
            ? result
            : right(input);
    });
}

var parser = Char('a').Or(Char('b'));
```

С помощью комбинатора `Many` можно сделать парсер для слова, состоящего из одних лишь букв:

```csharp
public static Parser<string> Letter()
{
    return (Parser<string>)(input => {
        if (char.IsLetter(input.Current))
            return Result.Success<string>(
                input.Current.ToString(),
                input.Advance());
        else
            return Result.Failure<string>(
                input);
    });
}

public static Parser<IEnumerable<T>> Many<T>(this Parser<T> parser)
{
    return Parser<IEnumerable<T>>(input => {
        var list = new List<T>();
        for (var result = parser(input);
            result.Success;
            result = parser(input))
        {
            list.Add(result.Value);
            input = result.Remainder;
        }
        return Result.Success<IEnumerable<T>>(
            (IEnumerable<T>) list, input);
    });
}

var parser = Letter().Many();
var result = parser("abc123");

//Результат - "abc"
Console.WriteLine(result.Value);
//Результат - "123"
Console.WriteLine(result.Remainder);
```

Или можно скомбинировать парсеры двух строк, идущих друг за другом, с помощью комбинатора `Then`:

```csharp
public static Parser<string> Then(
    this Parser<string> left,
    Func<string, Parser<string>> right)
{
    return (Parser<string>)(input => {
        var result = left(input);
        if (!result.Success)
        {
            return result;
        }
        return right(result.Value)(result.Remainder);
    });
}
```

Здесь вместо парсера справа в `Then` передается делегат `Func<string, Parser<string>>`, аргумент которого — результат предыдущего парсера. Это необходимо, чтобы скомбинировать результаты двух парсеров.

Этих комбинаторов достаточно, чтобы написать парсер слов, заключенных в круглые скобки:

```csharp
var wordParser = 
    Char('(')
    .Then(left => (left, AnyLetter().Many())
    .Then((left, word) => (left, word, Char(')')))
    //левая и правая скобка не нужны, делаем discard параметров
    //спасибо тебе, C# 7.0!
    .Then((_, word, _) => word); 
```

## При чем здесь монады?

Фишка в том, что парсер — это [монада](https://mikhail.io/2016/01/monads-explained-in-csharp/) в чистом виде. Монада — это контейнер некоторого значения. У монады есть:

- Конструктор, который собирает монаду. В случае парсера — конструктор, который создает парсер одного символа;

- Операция связывания (`bind`), которая позволяет комбинировать монады. В случае парсеров — это комбинаторы `Or`, `Then`, `Many` и другие. 

Абстракция монады часто применяется в отложенных вычислениях. Самые известные монады из языка C#:

- Монада списка `IEnumerable<T>` — позволяет выполнять отложенные вычисления над коллекциями элементов с помощью LINQ;

- Монада задачи `Task<T>` — нужна для комбинирования асинхронных операций. 

## Реализация парсера с помощью Sprache

Попробуем реализовать тот же парсер с помощью [Sprache](https://github.com/sprache/Sprache) — легковесной библиотеки, в которой уже реализованы разные комбинаторы парсеров. Исходники смотри на [Github](https://github.com/BurlakovNick/SpracheExamples/blob/master/Parsers/ParserExamples/Example1/FilterParser.cs).

На выходе парсер должен вернуть `IFilter` — комбинацию фильтров `Must` и `Should`, собранную по скобочному выражению. Выражение, которое парсим — это одно слово или выражение `Must` или выражение `Should`.

```csharp
private static Parser<IFilter> Expr => Should.XOr(Must).XOr(Word);
```

Для начала напишем парсер одного слова:

```csharp
private static Parser<IFilter> Word =>
    Parse
        .LetterOrDigit
        .AtLeastOnce()
        .Text()
        .Select(word => new WordFilter(word));
```

Теперь научимся комбинировать слова с помощью разделителей:

```csharp
private static Parser<IEnumerable<IFilter>> List =>
    Parse.Ref(() => Expr)
        .DelimitedBy(Parse.Chars(',', ';'));
```

Метод `Parse.Ref` позволяет неявно сослаться на другой парсер, чтобы разорвать циклическую зависимость (`Expr` -> `Should` -> `List` -> `Expr`). Фактическое вычисление (`() => Expr`) будет отложено до первого запроса к парсеру.

Распарсим выражение в скобках:

```csharp
private static Parser<IFilter> Should =>
    from left in Parse.Char('[')
    from expr in List.Optional()
    from right in Parse.Char(']')
    select new ShouldFilter(expr.GetOrDefault()?.ToList());

private static Parser<IFilter> Must =>
    from left in Parse.Char('(')
    from expr in List.Optional()
    from right in Parse.Char(')')
    select new MustFilter(expr.GetOrDefault()?.ToList());
```

Так, что за ерунда с `from` и `in`? Это хипстерский LINQ синтаксис — компилятор автоматически заменяет такие конструкции на цепочку вызовов, потому что в библиотеке Sprache объявлен метод `SelectMany`. По смыслу он делает то же самое, что и `Then`. То есть код с `from` эквивалентен такой записи:

```csharp
Parse
    .Char('(')
    .Then(left => (left, List.Optional()))
    .Then((left, expr) => (left, expr, Char(')')
    .Then((_, expr, _) =>
        new MustFilter(expr.GetOrDefault()?.ToList()));
```

Синтаксис с `from` намного читаемее, на мой вкус!

Метод `Optional` пропускает часть выражения — она становится необязательной. Метод `GetOrDefault()` нужен, чтобы получить результат парсинга `Optional`-выражения.

И наконец, можно воспользоваться готовым парсером:

```csharp
public static IFilter BuildFromText(string text)
{
    var normalizedText = new string(
        text
        .Where(c => !char.IsWhiteSpace(c))
        .ToArray());
    var input = new Input(normalizedText);
    var parser = Expr;
    var parsed = parser(input);
    if (!parsed.WasSuccessful)
    {
        throw new Exception(
            $"Message: {parsed.Message}, " +
            $"Offset: {parsed.Remainder}");
    }
    return parsed.Value;
}
```

Теперь давайте сравним [наивный парсер](https://github.com/BurlakovNick/SpracheExamples/blob/master/Parsers/ParserExamples/Example1/NaiveFilterParser.cs) и [умный](https://github.com/BurlakovNick/SpracheExamples/blob/master/Parsers/ParserExamples/Example1/FilterParser.cs). Реализация с помощью Sprache компактнее и точь-в-точь совпадает с исходной грамматикой!

## Эффективность

Большой ли оверхед у функционального подхода к написанию парсеров? Попробуем написать бенчмарк с помощью [Benchmark.net](https://github.com/dotnet/BenchmarkDotNet).

Тестировать будем на двух несложных тестах:
- Тест с большой вложенностью скобок. Генерируется так — букву `x` заворачиваем в скобки `()`, полученное выражение заворачиваем в квадратные скобки `[]` — и так далее 500 раз. Итоговое выражение размножить раз 100 и записать в один список.
- Тест с длинными списками. Запишем один большой список из 100 списков, в каждом из которых через запятую 500 раз повторим символ&nbsp;`x`.

Исходный код бенчмарка на [Github](https://github.com/BurlakovNick/SpracheExamples/blob/master/Parsers/ParserBenchmark/Program.cs).

Итоговые результаты:
```
  Method |                     Text |     Mean |     Error |    StdDev |   Median |
-------- |------------------------- |---------:|----------:|----------:|---------:|
 Sprache | (([([(...)])])) [100201] | 655.3 ms |  9.674 ms |  9.049 ms | 652.2 ms |
   Naive | (([([(...)])])) [100201] | 651.9 ms | 12.898 ms | 27.487 ms | 639.8 ms |
 Sprache | ([x,x(...)x,x]) [100201] | 206.8 ms |  1.667 ms |  1.559 ms | 206.4 ms |
   Naive | ([x,x(...)x,x]) [100201] | 209.1 ms |  3.591 ms |  2.999 ms | 208.2 ms |
```

Результаты практически не отличаются друг от друга. Разумеется, результат зависит от грамматики, поэтому если захотите распарсить мегабайтные файлы — лучше напишите свой бенчмарк.

## Пример DSL

Если пример с парсингом скобочных последовательностей слишком простой, то давайте посмотрим на задачку из жизни.

На моей работе я делаю биллинг. Одна из фич биллинга — корректное формирование юридических и платежных документов по сделке с клиентом. Документы формируются по сложным бизнес-правилам, в которых черт ногу сломит. Бизнес-правила активно меняются — вместе с изменениями законодательства, появлением в биллинге новых тарифов и продуктов.

Проблема — поддержкой этих бизнес-правил занимаются, конечно же, разработчики. Это скучно (правила довольно однотипные) и совершенно не тиражируемо (число правил растет, а разработчиков на рынке не так уж и много).

Ок, давайте заберем у разработчиков скучную задачу по поддержке бизнес-правил! И отдадим кому-нибудь еще. Юристам, например, или еще каким-нибудь экспертам в предметной области (кто придумывает эти правила). Но есть один нюанс — юристы не умеют писать код :( 

На помощь приходят [DSL](https://en.wikipedia.org/wiki/Domain-specific_language) — предметно-ориентированный язык. Это специализированный язык для применения в конкретной предметной области. Например, можно придумать свой язык для описания бизнес-правил формирования документов. С лунапарком и девочками!

Представим правило формирования документа на таком языке:

```csharp
ContractRule: {
    Product = Diadoc,
    Logic = {
        if (History.IsOfferScheme)
            return false;

        if (Bill.IsPostpay)
            return false;

        if (History.HasContract)
            return false;

        return Order.HasCloudCert || Order.HasSubscribeOrExtra;
    }
}
```

Правило — набор условий из `if`, `else` и булевых выражений. Эксперт может комбинировать простые знания о сделке — выставлен ли счет на постоплату (`Bill.IsPostpay`) или был ли договор с клиентом (`History.HasContract`), чтобы определить, нужно ли формировать документ по сделке. Проверку простых условий реализует разработчик (они меняются очень редко). А часто меняющиеся бизнес-правила живут отдельной жизнью.

Парсер для правил получается очень короткий. Исходный код [здесь](https://github.com/BurlakovNick/SpracheExamples/blob/master/Parsers/ParserExamples/Example2/Parsers/RuleParser.cs). Меньше 100 строк кода!

В парсере есть несколько ограничений, например:
- Нет поддержки ветви `else`;
- Не поддерживаются скобочные выражения;
- Сильно ограничен набор простых условий по сделке (на деле их намно-о-о-го больше!);
- Не выводятся красивые сообщения об ошибках, если правило не соответствует грамматике.

То есть это больше proof-of-concept, чем production-ready парсер. Но! Весь код системы правил и парсер я написал всего за 2 часа. Без Sprache ушло бы на порядок больше времени, чтобы распарсить такую грамматику. 

Ну и комбинировать парсеры друг с другом с API Sprache — это кайф для разработчика :)

## Итого

Если нужно распарсить сложный текст — не надо изобретать велосипед, используй готовые инструменты. Sprache прекрасно подходит, чтобы быстро наваять парсер.

Исходный код всех примеров есть на [github](https://github.com/BurlakovNick/SpracheExamples). Остались вопросы — пиши, пообщаемся.