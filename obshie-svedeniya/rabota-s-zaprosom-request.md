---
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/5NJeWEeBonlrBhBrVHEz/obshie-svedeniya/rabota-s-zaprosom-request
---

# Работа с запросом (Request)

Данные HTTP запроса в JohnCMS представлены классом **\Johncms\Http\Request**. Это тонкая обёртка над `Symfony\Component\HttpFoundation\Request`: доступны все методы HttpFoundation (`getClientIp()`, `isSecure()`, `getPathInfo()`, бэги `query`, `request`, `cookies`, `files`, `headers`, `server`, `attributes`), а обёртка добавляет к ним несколько коротких методов для самых частых операций чтения.

## Как получить запрос

Запрос принадлежит одному циклу обработки, поэтому он **не является сервисом контейнера**. Способ получить его ровно один: запрос передают туда, где он нужен.

### В контроллере — аргумент действия

Объявите параметр с типом `Request` — запрос подставится в него автоматически:

```php
<?php

declare(strict_types=1);

namespace Johncms\Modules\MyModule\Application\Controllers;

use Johncms\Http\Request;
use Symfony\Component\HttpFoundation\Response;

final class MyController
{
    public function view(Request $request, int $id): Response
    {
        $page = $request->queryInt('page', 1);

        // ...
    }
}
```

Запрос подставляется по типу параметра, а параметры маршрута — по имени, поэтому порядок аргументов роли не играет.

Запрос объявляют только те действия, которые действительно его читают. Если действие лишь показывает форму — параметр не нужен.

{% hint style="danger" %}
Не сохраняйте запрос в конструкторе или в свойстве контроллера. Контроллеры — синглтоны контейнера, поэтому сохранённый запрос переживёт цикл, которому принадлежит, и в worker-режиме следующие посетители получат ответ по данным первого. По той же причине приватным методам контроллера запрос передают параметром.
{% endhint %}

### В middleware — аргумент `handle()`

```php
public function handle(Request $request, callable $next): Response
{
    // проверка/подготовка
    return $next($request);
}
```

### В сервисе, который живёт дольше запроса

Варианты в порядке предпочтения:

1. принять нужный факт параметром (строку адреса, хост — см. `ClientInfoDTO`);
2. принять `Request` параметром того метода, который его читает, если нужен целый набор полей;
3. прочитать текущий запрос из `Symfony\Component\HttpFoundation\RequestStack`.

Стек — крайний вариант и допустим только в `system/src/`; в Application-слое модуля это тот же скрытый захват запроса, только в другой форме. Он оправдан, когда вызывающих десятки и передать запрос неоткуда (`PaginationFactory`, `Theme`, `Environment`).

### В шаблоне — факт, а не запрос

Шаблон не обращается к запросу. Нужный ему факт отдаёт тонкий сервис поверх стека, и шаблон резолвит именно этот сервис:

```php
$currentPage = di(\Johncms\Http\CurrentPage::class);

if ($currentPage->isHomePage()) {
    // ...
}
```

{% hint style="warning" %}
`di(\Johncms\Http\Request::class)` и `$container->get(Request::class)` бросают исключение: такого сервиса нет. Если вы встретили этот вызов в старом коде или стороннем модуле — его нужно заменить на аргумент действия.
{% endhint %}

## Получение данных из строки запроса ($\_GET)

Пользователь открыл `http://domain.com/?user_id=123&search=john`:

```php
$userId = $request->queryInt('user_id');          // 123, по умолчанию 0
$page   = $request->queryInt('page', 1);          // 1, если параметра нет
$search = $request->queryParam('search');         // 'john', по умолчанию ''
$ids    = $request->queryInts('ids');             // список чисел из ?ids[]=1&ids[]=2
```

Первым параметром идёт имя параметра запроса, вторым — значение по умолчанию. Отдельного аргумента с фильтром нет: тип задаёт сам метод.

## Получение данных из тела запроса ($\_POST и JSON)

Методы `body*` читают тело запроса независимо от того, пришло оно формой или JSON:

