---
description: Список изменений для обновления с версии JohnCMS 9.7
---

# Обновление с версии 9.7

### Работа с конфигурацией

Раньше конфиг хранился в контейнере. Теперь для конфига есть специальная функция config();

Пример с запросом конфига с помощью функции di()

```php
// Было ранее
di('config')
di('config')['johncms']
di('config')['news']

// Стало
config()
config('johncms')
config('news')
```

Пример с запросом из контейнера

```php
// Было
$container->get('config')
$container->get('config')['johncms']
$container->get('config')['news']

// Стало
config()
config('johncms')
config('news')
```

### Новый DI-контейнер (Symfony)

В новой версии контейнер зависимостей переведен на Symfony DI.

Что это значит для разработки:

* регистрируйте и переопределяйте сервисы через `services.php` / `services.local.php`
* в новом коде предпочитайте constructor injection вместо получения зависимостей из контейнера вручную
* для сервиса используйте идентификатор класса (`SomeService::class`), а не строковые ключи

Порядок загрузки сервисов:

1. core-сервисы (`system/config/services.php`)
2. сервисы модулей (`modules/*/config/services.php`)
3. проектные переопределения (`config/services.local.php`)

`config/services.local.php` имеет самый высокий приоритет и предназначен для локального переопределения сервисов проекта.

### Получение параметров текущего маршрута

Параметры текущего маршрута теперь сохраняются в объект запроса, а не хранятся в контейнере как было раньше. Из di('route') всё ещё можно достать данные о параметрах в рамках обратной совместимости, но из контейнера напрямую больше нельзя. В своих модулях вам необходимо провести замену кода сделующим образом:

```php
// Было
$route = di('route');
$route = $container->get('route')

// Стало
$route = di(\Johncms\System\Http\Request::class)->getCurrentRouteParams();
```

### Маршрутизация и middleware

Новая маршрутизация построена на `Johncms\Router\RouteCollection` и `Symfony Routing`.

Поддерживаются следующие варианты обработчиков маршрута:

* legacy include (строка пути к php-файлу)
* invokable-контроллер (`Controller::class` с `__invoke()`)
* массив `[Controller::class, 'method']`

Также добавлены middleware:

* для отдельного маршрута — `->addMiddleware(...)`
* для группы/коллекции маршрутов — `RouteCollection::addMiddleware(...)`

Middleware выполняются до handler и могут прерывать выполнение (например, при проверке прав доступа).

Если вы используете собственный файл `config/routes.local.php`, после обновления обязательно сверить его с актуальным `config/routes.local.php.example` и привести к текущему формату.

Что важно проверить в `routes.local.php`:

* файл подключается внутрь общего роутера, поэтому маршруты нужно добавлять напрямую через `$router`
* в файле доступны `$router` (и `$user`, если нужен доступ к текущему пользователю)
* используйте актуальные примеры объявления маршрутов (в том числе пресеты `{id:number}`, `{slug:slug}`, `{path:path}` и middleware через `->addMiddleware(...)`)

Минимальный актуальный каркас:

```php
<?php

declare(strict_types=1);

/** @var \Johncms\Router\RouteCollection $router */

$router->map(['GET', 'POST'], '/contacts', 'modules/contacts/index.php');
```

Подробно про объявление маршрутов, middleware и порядок dispatch:
[Маршрутизация (роутинг)](../moduli/marshrutizaciya-routing.md)

### Обновление cron-задачи

Если у вас была настроена старая cron-команда, её необходимо заменить:

* было: `php system/cron.php`
* стало: `php system/bin/console schedule:run --no-interaction`

Рекомендуемая периодичность запуска: 1 раз в минуту.

Подробнее про планировщик:
[Планировщик задач (schedule)](../konsol/planirovshchik-zadach-schedule.md)

### Переопределение сервисов

Раньше сервисы можно было переопределять через ‎config/autoload/dependencies.local.php.

Теперь переопределение можно выполнить в файле config/services.local.php. Для переопределения просто скопируйте файл services.local.php.example и назовите его services.local.php\
В файле-примере есть закомментированный пример переопределения.

### Базовый контроллер: отказ от наследования

Начиная с текущей версии, рекомендуется отказаться от наследования базового контроллера (`BaseController`) и использовать `ControllerContext`.\
Это упрощает контроллеры, делает зависимости явными, уменьшает скрытую магию и облегчает тестирование.

Контроллеры больше не обязаны наследоваться от общего класса — вся инициализация (шаблоны, переводы, навигация) вынесена в отдельный сервис.

Было (наследование):

```php
final class GuestbookController extends BaseController
{
    protected string $module_name = 'guestbook';

    public function index(): string
    {
        return $this->render->render('guestbook::index');
    }
}
```

Стало (ControllerContext + composition):

```php
final class GuestbookController
{
    public function __construct(
        private readonly ControllerContext $context,
        private readonly Render $render,
    ) {
        $this->context->initModule('guestbook');
    }

    public function __invoke(): string
    {
        return $this->render->render('guestbook::index');
    }
}
```

Преимущества:

* нет наследования
* явные зависимости через DI
* контроллер остаётся тонким HTTP-адаптером
* проще писать и тестировать

Для новых контроллеров рекомендуется использовать invokable-контроллеры и инициализацию через `\Johncms\Http\Controller\ControllerContext`.\
`BaseController` считается устаревшим и будет удалён в будущих версиях.

Для админских контроллеров используйте  `\Johncms\Http\Controller\AdminControllerContext`
