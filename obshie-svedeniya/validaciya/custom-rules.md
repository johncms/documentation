# Свои правила валидации

Модуль может добавить собственное правило, не изменяя ядро. Способ зависит от того, что именно вы проверяете.

## Обёртка над готовым правилом Symfony

Если проверка уже реализована в `symfony/validator` (например `Regex`, `Ip`, `Url`), напишите два класса: объект-правило и фабрику, которая превращает его в констрейнт.

**Объект-правило** описывает, что настраивает разработчик формы. Он не знает о движке валидации:

```php
<?php

declare(strict_types=1);

namespace Johncms\Modules\MyModule\Application\Validation;

use Johncms\Validator\Rules\RequiresValueInterface;

final readonly class PhoneNumber implements RequiresValueInterface
{
    public function __construct(
        private bool $allowEmpty = false,
        private ?string $message = null,
    ) {
    }

    public function allowEmpty(): bool
    {
        return $this->allowEmpty;
    }

    public function message(): ?string
    {
        return $this->message;
    }
}
```

Реализуйте `RequiresValueInterface`, если пустое значение должно считаться ошибкой (перед правилом будет добавлена проверка на заполненность). Если правило допускает пустое значение всегда — реализуйте `RuleInterface` с одним методом `message()`.

**Фабрика** собирает констрейнт:

```php
<?php

declare(strict_types=1);

namespace Johncms\Modules\MyModule\Application\Validation;

use Johncms\Validator\RuleConstraintFactoryInterface;
use Johncms\Validator\Rules\RuleInterface;
use Symfony\Component\Validator\Constraint;
use Symfony\Component\Validator\Constraints\Regex;

final readonly class PhoneNumberRuleFactory implements RuleConstraintFactoryInterface
{
    public static function ruleClass(): string
    {
        return PhoneNumber::class;
    }

    public function create(RuleInterface $rule): Constraint
    {
        return new Regex(
            pattern: '/^\+?\d{10,15}$/',
            message: d__('my-module', 'Укажите номер телефона в международном формате'),
        );
    }
}
```

Фабрика может вернуть и список констрейнтов, если одно правило проверяет несколько условий.

Регистрация не нужна: фабрики находятся по интерфейсу автоматически. Достаточно, чтобы классы лежали в `src/` модуля, который подключён к контейнеру.

## Правило с собственной проверкой

Когда проверка своя — обращение к базе, сессии, сведениям о посетителе, — обёртка ничего не даёт. В этом случае правило само является констрейнтом:

```php
<?php

declare(strict_types=1);

namespace Johncms\Modules\MyModule\Application\Validation;

use Johncms\Validator\Rules\RequiresValueInterface;
use Symfony\Component\Validator\Constraint;

final class PromoCode extends Constraint implements RequiresValueInterface
{
    public string $message;

    private readonly ?string $ruleMessage;

    public function __construct(
        private readonly bool $allowEmpty = false,
        ?string $message = null,
    ) {
        parent::__construct([]);

        $this->ruleMessage = $message;
        $this->message = $message ?? d__('my-module', 'Промокод недействителен');
    }

    public function allowEmpty(): bool
    {
        return $this->allowEmpty;
    }

    public function message(): ?string
    {
        return $this->ruleMessage;
    }

    public function validatedBy(): string
    {
        return PromoCodeValidator::class;
    }
}
```

Сама проверка живёт в классе-валидаторе. Зависимости он получает через конструктор — это обычный сервис контейнера:

```php
<?php

declare(strict_types=1);

namespace Johncms\Modules\MyModule\Application\Validation;

use Symfony\Component\Validator\Constraint;
use Symfony\Component\Validator\ConstraintValidator;
use Symfony\Component\Validator\Exception\UnexpectedValueException;

final class PromoCodeValidator extends ConstraintValidator
{
    public function __construct(private readonly PromoCodeRepositoryInterface $promoCodes)
    {
    }

    public function validate(mixed $value, Constraint $constraint): void
    {
        if (! $constraint instanceof PromoCode) {
            throw new UnexpectedValueException($constraint, PromoCode::class);
        }

        // Пустое значение — забота проверки на заполненность, которая стоит перед правилом
        if ($value === null || $value === '') {
            return;
        }

        if ($this->promoCodes->isActive((string) $value)) {
            return;
        }

        $this->context->buildViolation($constraint->message)->addViolation();
    }
}
```

{% hint style="danger" %}
Не обращайтесь к контейнеру внутри валидатора через `di(...)`. Зависимости передаются конструктором: так правило можно протестировать, не поднимая приложение целиком.
{% endhint %}

### Значения в тексте сообщения

Если сообщение содержит подстановку, заполните её при создании нарушения:

```php
$this->context->buildViolation($constraint->message)
    ->setParameter('%value%', (string) $remainingSeconds)
    ->addViolation();
```

## Переводы сообщений

Текст сообщения должен попасть в файлы переводов, а туда его собирает сканер `composer translate-scan`. Он видит только литеральные вызовы функций перевода, поэтому:

```php
// Правильно: строка попадёт в .pot
$this->message = $message ?? d__('my-module', 'Промокод недействителен');

// Неправильно: значение по умолчанию у свойства сканер не увидит,
// и сообщение останется без перевода
public string $message = 'Промокод недействителен';
```

Плейсхолдеры пишутся в формате `%имя%`. После добавления сообщения выполните `composer translate-scan`, затем `composer translate` — подробности в разделе [«Многоязычность»](../../multiyazychnost/README.md).

## Доступ к другим полям формы

Валидатор проверяет весь массив данных сразу, поэтому в своём валидаторе можно получить и остальные поля:

```php
$allFields = $this->context->getRoot();
```

Это пригодится для правил, сравнивающих поля между собой, — например «пароль и его подтверждение совпадают».
