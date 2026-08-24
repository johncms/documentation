---
description: Как добавить URL-адреса своего модуля в автоматически генерируемый Sitemap
metaLinks:
  alternates:
    - https://app.gitbook.com/s/5NJeWEeBonlrBhBrVHEz/moduli/sitemap-provider
---

# Sitemap-провайдер

JohnCMS автоматически генерирует XML-карту сайта (`sitemap.xml`) по расписанию. Каждый модуль может добавить свои URL в sitemap, реализовав интерфейс `SitemapUrlProviderInterface` и зарегистрировав провайдер в контейнере.

## Как это работает

При генерации sitemap система собирает все зарегистрированные провайдеры (помеченные тегом `johncms.sitemap_provider`), вызывает у каждого метод `getEntries()` и записывает результат в отдельный файл `sitemap-{groupName}-1.xml`. Итоговый `sitemap.xml` — это индексный файл со ссылками на все чанки.

## Создание провайдера

Реализуйте интерфейс `Johncms\Sitemap\SitemapUrlProviderInterface`:

```php
<?php

declare(strict_types=1);

namespace Johncms\Modules\MyModule\Application\Sitemap;

use Johncms\Modules\MyModule\Domain\Models\MyModel;
use Johncms\Sitemap\SitemapUrlEntry;
use Johncms\Sitemap\SitemapUrlProviderInterface;

final class MyModuleUrlsProvider implements SitemapUrlProviderInterface
{
    public function groupName(): string
    {
        // Уникальное имя группы — определяет имя файла: sitemap-my-module-1.xml
        return 'my-module';
    }

    /**
     * @return iterable<SitemapUrlEntry>
     */
    public function getEntries(string $homeUrl): iterable
    {
        $items = MyModel::query()
            ->select(['id', 'slug', 'updated_at'])
            ->orderBy('id')
            ->cursor(); // cursor() вместо get() экономит память на больших таблицах

        foreach ($items as $item) {
            yield new SitemapUrlEntry(
                loc: $homeUrl . '/my-module/' . $item->slug . '/',
                lastmod: $item->updated_at?->format('c'),
            );
        }
    }
}
```

### SitemapUrlEntry

Конструктор принимает два параметра:

| Параметр  | Тип            | Описание                                                     |
| --------- | -------------- | ------------------------------------------------------------ |
| `loc`     | `string`       | Полный URL страницы (включая домен)                          |
| `lastmod` | `string\|null` | Дата последнего изменения в формате ISO 8601 (необязательно) |

## Регистрация провайдера

В файле `modules/<вендор>/my-module/config/services.php` зарегистрируйте провайдер с тегом `johncms.sitemap_provider`:

```php
<?php

declare(strict_types=1);

use Johncms\Modules\MyModule\Application\Sitemap\MyModuleUrlsProvider;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;

return function (ContainerConfigurator $configurator): void {
    $services = $configurator->services();

    $services
        ->set(MyModuleUrlsProvider::class)
        ->autowire()
        ->tag('johncms.sitemap_provider')
        ->public();
};
```

После этого при следующем запуске планировщика провайдер будет автоматически подхвачен и его URL попадут в sitemap.

## Важные моменты

* `groupName()` должен быть уникальным среди всех провайдеров. При дублировании система выбросит исключение.
* Используйте `cursor()` вместо `get()` для выборки большого количества записей — это не загружает все записи в память сразу.
* Лимит URL на один файл — 45 000. При превышении система автоматически создаёт дополнительные чанки (`sitemap-my-module-2.xml` и т.д.).
