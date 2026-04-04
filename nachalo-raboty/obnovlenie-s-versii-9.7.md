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

### Получение параметров текущего маршрута

Параметры текущего маршрута теперь сохраняются в объект запроса, а не хранятся в контейнере как было раньше. Из di('route') всё ещё можно достать данные о параметрах в рамках обратной совместимости, но из контейнера напрямую больше нельзя. В своих модулях вам необходимо провести замену кода сделующим образом:

```php
// Было
$route = di('route');
$route = $container->get('route')

// Стало
$route = di(\Johncms\System\Http\Request::class)->getCurrentRouteParams();
```

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
