---
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/9pmJhK7QfUfjF0JYsZd2/moduli/marshrutizaciya-routing
---

# Маршрутизация (роутинг)

В JohnCMS маршрутизация построена на `Johncms\Router\RouteCollection` и `Symfony Routing`.

Эта страница описывает текущий подход: как объявлять маршруты, как подключать middleware, какие бывают обработчики и как происходит dispatch.

## Где описываются маршруты

* Основные маршруты системы: `config/routes.php`
* Ваши дополнительные маршруты: `config/routes.local.php`

Для собственного кода используйте `routes.local.php`, чтобы изменения не терялись при обновлениях.

Пример есть в файле `config/routes.local.php.example`.

## Базовый пример маршрута

```php
<?php

declare(strict_types=1);

/** @var \Johncms\Router\RouteCollection $router */

$router->get('/contacts', 'modules/contacts/index.php');
$router->map(['GET', 'POST'], '/feedback', 'modules/feedback/index.php');
```

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

Примеры можно посмотреть в `config/routes.php`.

## Middleware на маршрутах

Middleware — это промежуточный обработчик между совпавшим маршрутом и его handler.
Он получает `Request`, может выполнить проверку/подготовку и передать управление дальше через `$next($request)`.

Обычно middleware используют для:

* проверки доступа (права, авторизация, владение ресурсом)
* валидации обязательных условий перед действием
* логирования и других сквозных задач

### Middleware для конкретного маршрута

Добавление middleware к одному маршруту:

```php
$router
    ->map(['GET', 'POST'], '/guestbook/clean', App\Guestbook\ClearController::class)
    ->addMiddleware(App\Guestbook\GuestbookCleanAccessMiddleware::class);
```

Можно указывать несколько middleware, они будут вызваны по порядку добавления.

### Middleware для группы/коллекции маршрутов

Можно добавить middleware сразу на коллекцию (например, в группе):

```php
$router->group('/guestbook', static function (\Johncms\Router\RouteCollection $group): void {
    $group->addMiddleware(App\Guestbook\CommonAccessMiddleware::class);

    $group->get('/edit/{id:number}', App\Guestbook\EditController::class);
    $group->post('/reply/{id:number}', App\Guestbook\ReplyController::class);
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

namespace App\Guestbook;

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
    ->get('/contacts', App\Contacts\IndexController::class)
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
   - route params кладутся в request
   - запускается цепочка middleware
   - вызывается handler
4. При `METHOD_NOT_ALLOWED` возвращается `405 Method Not Allowed`
5. При `NOT_FOUND` вызывается `pageNotFound()`

## Типовые ошибки

### 404 Not Found

Причины:

* путь не совпадает с шаблоном маршрута
* route params не прошли `requirements`
* маршрут не зарегистрирован в `routes.php`/`routes.local.php`

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
