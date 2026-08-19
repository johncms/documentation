---
description: Как текст пользователя превращается в готовую разметку страницы
---

# Вывод пользовательского контента

Сообщение форума, комментарий, статья, описание файла — всё это текст, который написал посетитель, и по пути на страницу с ним нужно сделать несколько вещей подряд: очистить разметку, превратить вставленную ссылку на видео в плеер, нарисовать смайлы. Этим занимается **конвейер контента** — `Johncms\Content\ContentRendererInterface`.

## Главное правило

**Один вызов вместо цепочки.** Не собирайте последовательность «санитайзер → медиа → смайлы» вручную: конвейер и появился затем, чтобы она была в одном месте.

{% hint style="warning" %}
Раньше эта цепочка была написана в десяти местах, и каждый её шаг разбирал и снова собирал весь текст — поверх разбора, который уже сделал санитайзер. Сообщение с двумя шагами перед выводом разбиралось четыре раза. Сейчас текст разбирается один раз, и все шаги работают с одним деревом.
{% endhint %}

## Как пользоваться

Внедрите интерфейс через конструктор и передайте текст в `render()`:

```php
<?php

declare(strict_types=1);

namespace Johncms\Modules\MyModule\Application\Services;

use Johncms\Content\ContentContext;
use Johncms\Content\ContentRendererInterface;
use Twig\Markup;

final readonly class ReviewTextFormatter
{
    public function __construct(
        private ContentRendererInterface $content,
    ) {
    }

    public function format(Review $review): Markup
    {
        return $this->content->render(
            $review->text,
            new ContentContext(adminSmilies: $review->authorIsStaff),
        );
    }
}
```

Результат — `Twig\Markup`, то есть «готовая разметка» по договорённости. Шаблон печатает такое значение обычным способом, без `|raw`:

```twig
{{ review.formatted_text }}
```

### Три метода

| Метод | Что возвращает |
| --- | --- |
| `render()` | разметку всегда: у пустого текста будет пустой `Markup` |
| `renderOrNull()` | то же самое, но `null`, если показывать нечего |
| `toPlainText()` | тот же контент без разметки — для превью в списке, заголовка страницы, уведомления |

{% hint style="info" %}
`Markup` — объект, а объект истинный, даже когда он пустой, поэтому `{% if text %}` в шаблоне на пустом `Markup` сработает всегда. Если у источника «текста может не быть» — берите `renderOrNull()`.
{% endhint %}

```php
public function formatReply(GuestbookEntry $entry): ?Markup
{
    return $this->content->renderOrNull((string) $entry->otvet, new ContentContext(adminSmilies: true));
}
```

`toPlainText()` рисует медиа и смайлы и только потом снимает теги — так содержимое удалённого элемента не всплывёт видимым текстом:

```php
$preview = mb_strimwidth($this->content->toPlainText($message->text), 0, 300, '...');
```

### Контекст

`ContentContext` — это то, что конвейер знает о тексте. Оба поля необязательны:

| Поле | Значение |
| --- | --- |
| `policy` | Политика очистки: значение enum `HtmlPolicy` или имя политики, которую объявил модуль. По умолчанию `HtmlPolicy::RichContent` — см. [Очистка HTML](html-sanitizer.md) |
| `adminSmilies` | Рисовать ли смайлы, доступные только персоналу. По умолчанию `false` |

Один и тот же контекст получает каждый шаг конвейера, поэтому модулю, который добавляет свой шаг, не нужно придумывать отдельный способ передать в него условия вывода.

## Когда конвейер не нужен

Он предназначен для **контента**, который выводится на страницу как разметка. Отдельно от него остаётся санитайзер `HtmlSanitizerInterface` — для значений, которые чистят, но не выводят как контент: заголовок для тега `<title>`, подпись, текст для поиска. Такие вещи проходят через `sanitize()` или `toPlainText()` санитайзера, а не через конвейер.

## Шаги конвейера

Шаг — это сервис, реализующий `Johncms\Content\Transformer\ContentTransformerInterface`. Встроенных три:

| Шаг | Приоритет | Что делает |
| --- | --- | --- |
| `OembedTransformer` | 100 | превращает `<oembed url="…">` редактора в плеер |
| `ImagePopupTransformer` | 50 | оборачивает картинку в ссылку, открывающую просмотрщик |
| `SmiliesTransformer` | −100 | рисует смайлы по готовому тексту |

Приоритет задаёт порядок: чем больше, тем раньше.

### Свой шаг

Править ядро не нужно — контейнер вешает тег на сервис по одному тому, что он реализует интерфейс. Достаточно, чтобы сервис попал в контейнер обычным способом (`load()` с `autoconfigure()`).

```php
<?php

declare(strict_types=1);

namespace Johncms\Modules\MyModule\Application\Content;

use Dom\HTMLDocument;
use Johncms\Content\ContentContext;
use Johncms\Content\Transformer\ContentTransformerInterface;

/**
 * Внешние ссылки в сообщениях не передают вес сайта и открываются в новой вкладке.
 */
final readonly class ExternalLinkTransformer implements ContentTransformerInterface
{
    public function priority(): int
    {
        return 20;
    }

    public function transform(HTMLDocument $document, ContentContext $context): void
    {
        foreach ($document->querySelectorAll('a[href^="http"]') as $link) {
            $link->setAttribute('rel', 'nofollow noopener');
            $link->setAttribute('target', '_blank');
        }
    }
}
```

