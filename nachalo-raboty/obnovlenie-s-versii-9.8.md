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

* `->paginate(...)` в запросах Eloquent (в том числе прямо в контроллере, без репозитория и use case);
* тип `Illuminate\Contracts\Pagination\LengthAwarePaginator` в репозиториях, use cases и DTO;
* вызовы `$this->tools->displayPagination(...)` в контроллерах;
* вызов `$paginator->render()` в шаблонах (метод рендера старого пагинатора).

Типичная ошибка при пропущенной миграции:

```
Class "Illuminate\Pagination\Paginator" not found
```

Она возникает при первом же обращении к странице, где вызывается `->paginate()`: класс пагинатора удалён вместе с форком, поэтому метод больше не работает.

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

#### 4. Шаблон: `$paginator->render()` → готовая строка пагинации

Старый пагинатор передавался в шаблон целиком, а HTML строился вызовом его метода `render()`. Так делать больше нельзя — сам объект пагинатора удалён.

Было:

```php
<?php foreach ($commits as $commit): ?>
    <?php /* ... вывод элемента ... */ ?>
<?php endforeach; ?>
<div class="mt-4">
    <?= $commits->render(); ?>
</div>
```

Стало (контроллер передаёт обычный массив элементов и уже отрендеренную строку пагинации):

```php
<?php foreach ($commits as $commit): ?>
    <?php /* ... вывод элемента ... */ ?>
<?php endforeach; ?>
<div class="mt-4">
    <?= $pagination ?>
</div>
```

Где `$pagination` — результат `$pagination->render()`, переданный контроллером.

> Шаблон `system::app/pagination` теперь работает только с новым форматом элементов (ключ `type`). Легаси-ветка (элементы с `name`/`url` без `type`) из тем `default` и `admin` удалена. Если у вас своя тема с собственным шаблоном пагинации — приведите его к новому формату (см. шаблоны темы `default`).

#### Простой модуль: `->paginate()` прямо в контроллере

В старых компактных модулях пагинатор нередко вызывался напрямую в контроллере, без репозитория и use case. Такой код тоже ломается («class not found») и требует миграции.

Было:

```php
public function index(User $user): string
{
    $data = [
        'commits' => (new GitCommit())->orderByDesc('commit_date')->paginate($user->config->kmess),
    ];

    return $this->render->render('github::index', $data);
}
```

Стало (данные считаются через use case, а пагинация строится штатными сервисами):

```php
public function index(
    GetCommitListUseCase $getCommitListUseCase,
    PaginationFactory $paginationFactory,
    PaginationGuard $paginationGuard
): string {
    $pagination = $paginationFactory->create($getCommitListUseCase->count());

    $redirectUrl = $paginationGuard->redirectUrl($pagination);
    if ($redirectUrl !== null) {
        redirect($redirectUrl);
    }

    return $this->render->render('github::index', [
        'commits'    => $getCommitListUseCase->getPage($pagination->getPerPage(), $pagination->getOffset()),
        'pagination' => $pagination->render(),
    ]);
}
```

Сервисы `PaginationFactory`/`PaginationGuard` и use case можно получить как аргументы метода-экшена — контейнер подставит их по типу. Не забудьте связать интерфейс репозитория в `config/services.php` модуля:

```php
$services->set(GitCommitRepositoryInterface::class, EloquentGitCommitRepository::class)->public();
```

#### Готовые примеры в коде

Проще всего ориентироваться на уже переведённые штатные модули. В качестве компактных образцов посмотрите:

* **`modules/notifications`** — минимальный пример: репозиторий с `countNotifications()`/`getNotifications(limit, offset)`, use case с `count()`/`getPage()` и контроллер `IndexController` с `PaginationFactory`/`PaginationGuard`.
* **`modules/community`** — несколько списков с сортировками и фильтрами (учитывайте сохранение query-параметров в ссылках пагинации).

Если у вас несложный модуль, проще всего скопировать структуру из `modules/notifications` и адаптировать под свои сущности.

### Конвертация существующих данных (одноразовые команды)

В версии 9.9 редактор BB-кодов заменён на CKEditor (личные сообщения, библиотека, загрузки и комментарии к ним, а также комментарии фотоальбомов), а в библиотеке и загрузках добавлены ЧПУ на основе slug-ов. Чтобы привести **уже существующие** данные к новому формату, после обновления нужно один раз выполнить консольные команды.

Команды запускаются через CLI-вход `system/bin/console`:

```bash
# Конвертация BB-кодов в HTML
php system/bin/console mail:convert-bbcode          # личные сообщения
php system/bin/console library:convert-bbcode       # тексты статей библиотеки
php system/bin/console library:convert-comments     # комментарии библиотеки
php system/bin/console downloads:convert-bbcode      # описания файлов загрузок
php system/bin/console downloads:convert-comments    # комментарии загрузок
php system/bin/console album:convert-comments        # комментарии фотоальбомов

# Генерация slug-ов для ЧПУ
php system/bin/console library:generate-slugs       # разделы и статьи библиотеки
php system/bin/console downloads:generate-slugs     # разделы и файлы загрузок

# Нормализация устаревших ссылок в текстах постов форума
php system/bin/console forum:normalize-message-links # ссылки в сообщениях форума
```

Особенности:

