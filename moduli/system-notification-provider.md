---
description: Как модуль сообщает пользователю, что его что-то ждёт
---

# Системные уведомления

Когда пользователя что-то ждёт — непрочитанные письма, очередь модерации, которую он ведёт, бан, — это видно в двух местах: строкой на странице уведомлений со ссылкой туда, где это лежит, и числом на значке в шапке сайта. Каждую строку сообщает модуль, которому принадлежит то, что ждёт.

## Как это работает

Система собирает всех зарегистрированных провайдеров и спрашивает каждого, что ждёт текущего пользователя. Страница уведомлений выводит всё, что они вернули; значок в шапке складывает числа вместе с непрочитанными личными уведомлениями. Гостя система не спрашивает ни о чём.

У выключенного модуля провайдера нет, и его строки просто не появляются: модулю уведомлений не нужно знать ни о почте, ни о форуме.

## Провайдер

Реализуйте интерфейс `Johncms\Notifications\SystemNotificationProviderInterface`. Метод `notifications()` возвращает строки — объекты `Johncms\Notifications\SystemNotification`, — или ничего, если ничего не ждёт:

```php
<?php

declare(strict_types=1);

namespace Mysite\Blog\Application\Notifications;

use Johncms\Auth\Authorization\AccessCheckerInterface;
use Johncms\Notifications\SystemNotification;
use Johncms\Notifications\SystemNotificationProviderInterface;
use Mysite\Blog\Domain\Repository\CommentRepositoryInterface;

final readonly class ModerationNotificationProvider implements SystemNotificationProviderInterface
{
    public function __construct(
        private AccessCheckerInterface $accessChecker,
        private CommentRepositoryInterface $comments,
    ) {
    }

    public function notifications(): iterable
    {
        if (! $this->accessChecker->allows('blog.moderate')) {
            return;
        }

        $count = $this->comments->countAwaitingModeration();

        if ($count > 0) {
            yield new SystemNotification('blog_mod', d__('blog', 'Комментарии на модерации'), '/blog/moderation', $count);
        }
    }
}
```

Поля строки:

| Поле | Что это |
|---|---|
| `key` | имя числа в том, что функция шаблонов `notifications()` возвращает теме: `notifications().blog_mod` |
| `name` | текст строки на странице уведомлений |
| `url` | куда ведёт строка |
| `count` | число; входит в сумму на значке в шапке |
| `countShown` | `false` — строка без числа на ней, как у бана; в сумму число всё равно входит |

Что важно:

* **Права проверяет провайдер.** Очередь модерации видит тот, кто её ведёт, а не каждый вошедший.
* **Ничего не ждёт — ничего не возвращайте.** Строка с нулём на странице не нужна.
* **Провайдер вызывается на каждой странице**, где показан значок шапки. Держите подсчёт лёгким: один `COUNT` по индексу, без выборки строк.
* **Строки — в домене своего модуля**, `d__('blog', …)`: словарь модуля подгрузится сам.

## Регистрация

Отдельно регистрировать провайдер не нужно: тег `johncms.system_notification_provider` система ставит сама — по реализованному интерфейсу, если класс объявлен с `autoconfigure()`. Для класса, объявленного отдельной строкой `set()` без `autoconfigure()`, тег указывается явно:

```php
$services->set(ModerationNotificationProvider::class)->autowire()->tag('johncms.system_notification_provider');
```

Примеры в коде — `src/Application/Notifications/` модулей `mail`, `forum`, `library`, `downloads`, `album`, `profile` и `admin`.

## См. также

* [Место посетителя](visitor-place-provider.md)
* [Права доступа](prava-dostupa.md)
