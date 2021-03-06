---
title: Полнотекстовый поиск с Elastic Search
header:
  teaser: images/elastic-search/search_example.png
excerpt: Несколько рецептов приготовления
date: '2018-07-11 00:00:00'
---

В проекте, который я разрабатываю в [Контуре](https://kontur.ru/), мы решили сделать полнотекстовый поиск по основным сущностям системы. Мы не изобрели велосипед — взяли [Elastic Search](https://www.elastic.co/products/elasticsearch) и дотюнили до наших нужд. Elastic Search дает богатые возможности для полнотекстового поиска, предоставляет шардирование и репликацию данных, в общем — классный инструмент. Логика поиска с использованием Elastic умещается всего в пару сотен строк кода. Код компактный, но за ним спрятано парочка хаков и понимание устройства Elastic Search. В статье хочу рассказать о фишках в организации поиска.

## Что ищем

Мой проект — это внутренний биллинг компании (Контур.Биллинг), один из пользователей — продавцы. Продавцу нужно быстро найти клиента в нашей системе по любой информации, которая у него есть — ФИО, номер телефона, email, ИНН, номер заказа. Для продавца поиск выглядит примерно так:

![Пример поискового запроса](/images/elastic-search/autocomplete.gif 'Пример поискового запроса'){: .align-center}

Строка поиска — обычный autocomplete. Пока пользователь набирает запрос, всплывают подсказки с найденными клиентами.

Кроме записей о клиенте, в поиске ищут заявки по работе с клиентом, заказы, контакты, счета, юридические документы и прочее.

Для каждой сущности системы мы храним документ из нескольких полей:
  - Текст, по которому можно найти документ;
  - Мета-информация о документе — например, тип документа;
  - Список пользователей, которые могут видеть документ.

Текст по-разному формируется для разных сущностей — по клиенту это ИНН-КПП клиента и название организации (Контур работает в B2B, поэтому большинство клиентов идентифицируются по реквизитам), для заявки — ФИО клиента и телефон, который он оставил в заявке, и так далее.

## Индексирование данных

Данные в Биллинге хранятся в нескольких базах данных, в основном — в&nbsp;Microsoft SQL и Apache Cassandra. Есть индексирующий процесс, который просыпается по [расписанию](https://ru.wikipedia.org/wiki/Cron), вычитывает изменившиеся данные из базы, отправляет их в Elastic. Elastic хранит лишь копию данных, необходимых для поиска.

В чем плюсы такого подхода:
  - В отличие от синхронной записи (записали в БД — сразу записали в Elastic) получаем дополнительную отказоустойчивость. Бывает так, что Elastic тупит и не может записать данные ([долго собирает мусор](https://ru.wikipedia.org/wiki/%D0%A1%D0%B1%D0%BE%D1%80%D0%BA%D0%B0_%D0%BC%D1%83%D1%81%D0%BE%D1%80%D0%B0), [тупанула сеть](https://aphyr.com/posts/288-the-network-is-reliable)). Что делать при синхронной записи неясно — данные уже есть в БД, а в Elastic нет, транзакционно записать в Elastic нельзя. Асинхронный процесс гарантирует [eventual consistency](https://en.wikipedia.org/wiki/Eventual_consistency) — данные в конечном счете окажутся в Elastic. Если не получилось записать сразу, то процесс повторит попытку позже;
  - Elastic не используется как первичное хранилище. Данные можно безболезненно потерять и пересобрать индекс заново. Пару раз это здорово выручало меня, когда делал изменение схемы данных — я забил на поддержку обратной совместимости, создал новый индекс, накачал его данными и переключил пользователей на чтение из нового индекса.

В чем минусы:
  - Есть задержка на появление данных в поиске, поскольку индексирующий процесс работает по расписанию. Задержку можно уменьшать, настраивая время запуска процесса;
  - Конкретно в нашей однопоточной схеме — индексация слишком медленная, если данных много. В Биллинге небольшой индекс на десятки гигабайт и несколько десятков миллионов документов. Его индексация занимает 10-12 часов. Не слишком быстро — но пока нас устраивает.

Есть прикольный альтернативный подход к индексации с помощью очереди сообщений — записываем все события по изменению сущностей в очередь [Kafka](http://kafka.apache.org/), а потом несколько индексирующих процессов разгребают очередь сообщений и индексируют документы в Elastic. Подход с очередью лучше масштабируется. Пример индексации с помощью очереди можно посмотреть на [Github](https://github.com/BigDataDevs/kafka-elasticsearch-consumer).

## Анализ текста

Основная структура данных в Elastic — это [инвертированный индекс](https://ru.wikipedia.org/wiki/%D0%98%D0%BD%D0%B2%D0%B5%D1%80%D1%82%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%BD%D1%8B%D0%B9_%D0%B8%D0%BD%D0%B4%D0%B5%D0%BA%D1%81). Это индекс, в котором для каждого слова хранится, в каких документах оно встречается.

![Инвертированный индекс](/images/elastic-search/inverted-index.svg 'Инвертированный индекс'){: .align-center}

Такой индекс эффективен, когда мы ищем документы с вхождением слова. Если нужно найти вхождения комбинации слов, то можно взять списки для каждого слова и пересечь их.

Чтобы построить такой индекс, Elastic прогоняет текст через несколько шагов: 

![Этапы анализа](/images/elastic-search/analysis.png 'Этапы анализа'){: .align-center}

  1. CharFilter — фильтрация входных данных. Здесь отбрасываются символы, которые не несут полезной информации для поиска, например, служебные символы, html-верстка.
  2. Tokenizer — токенизация, то есть разбиение текста на слова. 
  3. TokenFilter — преобразование полученных слов. Например, каждое слово можно привести к нижнему регистру или заменить на слово-синоним. Можно вообще выкинуть слово из индекса, например, если это нецензурное слово.

## Поиск

Поисковый запрос в Elastic состоит из двух частей — [Filter и Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html).
  - Filter — отвечает на вопрос "подходит ли документ под условия поиска";
  - Query — "насколько хорошо документ подходит под условия поиска".

Отличие Query в том, что кроме формальной проверки "подходит" — "не&nbsp;подходит", вычисляется еще и [релевантность](https://www.elastic.co/guide/en/elasticsearch/guide/current/scoring-theory.html) подходящего документа. Все найденные документы затем ранжируются по релевантности. [Формула релевантности](https://www.elastic.co/guide/en/elasticsearch/guide/current/practical-scoring-function.html) — хитрый матанализ, но коротко поведение функции описывается так:
  - Релевантнее те документы, где больше вхождений искомых слов;
  - Менее релевантны те документы, где встречаются самые популярные слова в индексе. Например, союзы, предлоги, вводные слова встречаются во всех текстах — и слабо влияют на релевантность документа;
  - Короткие тексты более релевантны (вероятность встретить искомое слово в коротком тексте меньше, чем в длинном).

Самые простые способы найти что-то — запросы match и phrase.

[Match](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html) запрос — принимает на вход текст запроса, анализирует его, ищет документы со словами из текста. Например, если хотим найти все слова из запроса, то подойдет такой запрос:

```json
{
    "query": {
        "match" : {
            "message" : {
                "query" : "все слова должны встретиться в документе",
                "operator" : "and"
            }
        }
    }
}
```

[Match Phrase](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase.html) — то же самое, что Match, но требует от документа, чтобы слова встречались в правильном порядке. Не все слова обязаны идти строго друг за другом в тексте, можно настроить число слов, которые разделяют два искомых слова во фразе, с помощью параметра `slop`.

```json
{
    "query": {
        "match_phrase" : {
            "message" : "ищем точное вхождение фразы в тексте",
            "slop": 0
        }
    }
}
```

[Match Phrase Prefix](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase-prefix.html) — в отличие от Match Phrase, у последнего слова ищем совпадение префикса.

```json
{
    "query": {
        "match_phrase_prefix" : {
            "message" : {
                "query" : "ищем вхожде",
                "max_expansions" : 10
            }
        }
    }
}
```

Для простого autocomplete подходит `match_phrase_prefix`. Однако, этот запрос стоит использоваться с осторожностью — он работает недетерминировано, поскольку выбирает лишь `max_expansions` слов в индексе, которые начинаются с вхождения префикса (в нашем запросе — `вхожде`), а потом ищет документы с такими словами. При слишком маленьком max_expansions пользователь не найден нужный документ, при слишком большом — поиск будет работать медленно.

## Авторизация запроса

Биллинг хранит чувствительные данные компании. Поэтому любой запрос пользователя авторизуется. Для авторизации доступа к документам в поиске мы используем паттерн Access Control List.

[Access Control List](https://ru.wikipedia.org/wiki/ACL) (ACL) — паттерн для избирательного предоставления доступа к документу. В документе мы сохраняем список пользователей, которым доступен этот документ. В Биллинге размер ACL ограничен десятком пользователей, поэтому документ получается не слишком пухлый. Авторизация по ACL делается так — к любому запросу пользователя в Elastic добавляется запрос по вложенному документу [(Nested query)](https://www.elastic.co/guide/en/elasticsearch/reference/6.0//query-dsl-nested-query.html).

Пример фильтра:
```json
{
  "filter": [{
    "nested": {
      "path": "accessControlList",
      "query": {
        "bool": {
          "filter": [{
            "term": {
              "accessControlList.userId": {
                "value": "d32c608c4d484a058bcf759e3c68eb28"
              }
            }
          }]
        }
      }
    }
  }]
}
```

В Nested-фильтре указан путь `path` до вложенного документа. Во вложенном запросе фильтруем по `userId` — идентификатору пользователя.

## "Объясни" — API 

Для неискушенного инженера поиск в Elastic работает как магия. Иногда категорически непонятно, почему документ подошел под критерии поиска. Мне кажется, я потратил человеко-дни на медитацию над некоторыми запросами, когда только начинал изучать Elastic.

Чтобы понимать работу Elastic, не нужно разбираться в его исходниках. Разработчики дали два удобных API:
  - [Analyze API](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-analyze.html) — прогоняет текст через указанный анализатор и показывает, какие слова Elastic сохранит в индекс.
  - [Explain API](https://www.elastic.co/guide/en/elasticsearch/reference/6.3/search-explain.html) — объясняет, почему документ подходит или не подходит под критерии поиска, показывает релевантность документа и как она вычислена. 

Я использую эти API для отладки:
  - Если настраиваю свой анализатор и хочу проверить, как она работает (особенно если где-то фигурируют регулярные выражения);
  - Когда поиск не находит нужный документ или находит лишний;
  - Когда находятся правильные документы, но более релевантные оказываются в выдаче ниже менее релевантных. Тогда лезу в&nbsp;Explain&nbsp;API и зарываюсь в формулу расчета релевантности.

## Тюним удобство поиска

Сделать поиск удобным и интуитивным для пользователя можно с помощью настроек анализа.

Пример — поиск по названию организации. Не всегда пользователь точно знает, как название организации сохранено в Биллинге. Попробуйте запомнить название такой организации, как ООО "Союз святого Иоанна&nbsp;Воина!" Ок, будем искать неточные совпадения — например, совпадения по подстроке.

Настройка ниже включает поиск по подстрокам ([ngram'ам](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-ngram-tokenizer.html)).

```json
{
  "analysis": {
    "filter": {
      "tokenfilter_ngram": {
        "type": "nGram",
        "min_gram": "2",
        "max_gram": "20"
      }
    }
  }
}
```

При индексировании для каждого слова будут сохранены все его подстроки, длинной от `min_gram` до `max_gram`. Из ngram'ов в сочетании с запросом `match` можно состряпать неплохое автодополнение поиска.

Попробуем отправить запрос анализатору в Elastic:

```json
POST /index/_analyze

{
  "tokenizer": "tokenfilter_ngram",
  "text": "foo bar"
}
```

И увидим все возможные подстроки длины 2 и 3 (поскольку параметр `min_gram` равен 2). Часть ответа Elastic скрыта для наглядности

```json
{
  "tokens": [
    {
      "token": "fo"
    },
    {
      "token": "foo"
    },
    {
      "token": "oo"
    },
    {
      "token": "ba"
    },
    {
      "token": "bar"
    },
    {
      "token": "ar"
    },
  ]
}
```

Другой пример из Биллинга — пользователи ищут клиента по номеру телефона. Российские номера телефона начинаются с `+7` или `8`, при этом не важно, как хранится телефон в базе данных — хочется его найти и с `+7`, и с `8`.

Сделаем замену при индексации с регулярными выражениями с помощью [Pattern Replace Token Filter](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-pattern_replace-tokenfilter.html) и [Pattern Capture Token Filter](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-pattern-capture-tokenfilter.html).

Для этого возьмем исходный телефон (номер из 11 цифр, начинающийся с 7 или 8), заменим его на номер, начинающийся с `7`, с `8`, и без ведущего знака.

Плюсик не фигурирует в регулярках, потому что отбрасывается при индексировании как разделитель слов.

```json
{
  "analysis": {
    "filter": {
      "phone_multiplier": {
        "pattern": "(?<!\\d)[7|8](\\d{10})(?!\\d)",
        "type": "pattern_replace",
        "replacement": "7$1 8$1 $1"
      },
      "phone_splitter": {
        "type": "pattern_capture",
        "preserve_original": "false",
        "patterns": [
          "(7(?<number>\\d{10}))\\s(8\\k<number>)\\s(\\k<number>)"
        ]
      }
    }
  }
}
```

Как говорится в одной старой шутке:

> Если у вас есть проблема и вы решили использовать регулярные выражения, у вас уже две проблемы.

Еще один пример, где анализ упрощает жизнь — буквы `е` и `ё`. Пока зануды спорят, нужно ли в веб-сервисах использовать `ё`, мы поддержали преобразование всех `ё` в `е`, как при индексировании документов, так и при запросах пользователя. Преобразование легко сделать в коде приложения, но можно и вынести в настройку Elastic:

```json
{
  "analysis": {
    "filter:" {
      "char_filter": {
        "e_mapping": {
          "type": "mapping",
          "mapping": ["Ё=>Е", "ё=>е"]
        }
      }
    }
  }
}
```

Вишенка на торте — переключение раскладки за пользователя. Неудобно, когда начинаешь набирать русский текст на английском, забыв переключить раскладку. Биллинг пробует переключить раскладку за пользователя. Работает это на стороне приложения так — если в запросе нет русских символов, и под условия поиска не подошел ни один документ, то приложение пробует повторить запрос, но изменив английские символы на русские — в предположении, что у пользователя qwerty-раскладка клавиатуры. Работает как часы.

![Преобразование текста запроса](/images/elastic-search/eng_rus_convert.png 'Преобразование текста запроса'){: .align-center}

## Настраиваем релевантность

В выдаче поиска есть документы разных типов — все они выводятся в одном списке. Некоторые типы документов важнее других, их нужно поднимать в выдаче. Мы чуть-чуть подкрутили поиск, исходя из того, что пользователи ищут чаще:
  - Клиенты важнее, чем все остальное (заказы, документы, контакты и прочее);
  - Заказы не так важны, как клиенты, но важнее чем все остальное;
  - Точное вхождение намного лучше, чем совпадение подстрок.

Мы настроили приоритеты документов с помощью тюнинга запросов. В [формуле расчета релевантности](https://www.elastic.co/guide/en/elasticsearch/guide/current/practical-scoring-function.html) для каждого найденного слова настраивается [boost](https://www.elastic.co/guide/en/elasticsearch/guide/current/query-time-boosting.html) — множитель, который увеличивает вес документа, если слово встретилось в нем. 

```json
{
  "query": {
    "bool": {
      "minimum_should_match": 0,
      "should": [
        {
          "term": {
            "entityType": {
              "value": "client",
              "boost": 100
            }
          }
        },
        {
          "term": {
            "entityType": {
              "value": "bills",
              "boost": 50
            }
          }
        },
        {
          "match": {
            "textExact": {
              "value": "запрос пользователя",
              "operator": "and",
              "boost": 200
            }
          }
        }
      ]
    }
  }
}
```

Все запросы мы завернули в `should` с `minumum_should_match`, равным 0. Ни один из этих запросов не влияет на то, подойдет ли документ под критерии поиска, но каждый влияет на релевантность документа. Последний `match` ищет запрос пользователя в поле `textExact` — специальное поле, которое подвергается минимуму анализа, например, не разбивается на ngram'ы. Если запрос пользователя находится в почти не тронутом анализатором тексте — скорей всего, это именно то, что нужно пользователю.

С настройкой `boost` по типу документа есть проблема. Релевантность в Elastic зависит от редкости слова и длины документа. Например, в Биллинге на порядок больше клиентов, чем заявок на работу с клиентами — поэтому Elastic сам поднимает вес заявок, несмотря на множитель `boost`. Плюс короткие документы более релевантны. Все это приводит к тому, что некоторых клиентов почти невозможно найти, пока не введешь в поиск точное совпадение.

Можно решить проблему, дальше подтюнивая `boost`. Можно переделать UI, чтобы разделить найденные документы по типу. Мы решили пойти третьим путем — заигнорили формулу расчета релевантности при поиске по типу документа с помощью запроса [constant_score](https://www.elastic.co/guide/en/elasticsearch/guide/current/ignoring-tfidf.html). Этот запрос дает фиксированный вес в формуле подсчета релевантности, если документ удовлетворяет запросу. В итоге, поиск работает намного предсказуемее, упорядочивает документы детерминировано.

Последний ингридиент — `should` запрос по тексту с вычислением релевантности "по-честному". Он нужен, чтобы ранжировать документы одинакового типа (Elastic справляется с этим прекрасно). Веса подобраны так, что `constant_score` запросы имеют на порядок больше вес, чтобы не нарушать порядок документов по их типу.

```json
{
  "query": {
    "bool": {
      "minimum_should_match": 0,
      "should": [
        {
          "constant_score": {
            "query": {
              "term": {
                "entityType": {
                  "value": "client",
                  "boost": 200
                }
              }
            }
          }          
        },
        {
          "constant_score": {
            "query": {
              "term": {
                "entityType": {
                  "value": "bills",
                  "boost": 100
                }
              }
            }
          } 
        },
        {
          "constant_score": {
            "query": {
              "match": {
                "textExact": {
                  "value": "текст запроса",
                  "boost": 120,                  
                  "operator": "and"
                }
              }
            }
          } 
        },
        {
          "match": {
            "text": {
              "value": "текст запроса",
            }
          }
        }
      ]
    }
  }
}
```

## TL;DR

Итоговые рекомендации по приготовлению ElasticSearch от шефа:
  - Индексируй данные асинхронно;
  - Авторизуй запросы к чувствительным данным с помощью Access Control List;
  - Используй средства анализа, чтобы сделать поиск удобным для пользователя;
  - Настраивай `boost`, чтобы находить релевантные документы.
