---
description: Как модуль называет свои страницы там, где сайт показывает, кто где находится
---

# Место посетителя

Сайт показывает, где находится посетитель: в списке «Кто онлайн», в истории посещений и в профиле. Место — это адрес страницы, которую посетитель открыл последней. Постоянные адреса названы в словаре мест `config/places.global.php`, а страницы, которые модуль строит из своих данных — тему форума, запись блога, альбом, — может назвать только сам модуль.

## Как это работает

Сначала система опрашивает модули: каждый получает адрес и отвечает, его ли это страница. Показывается первый ответ, отличный от `null`. Если ни один модуль адрес не взял, место ищется в словаре, а если нет и там — выводится «Где-то на сайте».

У выключенного модуля провайдера нет, и его адреса уходят в словарь: модулю, который показывает место, ничего не нужно знать о модуле, которому место принадлежит.

## Провайдер

Реализуйте интерфейс `Johncms\Users\VisitorPlaceProviderInterface`:

```php
<?php

declare(strict_types=1);

namespace Mysite\Blog\Application\Services;

use Johncms\Users\VisitorPlaceProviderInterface;
use Johncms\Utils\PlainTextFormatter;
use Mysite\Blog\Domain\Repository\PostRepositoryInterface;
use Twig\Markup;

final readonly class BlogPlaceProvider implements VisitorPlaceProviderInterface
{
    public function __construct(private PostRepositoryInterface $posts)
    {
    }

    public function describe(string $place): ?Markup
    {
        if (! str_starts_with($place, '/blog')) {
            return null;
        }

        $post = preg_match('~^/blog/post/(\d+)~', $place, $matches) === 1
            ? $this->posts->findById((int) $matches[1])
            : null;

        if ($post === null) {
            return new Markup('<a href="/blog/">' . d__('blog', 'В блоге') . '</a>', 'UTF-8');
        }

        return new Markup(
            d__('blog', 'Читает запись') . ' <a href="/blog/post/' . $post->id . '">'
            . PlainTextFormatter::escape($post->title) . '</a>',
            'UTF-8'
        );
    }
}
```

Что важно:

* **Чужой адрес — `null`.** Провайдер, который отвечает на всё подряд, заберёт себе страницы других модулей.
* **Ответ — разметка, она выводится как есть.** Всё, что пришло из данных — названия, имена, — экранируйте.
* **Провайдер вызывается для каждой строки списка.** Если узнать название дорого, держите запрос лёгким или кэшируйте результат.
* **Строки — в домене своего модуля**, `d__('blog', …)`. Словарь модуля подгрузится сам, на какой бы странице ни показывалось место.

## Регистрация

Отдельно регистрировать провайдер не нужно: тег `johncms.visitor_place_provider` система ставит сама — по реализованному интерфейсу, если класс объявлен с `autoconfigure()`. Блок `load()` в `config/services.php` модуля это делает. Для класса, объявленного отдельной строкой `set()` без `autoconfigure()`, тег указывается явно:

```php
$services->set(BlogPlaceProvider::class)->autowire()->tag('johncms.visitor_place_provider');
```

Пример в коде: `modules/johncms/forum/src/Application/Services/ForumVisitorPlaceFormatter.php` — форум называет свои разделы и темы.

## См. также

* [Sitemap-провайдер](sitemap-provider.md)
* [Создание модуля](sozdanie-modulya.md)
