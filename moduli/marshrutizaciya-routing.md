---
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/5NJeWEeBonlrBhBrVHEz/moduli/marshrutizaciya-routing
---

# Маршрутизация (роутинг)

В JohnCMS маршрутизация построена на `Johncms\Router\RouteCollection` и `Symfony Routing`.

Эта страница описывает текущий подход: как объявлять маршруты, как подключать middleware, какие бывают обработчики и как происходит dispatch.

## Где описываются маршруты

Маршруты загружаются в следующем порядке:

1. `config/routes.php` — глобальные/системные маршруты (зарезервирован для ядра)
2. `modules/{name}/config/routes.php` — маршруты каждого модуля (подхватываются автоматически)

Каждый модуль регистрирует свои маршруты в `config/routes.php` внутри папки модуля. Файл подхватывается автоматически — вручную подключать его не нужно.

## Базовый пример маршрута

Файл `modules/{name}/config/routes.php` должен возвращать callable:

```php
<?php

declare(strict_types=1);

use Johncms\Modules\Partners\Application\Controllers\PartnersController;
use Johncms\Router\RouteCollection;

return static function (RouteCollection $router): void {
    $router->get('/partners', PartnersController::class)->name('partners.index');
    $router->map(['GET', 'POST'], '/feedback', FeedbackController::class)->name('feedback');
};
```

{% hint style="warning" %}
**Изменение в 10.0.** Раньше файл получал вторым аргументом `Johncms\System\Users\User`, и маршруты для персонала объявлялись внутри `if ($user->rights >= 7)`. Теперь аргумент один: коллекция маршрутов одинакова для всех посетителей, а кого пускать — решает middleware (см. «Доступ к маршруту»). Сторонним модулям нужно убрать второй параметр из сигнатуры и переписать условия на `permission()` / `RequireAuthMiddleware`.
{% endhint %}

## Доступ к маршруту

Маршрут объявляется всегда, а ограничение доступа описывается рядом с ним.

### Право на маршрут

```php
use Johncms\Modules\Partners\Application\Services\PartnersPermissions;

$router->get('/admin/partners', PartnersAdminController::class)
    ->name('partners.admin')
    ->permission(PartnersPermissions::MANAGE);
```

`permission()` кладёт ключ права в атрибут маршрута; ядро само подключает `RequirePermissionMiddleware`, который его читает. Ответ посетителю без права — **403**, гостю — редирект на `/login`. Если сам факт существования маршрута скрывать важнее, чем честно ответить, — `->permission('...', hidden: true)`, тогда вместо 403 будет 404.

Право должно быть объявлено провайдером модуля (класс, реализующий `PermissionProviderInterface`), иначе выдать его в редакторе ролей будет нечем.

То же самое можно задать сразу на группу — право получат все её маршруты, кроме тех, что назвали своё:

```php
$admin = $router->group('', static function (RouteCollection $router): void {
    $router->get('/admin/partners', PartnersAdminController::class)->name('partners.admin');
    $router->post('/admin/partners', [PartnersAdminController::class, 'save'])->name('partners.admin.save');
});
$admin->permission(PartnersPermissions::MANAGE);
```

### Маршрут только для авторизованных

Когда никакого права не нужно, а нужна только сессия:

```php
use Johncms\Http\Middleware\RequireAuthMiddleware;

$router->post('/partners/subscribe', SubscribeController::class)
    ->name('partners.subscribe')
    ->addMiddleware(RequireAuthMiddleware::class);
```

{% hint style="info" %}
**Конвенция завершающего слэша.** Маршруты принято регистрировать **без** завершающего слэша (`/partners`), а в ссылках (в шаблонах и контроллерах) — использовать слэш (`/partners/`). Перед сопоставлением `index.php` нормализует URI через `rtrim`, поэтому оба варианта работают.
{% endhint %}

## Именованные маршруты

Маршруту можно задать имя с помощью метода `name()`. Имя используется как внутренний идентификатор маршрута в системе.

```php
$router->map(['GET', 'POST'], '/guestbook', GuestbookController::class)->name('guestbook.index');
```

Рекомендуется именовать маршруты по схеме `модуль.действие` (например `guestbook.index`, `guestbook.edit`, `admin.contacts.save`). Это соглашение используется во всех штатных модулях. `name()` можно комбинировать с другими методами (`requirements()`, `addMiddleware()` и т.д.) в цепочке вызовов.

## Параметры и ограничения

Маршрут может содержать параметры и ограничения через `requirements()`:

```php
$router
    ->map(['GET', 'POST'], '/contacts/{city}/{id}/{street}', 'modules/contacts/index.php')
    ->defaults([
        'city' => null,
        'id' => null,
        'street' => null,
    ])
    ->requirements([
        'id' => '\\d+',
    ]);
```

Также поддерживаются пресеты в пути:

* `{id:number}`
* `{article_code:slug}`
* `{category:path}`

Примеры можно посмотреть в файлах `modules/*/config/routes.php`.

## Middleware на маршрутах

Middleware — это промежуточный обработчик между совпавшим маршрутом и его handler. Он получает `Request`, может выполнить проверку/подготовку и передать управление дальше через `$next($request)`.

Обычно middleware используют для:

