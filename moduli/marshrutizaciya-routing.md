# Маршрутизация (роутинг)

### **Для чего нужен роутер в JohnCMS?**

Как и в других CMS и фреймворках роутер в JohnCMS обрабатывает запрошенный URL адрес и определяет какой модуль запустить для обработки этого запроса.\
В свою очередь модуль может получить от роутера различные параметры в зависимости от настроек маршрута и использовать для реализации своего функционала.

Перейдем к практической части.

### Где хранятся настройки маршрутизации?

Все описания маршрутов хранятся в модулях по следующему пути: **config/routes.php**

Пример файла **routes.php**

{% code title="config/routes.local.php" %}
```php
<?php

declare(strict_types=1);

use Johncms\Contacts\Controllers\ContactseController;
use Johncms\Router\RouteCollection;

return function (RouteCollection $router) {
    $router->get('/contacts/', [ContactseController::class, 'index'])->setName('contacts');
};
```
{% endcode %}

Давайте теперь рассмотрим детально как работать с роутером и как использовать его в своих модулях?

Возьмём простой пример из кода выше.\
Маршрут у нас в нем задается такой строкой:

<pre class="language-php"><code class="lang-php"><strong>$router->get('/contacts/', [ContactseController::class, 'index'])->setName('contacts');
</strong></code></pre>

Эта строка говорит роутеру следующее:\
Если запрос пришел методом GET и он поступил на страницу site.ru/contacts/, то необходимо вызвать метод **index** в контроллере **Johncms\Contacts\Controllers\ContactseController**\
Таким образом, когда пользователь переходит по адресу site.ru/contacts/ он видит те данные, которые вернул метод index.

Давайте рассмотрим более сложные примеры маршрутизации.\
Для этого давайте изменим наш простой модуль контактов, который мы создавали в предыдущей статье: [Создание модуля](sozdanie-modulya.md)

Откроем файл /config/routes.local.php

Изменим нашу строку маршрута следующим образом:

```php
$router->get('/contacts/{action?}', [ContactseController::class, 'index'])->setName('contacts');
```

Мы добавили в неё дополнительный параметр {action?}\
Что это значит?\
Вопросительный знак сообщает роутеру, что этот параметр не обязателен (он может быть, а может отсутствовать).\
В фигурных скобках задается название параметра, чтобы модуль смог с ним работать.\
По умолчанию выбираться будет та часть адреса, которая расположена между /contacts/ и следующим слэшем.

