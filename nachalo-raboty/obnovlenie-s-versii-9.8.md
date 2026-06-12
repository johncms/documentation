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

### Пагинация: удалён форк `johncms/johncms-pagination` и `Tools::displayPagination()`

В версии 9.9 полностью удалены устаревший форк `johncms/johncms-pagination` (Laravel-метод `->paginate()` и `LengthAwarePaginator`) и метод `Tools::displayPagination()`. Вместо них используется компонент **`\Johncms\Http\Pagination`** (`PaginationFactory`, `PaginationGuard`, `Pagination`).

Если в ваших модулях встречается что-либо из перечисленного, их нужно перевести на новый компонент, иначе будут ошибки «class/method not found»:

* `->paginate(...)` в запросах Eloquent;
* тип `Illuminate\Contracts\Pagination\LengthAwarePaginator` в репозиториях, use cases и DTO;
* вызовы `$this->tools->displayPagination(...)` в контроллерах.

Полное описание компонента — на странице [«Пагинация»](../obshie-svedeniya/pagination.md). Ниже — краткая схема миграции.

#### 1. Репозиторий: `->paginate()` → `count*()` + `get*(limit, offset)`

Было:

```php
use Illuminate\Contracts\Pagination\LengthAwarePaginator;

public function paginateActive(int $page, int $perPage): LengthAwarePaginator
{
    return Article::query()->where('active', 1)->orderByDesc('id')->paginate($perPage, page: $page);
}
```

Стало:

```php
use Illuminate\Database\Eloquent\Collection;

public function countActive(): int
{
    return Article::query()->where('active', 1)->count();
}

/** @return Collection<int, Article> */
public function getActive(int $limit, int $offset): Collection
{
    return Article::query()
        ->where('active', 1)
        ->orderByDesc('id')
        ->offset($offset)
        ->limit($limit)
        ->get();
}
```

#### 2. Use case: разделите на `count()` и `getPage()`

```php
public function count(): int
{
    return $this->repository->countActive();
}

/** @return array<int, Article> */
public function getPage(int $limit, int $offset): array
{
    return $this->repository->getActive($limit, $offset)->all();
}
```

#### 3. Контроллер: `displayPagination()` / пагинатор → `PaginationFactory` + `PaginationGuard`

Было:

```php
$page  = max(1, (int) $this->request->getQuery('page', 1));
$total = $this->useCase->count();
// ...
'pagination' => $this->tools->displayPagination('/articles?', ($page - 1) * $perPage, $total, $perPage),
```

Стало:

```php
use Johncms\Http\Pagination\PaginationFactory;
use Johncms\Http\Pagination\PaginationGuard;

// В конструкторе: PaginationFactory $paginationFactory, PaginationGuard $paginationGuard

$pagination = $this->paginationFactory->create($this->useCase->count());

$redirectUrl = $this->paginationGuard->redirectUrl($pagination);
if ($redirectUrl !== null) {
    redirect($redirectUrl);
}

$items = $this->useCase->getPage($pagination->getPerPage(), $pagination->getOffset());
// ...
'total'      => $pagination->getTotal(),
'pagination' => $pagination->render(),
```

В шаблоне по-прежнему достаточно вывести готовый HTML строкой: `<?= $pagination ?>` (где переменная содержит результат `$pagination->render()`).

> Шаблон `system::app/pagination` теперь работает только с новым форматом элементов (ключ `type`). Легаси-ветка (элементы с `name`/`url` без `type`) из тем `default` и `admin` удалена. Если у вас своя тема с собственным шаблоном пагинации — приведите его к новому формату (см. шаблоны темы `default`).

#### Готовые примеры в коде

Проще всего ориентироваться на уже переведённые штатные модули. В качестве компактных образцов посмотрите:

* **`modules/notifications`** — минимальный пример: репозиторий с `countNotifications()`/`getNotifications(limit, offset)`, use case с `count()`/`getPage()` и контроллер `IndexController` с `PaginationFactory`/`PaginationGuard`.
* **`modules/community`** — несколько списков с сортировками и фильтрами (учитывайте сохранение query-параметров в ссылках пагинации).

Если у вас несложный модуль, проще всего скопировать структуру из `modules/notifications` и адаптировать под свои сущности.
