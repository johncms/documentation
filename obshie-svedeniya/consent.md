---
description: Как выводить согласия (Consent) в своих формах и записывать их принятие в лог
---

# Согласия (Consent)

Модуль `consent` — это общая подсистема согласий: чекбоксов вида «Я принимаю правила сайта», которые нужно показать в форме, проверить при отправке и зафиксировать факт принятия.

Тексты согласий создаёт администратор в админке (**Согласия**), а модули не хранят их у себя и не знают, сколько согласий настроено. Модуль только объявляет **контекст** — имя своей формы — и спрашивает у сервиса, что нужно показать.

Из коробки согласия подключены к форме регистрации (контекст `register`) и форме обратной связи (контекст `contacts`).

## Как это работает

1. В админке создаётся согласие: контекст, язык, заголовок (с ссылками при необходимости), текст, версия, флаги «обязательное» и «активное».
2. Контроллер формы запрашивает у `ConsentService` список согласий для своего контекста.
3. Шаблон формы выводит для каждого согласия чекбокс с именем `consent_{id}`.
4. При отправке формы обязательные согласия проверяются валидатором.
5. После успешного сохранения данных факт принятия каждого отмеченного согласия пишется в лог (`consent_log`) вместе с версией, ID пользователя и IP.

Согласия выбираются по языку текущего интерфейса. Если для него согласий нет — берутся согласия языка сайта по умолчанию.

## ConsentService

Единственная точка входа для других модулей — `Johncms\Modules\Consent\Application\Services\ConsentService`. Внедряется через конструктор.

| Метод | Назначение |
| --- | --- |
| `getFormConsents(string $context): list<FormConsentDTO>` | Активные согласия контекста, подготовленные для вывода в форме |
| `getActiveConsents(string $context): Collection<Consent>` | То же, но моделями — если нужен полный текст согласия |
| `getConsent(int $id): ?Consent` | Согласие по ID |
| `logAcceptance(int $consentId, ?int $userId, string $ipAddress, ?string $version = null): ConsentLog` | Запись принятия в лог |

### FormConsentDTO

```php
final readonly class FormConsentDTO
{
    public int $id;
    public string $titleHtml; // очищенный HTML, выводится как есть
    public ?string $url;      // ссылка на страницу текста согласия или null
    public bool $isRequired;
    public string $version;
}
```

`titleHtml` уже пропущен через HTMLPurifier с разрешёнными инлайн-тегами (`a`, `b`, `strong`, `i`, `em`, `u`, `br`), поэтому в шаблоне печатается без `$this->e()`.

`url` заполняется только тогда, когда у согласия есть текст и в заголовке нет собственных ссылок. Если администратор указал в заголовке свои ссылки, оборачивать заголовок ещё одной ссылкой нельзя.

## Подключение к своей форме

### 1. Выберите контекст

Контекст — произвольная строка, идентификатор вашей формы, например `feedback` или `my_module_order`. В админке поле контекста — свободный ввод со списком подсказок, поэтому свой код указывается вручную и работает без изменений в модуле `consent`.

Удобно хранить контекст константой контроллера:

```php
private const CONSENT_CONTEXT = 'my_module_order';
```

### 2. Получите согласия в контроллере

```php
use Johncms\Http\Environment;
use Johncms\Http\Request;
use Johncms\Modules\Consent\Application\Services\ConsentService;
use Johncms\Users\User;
use Johncms\Validator\Validator;
use Laminas\Validator\Identical;

final class OrderController
{
    private const CONSENT_CONTEXT = 'my_module_order';

    public function __construct(
        private ConsentService $consentService,
        private Environment $env,
        private User $user,
        // ...
    ) {
    }

    public function __invoke(Request $request): string
    {
        $consents = $this->consentService->getFormConsents(self::CONSENT_CONTEXT);

        $fields = [
            'comment' => $request->body('comment'),
        ];

        // Значения чекбоксов попадают в общий массив полей формы
        foreach ($consents as $consent) {
            $fields['consent_' . $consent->id] = $request->body('consent_' . $consent->id);
        }

        $errors = [];
        // ...
    }
}
```

