---
description: Список изменений для обновления с версии JohnCMS 9.8
---

# Обновление с версии 9.8

### Маршрутизация: новый формат routes.local.php

Файл `config/routes.local.php` теперь должен возвращать callable, а не использовать `$router` напрямую.

Было:

```php
<?php

/** @var \Johncms\Router\RouteCollection $router */

$router->map(['GET', 'POST'], '/contacts', 'modules/contacts/index.php');
```

Стало:

```php
<?php

declare(strict_types=1);

use Johncms\Router\RouteCollection;
use Johncms\System\Users\User;

return static function (RouteCollection $router, User $user): void {
    $router->map(['GET', 'POST'], '/contacts', 'modules/contacts/index.php');
};
```

Актуальный пример есть в `config/routes.local.php.example`.

### Маршрутизация: перенос маршрутов в модульный config/routes.php

Если у вас есть собственные модули с маршрутами в `routes.local.php`, рекомендуется перенести их в `config/routes.php` внутри папки модуля. Файл подхватывается автоматически и не требует ручного подключения.

```php
// modules/contacts/config/routes.php
<?php

declare(strict_types=1);

use Johncms\Router\RouteCollection;
use Johncms\System\Users\User;

return static function (RouteCollection $router, User $user): void {
    $router->map(['GET', 'POST'], '/contacts', 'modules/contacts/index.php');
};
```

`config/routes.local.php` по-прежнему поддерживается, но считается устаревшим и будет удалён в будущих версиях. При наличии файла будет выведено deprecation-предупреждение.