```php
$name  = $request->body('name');                  // строка, по умолчанию ''
$type  = $request->body('type', 'default');
$id    = $request->bodyInt('user_id');            // число, по умолчанию 0
$files = $request->bodyInts('attached_files');    // список чисел
$users = $request->bodyList('users');             // список без приведения типа
```

Проверить наличие ключа (например, галочки в форме) можно так:

```php
if ($request->hasBody('subscribe')) {
    // чекбокс отмечен
}
```

Метод запроса проверяется через `isPost()` или общий `isMethod()`:

```php
if ($request->isPost()) {
    // обработка отправленной формы
}
```

## Некорректные данные

`queryInt()` и `bodyInt()` мягко относятся к мусору: `?id=abc` вернёт значение по умолчанию, а не ошибку. Но массив в скалярном параметре (`?id[]=1`) — это попытка подмены типа, она намеренно не подавляется и превращается в ответ **400 Bad Request**.

Если нужна строгая семантика, обращайтесь к бэгам HttpFoundation напрямую — там неверное значение бросает исключение:

```php
$id = $request->query->getInt('id');
```

## Строки приходят обрезанными

Все строки в теле формы и в строке запроса обрезаются по краям (`trim`) глобальным middleware `TrimStringsMiddleware` до того, как отработает контроллер. Это касается и чтения через бэги напрямую. Тело в формате JSON не обрезается.

## Параметры маршрута

Параметры маршрута — это не данные запроса, их объявляют аргументами действия по имени, и они приводятся к типу аргумента:

```php
// маршрут: /forum/{id}/page/{page}
public function topic(Request $request, int $id, int $page = 1): Response
```

При необходимости все параметры совпавшего маршрута доступны как атрибуты запроса:

```php
$params = $request->attributes->all();
```

## Cookies, заголовки и данные сервера

Для них используются штатные бэги HttpFoundation:

```php
$theme     = $request->cookies->get('theme', 'default');
$userAgent = $request->headers->get('User-Agent', '');
$ip        = $request->getClientIp();
$isSecure  = $request->isSecure();
```

## Получение файлов ($\_FILES)

Загруженные файлы доступны в бэге `files`. Для одного поля:

```php
$uploaded = $request->files->get('imagefile');
```

Для всех сразу — `$request->files->all()`. Множественное поле (`<input type="file" name="photos[]" multiple>`) возвращается уже нормальным списком объектов, собирать структуру `$_FILES` вручную не нужно.

Элемент бэга — это `Symfony\Component\HttpFoundation\File\UploadedFile`, то есть HTTP-тип. Он не должен покидать слой HTTP: контроллер преобразует его в `\Johncms\Http\UploadedFileDTO` с помощью `\Johncms\Http\UploadedFileMapper`, и дальше — в use case, сервисы, хранилище — передаётся уже DTO.

```php
use Johncms\Http\UploadedFileMapper;
use Symfony\Component\HttpFoundation\File\UploadedFile;

final class PhotoUploadController
{
    public function __construct(
        private readonly SavePhotoUseCase $savePhotoUseCase,
        private readonly UploadedFileMapper $uploadedFileMapper,
    ) {
    }

    public function upload(Request $request, int $albumId): Response
    {
        $uploaded = $request->files->get('imagefile');
        if (! $uploaded instanceof UploadedFile) {
            // файл не пришёл или загрузка не удалась
        }

        $this->savePhotoUseCase->execute(
            $albumId,
            $this->uploadedFileMapper->fromUploadedFile($uploaded),
            $request->body('description'),
        );

        // ...
    }
}
```

Сам DTO умеет проверять успешность загрузки и переместить файл, поэтому оригинальный HTTP-объект дальше не нужен:

```php
if (! $file->isValid()) {
    throw new RuntimeException('Ошибка загрузки файла');
}

$file->moveTo(UPLOAD_PATH . 'photos/' . $newName);
```

Доступные поля DTO: `clientName`, `mimeType`, `size`, `tmpPath`, `error`.

{% hint style="danger" %}
В примерах рассмотрен простой вариант сохранения файлов без проверок допустимых типов и размеров. Имя файла, полученное от клиента, использовать как имя на диске нельзя — генерируйте своё.
{% endhint %}
