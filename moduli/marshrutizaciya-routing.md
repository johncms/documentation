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
3. `config/routes.local.php` — переопределения для конкретного проекта (наивысший приоритет, **устарел**)

Каждый модуль регистрирует свои маршруты в `config/routes.php` внутри папки модуля. Файл подхватывается автоматически — вручную подключать его не нужно.

> **`config/routes.local.php` устарел** и будет удалён в будущих версиях. Если у вас есть маршруты в этом файле, перенесите их в `modules/{name}/config/routes.php`.

## Базовый пример маршрута

Файл `modules/{name}/config/routes.php` должен возвращать callable:

```php
<?php

declare(strict_types=1);

use Johncms\Modules\Partners\Application\Controllers\PartnersController;
use Johncms\Router\RouteCollection;
use Johncms\System\Users\User;

return static function (RouteCollection $router, User $user): void {
    $router->get('/partners', PartnersController::class)->name('partners.index');
    $router->map(['GET', 'POST'], '/feedback', FeedbackController::class)->name('feedback');
};
```

Параметр `$user` доступен для регистрации маршрутов, зависящих от состояния пользователя:

```php
return static function (RouteCollection $router, User $user): void {
    $router->get('/posts', PostsController::class);

    if ($user->isValid()) {
        $router->post('/posts/create', CreatePostController::class);
    }
};
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
public function handle(Request $request, callable $next): mixed;
```

Пример класса middleware:

```php
<?php

declare(strict_types=1);

namespace Johncms\Modules\Guestbook\Application\Middlewares;

use Johncms\Router\MiddlewareInterface;
use Johncms\System\Http\Request;

final class GuestbookCleanAccessMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, callable $next): mixed
    {
        $params = $request->getCurrentRouteParams();

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
    ->addMiddleware(static function (\Johncms\System\Http\Request $request, callable $next): mixed {
        return $next($request);
    });
```

### Порядок выполнения

При обработке совпавшего маршрута выполняется цепочка:

1. middleware коллекции/группы
2. middleware самого маршрута
3. handler маршрута

Если middleware не вызывает `$next($request)`, цепочка останавливается и handler не будет вызван.

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
* маршрут не зарегистрирован в `modules/{name}/config/routes.php` или `config/routes.local.php`

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
