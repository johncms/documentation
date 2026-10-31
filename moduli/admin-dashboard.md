---
description: Как модуль выводит своё на главной панели управления — строки «Требует внимания», метрики, быстрые действия
---

# Главная панели управления

Главная панели (`/admin`) не знает, какие модули установлены. Всё, что на ней появляется от модулей, модуль сообщает сам — через одну из трёх точек расширения:

| Что | Где на главной | Интерфейс |
|---|---|---|
| Строка «Требует внимания» | первый блок: очереди модерации, проблемы | `Johncms\View\Dashboard\DashboardAttentionProviderInterface` |
| Метрика | карточки с числом за сутки и графиком за две недели | `Johncms\View\Dashboard\DashboardMetricProviderInterface` |
| Быстрое действие | кнопки над всеми блоками | `Johncms\View\Menu\MenuItemProviderInterface`, область `MenuArea::AdminActions` |

Комментарии в блоке «Последние комментарии» тоже приходят от модулей — это отдельная точка расширения, см. [Лента комментариев](comment-feed-source.md).

У выключенного модуля провайдеров нет, и его строки, карточки и кнопки просто не появляются.

## Требует внимания

Сюда идёт то, что администратору нужно сделать или заметить: очередь модерации, которую он ведёт, проблема сервера. Если ничего не ждёт, блок говорит «Всё разобрано».

Реализуйте `DashboardAttentionProviderInterface`. Метод `attentionItems()` возвращает строки — объекты `Johncms\View\Dashboard\AttentionItem`, — или ничего:

```php
<?php

declare(strict_types=1);

namespace Mysite\Blog\Application\Dashboard;

use Johncms\Auth\Authorization\AccessCheckerInterface;
use Johncms\View\Dashboard\AttentionItem;
use Johncms\View\Dashboard\AttentionLevel;
use Johncms\View\Dashboard\DashboardAttentionProviderInterface;
use Mysite\Blog\Domain\Repository\CommentRepositoryInterface;

final readonly class ModerationAttentionProvider implements DashboardAttentionProviderInterface
{
    public function __construct(
        private AccessCheckerInterface $accessChecker,
        private CommentRepositoryInterface $comments,
    ) {
    }

    public function attentionItems(): iterable
    {
        if (! $this->accessChecker->allows('blog.moderate')) {
            return;
        }

        $count = $this->comments->countAwaitingModeration();

        if ($count > 0) {
            yield new AttentionItem(d__('blog', 'Комментарии на модерации'), '/blog/moderation', AttentionLevel::Pending, $count, 'message-square');
        }
    }
}
```

| Поле | Что это |
|---|---|
| `name` | текст строки |
| `url` | куда ведёт строка — туда, где это разбирают |
| `level` | важность, см. ниже; строки идут от важных к очередям |
| `count` | число на строке; `null` — строка без числа |
| `icon` | имя иконки Lucide из спрайта панели (`themes/default/src/admin/icons/icons.json`); пусто — иконка уровня |

Уровни `AttentionLevel`:

| Уровень | Когда | Вид |
|---|---|---|
| `Danger` | что-то сломано: сайт не работает как должен, пока это не исправят | красная |
| `Warning` | ничего не сломано, но это стоит сайту скорости или безопасности | жёлтая |
| `Pending` | очередь персонала: то, что прислали посетители и кто-то должен посмотреть | зелёная |

Что важно:

* **Права проверяет провайдер.** Главную открывает любой, у кого есть доступ в панель; очередь модерации блога видит тот, кто её ведёт.
* **Ничего не ждёт — ничего не возвращайте.** Строка с нулём не нужна.
* **Провайдер вызывается при каждом открытии главной.** Держите подсчёт лёгким: один `COUNT` по индексу.
* **Строки — в домене своего модуля**, `d__('blog', …)`.

То же самое модуль часто сообщает и [системным уведомлением](system-notification-provider.md): уведомление видит пользователь в шапке сайта, строку «Требует внимания» — персонал на главной панели.

## Метрика

Карточка показывает, сколько чего-то сайт получил за последние сутки, насколько это больше или меньше, чем за сутки до них, и линию за две недели. «Сутки» здесь — последние 24 часа, а не календарный день.

Реализуйте `DashboardMetricProviderInterface`. Метод `metrics()` получает период — объект `Johncms\View\Dashboard\MetricPeriod` — и возвращает карточки `Johncms\View\Dashboard\DashboardMetric`:

