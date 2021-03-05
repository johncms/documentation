# Captcha - Проверка защитного кода

Валидатор Captcha предназначен для проверки защитного кода, который указал пользователь в форме.  
Перед проверкой, код должен быть сгенерирован и записан в сессию.

### Поддерживаемые параметры

* **sessionField**: Указывается ключ в сессии из которого валидатор будет использовать код. По умолчанию: **code**

### Примеры использования

```php
// Массив полей и значений
$data = [
    'test' => 'captcha_code',
];

// Настройки валидатора
$rules = [
    'test' => [
        'Captcha',
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

В указанном выше примере код код будет использоваться из переменной по умолчанию **$\_SESSION\['code'\]**

```php
// Массив полей и значений
$data = [
    'test' => 'captcha_code',
];

// Настройки валидатора
$rules = [
    'test' => [
        'Captcha' => [
            'sessionField' => 'captcha_code'
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

А в этом примере будет использован код из переменной **$\_SESSION\['captcha\_code'\]**

Более подробно работу с формами и с капчей рассмотрим в отдельной статье.

