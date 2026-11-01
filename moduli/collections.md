# Коллекции: сохранение записей через PHP API

Коллекция хранит базовые свойства записи и дополнительные поля. Для записи используйте
`Johncms\Modules\Collections\Application\Api\CollectionsApiInterface` через DI. Модуль-потребитель
объявляет зависимость от `johncms/collections` в `requires.modules` своего `module.php`.

## Создание и изменение

```php
use Johncms\Modules\Collections\Application\Api\CollectionsApiInterface;
use Johncms\Modules\Collections\Application\DTO\CollectionItemFormDTO;

$form = new CollectionItemFormDTO(
    collectionId: $collectionId,
    sectionId: null,
    code: 'Release.V10+RC_1',
    name: 'Release',
    active: true,
    activeFrom: '2026-11-02T12:30',
    activeTo: null,
    sort: 100,
    previewText: '<p>Preview</p>',
    detailText: '<p>Content</p>',
    values: ['author' => 'Alice', 'tag' => ['php', 'cms']],
);

$id = $collections->addItem($form, actorId: $currentUserId);
$collections->updateItem($id, $form, actorId: $currentUserId);
```

`$collections` — внедрённый `CollectionsApiInterface`. ID пользователя берётся из доверенного
контекста операции. Если `actorId` не передан, операция считается системной. Создание заполняет
`created_by`, изменение — `updated_by`, сохраняя первоначального автора.

`updateItem()` требует существующую запись именно этой коллекции. Отсутствующий ID и попытка
переноса в другую коллекцию возвращают ошибку. Обновление полностью заменяет дополнительные
значения: необходимо передать весь набор полей; отсутствующие значения очищаются.

## Формат данных и ошибки

Название обязательно и ограничено 255 символами. Явный код допускает латинские буквы обоих
регистров, цифры, точку, плюс, дефис и подчёркивание. Пустой код создаётся из названия как slug,
при совпадении получает числовой суффикс. Код уникален в коллекции и разделе, включая записи
без раздела. Формы разделов допускают тот же алфавит.

Даты публикации принимаются в формате `Y-m-d H:i:s`, `Y-m-d H:i`, `Y-m-dTH:i:s` или
`Y-m-dTH:i`; пустая дата означает отсутствие границы. Дата окончания не может предшествовать
дате начала. В БД даты сохраняются как `Y-m-d H:i:s` без форматирования для интерфейса.

Дополнительные значения передаются строками; множественные — списками строк. Проверяются
обязательность, множественность и тип. Некорректное число не становится нулём, некорректная дата
не исчезает. Boolean принимает `0`, `1` и пустое значение как false; integer ограничен диапазоном
PHP int, relation — положительным integer, double — конечным числом. Проверка существования цели
relation и настройка вариантов select относятся к последующим возможностям этих типов.

Date принимает `Y-m-d` либо каноническую дату с нулевым временем, datetime — форматы даты-времени
выше. Короткие строковые поля ограничены 255 символами; анонс — 65535 байт.

`CollectionItemFormException::$errors` содержит сообщения по полям: `name`, `code`,
`section_id`, `active_from`, `active_to`, `sort`, `values.<код поля>` и другим проверяемым
атрибутам. Для занятого кода сохраняется отдельный `CollectionItemCodeAlreadyExistsException`.
Форма админки использует тот же сценарий и показывает ошибки у полей.

## Транзакции и конкуренция

Запись и дополнительные значения сохраняются одной транзакцией. Для более крупной атомарной
операции внедряйте `CollectionTransactionInterface` из Domain/Repository и вызывайте `run()`:
вложенное сохранение использует savepoint. Поддерживается также внешняя транзакция общего PDO
через `Johncms\Database\ConnectionInterface`. Исключение при сохранении возвращает прежние
данные; откат вызывающего кода откатывает и созданные записи. В сценариях сохранения нет DDL.

