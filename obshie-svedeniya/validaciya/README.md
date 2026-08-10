---
metaLinks:
  alternates:
    - https://app.gitbook.com/s/5NJeWEeBonlrBhBrVHEz/obshie-svedeniya/validaciya
---

# Валидация

## Зачем нужен валидатор

Почти в каждой форме есть поля, обязательные для заполнения, значения, которые нужно проверить по базе данных, адреса электронной почты, длину текста и списки допустимых значений. Валидатор избавляет от рутины: вы описываете правила, а он проверяет данные и возвращает сообщения о том, что именно не так.

Внутри работает [symfony/validator](https://symfony.com/doc/current/validation.html), но в коде модуля вы его не видите: правила — это объекты JohnCMS, а движок скрыт за интерфейсом. Это позволяет заменить движок, не переписывая формы.

{% hint style="warning" %}
В версиях до 10.0 валидатор создавался через `new \Johncms\Validator\Validator($data, $rules)`, а правила задавались строками. Этот класс удалён. Как перенести старый код — в таблице соответствий в конце страницы.
{% endhint %}

## Быстрый старт

Валидатор внедряется через интерфейс `Johncms\Validator\ValidatorInterface`, создавать его вручную не нужно.

```php
<?php

declare(strict_types=1);

namespace Johncms\Modules\MyModule\Application\Controllers;

use Johncms\Http\Request;
use Johncms\Http\View\ViewResponse;
use Johncms\Validator\Rules\EmailAddress;
use Johncms\Validator\Rules\StringLength;
use Johncms\Validator\ValidatorInterface;

final readonly class FeedbackController
{
    public function __construct(private ValidatorInterface $validator)
    {
    }

    public function __invoke(Request $request): ViewResponse
    {
        $formData = [
            'name'    => $request->body('name', ''),
            'email'   => $request->body('email', ''),
            'message' => $request->body('message', ''),
        ];

        $result = $this->validator->validate($formData, [
            'name'    => [new StringLength(min: 2, max: 50)],
            'email'   => [new EmailAddress()],
            'message' => [new StringLength(min: 10, max: 5000)],
        ]);

        if ($result->isValid()) {
            // Сохраняем данные
        }

        return new ViewResponse('@my-module/public/form.twig', [
            'errors' => $result->getErrors(),
        ]);
    }
}
```

Первый аргумент `validate()` — массив данных, обычно собранный из запроса. Второй — массив правил: ключ совпадает с именем поля, значение — список правил, которые к этому полю применяются.

Данные могут содержать больше полей, чем описано правил: лишние валидатор не трогает. Поле, для которого правила описаны, но которого нет в данных, проверяется как пустое значение.

## Результат проверки

`validate()` возвращает объект `Johncms\Validator\ValidationResult`. Он неизменяемый: методы, меняющие состав ошибок, возвращают новый объект.

| Метод                          | Что делает                                                                     |
|--------------------------------|--------------------------------------------------------------------------------|
| `isValid()`                    | `true`, если ошибок нет                                                        |
| `getErrors()`                  | массив `имя поля => список сообщений`                                          |
| `hasError('email')`            | есть ли ошибки у поля                                                          |
| `getFirstError('email')`       | первое сообщение поля или `null`                                               |
| `withError('field', 'текст')`  | новый результат с добавленной ошибкой                                          |
| `merge($otherResult)`          | объединяет два результата                                                      |
| `throwIfInvalid($factory)`     | бросает исключение, если результат невалиден                                    |

Массив из `getErrors()` передаётся в шаблон как есть — компонент `@theme/components/field-errors.twig` и остальная вёрстка ожидают именно такую форму:

```twig
<input name="email" class="form-control {{ errors.email is defined ? 'is-invalid' }}">
{% include '@theme/components/field-errors.twig' with {errors: errors.email|default([])} only %}
```

### Ошибки, которые находятся после валидации

Иногда причина отказа известна только после проверки правил — например, раздел нельзя сделать родителем самого себя. Такую ошибку добавляют в результат, а не разворачивают его в массив:

```php
$result = $this->validator->validate($fields, ['name' => [new StringLength(min: 2, max: 150)]]);

if ($this->wouldCreateCycle($section, $fields['parent'])) {
    $result = $result->withError('parent', __('Выберите другой родительский раздел'));
}
```

### Исключение вместо проверки

Если ошибки формы обрабатывает не контроллер, а вызывающий код, удобнее сразу бросить доменное исключение:

```php
$this->validator->validate($formData, $rules)->throwIfInvalid(
    static fn (array $errors): EditProfileException => new EditProfileException($errors)
);
```

## Обязательные и необязательные поля

**Поле, у которого есть правила, обязательно, если явно не указано обратное.**

```php
$rules = [
    // Обязательное: пустое значение не пройдёт проверку
    'name'          => [new StringLength(min: 2, max: 50)],

    // Необязательное: пустым можно оставить, но если заполнено — не длиннее 250 символов
    'meta_keywords' => [new StringLength(max: 250, allowEmpty: true)],
];
```

Пустыми считаются `null`, пустая строка и строка из одних пробелов, пустой массив и `false`. Ноль в любом виде (`0`, `0.0`, `'0'`) пустым **не** считается — это значение.

Параметр `allowEmpty` есть у всех правил, для которых он имеет смысл. Указывайте его осознанно: это единственное место, где записана разница между полем, которое посетитель может не заполнять, и полем, которое заполнить обязан.

## Ошибки формы целиком

Некоторые правила проверяют не значение поля, а обстоятельства отправки: не флудит ли посетитель, нет ли у него бана. Такие правила помещают под зарезервированный ключ `ValidationResult::FORM_KEY` (`_form`):

```php
use Johncms\Validator\Rules\Ban;
use Johncms\Validator\Rules\Flood;
use Johncms\Validator\ValidationResult;

$rules = [
    'message'                  => [new StringLength(min: 4)],
    ValidationResult::FORM_KEY => [new Flood(), new Ban(bans: [1, 13])],
];
```

В шаблоне такие сообщения выводятся над формой:

```twig
{% include '@theme/components/alert.twig' with {alert_type: 'alert-danger', alert: errors._form|default([])} only %}
```

## Порядок правил

Правила поля выполняются по очереди и останавливаются на первой ошибке, поэтому под полем всегда одно сообщение, а не по одному на каждое нарушенное правило. Ставьте правила от простого к сложному: сначала длина, потом обращение к базе данных.

## Свои сообщения

У каждого правила есть параметр `message` — он заменяет стандартный текст:

```php
new Identical(token: '1', message: __('Необходимо принять соглашение'))
```

Сообщение принадлежит одному правилу и не влияет на остальные поля формы.

## Что дальше

* [Правила валидации](rules.md) — справочник по всем встроенным правилам.
* [Свои правила](custom-rules.md) — как добавить правило в своём модуле.
* [Защита от CSRF](../zashita-ot-csrf.md) — проверка подлинности запроса, которая больше не является правилом валидации.

## Что изменилось в 10.0

| Было                                                        | Стало                                                      |
|-------------------------------------------------------------|------------------------------------------------------------|
| `new \Johncms\Validator\Validator($data, $rules)`           | внедрение `ValidatorInterface`, метод `validate()`         |
| `'name' => ['NotEmpty', 'StringLength' => ['min' => 2]]`    | `'name' => [new StringLength(min: 2)]`                     |
| `$validator->isValid()` / `getErrors()`                     | `$result->isValid()` / `$result->getErrors()`              |
| третий аргумент конструктора — сообщения для всей формы     | параметр `message` у конкретного правила                   |
| правило `Csrf`                                              | проверку выполняет middleware, правила больше нет           |
| `Flood` и `Ban` на поле `csrf_token`                        | ключ `ValidationResult::FORM_KEY`                          |
| обязательность поля зависела от правила                     | правило требует значение, пока не указан `allowEmpty: true` |

Правила прежнего движка, которые нигде в JohnCMS не используются (файловые, `Date`, `Ip`, `Uri`, `Regex`, `Hostname` и другие), не переносились. Если такое правило нужно вашему модулю — опишите его как [своё правило](custom-rules.md).
