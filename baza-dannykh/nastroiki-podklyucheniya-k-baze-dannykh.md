# Настройки подключения к базе данных

JohnCMS как и многие другие системы для работы использует базу данных.  
Когда вы устанавливаете систему, создается файл **config/autoload/database.local.php** в этом файле хранятся настройки подключения к базе данных.

На данный момент по умолчанию он выглядит так:

```php
array (
    'db_host' => 'localhost',
    'db_name' => 'johncms',
    'db_user' => 'database_user',
    'db_pass' => 'password',
  ),
);
```

Это минимально необходимый список параметров для работы системы. В некоторых случаях может понадобиться задать дополнительные параметры, такие как порт и драйвер.  
На данный момент максимально полный файл конфигурации подключения выглядит так:

```php
array (
    'db_driver' => 'mysql',
    'db_host' => 'localhost',
    'db_name' => 'johncms',
    'db_user' => 'database_user',
    'db_pass' => 'password',
    'db_port' => '3306',
  ),
);
```

В параметре db\_port указывается порт, который используется для подключения к БД.  
В параметре **db\_driver** указывается драйвер для работы с базой данных.

{% hint style="info" %}
В настоящее время полностью поддерживается работа с **MySQL 5.6.4** и выше.
{% endhint %}
