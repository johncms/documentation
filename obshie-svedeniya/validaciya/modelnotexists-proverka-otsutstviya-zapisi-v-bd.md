# ModelNotExists - Проверка отсутствия записи в БД

Валидатор **ModelNotExists** позволяет проверить отсутствие записи в базе данных. Это подойдет для тех случаев, когда вам нужно проверить отсутствие записи в таблице прежде чем её добавить. Например, с помощью этого валидатора, в форме регистрации пользователя вы можете проверить существует ли пользователь с введенным логином или нет.

Для работы этого валидатора вам потребуется существующая [модель](https://johncms.com/documentation/eloquent-orm/). 

## Поддерживаемые параметры

* **model**: Класс модели, который будет использоваться для построения запроса к БД.
* **field**: Столбец в БД по которому будет осуществляться поиск записи.
* **exclude**: Параметры для задания условий исключения из выборки. Может содержать анонимную функцию или массив с полями **field** и **value**.

## Примеры использования

```php
// Массив полей и значений
$data = [
    'test' => 'admin@admin.ru',
];

// Настройки валидатора
$rules = [
    'test' => [
        'ModelNotExists' => [
            'model'   => \Johncms\Users\User::class,
            'field'   => 'mail',
            'exclude' => static function ($query) {
                return $query->where('name', '!=', 'admin')->where('id', '!=', 1);
            },
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

В примере выше выполняется проверка наличия в таблице users пользователя с полем mail, содержащим admin@admin.ru. При этом из выборки исключаются строки с name = admin и id = 1. Для расширения запроса на выборку используется анонимная функция. Она позволяет дополнять запрос любыми условиями.  
Валидатор выполнит следующий запрос:

```sql
select * from `users` where (`name` != 'admin' and `id` != 1) and `mail` = 'admin@admin.ru' limit 1
```

Рассмотрим более простой пример, где в параметр **exclude** передается массив:

```php
// Массив полей и значений
$data = [
    'test' => 'admin@admin.ru',
];

// Настройки валидатора
$rules = [
    'test' => [
        'ModelNotExists' => [
            'model'   => \Johncms\Users\User::class,
            'field'   => 'mail',
            'exclude' => [
                'field' => 'name',
                'value' => 'admin',
            ],
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

Если вам не требуется сложное условие для исключения записей из выборки, то вы можете использовать такой вариант задания исключений. При таких настройках валидатор выполнит следующий запрос:

```sql
select * from `users` where `name` != 'admin' and `mail` = 'admin@admin.ru' limit 1
```

Ну и давайте рассмотрим минимальный вариант использования, вообще без исключений.

```php
// Массив полей и значений
$data = [
    'test' => 'admin@admin.ru',
];

// Настройки валидатора
$rules = [
    'test' => [
        'ModelNotExists' => [
            'model'   => \Johncms\Users\User::class,
            'field'   => 'mail',
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

При таких настройках валидатор просто проверит наличие записи с mail = admin@admin.ru. Если запись будет найдена, то валидатор вернёт ошибку. Если нет, проверка пройдет успешно.

Запрос, который выполнит валидатор при этих настройках будет таким:

```sql
select * from `users` where `mail` = 'admin@admin.ru' limit 1
```