Чтобы было понятнее, давайте разберем на примерах.\
1\. **site.ru/contacts/** - В таком варианте у нас параметр action будет игнорироваться т.к. роутер считает его необязательным и откроет нашу страницу контактов.\
2\. **site.ru/contacts/moscow**  - Такой вариант откроет нашу страницу контактов и в модуле будет доступен параметр action. В этом параметре будет содержаться слово "moscow".\
3\. **site.ru/contacts/new\_york** - Тоже самое что и в варианте 3, только в параметре action будет "new\_york"\
4\. **site.ru/contacts/new\_york/test1/** - Выдаст ошибку 404 т.к. роутер видит, что маршрут не подходит нам (содержит больше данных чем нужно для нашего маршрута).

### Получение параметров маршрута в контроллере

Давайте теперь разберемся как в модуле нам получить параметры, которые мы указываем в роутере.\
Откроем файл контроллера **Controllers/ContactsController.php**\
В методе index вставим следующий код

{% code title="ContactsController.php" %}
```php
<?php

declare(strict_types=1);

namespace Johncms\Contacts\Controllers;

class ContactsController
{
    public function index(string $action = '')
    {
        // Получаем массив параметров, которые вернул нам роутер
        $route = di('route');
        // Выведем их на экран
        dd($route);
    }
}
```
{% endcode %}

Перейдем по адресу: **site.ru/contacts/moscow/**

В браузере у вас отобразится следующий текст:

```
Array
(
    [action] => moscow
)
```

Как мы видим, параметр, который мы назвали в настройках маршрута action, появился у нас в массиве и содержит слово moscow.

В модуле мы можем обратиться к этому параметру и в зависимости от его содержимого управлять логикой работы модуля.\
Получить этот параметр можно, как вы наверное уже догадались, следующим образом:\
&#x20;$route\['action']

Также в методе контроллера можно указывать одноименные переменные и в таком случае в этих переменных будет содержаться то, что вы указали в параметрах марщрута.

В нашем случае в переменной $action будет содержаться слово moscow.

{% hint style="info" %}
При автоматическом внедрении переменных в методы контроллеров происходит приведение типов к указанному в сигнатуре метода. Преобразование выполняется только для типов int, float, string, bool. \
Оригинальные типы можно получить из массива $route = di('route');
{% endhint %}

### Прочие примеры использования

Указание регулярного выражения для сопоставления маршрута

```php
$router->get('/contacts/{page<\d+>}', [ContactseController::class, 'index']);
```

В примере выше регулярное выражение ограничивает тип параметра page до числа. Если после строки /contacts/ будут буквы - маршрут не будет сопоставлен и отобразится 404 страница.

### Шаблоны популярных выражений

Для упрощения ограничений типов параметров в JohnCMS существуют заготовленные регулярные выражения которые можно указать следующим образом:

<pre class="language-php"><code class="lang-php">// Число. /contacts/1234
$router->get('/contacts/{page:number}', [ContactseController::class, 'index']);
<strong>
</strong><strong>// Буквы, цифры и некоторые знаки. /contacts/my-page_1-2-3
</strong>$router->get('/contacts/{page:slug}', [ContactseController::class, 'index']);

// Буквы латинского алфафита. /contacts/word
$router->get('/contacts/{page:word}', [ContactseController::class, 'index']);

// Путь. /contacts/path/level2/level3/level4/any-level
$router->get('/contacts/{page:path}', [ContactseController::class, 'index']);

// Опциональный параметр. /contacts/1 или /contacts
$router->get('/contacts/{page:number?}', [ContactseController::class, 'index']);
</code></pre>

### Группы маршрутов

Поддерживается возможность объединения маршрутов в группу

```php
$router->group('/admin/', function (RouteCollection $routeGroup) {
    $routeGroup->get('/login/', [AuthController::class, 'index'])->setName('login');
    $routeGroup->post('/login/authorize/', [AuthController::class, 'authorize'])->setName('authorize');
})->setNamePrefix('admin.');
```

В примере описанном выше будут обрабатываться следующие ссылки:\
/admin/login/\
/admin/login/authorize/

Имена маршрутов так же будут иметь префикс admin. (admin.login, admin.authorize)

### Генерация URL по имени маршрута

При объявлении маршрута можно указывать его имя с помощью метода ->setName('routeName').\
Далее это имя можно использовать для генерации ссылок:

```php
echo route('admin.login'); // /admin/login/
echo route('admin.authorize'); // /admin/login/authorize/
```

Можно так же генерировать маршруты с параметрами:

```php
$router->get('/contacts/{action?}', [ContactseController::class, 'index'])->setName('contacts');
echo route('contacts', ['action' => 'action-name']); // /contacts/action-name
```

### Установка приоритета

Бывают случаи когда один адрес может попадать под одно и то же регулярное выражение. Например если у вас есть маршрут с динамическим параметром, но для конкретной страницы которая попадает под этот маршрут вам нужно сделать отдельный контроллер.

```php
$router->get('/path/{action}', ['class', 'method'])->setPriority(20);
$router->get('/path/test-action', ['class2', 'method'])->setPriority(100);
```

В этом примере URL **/path/test-action** попадает под шаблон первого маршрута, но т.к. мы указали приоритет второго маршрута выше чем первого, второй маршрут отработает раньше.

### Добавление middleware

```php
// Для одного маршрута
$router->get('/admin/', [DashboardController::class, 'index'])
    ->setName('admin.dashboard')
    ->addMiddleware(AdminAuthorizedUserMiddleware::class)
    ->addMiddleware(HasAnyRoleMiddleware::class);

// Для группы
$router->group('/admin/', function (RouteCollection $routeGroup) {
    $routeGroup->get('/login/', [AuthController::class, 'index'])->setName('admin.login');
    $routeGroup->post('/login/authorize/', [AuthController::class, 'authorize'])->setName('admin.authorize');
})->addMiddleware(AdminUnauthorizedUserMiddleware::class);
```

Middleware будут выполняться в следующем порядке:

1. Глобальные для всех маршрутов (из конфигов)
2. Для группы
3. Для конкретных маршрутов

Мы рассмотрели наиболее частые варианты использования маршрутизации и надеемся дальше вы сможете самостоятельно строить ещё более сложные маршруты.