### 3. Проверьте обязательные согласия

Обязательное согласие считается принятым, только если чекбокс отмечен, то есть пришло значение `1`. Проверяется валидатором `Identical`:

```php
$rules = [
    'comment' => ['NotEmpty' => []],
];

foreach ($consents as $consent) {
    if ($consent->isRequired) {
        $rules['consent_' . $consent->id] = ['Identical' => ['token' => '1']];
    }
}

$consentMessage = __('You must accept the consent to continue');
$messages = [
    'Identical' => [
        Identical::NOT_SAME      => $consentMessage,
        Identical::MISSING_TOKEN => $consentMessage,
    ],
];

$validator = new Validator($fields, $rules, $messages);
```

Необязательные согласия не валидируются: пользователь может их не отмечать.

### 4. Запишите принятие в лог

Лог заполняется **после** успешного сохранения основных данных формы — записывать нужно только реально отмеченные согласия. Версия берётся из DTO, чтобы в логе остался снимок той версии, которую пользователь видел в момент отправки.

```php
if ($validator->isValid()) {
    $order = $this->createOrder->execute($dto);

    $ip = (string) $this->env->getIp(false);
    foreach ($consents as $consent) {
        if ($fields['consent_' . $consent->id] === '1') {
            $this->consentService->logAcceptance(
                $consent->id,
                $this->user->isValid() ? $this->user->id : null,
                $ip,
                $consent->version
            );
        }
    }
}
```

Для гостевых форм вместо ID пользователя передаётся `null`.

### 5. Выведите чекбоксы в шаблоне

Готовый шаблон чекбокса `system::app/consent-checkbox` уже умеет выводить заголовок, ссылку на текст согласия, звёздочку обязательности и ошибку валидации. Свою разметку писать не нужно:

```php
<?php
/**
 * @var list<Johncms\Modules\Consent\Application\DTO\FormConsentDTO> $consents
 * @var array<string, mixed> $fields
 * @var array<string, array<int, string>> $errors
 */
?>

<?php foreach ($consents as $consent): ?>
    <?php $consentField = 'consent_' . $consent->id ?>
    <?= $this->fetch('system::app/consent-checkbox', [
        'consent' => $consent,
        'field'   => $consentField,
        'checked' => ($fields[$consentField] ?? null) === '1',
        'errors'  => $errors[$consentField] ?? [],
    ]) ?>
<?php endforeach ?>
```

Не забудьте передать `consents` в шаблон из контроллера:

```php
return $this->render->render('my-module::order', [
    'consents' => $consents,
    'fields'   => $fields,
    'errors'   => $errors,
]);
```

## Страница текста согласия

Если у согласия заполнен текст, оно доступно по адресу `/consent/{id}` — именно на неё ведёт ссылка из чекбокса. Отдельный роут в своём модуле создавать не нужно.

Согласия без текста — это просто заголовок со своими ссылками. Так делают, когда правила уже опубликованы отдельной страницей сайта.

## Структура таблиц

`consents` — сами согласия:

| Поле | Описание |
| --- | --- |
| `context` | Код формы, в которой выводится согласие |
| `language` | Код языка согласия |
| `title` | Заголовок рядом с чекбоксом, допускает инлайн-HTML |
| `text` | Полный текст, показывается на `/consent/{id}` |
| `version` | Версия согласия, попадает в лог при принятии |
| `is_required` | Обязательно ли принять для отправки формы |
| `is_active` | Выводится ли согласие в форме |

`consent_log` — журнал принятий: `user_id`, `consent_id`, `version`, `ip_address`, `accepted_at`. Просматривается в админке: **Согласия → Лог**.

## Cookie-баннер

Модуль также отвечает за баннер о cookie. Он настраивается отдельно в админке (**Cookie-баннер**): включение, тексты по языкам и версия. Со стороны кода модулей ничего подключать не нужно — баннер выводится сам.
