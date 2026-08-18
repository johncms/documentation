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

Ниже примеры команд, доступных в текущей версии:

* `i18n:scan` — сканирование исходников и генерация POT-файлов
* `i18n:translate` — генерация `*.lng.php` из `*.po`
* `mail:send-pending` — отправка накопившейся почтовой очереди
* `mail:cleanup` — удаление давно доставленных писем из очереди
* `mail:upgrade-schema` — добавление в таблицу очереди колонок, отвечающих за повторные попытки (для сайтов, обновляющихся со старых версий)
* `forum:cleanup-orphan-files` — очистка orphan-файлов форума
* `sitemap:generate` — генерация sitemap
* `schedule:list` — список задач планировщика
* `schedule:run` — запуск задач, которые должны выполниться в текущую минуту
* `router:list` — список зарегистрированных маршрутов
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
2. Добавьте атрибут `#[AsCommand(...)]` с именем и описанием команды.
3. Для вывода используйте `SymfonyStyle`.

Команды автоматически подхватываются контейнером и появляются в `system/bin/console list`.

### Примеры в коде

Если хотите быстро посмотреть рабочие реализации, ориентируйтесь на эти классы:

* `system/src/Console/Commands/I18nScanCommand.php`
* `system/src/Console/Commands/I18nTranslateCommand.php`
* `system/src/Console/Commands/ScheduleListCommand.php`
* `system/src/Console/Commands/ScheduleRunCommand.php`
* `system/src/Console/Commands/CronSendEmailCommand.php`

Для примера планировщика (расписание через атрибут) см.:

* `system/src/Scheduler/AsScheduledTask.php`
* `system/src/Scheduler/ScheduleRunner.php`
* `system/src/Scheduler/ScheduledTaskRegistry.php`

## См. также

* [Планировщик задач (schedule)](planirovshchik-zadach-schedule.md)
* [Задачи обслуживания в админке](zadachi-obsluzhivaniya-v-adminke.md)
* [Конфигурационные файлы (configs)](../obshie-svedeniya/konfiguracionnye-faily-configs.md)
