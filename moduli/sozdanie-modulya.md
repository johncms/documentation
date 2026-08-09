---
metaLinks:
  alternates:
    - https://app.gitbook.com/s/5NJeWEeBonlrBhBrVHEz/moduli/sozdanie-modulya
---

# Создание модуля

Давайте создадим свой первый простой модуль.\
Это будет обычная простая страница со списком наших партнёров.

Как нам уже известно, модули располагаются в папке **modules**.

Сначала создадим папку с модулем и назовём её **partners**, путь к папке получится такой: **modules/partners**.

Пока создадим простой модуль без мультиязычности.

Начиная с версии **9.9** рекомендуется использовать слоистую [структуру модуля](struktura-modulya.md). Исходный код располагается в папке **src** и делится по слоям (Application, Domain, Infrastructure). Для нашего простого модуля понадобится только прикладной слой (Application), конфигурация и шаблоны.

После выполнения всех действий из этой статьи у нас получится такая структура:

* modules
  * partners
    * config
      * routes.php
    * src
      * Application
        * Controllers
          * PartnersController.php
    * templates
      * public
        * index.twig

{% hint style="info" %}
Старая структура (папка `Controllers` прямо в корне модуля) по-прежнему работает. Но для новых модулей рекомендуется использовать новую структуру, описанную здесь.
{% endhint %}

Контроллеры позволяют избавиться от большого количества базового кода, который необходимо написать для начала работы, а также упрощают настройку маршрутов: не нужно самостоятельно писать логику по определению страницы, которую необходимо показать пользователю.

