# Маршрутизация \(роутинг\)

## **Для чего нужен роутер в JohnCMS?**

Как и в других CMS и фреймворках роутер в JohnCMS обрабатывает запрошенный URL адрес и определяет какой модуль запустить для обработки этого запроса.  
В свою очередь модуль может получить от роутера различные параметры в зависимости от настроек маршрута и использовать для реализации своего функционала.

Перейдем к практической части.

## Где хранятся настройки маршрутизации?

Настройки для системных модулей JohnCMS хранятся в файле **/config/routes.php**

Так же система позволяет задавать маршруты для сторонних модулей.  
Для этого предназначен файл **/config/routes.local.php**

Почему для маршрутов сторонних модулей используется отдельный файл?  
Дело в том, что при обновлениях JohnCMS файл **/config/routes.php** может меняться и при очередном обновлении все Ваши изменения в нем, будут утеряны.  
Чтобы решить эту проблему, используется файл **/config/routes.local.php**

Пример файла **/config/routes.local.php**

{% code title="/config/routes.local.php" %}
```php
<?php

declare(strict_types=1);

/**
 * @var FastRoute\RouteCollector $map
 */

/*
 * /contacts/ - Это адрес страницы по которому будет доступен наш модуль
 * modules/contacts/index.php - Это путь к точке входа в наш модуль
 */

$map->addRoute(['GET', 'POST'], '/contacts/', 'modules/contacts/index.php');
```
{% endcode %}

Давайте теперь рассмотрим детально как работать с роутером и как использовать его в своих модулях?

Возьмём простой пример из примера выше.  
Маршрут у нас в нем задается такой строкой:

```php
$map->addRoute(['GET', 'POST'], '/contacts/', 'modules/contacts/index.php');
```

Эта строка говорит роутеру следующее:  
Если запрос пришел методом GET или POST и он поступил на страницу site.ru/contacts/, то необходимо выполнить файл modules/contacts/index.php  
Таким образом, когда пользователь переходит по адресу site.ru/contacts/ он видит результат выполнения файла modules/contacts/index.php

