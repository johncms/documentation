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

namespace Mysite\MyModule\Application\Sitemap;

use Johncms\Sitemap\SitemapUrlEntry;
use Johncms\Sitemap\SitemapUrlProviderInterface;
use Mysite\MyModule\Domain\Models\MyModel;

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

Отдельно регистрировать провайдер не нужно: он попадает в контейнер вместе с остальным прикладным слоем модуля, а тег `johncms.sitemap_provider` система ставит сама — по реализованному интерфейсу. Достаточно, чтобы класс лежал внутри каталога, который загружает `config/services.php` модуля:

```php
<?php

declare(strict_types=1);

namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return static function (ContainerConfigurator $container): void {
    $services = $container->services();

    $services->load(
        'Mysite\\MyModule\\Application\\',
        dirname(__DIR__) . '/src/Application'
    )
        ->autowire()
        ->autoconfigure()
        ->public();
};
```

Ключевое здесь — `autoconfigure()`: именно он разрешает системе пометить провайдер тегом. Если вы объявляете класс отдельной строкой `set()`, не забудьте добавить `->autoconfigure()` к ней — иначе тег не проставится и провайдер молча не попадёт в sitemap.

После этого при следующем запуске планировщика провайдер будет автоматически подхвачен и его URL попадут в sitemap.

## Важные моменты

* `groupName()` должен быть уникальным среди всех провайдеров. При дублировании система выбросит исключение.
* Используйте `cursor()` вместо `get()` для выборки большого количества записей — это не загружает все записи в память сразу.
* Лимит URL на один файл — 45 000. При превышении система автоматически создаёт дополнительные чанки (`sitemap-my-module-2.xml` и т.д.).