Сохранения одной коллекции сериализуются блокировкой её строки. Проверка занятого кода и
изменяемой записи выполняется locking read, а не чтением старого снимка внешней транзакции.
Гарантии действуют для записи через API или `SaveCollectionItemUseCase`; прямые записи в
репозитории/таблицы обходят этот протокол.

`CollectionWriteConflictException` означает, что из-за конкуренции нужно повторить всю внешнюю
операцию. Сам savepoint не обновляет старый снимок внешней транзакции. Такое поведение возможно,
в частности, при включённом [snapshot isolation MariaDB](https://mariadb.com/docs/server/server-usage/storage-engines/innodb/innodb-system-variables#innodb_snapshot_isolation).
Настройки сервера модуль не меняет. Админка предлагает повторить сохранение, сохраняя введённые данные.

## Чтение и постраничная выборка

Методы `findCollection()`, `getItem()` и `getItemByCode()` возвращают неизменяемые DTO,
а `getItems()` — обычный список `CollectionItemDTO`, без моделей и коллекций Eloquent.
Свойства именуются в camelCase: `collectionId`, `sectionId`, `previewText`, `createdAt`,
`createdBy`, `viewCount`. Значения дополнительных полей доступны в `$item->values` по коду.
Незаполненные и не загруженные поля отсутствуют в этой карте.

Даты записи и дополнительные поля Date/Datetime возвращаются как `DateTimeImmutable` в UTC;
исторически отсутствующие даты — как null. HTML возвращается исходным: подготовка для Twig
выполняется общим content pipeline в потребителе.

```php
use Johncms\Modules\Collections\Domain\Enums\FilterOperator;
use Johncms\Modules\Collections\Domain\Enums\SortDirection;
use Johncms\Modules\Collections\Domain\Query\ContentCollectionItemQuery;
use Johncms\Modules\Collections\Domain\Query\FieldFilterDTO;
use Johncms\Modules\Collections\Domain\Query\OrderByDTO;

$page = $collections->getPage(new ContentCollectionItemQuery(
    collectionId: $collectionId,
    filters: [new FieldFilterDTO('field.author', FilterOperator::Eq, 'Alice')],
    orderBy: [new OrderByDTO('item.created_at', SortDirection::Desc)],
    limit: 20,
    offset: 0,
    fields: ['author', 'tag'],
));

foreach ($page->items as $item) {
    // $item->id, $item->name, $item->values['author']
}
$total = $page->total;
```

`getPage()` возвращает `items`, `total`, `limit`, `offset`. Total не ограничивается limit/offset.
По умолчанию limit равен 100; допустимы 1–1000, offset — 0–1000000. Сортировка при равенстве
значений дополняется ID, если он не указан явно. Сортировка множественного поля использует
первое значение в порядке sort/id. Произвольного неограниченного чтения через этот API нет.

`fields: null` загружает все дополнительные значения, `fields: []` — ни одного,
список кодов — только выбранные. Загрузка выполняется пакетно, без отдельного запроса на запись.

### Адреса полей и фильтры

Базовые колонки адресуются как `item.name`, `item.id`, `item.active_from`, дополнительные —
как `field.name`, `field.author`. Одинаковый код дополнительного и базового поля не меняет
смысл запроса. Bare-коды без префикса не принимаются. Неизвестное поле, сортировка или выбранный
код, неверный тип значения и неподдерживаемый оператор дают `InvalidArgumentException`.

Integer/Relation-фильтры принимают int, Double — int/конечный float, Boolean — bool,
Date/Datetime — `DateTimeImmutable`, текстовые — string. `In` принимает список до 1000 значений
одного подходящего типа. `Like` поддерживается текстовыми типами, сравнения порядка — числовыми
и датами. `Eq` с null выбирает базовое NULL либо отсутствие дополнительного значения; null
внутри `In` не поддерживается. Фильтры соединяются через AND.

Сейчас `includeSubsections: true` отклоняется явно; поддержка поддерева относится к следующему
этапу API разделов. `onlyActive` пока проверяет активность записи и её временное окно;
полная политика видимости коллекции и разделов развивается отдельно.

## Частичное обновление

`updateItem()` сохраняет полную форму. Для изменения отдельных свойств используйте `patchItem()`:

```php
use Johncms\Modules\Collections\Application\DTO\CollectionItemPatchDTO;

$collections->patchItem($id, new CollectionItemPatchDTO(
    attributes: ['name' => 'New title', 'previewText' => null],
    values: ['tag' => ['php', 'news'], 'author' => null],
), actorId: $currentUserId);
```

Пропущенные базовые свойства и дополнительные поля сохраняются. Ключи `attributes` совпадают
со свойствами формы: `name`, `code`, `sectionId`, `active`, `activeFrom`, `activeTo`, `sort`,
`previewText`, `detailText`. Null допустим для sectionId, текстов и дат; название очищать нельзя.

Дополнительные значения передаются строками, множественные — списками строк, как в полной
форме. Переданный список заменяет весь набор значений поля. Null удаляет значение, включая
необязательный Boolean; строка `0` сохраняет Boolean false. Обязательное поле очистить нельзя.
Пропущенное поле не удаляется и не создаётся заново. Изменение коллекции, системных дат,
авторов или просмотров через `attributes` не допускается.

Чтение актуальной записи, слияние и сохранение находятся под одной транзакционной блокировкой.
Параллельные изменения разных свойств не перезаписывают друг друга.

## Просмотры

Чтение через API не меняет счётчики. Для публичного просмотра вызывайте
`$collections->incrementViews($id)` отдельно: увеличение атомарно и не меняет `updated_at`,
автора, редактора и содержимое. Отсутствующие, выключенные и находящиеся вне временного окна
записи, а также записи выключенной коллекции отклоняются. Проверка цепочки разделов будет
добавлена вместе с общей политикой публичной доступности.

Предпросмотр использует только чтение. Правило повторных просмотров (например, один просмотр
за сессию) остаётся у модуля-потребителя.

## Исторический импорт из хука обновления

`CollectionsImportApiInterface` создаёт записи и разделы с исходными датами. Он доступен внутри
доверенного контекста `SystemOperationContext`, который ядро открывает на время
`Installer::update()`. Обычный вызов импорта вне такого контекста отклоняется `LogicException`;
HTTP-параметра для включения контекста нет. Контекст закрывается и при исключении в хуке.

Внутри хука:

```php
use Johncms\Modules\Collections\Application\Api\CollectionsImportApiInterface;
use Johncms\Modules\Collections\Application\DTO\CollectionItemHistoryDTO;

$importer = di(CollectionsImportApiInterface::class);
$id = $importer->importItem($form, new CollectionItemHistoryDTO(
    createdAt: new DateTimeImmutable('2001-02-03T04:05:06Z'),
    updatedAt: new DateTimeImmutable('2002-03-04T05:06:07Z'),
    createdBy: 7,
    updatedBy: 8,
    viewCount: 123,
));
$sectionId = $importer->importSection($sectionForm, $createdAt, $updatedAt);
```

Импорт создаёт новые ID. Сопоставление старых и новых ID ведёт вызывающий код. Исторические
атрибуты не заменяются текущими; даты приводятся к UTC и проверяются на диапазон DATETIME.
У записи сохраняются исходные авторы, редактор и просмотры; у раздела — даты создания и изменения.
Проверки содержимого и принадлежности коллекции сохраняются. Ошибки раздела представлены
`CollectionSectionFormException`, записи — `CollectionItemFormException`.

Импорт использует общий транзакционный механизм, включая внешнюю транзакцию вызывающего.
DDL внутри операций импорта не выполняется. Изменять исторические атрибуты обычным patch нельзя.
