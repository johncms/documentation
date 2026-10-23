---
description: Как модуль выводит свой блок на главной странице
---

# Блок главной страницы

Главная страница не знает, какие модули установлены: всё, что на ней появляется от модулей — последние новости, например, — модуль выводит сам, своим блоком. Блок — это шаблон модуля и данные для него.

## Как это работает

Система собирает зарегистрированных провайдеров, спрашивает у каждого его блоки и выводит их по порядку: блок с меньшим весом — выше. У выключенного модуля провайдера нет, и его блок просто не появляется.

## Провайдер

Реализуйте интерфейс `Johncms\View\Homepage\HomepageBlockProviderInterface`. Метод `homepageBlocks()` возвращает блоки — объекты `Johncms\View\Homepage\HomepageBlock`, — или ничего, если показывать нечего:

```php
<?php

declare(strict_types=1);

namespace Mysite\Blog\Application\Homepage;

use Johncms\View\Homepage\HomepageBlock;
use Johncms\View\Homepage\HomepageBlockProviderInterface;
use Mysite\Blog\Domain\Repository\PostRepositoryInterface;

final readonly class LatestPostsBlockProvider implements HomepageBlockProviderInterface
{
    public function __construct(private PostRepositoryInterface $posts)
    {
    }

    public function homepageBlocks(): iterable
    {
        $posts = $this->posts->getLatest(5);

        if ($posts !== []) {
            yield new HomepageBlock('@blog/homepage/latest.twig', ['posts' => $posts], weight: 20);
        }
    }
}
```

| Поле | Что это |
|---|---|
| `template` | шаблон модуля по его пространству имён: `@blog/homepage/latest.twig` |
| `data` | переменные шаблона; кроме них шаблон видит только глобальные (`app`) |
| `weight` | чем меньше, тем выше на странице; блоки с равным весом идут в порядке модулей. Блок новостей — 10 |

Что важно:

* **Нечего показать — нет блока.** Пустой заголовок без содержимого главной не нужен.
* **Строки шаблона — в домене своего модуля**, `d__('blog', …)`: страница принадлежит модулю главной, и обычный `__()` искал бы перевод в его домене. Словарь модуля подгрузится сам.
* **Блок собирается при каждом открытии главной.** Если выборка тяжёлая, кэшируйте её.

## Регистрация

Отдельно регистрировать провайдер не нужно: тег `johncms.homepage_block_provider` система ставит сама — по реализованному интерфейсу, если класс объявлен с `autoconfigure()`. Для класса, объявленного отдельной строкой `set()` без `autoconfigure()`, тег указывается явно:

```php
$services->set(LatestPostsBlockProvider::class)->autowire()->tag('johncms.homepage_block_provider');
```

Пример в коде: `modules/johncms/news/src/Application/Homepage/LatestNewsBlockProvider.php` и шаблон `modules/johncms/news/templates/homepage/latest.twig`.

## См. также

* [Публикация модуля](publikaciya-modulya.md) — пункт меню и плитка главной
* [Системные уведомления](system-notification-provider.md)
