# Конфигурационные файлы \(configs\)

Наверное Вы уже задавались вопросом "Где хранятся настройки JohnCMS и как добавлять свои настройки?". Давайте рассмотрим подробнее.

Ранее когда мы рассматривали [структуру папок](https://johncms.com/documentation/structure/), мы уже упоминали в ней папку [config](https://johncms.com/documentation/structure/#config). Теперь рассмотрим, что и за что отвечает...

Когда мы открываем папку config, то видим в ней примерно такую структуру:

![&#x421;&#x43F;&#x438;&#x441;&#x43E;&#x43A; &#x43A;&#x43E;&#x43D;&#x444;&#x438;&#x433;&#x443;&#x440;&#x430;&#x446;&#x438;&#x43E;&#x43D;&#x43D;&#x44B;&#x445; &#x444;&#x430;&#x439;&#x43B;&#x43E;&#x432; &#x432; JohnCMS](../.gitbook/assets/image%20%289%29.png)

Файлов достаточно много, давайте разберёмся за что они отвечают.

## Файлы в директории autoload:

Директория autoload содержит все конфигурационные файлы, которые автоматически загружаются системой.  
Как вы наверное заметили есть файлы содержащие в названии **global** и **local**.  
Файлы **global** это обычно файлы, которые могут обновляться при выходе новых версий JohnCMS. Не рекомендуем их редактировать, т.к. это осложнит обновление CMS.

Файлы **local** - это локальные файлы конкретно для вашего сайта. Они не содержаться в дистрибутиве JohnCMS. Некоторые из них создаются автоматически при установке системы, а некоторые вы можете создавать вручную.

### Как же быть если вы хотите изменить какие-то параметры, которые есть в global файле?

Всё очень просто. Нужно создать файл с таким же названием, но заменить global на local.

Например, вы хотите изменить настройки в файле **mail.global.php**, для этого скопируйте этот файл и сохраните под именем **mail.local.php**. Далее измените в нем нужные параметры и они переопределят те параметры, которые уже содержатся в **mail.global.php**.

{% hint style="info" %}
Обратите внимание. При необходимости Вы можете изменить только определенные параметры, а остальные останутся стандартными.
{% endhint %}

Давайте рассмотрим пример:

### Содержимое mail.global.php

```php
return [
    'mail' => [
        // Default transport (can be sendmail, smtp, file or memory)
        'transport' => 'sendmail',

        // Transport settings
        'options'   => [
            'smtp' => [
                'name'              => 'localhost.localdomain',
                'host'              => '127.0.0.1',
                'connection_class'  => 'plain',
                'connection_config' => [
                    'username' => 'user',
                    'password' => 'pass',
                ],
            ],
            'file' => [
                'path'     => DATA_PATH . 'mail/',
                'callback' => static function (FileTransport $transport) {
                    return 'Message_' . microtime(true) . '_' . mt_rand() . '.txt';
                },
            ],
        ],
    ],
];
```

Допустим нам нужно изменить имя пользователя: username. Это можно сделать так:

### Содержимое файла mail.local.php

```php
return [
    'mail' => [
        // Transport settings
        'options'   => [
            'smtp' => [
                'connection_config' => [
                    'username' => 'my_user',
                ],
            ],
        ],
    ],
];
```

Давайте теперь получим итоговый результат.

{% hint style="info" %}
Содержимое всех конфигурационных файлов можно получить следующим образом:  
**$config = di\('config'\);**  
Это вернет содержимое всех конфигурационных файлов из папки **config/autoload**.
{% endhint %}

Чтобы получить содержимое файла mail, выполним следующий код:

```php
d($config['mail']);
```

Это вернет следующий результат:

```php
Array
(
    [transport] => sendmail
    [options] => Array
        (
            [smtp] => Array
                (
                    [name] => localhost.localdomain
                    [host] => 127.0.0.1
                    [connection_class] => plain
                    [connection_config] => Array
                        (
                            [username] => my_user
                            [password] => pass
                        )
                )
            [file] => Array
                (
                    [path] => /Users/maksim/MyProjects/johncms_public/data/mail/
                    [callback] => Closure Object
                        (
                            [parameter] => Array
                                (
                                    [$transport] => 
                                )
                        )
                )
        )
)
```

Как видите, в итоговом результате username переопределился тем, что мы указали в файле **mail.local.php**

Вы можете самостоятельно поэкспериментировать, создать свой конфигурационный файл \(global/local\), а так же можете переопределить настройки из других файлов.

Для удобства можете создать файл **test.php** в корне вашего сайта со следующим содержимым:

```php
<?php

require 'system/bootstrap.php';
$config = di('config');

// Выведем содержимое конфига mail
d($config['mail']);
```

После этого в браузере перейдите по адресу site.com/test.php и увидите результат. \(site.com необходимо заменить на адрес вашего сайта\).

{% hint style="warning" %}
Обратите внимание.  
Хоть технически вы можете создавать конфигурационные файлы любой структуры и с любыми именами содержащими **local.php** или **global.php**, мы бы рекомендовали создавать осмысленные названия и первый элемент массива называть так же как и сам конфигурационный файл чтобы избежать путаницы и пересечения параметров.  
Например файл **my.global.php**, должен возвращать следующую структуру:  
**return \[**  
    **'my' =&gt; \[**  
        **'name' =&gt; 'value'**  
    **\],**  
**\];**
{% endhint %}

Autoload рассмотрели, теперь кратко рассмотрим остальные файлы.

## Прочие конфигурационные файлы:

constants.php - Файл содержит различные константы. В нем вам скорее всего понадобятся константы USE\_CRON \(для перевода отправки email на cron\) и DEBUG для включения режима отладки при возникновении ошибок или при разработке модулей.

notifications.global.php - Этот файл содержит шаблоны уведомлений. Параметры в данном файле можно переопределить или дополнить с помощью файла notifications.local.php

places.global.php - Файл содержит информацию о местоположении пользователей. Параметры в данном файле можно переопределить или дополнить с помощью файла places.local.php

routes.php - файл для настройки маршрутизации. Подробно работу с ним мы рассматривали в этой статье: [Маршрутизация \(роутинг\)](https://johncms.com/documentation/routing/)

