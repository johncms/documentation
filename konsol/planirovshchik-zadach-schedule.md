# Планировщик задач (schedule)

Для периодических задач в JohnCMS используется встроенный планировщик:

* `schedule:list` — посмотреть список зарегистрированных задач
* `schedule:run` — выполнить задачи, которые должны сработать в текущую минуту

## Быстрый старт

### Показать задачи

```bash
php system/bin/console schedule:list
```

### Выполнить due-задачи

```bash
php system/bin/console schedule:run --no-interaction
```

## Как зарегистрировать задачу

Задачи описываются атрибутом `AsScheduledTask` прямо на классе консольной команды:

```php
#[AsCommand(name: 'sitemap:generate', description: 'Generate sitemap')]
#[AsScheduledTask(expression: '0 3 * * *', withoutOverlapping: true)]
final class SitemapGenerateCommand extends Command
{
    // ...
}
```

Поддерживаемые параметры:

* `expression` — cron-выражение (обязательно)
* `timezone` — таймзона для вычисления расписания
* `arguments` — аргументы/опции, передаваемые в команду при запуске планировщиком
* `withoutOverlapping` — запрет параллельного запуска одной и той же задачи
* `description` — описание задачи для вывода в `schedule:list`

## withoutOverlapping

При `withoutOverlapping: true` используется file-lock.
Lock-файлы создаются в:

```text
data/cache/schedule
```

Это защищает от повторного запуска одной задачи, если предыдущий запуск еще не завершился.

## Запуск по cron

Планировщик рассчитан на запуск раз в минуту любым внешним cron-механизмом
(системный cron, scheduler панели хостинга, контейнерный scheduler и т.д.).

Рекомендуемая команда:

```text
php /app/system/bin/console schedule:run --no-interaction
```

`/app/system/bin/console` — это пример пути для docker-окружения.
В вашем cron нужно указать корректный абсолютный путь к файлу `system/bin/console`
для вашего размещения проекта (с учетом document root/рабочей директории на сервере).

## Типовые cron-выражения

* Каждую минуту: `* * * * *`
* Каждый час (в начале часа): `0 * * * *`
* Каждый день в 03:00: `0 3 * * *`

## Частые проблемы

* Неверное cron-выражение — задача пропускается, ошибка уходит в лог.
* Команда не найдена — проверьте `#[AsCommand(name: ...)]`.
* Задача постоянно пропускается с overlap — проверьте, не висит ли старый процесс.

## См. также

* [Консольные команды](konsolnye-komandy.md)
* [Проблемы и их решение](../obshie-svedeniya/problemy-i-ikh-reshenie.md)
