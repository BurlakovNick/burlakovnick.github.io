---
title: Название статьи
header:
teaser: images/about/beer.png
excerpt: И какое-то описание
date: '2018-04-22 00:00:00'
toc: true
toc_label: "Для больших постов лучше включить навигацию в виде Table of contents"
---

*курсив*

**жирный шрифт**

# Заголовок 1

## Заголовок 2

###### Заголовок 6

> Цитата очень умного человека

<cite>Steve Jobs</cite> --- Apple Worldwide Developers' Conference, 1997
{: .small}

```csharp
public class Calculator
{
    public int Add(int a, int b) => a + b;
}
```

Табличка

| Employee         | Salary |                                                              |
| --------         | ------ | ------------------------------------------------------------ |
| [John Doe](#)    | $1     | Because that's all Steve Jobs needed for a salary.           |
| [Jane Doe](#)    | $100K  | For all the blogging she does.                               |
| [Fred Bloggs](#) | $100M  | Pictures are worth a thousand words, right? So Jane × 1,000. |
| [Jane Bloggs](#) | $100B  | With hair like that?! Enough said.                           |

![Картинка](/images/birthday/kid.jpg 'Папин стиляга, мамин симпатяга'){: .align-center}

<figure class="half">
	<a href="http://placehold.it/1200x600.JPG"><img src="http://placehold.it/600x300.jpg"></a>
	<a href="http://placehold.it/1200x600.jpeg"><img src="http://placehold.it/600x300.jpg"></a>
	<figcaption>Two images.</figcaption>
</figure>

{% include gallery caption="This is a sample gallery with **Markdown support**." %}

Видео на YouTube

<figure>
<iframe width="640" height="360" src="https://www.youtube-nocookie.com/embed/l2Of1-d5E5o?controls=0&showinfo=0" frameborder="0" allowfullscreen></iframe> 
</figure>

Определение 1
:   Это значит то-то и то-то

Определение 2
:   Это уже что-то другое

Список
    * Заботать английский
        * Записаться на курсы
        * Сдать экзамен
    * Купить молока
    * Посадить дерево

Нумерованный список
    1. Заботать английский
        1. Записаться на курсы
        2. Сдать экзамен
    2. Купить молока
    3. Посадить дерево


**Watch out!** This paragraph of text has been [emphasized](#) with the `{: .notice}` class.
{: .notice}

**Watch out!** This paragraph of text has been [emphasized](#) with the `{: .notice--primary}` class.
{: .notice--primary}

**Watch out!** This paragraph of text has been [emphasized](#) with the `{: .notice--info}` class.
{: .notice--info}

**Watch out!** This paragraph of text has been [emphasized](#) with the `{: .notice--warning}` class.
{: .notice--warning}

**Watch out!** This paragraph of text has been [emphasized](#) with the `{: .notice--success}` class.
{: .notice--success}

**Watch out!** This paragraph of text has been [emphasized](#) with the `{: .notice--danger}` class.
{: .notice--danger}