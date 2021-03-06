# StringLength - длина строки

Валидатор StringLength позволяет проверить находится ли длина строки в диапазоне заданных значений или нет.

По умолчанию этот валидатор проверяет, находится ли значение между min и max, используя минимальное значение по умолчанию, равное 0, и максимальное значение по умолчанию, равное NULL \(то есть неограниченное\).  
Таким образом, без каких-либо опций, валидатор только проверяет, что ввод является строкой.

## Поддерживаемые параметры

* **encoding**: Устанавливает кодировку ICONV в которой будет проверяться строка.
* **min**: Устанавливает минимально допустимую длину строки.
* **max**: Устанавливает максимально допустимую длину для строки.

## Примеры использования

```php
// Массив полей и значений
$data = [
    'test' => 'Строка',
];

// Настройки валидатора
$rules = [
    'test' => [
        'StringLength' => [
            'min'      => 3,
            'max'      => 60,
            'encoding' => 'UTF-8',
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

В указанном примере валидатор проверит строку в кодировке UTF-8 на длину от 3 до 60 символов.

### Проверка только минимальной длины:

```php
// Массив полей и значений
$data = [
    'test' => 'Строка',
];

// Настройки валидатора
$rules = [
    'test' => [
        'StringLength' => [
            'min' => 3
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

В этом примере будет проверяться только минимальная длина строки \(3 символа\). Максимальная будет считаться не ограниченной.

### Проверка только максимальной длины:

```php
// Массив полей и значений
$data = [
    'test' => 'Строка',
];

// Настройки валидатора
$rules = [
    'test' => [
        'StringLength' => [
            'max' => 50
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

В этом же примере будет проверяться только максимальная длина строки. Если строка будет длиннее 50 символов, проверка не пройдет, если менее 50 символов, то проверка пройдет.

### Строгое ограничение длины строки:

```php
// Массив полей и значений
$data = [
    'test' => 'Строка',
];

// Настройки валидатора
$rules = [
    'test' => [
        'StringLength' => [
            'max' => 6,
            'min' => 6,
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

А в этом примере мы строго ограничили длину строки 6 символами. Т.е. проверка пройдет только если строка будет длиной в 6 символов. Больше или меньше не допускается.

