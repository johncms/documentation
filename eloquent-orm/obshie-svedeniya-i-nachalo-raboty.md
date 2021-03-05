# Общие сведения и начало работы

## Введение

Система объектно-реляционного отображения \(ORM\) Eloquent — простая реализация шаблона ActiveRecord для работы с базами данных. Каждая таблица имеет соответствующий класс-модель, который используется для работы с этой таблицей. Модели позволяют запрашивать данные из таблиц, а также вставлять, обновлять и удалять в них записи.

## Определение моделей

Для начала создадим модель Eloquent. Все модели Eloquent наследуют класс **Illuminate\Database\Eloquent\Model**.  
Допустим мы делаем модуль блогов. Создадим базовую структуру модуля как [описано здесь](https://johncms.com/documentation/create_module/). Модуль назовём **blog.**  
Теперь давайте создадим в нем папку **lib** в которой будем хранить классы нашего модуля и в этой папке создадим подпапку **models** в которой уже будем размещать наши модели.  
В итоге должен получиться такой путь: **lib/models**  
Теперь настроим автозагрузку классов из папки lib. Для этого в файле index.php нашего модуля поместим следующий код:

```php
$loader = new Aura\Autoload\Loader();
$loader->register();
$loader->addPrefix('Blog', __DIR__ . '/lib');
```

С помощью метода **addPrefix** первым параметром мы указываем пространство имен \(namespace\) **Blog** и указываем папку в которой располагаются классы для этого пространства имен.  
Создайте таблицу posts с примерно таким набором полей:

* id
* user\_id
* name
* text
* created\_at
* updated\_at

Пример запроса на создание таблицы:

```sql
CREATE TABLE `posts`
(
    `id`         INT          NOT NULL AUTO_INCREMENT,
    `user_id`    INT          NOT NULL,
    `name`       VARCHAR(255) NOT NULL,
    `text`       LONGTEXT     NULL DEFAULT NULL,
    `created_at` TIMESTAMP    NULL DEFAULT NULL,
    `updated_at` TIMESTAMP    NULL DEFAULT NULL,
    PRIMARY KEY (`id`),
    INDEX `user_id` (`user_id`)
) ENGINE = InnoDB;
```

 Теперь создадим модель.  
 В папке **lib/models** создайте файл **Post.php** со следующим содержимым

{% code title="lib/models/Post.php" %}
```php
<?php

namespace Blog\Models;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

/**
 * @mixin Builder
 */
class Post extends Model
{

}
```
{% endcode %}

На этом модель готова и она уже работоспособна.  

### Имена таблиц

Заметьте, что мы не указали, какую таблицу Eloquent должен привязать к нашей модели. Если это имя не указано явно, то в соответствии с принятым соглашением будет использовано имя класса в нижнем регистре \(snake case\) и во множественном числе. В нашем случае Eloquent предположит, что модель Post хранит свои данные в таблице posts. Вы можете указать произвольную таблицу, определив свойство table в классе модели:

```php
<?php

namespace Blog\Models;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

/**
 * @mixin Builder
 */
class Post extends Model
{
    /**
     * Таблица, связанная с моделью.
     *
     * @var string
     */
    protected $table = 'my_table';

}
```

### Первичные ключи

Eloquent также предполагает, что каждая таблица имеет первичный ключ с именем id. Вы можете определить свойство $primaryKey для указания другого имени.  
Вдобавок, Eloquent предполагает, что первичный ключ является инкрементным числом, и автоматически приведёт его к типу int. Если вы хотите использовать неинкрементный или нечисловой первичный ключ, задайте открытому свойству $incrementing вашей модели значение false.

### Отметки времени

По умолчанию Eloquent ожидает наличия в ваших таблицах столбцов `created_at` и `updated_at`. Если вы не хотите, чтобы они автоматически обрабатывались в Eloquent, установите свойство `$timestamps` класса модели в `false`:

```php
<?php

namespace Blog\Models;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

/**
 * @mixin Builder
 */
class Post extends Model
{
    /**
     * Определяет необходимость отметок времени для модели.
     *
     * @var bool
     */
    public $timestamps = false;

}
```

Если вы хотите изменить формат отметок времени, задайте свойство `$dateFormat` вашей модели. Это свойство определяет, как атрибуты времени будут храниться в базе данных, а также задаёт их формат при сериализации модели в массив или JSON:

```php
<?php

namespace Blog\Models;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

/**
 * @mixin Builder
 */
class Post extends Model
{
    /**
     * Формат хранения отметок времени модели.
     *
     * @var string
     */
    protected $dateFormat = 'U';

}
```

Если вам надо изменить имена столбцов для хранения отметок времени, вы можете задать константы `CREATED_AT` и `UPDATED_AT`:

```php
<?php

namespace Blog\Models;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

/**
 * @mixin Builder
 */
class Post extends Model
{
    const CREATED_AT = 'creation_date';
    const UPDATED_AT = 'last_update';
}
```

### Получение моделей

После создания модели и связанной с ней таблицы, вы можете начать получать данные из вашей БД. Каждая модель Eloquent представляет собой мощный конструктор запросов, позволяющий удобно выполнять запросы к связанной таблице. Например:

```php
$post = new \Blog\Models\Post();
$all_posts = $post->all();

foreach ($all_posts as $post) {
    echo $post->name . '<br>';
}
```

Этот код выведет все записи из таблицы posts.

### Добавление дополнительных ограничений

Метод all в Eloquent возвращает все результаты из таблицы модели. Поскольку модели Eloquent работают как конструктор запросов, вы можете также добавить ограничения в запрос, а затем использовать метод get для получения результатов:

```php
$post = new \Blog\Models\Post();
$all_posts = $post->where('user_id', '=', 1)
    ->orderBy('name', 'desc')
    ->get();

foreach ($all_posts as $post) {
    echo $post->name . '<br>';
}
```

{% hint style="info" %}
Все методы, доступные в конструкторе запросов, также доступны при работе с моделями Eloquent. Вы можете использовать любой из них в запросах Eloquent.
{% endhint %}

