# Пагинация

Начиная с версии 9.9 в JohnCMS появился собственный компонент пагинации **\Johncms\Http\Pagination**. Он заменил устаревший форк `johncms/johncms-pagination` (Laravel `LengthAwarePaginator` / метод `->paginate()`) и метод `Tools::displayPagination()` — оба **удалены** в версии 9.9. Весь код должен использовать только новый компонент.

> Обновляетесь с 9.8 и в ваших модулях есть `->paginate()`, `LengthAwarePaginator` или `Tools::displayPagination()`? Переход на новый компонент описан в инструкции по обновлению с 9.8 — она осталась в [документации ветки 9.9](https://github.com/johncms/documentation/blob/9.9/nachalo-raboty/obnovlenie-s-versii-9.8.md), так как обновляться на 10.0 нужно через 9.9.

## Состав компонента

* **`PaginationFactory`** — сервис DI-контейнера, создаёт объект `Pagination` из общего количества записей. Внедряется в контроллеры через конструктор.
* **`Pagination`** — неизменяемый (immutable) объект без обращения к контейнеру. Содержит математику страниц (`getOffset()`, `getPerPage()`, `getCurrentPage()`, `getTotalPages()`, `getTotal()`, `hasPages()`), строит URL страниц (`getUrl()`), отдаёт элементы для шаблона (`getItems()`) и рендерит готовый HTML (`render()`).
* **`PaginationGuard`** — сервис DI-контейнера, возвращает URL для редиректа с неканонических страниц. Сам редирект не выполняет (это упрощает тестирование) — контроллер вызывает `redirect()` явно.

## Использование в контроллере

```php
use Johncms\Http\PageMeta;
use Johncms\Http\Pagination\PaginationFactory;
use Johncms\Http\Pagination\PaginationGuard;

// В конструкторе контроллера: PaginationFactory $paginationFactory, PaginationGuard $paginationGuard

// 1. Строим пагинацию из общего количества записей (дешёвый COUNT-запрос).
$pagination = $this->paginationFactory->create($this->listUseCase->count());

// 2. Редирект с неканонических страниц.
$redirectUrl = $this->paginationGuard->redirectUrl($pagination);
if ($redirectUrl !== null) {
    redirect($redirectUrl);
}

// 3. Получаем срез данных по limit/offset из пагинации.
$items = $this->listUseCase->getPage($pagination->getPerPage(), $pagination->getOffset());

// 4. Строим мета-данные страницы и отдаём их шаблону вместе с пагинацией.
$meta = new PageMeta($pageTitle, $pagination->getCurrentPage());
return new ViewResponse('@module/public/index.twig', [
    'title'       => $meta->title,
    'description' => $meta->description,
    'items'       => $items,
    'pagination'  => $pagination->render(),
]);
```

`render()` возвращает готовую разметку (`Twig\Markup`), поэтому в шаблоне она выводится как
обычная переменная — фильтр `|raw` не нужен:

```twig
{{ pagination }}
```

### Параметры `PaginationFactory::create()`

```php
create(int $total, ?int $perPage = null, string $pageParamName = 'page', ?int $currentPage = null): Pagination
```

* `total` — общее количество записей.
* `perPage` — размер страницы. По умолчанию берётся из настроек текущего пользователя (`config->kmess`). Передавайте явно только если у списка собственный размер страницы.
* `pageParamName` — имя GET-параметра страницы, по умолчанию `page`.
* `currentPage` — номер текущей страницы. По умолчанию берётся из GET-параметра; передавайте явно только вне HTTP-контекста (например, в консольных командах).

## Канонические URL страниц

Первая страница канонична **без** параметра `page` — метод `getUrl(1)` возвращает URL без него. `PaginationGuard` приводит остальные случаи к каноническому виду:

* `?page=1`, мусорное значение (`?page=abc`) или `page < 1` → URL без параметра `page`;
* `page > totalPages` → URL последней страницы.

Если страница уже каноничная, `redirectUrl()` возвращает `null`. Редиректы выполняются со статусом 302.

## Разделение use case / репозиторий

Use case не должен знать ни про HTML-рендер, ни про пагинатор. Доступ к данным разделяется на два метода, а срезом управляет контроллер:

* `count(): int` → в репозитории метод `count*(...)` (запрос `COUNT`);
* `getPage(int $limit, int $offset): array` → в репозитории метод `get*(..., int $limit, int $offset): Collection`, результат маппится в DTO.

Репозитории принимают явные `limit`/`offset` и возвращают `Collection`. Метод `->paginate()` использовать нельзя.

## Рендеринг

* В стандартном случае используйте `$pagination->render()` в контроллере — HTML строится из шаблона `system::app/pagination` (темы `default` и `admin`).
* Для собственной разметки или JSON-эндпоинтов используйте `$pagination->getItems()` — массив элементов с сырыми полями `type`/`page`/`url`/`active`.
* Экранирование происходит на выводе: шаблон сам экранирует URL через `$this->e(...)`. Не экранируйте данные заранее в PHP.

## Заголовок и описание страницы (PageMeta)

Для формирования `title` и meta `description` на страницах с пагинацией используйте `Johncms\Http\PageMeta` — начиная со 2-й страницы он автоматически добавляет суффикс с номером страницы (разделитель ` — ` и переведённое слово «Page»):

```php
use Johncms\Http\PageMeta;

$meta = new PageMeta($documentTitle, $pagination->getCurrentPage());
// или с собственным описанием:
$meta = new PageMeta($documentTitle, $pagination->getCurrentPage(), $description);
```

* Первая страница остаётся без суффикса — `title` и `description` не изменяются.
* Если `description` не передан или пуст, базой для него служит `title`.
