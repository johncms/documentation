# ModelExists - Проверка существования записи в БД

Валидатор **ModelExists** позволяет проверить существование записи в базе данных. Это хорошо подходит для тех случаев, когда у вас в форме есть привязка к каким-то существующим записям в базе данных.

Для работы этого валидатора вам потребуется существующая [модель](https://johncms.com/documentation/eloquent-orm/). 

### Поддерживаемые параметры

* **model**: Класс модели, который будет использоваться для построения запроса к БД.
* **field**: Столбец в БД по которому будет осуществляться поиск записи.

### Примеры использования

```php
// Массив полей и значений
$data = [
    'test' => 45,
];

// Настройки валидатора
$rules = [
    'test' => [
        'ModelExists'   => [
            'model' => \Johncms\Users\User::class,
            'field' => 'id',
        ],
    ],
];

// Валидация
$validator = new \Johncms\Validator\Validator($data, $rules);
if ($validator->isValid()) {
    echo 'OK';
} else {
    d($validator->getErrors());
}
```

В указанном примере будет выполнена проверка наличия пользователя c идентификатором 45 в таблице users.

Запрос который будет выполнен:

```sql
SELECT * FROM `users` WHERE `id` = 45
```

В результате, если будет найдена запись с id = 45, то валидатор будет считать проверку успешной, если не найдет, то вернёт ошибку.

Рассмотрим ещё один пример:

```php
// Массив полей и значений
$data = [
    'test' => 'admin',
];

// Настройки валидатора
$rules = [
    'test' => [
        'ModelExists'   => [
            'model' => \Johncms\Users\User::class,
            'field' => 'name',
        ],
    ],
];

// Валидация
$validator = new \Johncms\Validator\Validator($data, $rules);
if ($validator->isValid()) {
    echo 'OK';
} else {
    d($validator->getErrors());
}
```

В этом примере будет выполнен поиск записи у которой поле name = admin.

Будет выполнен следующий запрос:

```sql
SELECT * FROM `users` WHERE `name` = 'admin'
```

Результат будет такой же как и в случае с id. Если будет найдена строка с полем name = admin, то валидация пройдет успешно, если нет, будет возвращена ошибка.

