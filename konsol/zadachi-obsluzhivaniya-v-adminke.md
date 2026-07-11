# Задачи обслуживания в админке

Часть консольных команд можно запускать прямо из админ-панели на странице **Обслуживание** (`/admin/maintenance`, доступна пользователям с правами ≥ 9). Это удобно для рутинных операций вроде очистки кэша или генерации карты сайта, когда нет доступа к консоли.

## Как это работает

Команда попадает на страницу обслуживания, если её класс помечен атрибутом `#[AsAdminTask]`. Страница показывает список таких команд, их статус и последний вывод, а также кнопку запуска.

Предусмотрено два режима выполнения:

* **Синхронный (foreground)** — команда выполняется прямо в запросе, а её вывод показывается на странице сразу после завершения. Подходит для быстрых операций.
* **Фоновый (background)** — при нажатии кнопки задача только ставится в очередь, а выполняется отдельно планировщиком. Это защищает от таймаутов веб-сервера и повторного запуска при обновлении страницы. Подходит для долгих операций.

## Как пометить команду

Добавьте атрибут `#[AsAdminTask]` на класс консольной команды:

```php
use Johncms\AdminTasks\AsAdminTask;
use Johncms\Scheduler\AsScheduledTask;
use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;

#[AsCommand(name: 'cache:clear', description: 'Clear application cache files')]
#[AsAdminTask(title: 'Clear cache', description: 'Remove all application cache files')]
final class CacheClearCommand extends Command
{
    // ...
}
```

Параметры атрибута:

* `title` — заголовок задачи на странице (по умолчанию — имя команды).
* `description` — описание задачи (по умолчанию — описание из `#[AsCommand]`).
* `background` — если `true`, задача выполняется в фоне через планировщик; по умолчанию `false` (синхронно).

Пример фоновой задачи:

```php
#[AsCommand(name: 'sitemap:generate', description: 'Generate sitemap and update robots.txt')]
#[AsScheduledTask(expression: '0 3 * * *', withoutOverlapping: true)]
#[AsAdminTask(title: 'Generate sitemap', description: 'Generate sitemap and update robots.txt', background: true)]
final class CronGenerateSitemapCommand extends Command
{
    // ...
}
```

Атрибуты независимы: `#[AsScheduledTask]` отвечает за запуск по расписанию, `#[AsAdminTask]` — за появление на странице обслуживания. Их можно использовать вместе или по отдельности.

## Обработка фоновой очереди

Фоновые задачи выполняет команда `admin-tasks:run-queued`. Она помечена `#[AsScheduledTask(expression: '* * * * *')]`, поэтому запускается автоматически при каждом вызове планировщика — **отдельная настройка cron не требуется**, достаточно уже настроенного `schedule:run` (см. [Планировщик задач](planirovshchik-zadach-schedule.md)).

Команда забирает задачи из очереди, помечает «зависшие» (например, после фатальной ошибки процесса) как завершившиеся с ошибкой и запускает каждую с защитой от параллельного выполнения.

При желании очередь можно обработать вручную:

```bash
php system/bin/console admin-tasks:run-queued
```

## Хранение данных

Состояние задач (статус, код выхода, вывод, метки времени) и файлы блокировок хранятся в:

```
data/admin_tasks
```

Каталог намеренно расположен вне `data/cache`, чтобы задача `cache:clear` не удаляла очередь и результаты во время работы. Он создаётся автоматически при первом запуске и не требует ручной подготовки.

## См. также

* [Консольные команды](konsolnye-komandy.md)
* [Планировщик задач (schedule)](planirovshchik-zadach-schedule.md)