Правила для шага:

* **Документ разбирается один раз и общий для всех.** Шаг правит дерево, которое ему дали, и ничего не возвращает. Он не разбирает и не собирает текст сам и не работает с HTML как со строкой.
* **Разметка попадает в дерево через DOM.** `setAttribute()` и `createElement()` экранируют значения сами. Для куска разметки крупнее одного элемента есть `HtmlFragment::nodes()`.
* **Шаг выполняется на каждом тексте сайта.** Ограничивайтесь тем селектором, который вам действительно нужен.

{% hint style="danger" %}
Не собирайте теги конкатенацией строк. Именно так раньше строилась ссылка просмотрщика, и кавычка в атрибуте `alt` картинки выходила из атрибута наружу — то есть в XSS. Через DOM это невозможно в принципе.
{% endhint %}

## Медиа: свой сайт видеохостинга

Редактор сохраняет только то, что вставил автор — `<oembed url="…">`. Разметки плеера в базе нет намеренно: иначе внешний вид всех старых сообщений застыл бы навсегда. Плеер строится на выводе, поэтому правка шаблона меняет и уже написанные сообщения.

Провайдер узнаёт адрес и называет шаблон. С документом он не работает — обход `<oembed>` делает `OembedTransformer` один раз для всех провайдеров:

```php
<?php

declare(strict_types=1);

namespace Johncms\Modules\MyModule\Application\Content;

use Johncms\Content\Embed\EmbeddedMedia;
use Johncms\Content\Embed\EmbedProviderInterface;

final readonly class RutubeEmbedProvider implements EmbedProviderInterface
{
    public function priority(): int
    {
        return 0;
    }

    public function embed(string $url): ?EmbeddedMedia
    {
        $parts = parse_url($url);
        if (($parts['host'] ?? '') !== 'rutube.ru') {
            return null; // не наш адрес, спросят следующего
        }

        // Идентификатор проверяем, а не доверяем ему: он попадёт в адрес iframe.
        if (preg_match('~^/video/([0-9a-f]{32})/~', $parts['path'] ?? '', $matches) !== 1) {
            return null;
        }

        return new EmbeddedMedia('@mymodule/embeds/rutube.twig', ['id' => $matches[1]]);
    }
}
```

Шаблон — обычный шаблон, поэтому тема переопределяет внешний вид плеера, повторив его путь (см. [Создание собственного шаблона](../shablony/sozdanie-sobstvennogo-shablona.md)). Шаблоны встроенных плееров лежат в `themes/default/templates/content/embeds/`:

```twig
{#
    Плеер Rutube.

    @var id string
#}
<div class="ratio ratio-16x9 my-2">
    <iframe src="https://rutube.ru/play/embed/{{ id }}" allowfullscreen="allowfullscreen" loading="lazy"></iframe>
</div>
```

Провайдеры опрашиваются по убыванию приоритета до первого, который вернул не `null`. Так модуль может заменить встроенный провайдер: пусть он объявляет те же адреса с приоритетом выше.

{% hint style="info" %}
Адрес, который не признал никто, — не ошибка. Элемент остаётся в тексте как есть, ссылка не теряется, и провайдер, добавленный позже, подхватит уже написанные сообщения.
{% endhint %}

## Разбор HTML

Разбирает текст единственный класс — `Johncms\Content\Html\HtmlFragment`. За ним стоит HTML5-парсер, который PHP 8.4 поставляет в расширении `ext-dom` (`Dom\HTMLDocument`); отдельной библиотеки для работы с DOM в JohnCMS нет.

Сообщение — это фрагмент страницы, а не документ, поэтому оно разбирается как содержимое `body` (там, где фрагменту и положено быть по правилам HTML5), и наружу отдаётся только это содержимое.

{% hint style="warning" %}
Не разбирайте контент через `DOMDocument` и не срезайте обёртку `html/body` из строки вручную. Это старый способ, и он приводил к тому, что обёртка попадала в текст на странице.

Есть и правило DOM, о которое легко споткнуться: у документа может быть только один дочерний элемент. Значит, узел можно заменять лишь **внутри** `body`, а не на уровне самого документа. Работа через `HtmlFragment` это обеспечивает.
{% endhint %}

## Как проверить свой шаг

Шаг тестируется без контейнера и без базы: разобрать фрагмент, выполнить шаг, собрать обратно.

```php
$fragment = new HtmlFragment();
$document = $fragment->parse('<p>Текст со <a href="https://example.org">ссылкой</a></p>');

(new ExternalLinkTransformer())->transform($document, new ContentContext());

self::assertStringContainsString('rel="nofollow noopener"', $fragment->serialize($document));
```

Готовые тесты конвейера лежат в `tests/Unit/Content/` — их можно взять за образец.