* проверки доступа (права, авторизация, владение ресурсом)
* валидации обязательных условий перед действием
* логирования и других сквозных задач

### Middleware для конкретного маршрута

Добавление middleware к одному маршруту:

```php
use Johncms\Modules\Guestbook\Application\Controllers\ClearGuestbookController;
use Johncms\Modules\Guestbook\Application\Middlewares\GuestbookCleanAccessMiddleware;

$router
    ->map(['GET', 'POST'], '/guestbook/clean', ClearGuestbookController::class)
    ->addMiddleware(GuestbookCleanAccessMiddleware::class);
```

Можно указывать несколько middleware, они будут вызваны по порядку добавления.

### Middleware для группы/коллекции маршрутов

Можно добавить middleware сразу на коллекцию (например, в группе):

```php
$router->group('/guestbook', static function (\Johncms\Router\RouteCollection $group): void {
    $group->addMiddleware(GuestbookCommonAccessMiddleware::class);

    $group->get('/edit/{id:number}', EditEntryController::class);
    $group->post('/reply/{id:number}', ReplyController::class);
});
```

Такой middleware будет применяться ко всем маршрутам внутри этой коллекции.

### Допустимые типы middleware

Middleware может быть:

* классом (получается из контейнера), реализующим `Johncms\Router\MiddlewareInterface`
* callable

Контракт middleware для класса:

```php
public function handle(Request $request, callable $next): Response;
```

Пример класса middleware:

```php
<?php

declare(strict_types=1);

namespace Johncms\Modules\Guestbook\Application\Middlewares;

use Johncms\Http\Request;
use Johncms\Router\MiddlewareInterface;
use Symfony\Component\HttpFoundation\Response;

final class GuestbookCleanAccessMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, callable $next): Response
    {
        // Параметры совпавшего маршрута доступны как атрибуты запроса
        $params = $request->attributes->all();

        // Если условие не выполнено, можно прервать цепочку (throw/return)
        // throw new \Johncms\Exceptions\PageNotFoundException();

        return $next($request);
    }
}
```

Пример callable middleware:

```php
$router
    ->get('/partners', Johncms\Modules\Partners\Application\Controllers\PartnersController::class)
    ->addMiddleware(static function (\Johncms\Http\Request $request, callable $next): \Symfony\Component\HttpFoundation\Response {
        return $next($request);
    });
```

### Порядок выполнения

При обработке совпавшего маршрута выполняется цепочка:

1. middleware коллекции/группы
2. middleware самого маршрута
3. handler маршрута

Если middleware не вызывает `$next($request)`, цепочка останавливается и handler не будет вызван.

## Кэш маршрутов

Коллекция маршрутов не зависит от посетителя, поэтому её можно собрать один раз. В
`config/constants.php`:

```php
const CACHE_ROUTES = true;
```

При включённом кэше маршруты один раз выгружаются в `data/cache/routes.php` (обычный массив,
который попадает в opcache), и файлы `config/routes.php` больше не читаются на каждый запрос.
После изменения маршрутов файл нужно удалить — или выполнить `php system/bin/console cache:clear`.

{% hint style="info" %}
Обработчик и middleware маршрута при этом должны быть данными: строкой с именем класса или
массивом `[Класс::class, 'метод']`. Замыкание выгрузить нельзя — такой маршрут ломает кэш, о чём
в лог попадёт предупреждение, а сайт продолжит работать без кэша.
{% endhint %}

## Какие обработчики поддерживаются

В маршруте можно указать:

1. Строку с путем legacy-файла (`'modules/contacts/index.php'`)
2. Invokable-контроллер (`SomeController::class` с `__invoke()`)
3. Массив `[ControllerClass::class, 'method']`

Это обрабатывается в `index.php` через `ActionInvoker` и `MiddlewareDispatcher`.

## Как работает dispatch

Схема обработки запроса:

1. URI нормализуется в `index.php`
2. `SymfonyRouteMatcher::dispatch()` пытается сопоставить маршрут
3. При `FOUND`:
   * route params кладутся в request
   * запускается цепочка middleware
   * вызывается handler
4. При `METHOD_NOT_ALLOWED` возвращается `405 Method Not Allowed`
5. При `NOT_FOUND` вызывается `pageNotFound()`

## Типовые ошибки

### 404 Not Found

Причины:

* путь не совпадает с шаблоном маршрута
* route params не прошли `requirements`
* маршрут не зарегистрирован в `modules/{name}/config/routes.php`

### 405 Method Not Allowed

Причина:

* URL найден, но HTTP-метод не разрешен для маршрута (например, `POST` вместо `GET`).

### Ошибка middleware

Причины:

* middleware-класс не зарегистрирован в контейнере
* middleware не callable и не реализует `MiddlewareInterface`

В этом случае `MiddlewareDispatcher` выбросит `InvalidArgumentException`.

## См. также

* [Создание модуля](sozdanie-modulya.md)
* [Структура модуля](struktura-modulya.md)
* [Конфигурационные файлы (configs)](../obshie-svedeniya/konfiguracionnye-faily-configs.md)
* [Проблемы и их решение](../obshie-svedeniya/problemy-i-ikh-reshenie.md)