```php
<?php

declare(strict_types=1);

namespace Mysite\Blog\Application\Dashboard;

use Johncms\View\Dashboard\DashboardMetric;
use Johncms\View\Dashboard\DashboardMetricProviderInterface;
use Johncms\View\Dashboard\MetricPeriod;
use Mysite\Blog\Domain\Repository\PostRepositoryInterface;

final readonly class BlogMetricsProvider implements DashboardMetricProviderInterface
{
    public function __construct(
        private PostRepositoryInterface $posts,
    ) {
    }

    public function metrics(MetricPeriod $period): iterable
    {
        yield DashboardMetric::fromTrend(
            d__('blog', 'записей в блоге сегодня'),
            $this->posts->countPublishedByDay($period->start, $period->days),
            '/blog',
            80
        );
    }
}
```

`DashboardMetric::fromTrend()` принимает ряд — по одному числу на сутки периода, от старых к новым — и берёт значение карточки из последнего. Число без истории (например, сколько посетителей было за сутки, когда хранится только последний визит) создаётся конструктором без ряда: `new DashboardMetric($label, $value)`; у такой карточки нет ни сравнения, ни линии.

| Поле | Что это |
|---|---|
| `label` | подпись карточки |
| `value` | число за последние сутки |
| `trend` | ряд по суткам периода, от старых к новым; пусто — карточка без истории |
| `url` | куда ведёт карточка; пусто — карточка не ссылка |
| `weight` | чем меньше, тем раньше; у пользователей 10 и 20, у форума 30, гостевой 40, загрузок 50, библиотеки 60, альбомов 70 |

Ряд по суткам удобно считать помощником `Johncms\Database\Query\DailyCounts` — одним запросом на весь период, одинаково на MySQL, MariaDB и SQLite. Колонка должна хранить unix-время:

```php
use Johncms\Database\Query\DailyCounts;

public function countPublishedByDay(int $start, int $days): array
{
    return DailyCounts::count(Post::query()->where('published', 1), 'created_at', $start, $days);
}
```

Что важно:

* **Карточка видна всем, кто открыл главную.** Если число не для всех, проверьте права в провайдере и ничего не возвращайте.
* **Ряд должен быть ровно на `$period->days` суток** — тех, что пришли в `MetricPeriod`, а не своих.
* **Карточки считаются при каждом открытии главной.** Колонка времени должна быть с индексом.

## Быстрое действие

Кнопки над всеми блоками главной — то, что персонал делает часто: написать новость, найти пользователя. Это обычный пункт меню в области `MenuArea::AdminActions` — объявите его тем же провайдером, что и остальные пункты меню модуля (см. раздел «Пункт меню» в [Публикации модуля](publikaciya-modulya.md)):

```php
new MenuItem(
    MenuArea::AdminActions,
    d__('blog', 'Написать запись'),
    '/admin/blog/posts/new',
    icon: 'pencil',
    permission: 'blog.manage',
    weight: 40,
),
```

Кнопку без права посетитель не увидит. `icon` — имя иконки Lucide из спрайта панели. Вес определяет порядок: у поиска пользователя 10, поиска по IP 20, новости 30, рекламы 60, настроек 90.

## Регистрация

Отдельно регистрировать провайдеры не нужно: теги `johncms.dashboard_attention_provider` и `johncms.dashboard_metric_provider` система ставит сама — по реализованному интерфейсу, если класс объявлен с `autoconfigure()`. Для класса, объявленного отдельной строкой `set()` без `autoconfigure()`, тег указывается явно:

```php
$services->set(BlogMetricsProvider::class)->autowire()->tag('johncms.dashboard_metric_provider');
```

Примеры в коде:

* строки «Требует внимания» — `src/Application/Dashboard/*AttentionProvider.php` модулей `admin`, `downloads` и `library`;
* метрики — `src/Application/Dashboard/*MetricsProvider.php` модулей `admin`, `forum`, `guestbook`, `downloads`, `library` и `album`;
* быстрые действия — `modules/johncms/admin/src/Application/Menu/AdminPanelMenuItemProvider.php` и `modules/johncms/news/src/Application/Menu/NewsMenuItemProvider.php`.

## См. также

* [Лента комментариев](comment-feed-source.md)
* [Системные уведомления](system-notification-provider.md)
* [Права доступа](prava-dostupa.md)