Давайте рассмотрим более сложные примеры маршрутизации.  
Для этого давайте изменим наш простой модуль контактов, который мы создавали в предыдущей статье [Создание модуля](https://johncms.com/documentation/create_module/)

Откроем файл /config/routes.local.php

Изменим нашу строку маршрута следующим образом:

```php
$map->addRoute(['GET', 'POST'], '/contacts/[{action}/]', 'modules/contacts/index.php');
```

Мы добавили в неё дополнительный параметр \[{action}/\]  
Что это значит?  
Квадратные скобки говорят роутеру, что этот параметр у нас не обязателен \(он может быть, а может и не быть\).  
В фигурных скобках задается название параметра, чтобы модуль смог с ним работать.  
Слэш мы ставим чтобы ограничить выбор т.е. выбираться будет та часть адреса, которая расположена между /contacts/ и следующим слешем.

Чтобы было понятнее, давайте разберем на примерах.  
1. **site.ru/contacts/** - В таком варианте у нас параметр action будет игнорироваться т.к. роутер считает его необязательным и откроет нашу страницу контактов.  
2. **site.ru/contacts/moscow** - В таком варианте роутер откроет страницу ошибки 404 т.к. URL у нас не заканчивается обратным слешем, а в настройках маршрута мы явно указали, что если после /contacts/ есть ещё что-то, то обрабатываем этот маршрут только если он заканчивается слешем \(/\).  
3. **site.ru/contacts/moscow/**  - Такой вариант откроет нашу страницу контактов и в модуле будет доступен параметр action. В этом параметре будет содержаться слово "moscow".  
4. **site.ru/contacts/new\_york/** - Тоже самое что и в варианте 3, только в параметре action будет "new\_york"  
5. **site.ru/contacts/new\_york/test1/** - Выдаст ошибку 404 т.к. роутер видит, что маршрут не подходит нам \(содержит больше данных чем нужно для нашего маршрута\).

Давайте теперь разберемся как в модуле нам получить параметры, которые мы указываем в роутере.  
Откроем файл **modules/contacts/index.php**  
В начале файла после строки **defined\('\_IN\_JOHNCMS'\) \|\| die\('Error: restricted access'\);** вставим следующий код:

{% code title="modules/contacts/index.php" %}
```php
// Получаем массив параметров, которые вернул нам роутер

$route = di('route');
// Выведем их на экран
d($route);

// Прекратим выполнение скрипта
exit;
```
{% endcode %}

Перейдем по адресу: **site.ru/contacts/moscow/**

В браузере у вас отобразится следующий текст:

```php
Array
(
    [action] => moscow
)
```

Как мы видим, параметр, который мы назвали в настройках маршрута action, появился у нас в массиве и содержит слово moscow.

В модуле мы можем обратиться к этому параметру и в зависимости от его содержимого управлять логикой работы модуля.  
Получить этот параметр можно, как вы наверное уже догадались, следующим образом:  
 $route\['action'\]

Давайте усложним маршрут.  
Откроем файл **/config/routes.local.php**  
Изменим нашу строку маршрута следующим образом:

```php
$map->addRoute(['GET', 'POST'], '/contacts/[{action}/[{id:\d+}/]]', 'modules/contacts/index.php');
```

В этом параметре мы добавили ещё один необязательный параметр и назвали его id. Через двоеточие мы указали регулярное выражение по которому будем вызывать этот маршрут. Указанное регулярное выражение принимает только цифры.  
Теперь у нас модуль контактов открывается по адресам:  
site.ru/contacts/  
site.ru/contacts/moscow/  
site.ru/contacts/moscow/123456/

Вместо слова moscow может быть любое слово, а вместо 123456 может быть любое число.  
Перейдем по адресу site.ru/contacts/moscow/123456/ и посмотрим что у нас выведется.

Вывелось следующее:

```php
Array
(
    [action] => moscow
    [id] => 123456
)
```

Как видим, пришел параметр action и id

Давайте добавим третий параметр и ещё усложним наш маршрут.  
Откроем файл /config/routes.local.php  
Изменим нашу строку маршрута следующим образом:

```php
$map->addRoute(['GET', 'POST'], '/contacts/[{action}/[{id:\d+}/[{street}/]]]', 'modules/contacts/index.php');
```

В этом маршруте мы добавили параметр street, он не ограничен только цифрами и может принимать любую строку.  
Рассмотрим примеры адресов, которые будут доступны для такого маршрута:  
**site.ru/contacts/**  
**site.ru/contacts/moscow/**  
**site.ru/contacts/moscow/123456/**  
**site.ru/contacts/moscow/123456/sadovaya/**

Перейдем по адресу:  
**site.ru/contacts/moscow/123456/sadovaya/**

Отобразилось следующее:

```php
Array
(
    [action] => moscow
    [id] => 123456
    [street] => sadovaya
)
```

Давайте переименуем параметр action в city чтобы на различных примерах посмотреть на что влияет это название

```php
$map->addRoute(['GET', 'POST'], '/contacts/[{city}/[{id:\d+}/[{street}/]]]', 'modules/contacts/index.php');
```

Перейдем по тому же адресу:  
site.ru/contacts/moscow/123456/sadovaya/

Получим результат:

```php
Array
(
    [city] => moscow
    [id] => 123456
    [street] => sadovaya
)
```

Как видим в результате тоже поменялось название параметра.

Мы рассмотрели наиболее частые варианты использования маршрутизации и надеемся дальше вы сможете самостоятельно строить ещё более сложные маршруты.  
С другими примерами маршрутов, вы так же можете ознакомиться в документации к библиотеке [https://github.com/nikic/FastRoute](https://github.com/nikic/FastRoute) , которая используется в JohnCMS для работы с маршрутами.
