# Проблемы и их решение

Иногда при переносе сайта на другой хостинг или после каких-то изменений в коде вы можете столкнуться с ошибками. Здесь мы рассмотрим распространенные проблемы и варианты их решений.

### Ошибка 500.

Причин появления этой ошибки много. Каждую причину нужно рассматривать индивидуально. Для начала чтобы понять от чего отталкиваться нужно включить вывод ошибок.

Для включения вывода ошибок откройте файл **config/constants.php**, найдите строки﻿

```php
// Включаем режим отладки
const DEBUG = false;
```

Замените false на true

```php
const DEBUG = true;
```

После этих действий на сайте должен отображаться текст ошибки.

Если этого не произошло, нужно смотреть журнал ошибок на сервере.

