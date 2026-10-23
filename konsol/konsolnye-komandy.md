---
metaLinks:
  alternates:
    - https://app.gitbook.com/s/5NJeWEeBonlrBhBrVHEz/konsol/konsolnye-komandy
---

# Консольные команды

В JohnCMS есть единая консольная точка входа:

```bash
php system/bin/console
```

Команда выводит список всех доступных CLI-команд и их описания.

Консоль создаёт только ту команду, которую запускают. Поэтому команды, которым не нужна база данных, — `list`, `cache:clear`, `twig:lint` и другие — работают и тогда, когда база недоступна.

## Базовые команды

### Показать список команд

```bash
php system/bin/console list
```

### Показать справку по команде

```bash
php system/bin/console help schedule:run
```

## Текущие команды проекта

Команды, доступные в текущей версии:

* `migrate` — применение миграций базы данных (см. [Миграции базы данных](../baza-dannykh/migracii.md))
* `migrate:status` — что применено, а что ждёт своей очереди
* `migrate:rollback` — откат последнего прогона миграций (для разработки)
* `make:migration` — создание новой миграции
* `i18n:scan` — сканирование исходников и генерация POT-файлов
* `i18n:translate` — генерация `*.lng.php` из `*.po`
* `mail:send-pending` — отправка накопившейся почтовой очереди
* `mail:cleanup` — удаление давно доставленных писем из очереди
* `module:list` — все модули, которые знает сайт, и их состояние (см. [Установка и удаление модулей](../moduli/ustanovka-modulei.md))
* `module:install`, `module:update`, `module:enable`, `module:disable`, `module:uninstall` — установка, обновление, включение, выключение и удаление модуля
* `module:sync` — запись модулей поставки в файл состояния сайта и удаление из него исчезнувших
* `module:publish-assets` — копирование ассетов установленных модулей в `public/modules`
* `auth:sync-roles` — создание встроенных ролей текущей версии и выдача им недостающих прав (см. [Права доступа](../moduli/prava-dostupa.md))
* `auth:cleanup` — удаление недействительных сессий, истёкших ссылок восстановления пароля и старых записей журнала входов (см. [Журнал входов](../polzovateli/zhurnal-vkhodov.md))
* `auth:migrate-legacy-access`, `auth:drop-legacy-columns` — одноразовые шаги перехода с 9.9 на роли (см. [Обновление с версии 9.9](../nachalo-raboty/obnovlenie-s-versii-9.9.md))
* `forum:cleanup-orphan-files` — очистка orphan-файлов форума
* `sitemap:generate` — генерация sitemap
* `schedule:list` — список задач планировщика
* `schedule:run` — запуск задач, которые должны выполниться в текущую минуту
* `router:list` — список зарегистрированных маршрутов
* `twig:lint` — проверка синтаксиса всех Twig-шаблонов
* `twig:compile` — компиляция шаблонов активной темы в кэш
* `twig:ide-config` — генерация `ide-twig.json`, по которому IDE находит шаблоны вида `@namespace/...`
* `cache:clear` — полная очистка `data/cache`: кэш приложения, скомпилированный контейнер, маршруты и шаблоны
* `cache:pool:clear` — очистка кэша приложения; с указанием тегов сбрасывает только их (см. [Кэширование](../obshie-svedeniya/caching.md))
* `cache:pool:prune` — освобождение места, занятого истёкшими и сброшенными записями кэша
* `admin-tasks:run-queued` — запуск задач обслуживания, поставленных в очередь из админки (см. [Задачи обслуживания в админке](zadachi-obsluzhivaniya-v-adminke.md))

### Показать все маршруты роутера

```bash
php system/bin/console router:list
```

### Показать маршруты с деталями

```bash
php system/bin/console router:list --details
```

### Очистить кэш приложения

Полностью, вместе со скомпилированным контейнером и шаблонами — это то, что нужно после обновления CMS:

```bash
php system/bin/console cache:clear
```

Только данные, накэшированные модулями, — контейнер и шаблоны при этом не трогаются:

```bash
php system/bin/console cache:pool:clear
```

Только записи с определёнными тегами:

```bash
php system/bin/console cache:pool:clear news counters
```

## Запуск в Docker

Если проект запущен через docker compose, команды обычно выполняют внутри контейнера `php-fpm`:

```bash
docker exec -it $(docker ps -q -f name=${COMPOSE_PROJECT_NAME}.php-fpm) php system/bin/console list
```

## Как добавить свою команду

1. Создайте класс, наследующий `Symfony\Component\Console\Command\Command`.
2. Добавьте атрибут `#[AsCommand(...)]` — и с именем, и с описанием команды.
3. Для вывода используйте `SymfonyStyle`.

```php
<?php

declare(strict_types=1);

namespace Mysite\MyModule\Application\Console;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Style\SymfonyStyle;

#[AsCommand(name: 'my-module:cleanup', description: 'Delete drafts nobody has touched for a year')]
final class CleanupDraftsCommand extends Command
{
    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $io = new SymfonyStyle($input, $output);
        // ...
        $io->success('Done.');

        return self::SUCCESS;
    }
}
```

Команда, которая работает с данными модуля, должна лежать в самом модуле: иначе выключение модуля ломает сборку контейнера.

### Регистрация

Отдельно регистрировать команду не нужно. Система сама вешает на неё тег `console.command` — стандартный тег Symfony — по тому, что класс наследует `Command`. Условие одно: класс должен быть объявлен с `autoconfigure()`. Блок `load()` в `config/services.php` модуля это делает:

```php
$services->load(
    'Mysite\\MyModule\\Application\\',
    dirname(__DIR__) . '/src/Application'
)
    ->autowire()
    ->autoconfigure()
    ->public();
```

Если класс объявлен отдельной строкой `set()` без `autoconfigure()`, укажите тег явно:

```php
$services->set(CleanupDraftsCommand::class)->autowire()->tag('console.command');
```

После этого команда появляется в `system/bin/console list`.

### Команда создаётся только при запуске

Список команд консоль составляет по атрибуту `#[AsCommand]`, не создавая их. Объект команды вместе со всеми зависимостями из конструктора строится только тогда, когда её запускают. Отсюда три следствия:

* Зависимости команды не мешают остальным. Если команде нужна база, без базы не запустится только она.
* Описание тоже должно быть в атрибуте. Команду без описания консоли приходится создавать, чтобы показать её в `list`.
* Команда, которая задаёт имя через `setName()` в `configure()`, работает, но создаётся при каждом запуске консоли вместе со всем, от чего зависит. Если ей нужна база, без базы не запустится ни одна команда.

### Примеры в коде

Если хотите быстро посмотреть рабочие реализации, ориентируйтесь на эти классы:

* `system/src/Console/Commands/I18nScanCommand.php`
* `system/src/Console/Commands/I18nTranslateCommand.php`
* `system/src/Console/Commands/ScheduleListCommand.php`
* `system/src/Console/Commands/ScheduleRunCommand.php`
* `system/src/Console/Commands/CronSendEmailCommand.php`
* `modules/johncms/forum/src/Application/Console/CleanupOrphanFilesCommand.php` — команда модуля

Для примера планировщика (расписание через атрибут) см.:

* `system/src/Scheduler/AsScheduledTask.php`
* `system/src/Scheduler/ScheduleRunner.php`
* `system/src/Scheduler/ScheduledTaskRegistry.php`

## См. также

* [Планировщик задач (schedule)](planirovshchik-zadach-schedule.md)
* [Задачи обслуживания в админке](zadachi-obsluzhivaniya-v-adminke.md)
* [Конфигурационные файлы (configs)](../obshie-svedeniya/konfiguracionnye-faily-configs.md)