* Команды **одноразовые**: после успешного запуска факт выполнения сохраняется в файле `config/autoload/one_time_tasks.local.php`, и повторный запуск будет пропущен с предупреждением. Чтобы выполнить команду повторно намеренно, добавьте флаг `--force`.
* Команды конвертации BB-кодов поддерживают флаг `--dry-run` (показывает, сколько записей будет затронуто, без сохранения изменений) и `--batch-size` (размер пакета обработки, по умолчанию 500).
* Конвертация BB-кодов идемпотентна: записи, уже сохранённые в HTML, повторно не обрабатываются.

#### Нормализация ссылок в постах форума

Команда `forum:normalize-message-links` нужна, если при переносе форума все ссылки в текстах постов оказались завёрнуты во внутренний редирект `/redirect/?url=…`, а ссылки на посты и темы остались в устаревшем виде (`?act=show_post&id=N`, `?type=topic&id=N`) и ходят через 301-редиректы. Команда:

* снимает обёртку `/redirect/` со **своих** (внутренних) ссылок, делая их прямыми; внешние ссылки остаются нетронутыми;
* переписывает `?act=show_post&id=N` в канонический адрес поста `/forum/post/N/`;
* переписывает `?type=topic&id=N` в ЧПУ темы (если тема существует).

Свой домен определяется по `johncms.homeurl`. Если исходные ссылки в постах содержат другой домен (например, старый адрес сайта), передайте его через `--domain` (опцию можно повторять):

```bash
# сначала оценить объём изменений, ничего не записывая
php system/bin/console forum:normalize-message-links --dry-run

# применить, указав при необходимости домен(ы) исходных ссылок
php system/bin/console forum:normalize-message-links --domain=example.com
```

Команда одноразовая (`--force` для повторного запуска) и поддерживает `--dry-run` и `--batch-size` (по умолчанию 500).

### Модуль «Коллекции»: создание таблиц

В версии 9.9 добавлен новый модуль **«Коллекции»** (`collections`). При обновлении существующей установки его таблицы не создаются автоматически — веб-инсталлятор запускается только на чистой системе. Поэтому после обновления нужно один раз выполнить скрипт из папки `install`:

```bash
cd install
php install_collections.php
```

Скрипт создаёт пять таблиц модуля: `collections`, `collection_fields`, `collection_sections`, `collection_items`, `collection_item_values`.

Особенности:

* Скрипт **безопасен при повторном запуске**: если таблица `collections` уже существует, он ничего не делает и сообщает, что модуль установлен.
* По умолчанию демо-данные не создаются. Чтобы завести демонстрационную коллекцию «Blog» с парой полей, разделом и элементом, добавьте флаг `--demo`:

  ```bash
  php install_collections.php --demo
  ```

* Модуль должен быть перечислен в `installed_modules` в файле `config/autoload/modules.global.php`. В штатной поставке он там уже есть; если конфигурационный файл правился вручную, добавьте `'collections'` в список, иначе скрипт не найдёт класс установщика модуля.

После установки управление коллекциями доступно в админ-панели по адресу `/admin/collections`.

### Модуль «Согласия»: создание таблиц

В версии 9.9 добавлен новый модуль **«Согласия»** (`consent`) — справочник согласий на обработку персональных данных и журнал их принятия. При обновлении существующей установки его таблицы не создаются автоматически. Поэтому после обновления нужно один раз выполнить скрипт из папки `install`:

```bash
cd install
php install_consent.php
```

Скрипт создаёт две таблицы модуля: `consents` (справочник согласий) и `consent_log` (журнал принятых согласий).

Особенности:

* Скрипт **безопасен при повторном запуске**: если таблица `consents` уже существует, он ничего не делает и сообщает, что модуль установлен.
* Модуль должен быть перечислен в `installed_modules` в файле `config/autoload/modules.global.php`. В штатной поставке он там уже есть; если конфигурационный файл правился вручную, добавьте `'consent'` в список, иначе скрипт не найдёт класс установщика модуля.
* Так как добавлен новый PSR-4 неймспейс `Johncms\Modules\Consent\`, после выкладки кода нужно обновить автозагрузчик Composer: `composer dump-autoload`.

После установки управление согласиями доступно в админ-панели по адресу `/admin/consents`, а журнал принятых согласий — по `/admin/consents/log`.

### Cookie-баннер и счётчики аналитики: столбец согласия

В версии 9.9 в модуль «Согласия» добавлена страница **«Cookie-баннер»** (`/admin/cookie-banner`), а у счётчиков аналитики появилась опция **«Загружать только после согласия на cookie»**. Настройки самого баннера хранятся в конфигурации (`config('johncms')`) и не требуют изменений в базе, но для опции счётчиков в таблицу `cms_counters` добавлен столбец `require_cookie_consent`.

При обновлении существующей установки столбец не добавляется автоматически. Поэтому после обновления нужно один раз выполнить скрипт из папки `install`:

```bash
cd install
php update_counters_cookie_consent.php
```

Особенности:

* Скрипт **безопасен при повторном запуске**: если столбец `require_cookie_consent` уже существует, он ничего не делает и сообщает, что таблица актуальна.
* Опция счётчика работает **вместе с cookie-баннером**: счётчик с включённым флагом подключается на странице только после того, как посетитель примет баннер (`/admin/cookie-banner` → включить баннер). До согласия код счётчика изолирован в `<template>` и не выполняется. Если баннер выключен, получить согласие нельзя, и такой счётчик загружаться не будет.
* Cookie-баннер выводится только в публичной теме `default`.