Контроллеры являются обычными PHP-классами. В JohnCMS используется автозагрузка классов модулей по стандарту [PSR-4](https://www.php-fig.org/psr/psr-4/). Чтобы она работала, нужно придерживаться некоторых правил:

1. Классы модуля должны располагаться в папке **src** внутри модуля, а [пространство имён](https://www.php.net/manual/ru/language.namespaces.rationale.php) должно соответствовать структуре папок. Разрешено использовать любые директории внутри **src** для логического разделения классов.
2. Для работы автозагрузки пространство имён модуля должно быть зарегистрировано в `composer.json`, а сам модуль — в конфигурации. Как это сделать, рассмотрим ниже.

## Регистрация пространства имён

Чтобы классы модуля загружались автоматически, зарегистрируйте пространство имён в секции `autoload.psr-4` файла `composer.json`:

```json
"Johncms\\Modules\\Partners\\": "modules/partners/src/"
```

Таким образом, пространством имён нашего модуля будет **Johncms\Modules\Partners**, и оно указывает на папку **modules/partners/src**.

После регистрации пространства имён нужно обновить карту автозагрузки, выполнив команду:

```bash
composer dump-autoload
```

## Регистрация модуля

Помимо автозагрузки классов, модуль нужно зарегистрировать в системе, добавив его в список установленных модулей.

Создайте в папке **config/autoload** файл с именем **modules.local.php** (если его ещё нет) со следующим содержимым:

{% code title="/config/autoload/modules.local.php" %}
```php
<?php

return [
    'modules' => [
        'installed_modules' => [
            'partners', // Название папки с модулем
        ],
    ],
];
```
{% endcode %}

В данном случае **partners** — это название папки с модулем. При добавлении дополнительных модулей просто добавьте их названия по аналогии.

## Создание контроллера

Создадим наш первый контроллер, который будет отвечать за отображение страницы партнёров.

Исходя из типовой [структуры модуля](struktura-modulya.md), классы контроллеров располагаются в папке **src/Application/Controllers**. С учётом зарегистрированного пространства имён полное пространство имён для нашего контроллера будет таким: `Johncms\Modules\Partners\Application\Controllers`.

Давайте создадим файл **PartnersController.php** со следующим содержимым:

{% code title="modules/partners/src/Application/Controllers/PartnersController.php" %}
```php
<?php

declare(strict_types=1);

namespace Johncms\Modules\Partners\Application\Controllers;

use Johncms\Http\View\ViewResponse;
use Johncms\NavChain;

final readonly class PartnersController
{
    public function __construct(
        private NavChain $navChain,
    ) {
    }

    public function __invoke(): ViewResponse
    {
    }
}
```
{% endcode %}

Разберёмся с этим кодом:

* Контроллер объявлен как `final readonly class` — это рекомендуемый стиль для новых классов.
* Зависимости передаются через конструктор (constructor injection) и разрешаются автоматически контейнером зависимостей. Нам понадобятся:
  * `NavChain` — сервис для работы с цепочкой навигации (хлебными крошками).
* Метод `__invoke()` делает контроллер «вызываемым»: именно он выполняется при обращении к маршруту. Он возвращает `ViewResponse` — имя шаблона и данные для него. Сам контроллер страницу не рендерит: этим занимается ядро.
* Подключать локализацию модуля не нужно: маршруты, объявленные в `modules/partners/config/routes.php`, помечены модулем `partners`, и ядро подключает его переводы на каждый запрос. Поэтому `__()` в контроллере и в его шаблонах берёт строки из `modules/partners/locale`.

Теперь дополним метод `__invoke()`.

Добавим нашу страницу в цепочку навигации:

```php
$this->navChain->add('Партнёры', '/partners/');
```

Подготовим данные для шаблона. Наполним массив нашими партнёрами и передадим его в шаблон:

```php
// Собираем массив данных, который будет передан в шаблон
$data = [
    'partners' => [
        [
            'name' => 'JohnCMS', // Название партнёра
            'url'  => 'https://johncms.com', // Ссылка на сайт партнёра
        ],
        [
            'name' => 'Партнёр 2',
            'url'  => 'https://example.com',
        ],
        [
            'name' => 'Партнёр 3',
            'url'  => 'https://example.org',
        ],
    ],
];

return new ViewResponse(
    '@partners/public/index.twig',
    [
        // Заголовок в теге title и заголовок страницы (h1)
        'title'      => 'Партнёры',
        'page_title' => 'Наши партнёры',
        'partners'   => $data['partners'],
    ]
);
```

Обратите внимание на последние строки. У каждого модуля есть своё пространство имён для шаблонов, совпадающее с названием его папки; оно появляется само, как только модуль перечислен в `config/autoload/modules.global.php`. В имени `'@partners/public/index.twig'` после `@` — название модуля, дальше — путь к файлу внутри папки **templates**.

Вторым аргументом передаётся массив данных: его ключи становятся именами переменных шаблона. В нашем примере в шаблоне будут доступны `title`, `page_title` и `partners`.

### Полный код файла контроллера

{% code title="modules/partners/src/Application/Controllers/PartnersController.php" %}
```php
<?php

declare(strict_types=1);

namespace Johncms\Modules\Partners\Application\Controllers;

use Johncms\Http\View\ViewResponse;
use Johncms\NavChain;

final readonly class PartnersController
{
    public function __construct(
        private NavChain $navChain,
    ) {
    }

    public function __invoke(): ViewResponse
    {
        // Добавляем страницу в цепочку навигации
        $this->navChain->add('Партнёры', '/partners/');

        // Собираем массив данных, который будет передан в шаблон
        $data = [
            'partners' => [
                [
                    'name' => 'JohnCMS', // Название партнёра
                    'url'  => 'https://johncms.com', // Ссылка на сайт партнёра
                ],
                [
                    'name' => 'Партнёр 2',
                    'url'  => 'https://example.com',
                ],
                [
                    'name' => 'Партнёр 3',
                    'url'  => 'https://example.org',
                ],
            ],
        ];

        // Контроллер не рендерит страницу сам, а возвращает имя шаблона и данные для него.
        // Заголовок в теге title и заголовок страницы (h1) передаются там же.
        return new ViewResponse(
            '@partners/public/index.twig',
            [
                'title'      => 'Партнёры',
                'page_title' => 'Наши партнёры',
                'partners'   => $data['partners'],
            ]
        );
    }
}
```
{% endcode %}

## Создание шаблона

Шаблоны пишутся на Twig. Публичные страницы модуля лежат в папке **templates/public**,
админские — в **templates/admin**. Наша страница будет называться **index.twig**.

Имя шаблона в контроллере складывается из пространства имён модуля и пути к файлу:
`@partners/public/index.twig`. Регистрировать пространство имён не нужно — оно появляется
само, как только модуль перечислен в `config/autoload/modules.global.php`.

{% code title="modules/partners/templates/public/index.twig" %}
```twig
{#
    Список партнёров.

    @var partners array
#}
{% extends '@theme/layouts/default.twig' %}

{% block content %}
    <div>
        Мы сотрудничаем со следующими партнёрами:
    </div>

    <ul>
        {# Перебираем массив партнёров и выводим название со ссылкой на сайт #}
        {% for partner in partners %}
            <li><a href="{{ partner.url }}">{{ partner.name }}</a></li>
        {% endfor %}
    </ul>
{% endblock %}
```
{% endcode %}

{% hint style="info" %}
Twig экранирует всё, что печатает, поэтому отдельно вызывать `htmlspecialchars()` не нужно —
и наоборот, не стоит экранировать данные при сохранении в базу.
{% endhint %}

## Добавление маршрута

Наш модуль готов, но пока ещё не доступен в браузере. Давайте это исправим.\
Чтобы модуль стал доступен, нужно создать файл `config/routes.php` внутри папки модуля. Система подхватит его автоматически.

{% code title="modules/partners/config/routes.php" %}
```php
<?php

declare(strict_types=1);

use Johncms\Modules\Partners\Application\Controllers\PartnersController;
use Johncms\Router\RouteCollection;

return static function (RouteCollection $router): void {
    /*
     * /partners - Это адрес страницы, по которому будет доступен наш модуль.
     *
     * Вторым параметром передаётся класс контроллера. Так как контроллер
     * является вызываемым (реализует метод __invoke), название метода указывать не нужно.
     */
    $router->map(['GET', 'POST'], '/partners', PartnersController::class)->name('partners');
};
```
{% endcode %}

Теперь наш модуль доступен по адресу **ваш.сайт/partners/**

Теперь давайте сообщим модулю online, что у нас появился модуль партнёров и нужно в списке пользователей онлайн отображать тех, кто смотрит эту страницу.\
Для этого перейдём в папку **config** и создадим в ней файл **places.local.php**, если его ещё нет.

```php
<?php

return [
    '/partners' => '<a href="/partners/">Смотрит партнёров</a>',
];
```

Отлично, наш модуль теперь полностью работоспособен, вам останется только добавить на него ссылку в основном шаблоне или на любой другой странице на ваше усмотрение.
